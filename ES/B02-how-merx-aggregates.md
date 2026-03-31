# Como MERX agrega todos los proveedores de energia en una sola API

The TRON mercado de energia in 2026 has a fragmentation problem. At least seven major providers offer delegacion de energia services, each with their own API, pricing model, and availability patterns. If you want el mejor precio, you need to integrate with all of them, monitor their prices continuously, handle their individual quirks, and build respaldo logic for when one goes down.

Or you can make una sola llamada a la API to MERX.

This article explains how MERX aggregates all major proveedor de energias into a una sola API - the architecture behind monitor de preciosing, enrutamiento al mejor precio, respaldo automatico, and the operational simplification that comes from replacing seven integrations with one.

---

## The Provider Landscape

The TRON mercado de energia includes multiple providers, each operating independently. As of early 2026, the major providers include:

- **TronSave** - one of the earliest alquiler de energia services
- **Feee** - competitive pricing with API access
- **itrx** - focus on bulk energy orders
- **CatFee** - mid-market positioning
- **Netts** - newer entrant with aggressive pricing
- **SoHu** - provider with Chinese market focus
- **PowerSun** - direct staking and delegation

Each provider has their own:

- API format and authentication method
- Pricing structure (some charge per unidad de energia, others per TRX)
- Minimum and maximum order sizes
- Available durations (1h, 1d, 3d, 7d, etc.)
- Supported payment methods
- Status pages and uptime characteristics

### The Integration Tax

Integrating with a single provider is straightforward. Integrating with all of them to get best pricing is a significant engineering undertaking:

```
Per provider:
  - API client implementation:    2-3 days
  - Price normalization:          1 day
  - Error handling:               1 day
  - Testing:                      1-2 days
  - Ongoing maintenance:          2-4 hours/month

7 providers x 5-7 days = 35-49 days of initial integration
7 providers x 3 hours/month = 21 hours/month ongoing maintenance
```

This is the integration tax that MERX eliminates. Instead of maintaining seven provider integrations, you maintain one MERX integration. MERX handles the rest.

---

## The MERX Architecture

MERX sits between your application and the provider ecosystem. The architecture has three core components:

### 1. Price Monitor

The monitor de precios is a dedicated service that continuously polls every integrated provider for current pricing. Every 30 seconds, it queries each provider's API, normalizes the response into a standard format, and publishes the result to a Redis pub/sub channel.

```
Every 30 seconds:
  For each provider:
    1. Query provider API for current prices
    2. Normalize to standard format (SUN per energy unit)
    3. Validate response (reject outliers, stale data)
    4. Publish to Redis: channel "prices:{provider}"
    5. Store in price history (PostgreSQL)
```

The 30-second interval is deliberately chosen. Faster polling would stress provider APIs and add minimal value (prices rarely change second-to-second). Slower polling would risk serving stale prices.

### 2. Redis Price Cache

Redis serves as the tiempo real price cache. Every actualizacion de precios from the monitor de precios is stored in Redis with a TTL (tiempo de vida) of 60 seconds - twice the intervalo de sondeo. If a provider's price data is older than 60 seconds, it is automatically expired and excluded from routing decisions.

```
Redis key structure:
  prices:tronsave     -> { energy: 88, bandwidth: 2, updated: 1711756800 }
  prices:feee         -> { energy: 92, bandwidth: 3, updated: 1711756800 }
  prices:itrx         -> { energy: 85, bandwidth: 2, updated: 1711756800 }
  prices:catfee       -> { energy: 95, bandwidth: 3, updated: 1711756800 }
  ...

  prices:best         -> { provider: "itrx", energy: 85, updated: 1711756800 }
```

The `prices:best` key is recomputed on every actualizacion de precios, giving the API instant access to the current mejor precio without scanning all providers.

### 3. Order Executor

When you place an order through the MERX API, the ejecutor de ordenes receives it and determines the optimal routing:

```
Order received: 65,000 energy for TBuyerAddress

1. Read prices:best from Redis -> itrx at 85 SUN/unit
2. Check itrx availability for 65,000 energy -> available
3. Submit order to itrx
4. Monitor for on-chain delegation confirmation
5. Verify energy arrived at TBuyerAddress
6. Notify buyer (webhook + WebSocket)
```

If el proveedor mas barato cannot fill the order (insufficient stock, API error, timeout), the executor automatically falls through to the next cheapest provider.

---

## Price Normalization

Different providers quote prices in different formats. Some quote in SUN per unidad de energia. Some quote total TRX for a given cantidad de energia. Some include bandwidth in the price; others charge separately.

MERX normalizes everything to a single format:

```typescript
interface NormalizedPrice {
  provider: string;
  energyPricePerUnit: number;    // SUN per energy unit
  bandwidthPricePerUnit: number; // SUN per bandwidth unit
  minOrder: number;              // Minimum energy units
  maxOrder: number;              // Maximum energy units
  availableEnergy: number;       // Currently available
  durations: string[];           // Supported durations
  lastUpdated: number;           // Unix timestamp
}
```

This normalization is critical. Without it, comparing prices across providers would require the consumer to understand each provider's pricing model. With it, comparacion de precios is a simple numerical sort.

---

## Best-Price Routing in Detail

The routing algorithm is not just "pick the cheapest." Several factors influence the routing decision:

### Factor 1: Price

The primary factor. All else being equal, el proveedor mas barato wins.

### Factor 2: Availability

A provider quoting 80 SUN but with only 10,000 energia disponible cannot fill a 65,000 energy order. The router must check available inventory.

### Factor 3: Reliability

MERX tracks each provider's historical tasa de ejecucion, tiempo de respuesta, and failure rate. A provider with a 95% tasa de ejecucion is penalized relative to one with a 99% tasa de ejecucion, even if the 95% provider is slightly cheaper.

