# MERX Price Monitor: How We Track Every Provider Every 30 Seconds

The price monitor is the heartbeat of MERX. Every 30 seconds, it reaches out to every integrated energy provider, fetches their current pricing, normalizes the data, and publishes it to the rest of the system. Without it, best-price routing would be guessing. With it, every order is routed to the cheapest available provider based on data no older than 30 seconds.

This article is a technical deep-dive into the price monitor's architecture: how it polls providers, how the adapter pattern keeps the system extensible, how Redis pub/sub distributes price data in real time, and how price history powers analytics and decision-making.

---

## Why 30 Seconds

The polling interval is a deliberate design choice. Energy prices on TRON do not change every second - they are not like spot forex or crypto order books. Provider pricing typically shifts a few times per hour, sometimes less. A 30-second interval captures every meaningful price change while avoiding several problems:

- **Provider API rate limits**: most providers allow 1-2 requests per second. At 30-second intervals, we stay well within limits even with retries.
- **Network overhead**: polling 7+ providers creates HTTP traffic. At 30 seconds, this is trivial. At 1 second, it would be substantial.
- **Data freshness vs noise**: sub-30-second price changes on energy markets are almost always noise, not signal. 30 seconds filters noise while capturing real movements.
- **System resource usage**: the price monitor runs alongside other services. Aggressive polling would compete for CPU and memory without adding value.

The TTL on cached prices is set to 60 seconds - twice the polling interval. If a poll fails, the previous price remains valid for one more cycle before being expired. This prevents a single failed poll from removing a provider from the order book.

---

## The Adapter Pattern

Each energy provider has a different API. Different endpoints, different authentication methods, different response formats, different error codes. The price monitor uses the adapter pattern to isolate these differences from the core polling logic.

### The Provider Interface

Every provider adapter implements a common interface:

```typescript
interface IEnergyProvider {
  name: string;

  // Fetch current pricing
  getPrices(): Promise<ProviderPriceResponse>;

  // Check if provider is operational
  healthCheck(): Promise<boolean>;

  // Get available inventory
  getAvailability(): Promise<AvailabilityResponse>;
}

interface ProviderPriceResponse {
  energyPricePerUnit: number;   // SUN per energy unit
  bandwidthPricePerUnit: number;
  minOrder: number;
  maxOrder: number;
  durations: Duration[];
  timestamp: number;
}
```

### A Provider Adapter

Each provider gets its own adapter file. Here is a simplified example of what a provider adapter looks like:

```typescript
// providers/tronsave/index.ts
import { IEnergyProvider, ProviderPriceResponse } from '../base';

export class TronSaveProvider implements IEnergyProvider {
  name = 'tronsave';
  private apiUrl: string;
  private apiKey: string;

  constructor(config: ProviderConfig) {
    this.apiUrl = config.apiUrl;
    this.apiKey = config.apiKey;
  }

  async getPrices(): Promise<ProviderPriceResponse> {
    const response = await fetch(`${this.apiUrl}/v1/prices`, {
      headers: { 'Authorization': `Bearer ${this.apiKey}` }
    });

    const data = await response.json();

    // Normalize provider-specific format to standard format
    return {
      energyPricePerUnit: this.normalizePrice(data.energy_price),
      bandwidthPricePerUnit: this.normalizePrice(data.bandwidth_price),
      minOrder: data.min_energy || 10000,
      maxOrder: data.max_energy || 10000000,
      durations: this.normalizeDurations(data.available_durations),
      timestamp: Date.now()
    };
  }

  async healthCheck(): Promise<boolean> {
    try {
      const response = await fetch(`${this.apiUrl}/health`);
      return response.ok;
    } catch {
      return false;
    }
  }

  // ... provider-specific normalization methods
}
```

### Adding a New Provider

One of the key benefits of this architecture is that adding a new provider requires exactly one new file:

1. Create `providers/newprovider/index.ts` implementing `IEnergyProvider`.
2. Register the provider in the configuration.
3. The price monitor automatically starts polling it.
4. The order executor automatically includes it in routing decisions.

No changes to the price monitor, no changes to the order executor, no changes to the API. The adapter pattern ensures that provider-specific complexity is encapsulated.

---

## The Polling Loop

The price monitor's core loop is straightforward but carefully designed for reliability:

```typescript
class PriceMonitor {
  private providers: IEnergyProvider[];
  private redis: RedisClient;
  private pollInterval = 30_000; // 30 seconds

  async start() {
    // Initial poll on startup
    await this.pollAll();

    // Schedule recurring polls
    setInterval(() => this.pollAll(), this.pollInterval);
  }

  async pollAll() {
    const results = await Promise.allSettled(
      this.providers.map(provider => this.pollProvider(provider))
    );

    // Compute best price from successful results
    const validPrices = results
      .filter(r => r.status === 'fulfilled')
      .map(r => r.value);

    if (validPrices.length > 0) {
      await this.updateBestPrice(validPrices);
    }
  }

  async pollProvider(provider: IEnergyProvider) {
    const startTime = Date.now();

    try {
      const prices = await provider.getPrices();
      const responseTime = Date.now() - startTime;

      // Store in Redis with 60s TTL
      await this.redis.setex(
        `prices:${provider.name}`,
        60,
        JSON.stringify(prices)
      );

      // Publish price update event
      await this.redis.publish(
        'price-updates',
        JSON.stringify({
          provider: provider.name,
          prices,
          responseTime
        })
      );

      // Update health metrics
      await this.updateHealthMetrics(provider.name, {
        success: true,
        responseTime,
        timestamp: Date.now()
      });

      return { provider: provider.name, prices };

    } catch (error) {
      await this.updateHealthMetrics(provider.name, {
        success: false,
        error: error.message,
        timestamp: Date.now()
      });

      throw error; // Let Promise.allSettled handle it
    }
  }
}
```

### Key Design Decisions

**Promise.allSettled, not Promise.all**: A single provider failure must not block updates from other providers. `allSettled` ensures every provider is polled independently.

**60-second TTL**: If a provider fails to respond for two consecutive cycles (60 seconds), its cached price expires automatically. The order executor will not route to a provider with no cached price.

**Health metrics alongside prices**: Every poll records response time and success/failure. This data feeds into the routing algorithm's reliability scoring.

---

## Redis Pub/Sub Distribution

The price monitor does not serve prices directly to API consumers. Instead, it publishes to Redis, and other services subscribe to the updates they need.

### Channel Structure

```
Channel: price-updates
  -> All price update events (consumed by API service, WebSocket broadcast)

Channel: price-alerts
  -> Significant price changes (consumed by notification service)

Channel: provider-health
  -> Health status changes (consumed by admin dashboard)
```

### Why Pub/Sub Instead of Direct Calls

The price monitor and the API service are separate processes (separate Docker containers, in fact). They communicate exclusively through Redis - no direct imports, no function calls, no shared memory. This isolation means:

- The price monitor can be restarted without affecting the API.
- The API can scale horizontally (multiple instances) and all receive the same price updates.
- A bug in the price monitor cannot crash the API service.
- Each service can be deployed independently.

### WebSocket Broadcast

The API service subscribes to the `price-updates` channel and broadcasts updates to connected WebSocket clients:

```
Price Monitor -> Redis pub/sub -> API Service -> WebSocket -> Client

Latency: ~5ms from provider response to client notification
```

Clients subscribing to the WebSocket feed receive price updates in near real-time, enabling live dashboards and responsive trading interfaces.

---

## Price History Storage

Every price data point is stored in PostgreSQL for historical analysis. The schema captures the full price snapshot:

```sql
CREATE TABLE price_history (
  id BIGSERIAL PRIMARY KEY,
  provider VARCHAR(50) NOT NULL,
  energy_price_sun BIGINT NOT NULL,
  bandwidth_price_sun BIGINT NOT NULL,
  min_order INTEGER,
  max_order INTEGER,
  available_energy BIGINT,
  response_time_ms INTEGER,
  recorded_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_price_history_provider_time
  ON price_history(provider, recorded_at DESC);
```

### Data Volume

At 30-second intervals across 7 providers:

```
7 providers x 2 polls/minute x 60 minutes x 24 hours = 20,160 rows/day
Monthly: ~604,800 rows
Yearly: ~7,257,600 rows
```

Each row is small (roughly 100 bytes), so annual storage is under 1 GB. PostgreSQL handles this volume trivially with appropriate indexing.

### Analytics Queries

The price history enables several valuable analyses:

```sql
-- Average price by provider over the last 24 hours
SELECT provider,
       AVG(energy_price_sun) as avg_price,
       MIN(energy_price_sun) as min_price,
       MAX(energy_price_sun) as max_price
FROM price_history
WHERE recorded_at > NOW() - INTERVAL '24 hours'
GROUP BY provider
ORDER BY avg_price;

-- Price trend for a specific provider
SELECT date_trunc('hour', recorded_at) as hour,
       AVG(energy_price_sun) as avg_price
FROM price_history
WHERE provider = 'tronsave'
  AND recorded_at > NOW() - INTERVAL '7 days'
GROUP BY hour
ORDER BY hour;
```

These queries power the MERX price history API endpoint and the admin dashboard.

---

## Handling Edge Cases

### Provider Returns Invalid Data

The price monitor validates every response before caching:

```typescript
function validatePrice(price: ProviderPriceResponse): boolean {
  // Price must be positive
  if (price.energyPricePerUnit <= 0) return false;

  // Price must be within reasonable bounds (10-500 SUN)
  if (price.energyPricePerUnit < 10_000_000) return false;  // < 10 SUN
  if (price.energyPricePerUnit > 500_000_000) return false;  // > 500 SUN

  // Must have valid timestamp
  if (price.timestamp > Date.now() + 60_000) return false;  // future
  if (price.timestamp < Date.now() - 300_000) return false;  // > 5min old

  return true;
}
```

Invalid data is logged and discarded. The previous valid price remains in cache until it expires naturally.

### Provider API Changes

Provider APIs change occasionally - new fields, deprecated endpoints, modified response formats. Because each provider has its own adapter, API changes are isolated to a single file. The adapter is updated, tested, and deployed without touching any other part of the system.

### Network Partitions

If the price monitor loses network connectivity, all provider polls fail simultaneously. The 60-second TTL ensures that cached prices expire within a minute, and the order executor stops routing to all providers. When connectivity is restored, the next poll cycle repopulates the cache automatically.

### Clock Drift

The price monitor runs on a single server, so clock drift between services is not a concern for relative timing. Timestamps in price history use `NOW()` from PostgreSQL, ensuring consistency. For absolute timestamps in API responses, the server runs NTP.

---

## Monitoring the Monitor

The price monitor itself is monitored through several mechanisms:

- **Health endpoint**: the service exposes a `/health` endpoint that reports the last successful poll time for each provider.
- **Alerting**: if no successful price update has been published for 5 minutes, an alert is triggered.
- **Metrics**: poll count, success rate, average response time, and cache hit rate are tracked.
- **Logs**: every poll result (success or failure) is logged with structured JSON for analysis.

---

## Performance Characteristics

```
Typical poll cycle (7 providers):
  Total time: 1-3 seconds (parallel HTTP requests)
  Redis writes: 8 (7 provider prices + 1 best price)
  Redis publishes: 7 (one per provider update)
  PostgreSQL inserts: 7 (one per provider)
  Memory usage: < 50 MB
  CPU usage: < 2% average
```

The price monitor is intentionally lightweight. It does one thing - fetch and distribute prices - and does it efficiently. The 30-second interval means the service is idle 90% of the time, leaving resources available for the more compute-intensive order executor and API services.

---

## Conclusion

The price monitor is conceptually simple - poll providers, normalize data, publish updates. But the details matter. The adapter pattern makes adding providers trivial. Redis pub/sub decouples price collection from price consumption. 60-second TTLs automatically exclude stale data. Health metrics feed into routing decisions. And price history enables analytics that help users make better decisions.

Every price you see on MERX, every best-price recommendation, every automatic routing decision traces back to the price monitor's 30-second heartbeat. It is the foundation that makes aggregation possible.

Explore live pricing and historical data at [https://merx.exchange](https://merx.exchange). For API access, see the documentation at [https://merx.exchange/docs](https://merx.exchange/docs).

---

*This article is part of the MERX technical series. MERX is the first blockchain resource exchange. Source code for SDKs: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js), [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python).*


## Try It Now with AI

Add MERX to Claude Desktop or any MCP-compatible client -- zero install, no API key needed for read-only tools:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Ask your AI agent: "What is the cheapest TRON energy right now?" and get live prices from all connected providers.

Full MCP documentation: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)