```
Effective price = quoted_price / fill_rate

Provider A: 85 SUN, 99% fill rate -> 85.86 effective
Provider B: 82 SUN, 94% fill rate -> 87.23 effective
Winner: Provider A despite higher quoted price
```

### Factor 4: Duration Support

Not all providers support all durations. If you need a 1-hour delegation, providers that only offer daily minimums are excluded.

### Order Splitting

For large orders that exceed any single provider's capacity, the router splits the order across multiple providers:

```
Order: 500,000 energy

Provider A: 200,000 available at 85 SUN -> fill 200,000
Provider B: 180,000 available at 87 SUN -> fill 180,000
Provider C: 300,000 available at 92 SUN -> fill 120,000

Total filled: 500,000 energy
Blended rate: 87.28 SUN/unit
```

The buyer sees a single order with a blended rate. The complexity of multi-provider execution is entirely hidden.

---

## Automatic Failover

Failover is where aggregation truly earns its value. When you integrate directly with a provider and they go down, your application stops functioning. With MERX, provider failures are handled transparently.

### Failover Chain

```
Primary provider fails
  |
  v
Mark provider as unhealthy (exclude from routing for 5 minutes)
  |
  v
Retry with next-cheapest provider
  |
  v
If second provider fails, try third
  |
  v
If all providers fail, return error to buyer with retry guidance
```

### Health Tracking

The monitor de precios maintains a puntuacion de salud for each provider:

```
Health score components:
  - Last successful price fetch: must be within 60s
  - API response time: penalize > 2 seconds
  - Recent order fill rate: penalize < 95%
  - Recent error rate: penalize > 5%
```

Unhealthy providers are excluded from routing until they recover. Recovery is detected automatically when the monitor de precios successfully fetches prices from them again.

### Zero-Downtime for Buyers

From the buyer's perspective, provider failures are invisible. Their API call succeeds as long as at least one provider is operational. En la practica, having seven or more providers means that total market outage is essentially impossible - the odds of all providers being down simultaneously are negligible.

---

## One API Replaces Many

Aqui esta what your integration looks like with MERX versus direct provider integration:

### Without MERX

```typescript
// Pseudo-code: direct multi-provider integration

// Initialize 7 provider clients
const tronsave = new TronSaveClient(apiKey1);
const feee = new FeeeClient(apiKey2);
const itrx = new ItrxClient(apiKey3);
// ... 4 more

// Fetch prices from all providers
const prices = await Promise.allSettled([
  tronsave.getPrice(65000),
  feee.getPrice(65000),
  itrx.getPrice(65000),
  // ... 4 more
]);

// Normalize different response formats
const normalized = prices
  .filter(p => p.status === 'fulfilled')
  .map(p => normalizePrice(p.value)); // complex per-provider logic

// Sort by price, check availability, handle errors...
const best = normalized.sort((a, b) => a.price - b.price)[0];

// Place order with best provider
try {
  const order = await getClient(best.provider).createOrder({
    energy: 65000,
    target: buyerAddress,
    // Provider-specific parameters...
  });
} catch (e) {
  // Failover to next provider...
  // More provider-specific error handling...
}
```

### With MERX

```typescript
import { MerxClient } from 'merx-sdk';

const client = new MerxClient({ apiKey: 'your-merx-key' });

// Get best price across all providers
const prices = await client.getPrices({ energy: 65000 });
console.log(`Best: ${prices.bestPrice.provider} at ${prices.bestPrice.perUnit} SUN`);

// Place order - automatically routed to best provider
const order = await client.createOrder({
  energy: 65000,
  targetAddress: buyerAddress,
  duration: '1h'
});

// Done. Failover, retries, verification handled automatically.
```

Seven integrations become one. Hundreds of lines of routing and respaldo code become four lines. Ongoing maintenance of provider API changes drops to zero.

---

## Real-Time Price Feed

For applications that want to display live prices or make tiempo real routing decisions, MERX provides a WebSocket flujo de precios:

```typescript
const client = new MerxClient({ apiKey: 'your-key' });

client.onPriceUpdate((update) => {
  console.log(`${update.provider}: ${update.energyPrice} SUN/unit`);
  console.log(`Best price: ${update.bestPrice} SUN/unit`);
});
```

The WebSocket feed publishes every actualizacion de precios from the monitor de precios - roughly every 30 seconds per provider. This enables applications to show live pricing without polling.

---

## Provider Transparency

MERX does not hide which provider filled your order. Every order response includes the provider name, the price paid, and the en cadena delegation hash de transaccion:

```json
{
  "orderId": "ord_abc123",
  "status": "completed",
  "provider": "itrx",
  "energy": 65000,
  "pricePerUnit": 85,
  "totalCostSun": 5525000,
  "delegationTxHash": "abc123def456...",
  "verifiedAt": "2026-03-30T12:00:00Z"
}
```

You always know where your energy came from, what you paid, and can verify the delegation en cadena independently.

---

## Como comenzar

MERX aggregation is available through the REST API, the JavaScript SDK, the Python SDK, and the Servidor MCP para agentes de IA:

- **API documentation**: [https://merx.exchange/docs](https://merx.exchange/docs)
- **JavaScript SDK**: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- **Python SDK**: [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)
- **MCP Server**: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)

Cree una cuenta en [https://merx.exchange](https://merx.exchange), get an clave de API, and start routing energy orders to the best available price with a una sola API call.

---

*Este articulo es parte de la serie tecnica de MERX. MERX es el primer exchange de recursos blockchain, aggregating all major TRON proveedor de energias into a una sola API with enrutamiento al mejor precio and respaldo automatico.*
