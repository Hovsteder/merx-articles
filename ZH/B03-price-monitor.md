# MERX 价格监控：我们如何每 30 秒追踪每个供应商

价格监控是 MERX 的心跳。每 30 秒，它向每个集成的能量供应商发起请求、获取当前定价、规范化数据，并发布到系统的其余部分。没有它，最优价格路由就是在猜测。有了它，每笔订单都基于不超过 30 秒的数据路由到最便宜的可用供应商。

本文是对价格监控架构的技术深度解析：它如何轮询供应商、适配器模式如何保持系统的可扩展性、Redis pub/sub 如何实时分发价格数据，以及价格历史如何驱动分析和决策。

---

## 为什么是 30 秒

轮询间隔是经过深思熟虑的设计选择。TRON 上的能量价格不会每秒变化——它们不像即期外汇或加密货币订单簿那样。供应商定价通常每小时变化几次，有时更少。30 秒的间隔在捕获每个有意义的价格变化的同时避免了几个问题：

- **供应商 API 限频**：大多数供应商允许每秒 1-2 个请求。以 30 秒为间隔，即使有重试也远在限制范围内。
- **网络开销**：轮询 7 个以上供应商会产生 HTTP 流量。30 秒间隔下这微不足道。1 秒间隔下则开销可观。
- **数据新鲜度与噪音**：能量市场上 30 秒以下的价格变化几乎都是噪音而非信号。30 秒在过滤噪音的同时捕获真实变动。
- **系统资源使用**：价格监控与其他服务一起运行。激进的轮询会与 CPU 和内存竞争，却不增加价值。

缓存价格的 TTL 设定为 60 秒——轮询间隔的两倍。如果一次轮询失败，前一次的价格在再过一个周期内仍然有效。这防止了单次轮询失败就将供应商从订单簿中移除。

---

## 适配器模式

每个能量供应商都有不同的 API。不同的端点、不同的认证方式、不同的响应格式、不同的错误码。价格监控使用适配器模式将这些差异与核心轮询逻辑隔离。

### 供应商接口

每个供应商适配器都实现一个共同接口：

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

### 供应商适配器

每个供应商有自己的适配器文件。以下是一个简化的供应商适配器示例：

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

### 添加新供应商

这种架构的一个关键好处是添加新供应商只需一个新文件：

1. 创建 `providers/newprovider/index.ts` 实现 `IEnergyProvider`。
2. 在配置中注册供应商。
3. 价格监控自动开始轮询它。
4. 订单执行器自动将其纳入路由决策。

无需更改价格监控，无需更改订单执行器，无需更改 API。适配器模式确保供应商特定的复杂性被封装。

---

## 轮询循环

价格监控的核心循环简洁但针对可靠性精心设计：

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

### 关键设计决策

**Promise.allSettled 而非 Promise.all**：单个供应商的失败不能阻塞其他供应商的更新。`allSettled` 确保每个供应商被独立轮询。

**60 秒 TTL**：如果供应商连续两个周期（60 秒）未响应，其缓存价格自动过期。订单执行器不会路由到没有缓存价格的供应商。

**健康指标伴随价格**：每次轮询记录响应时间和成功/失败状态。这些数据供路由算法的可靠性评分使用。

---

## Redis Pub/Sub 分发

价格监控不直接向 API 消费者提供价格。它发布到 Redis，其他服务订阅它们需要的更新。

### 频道结构

```
Channel: price-updates
  -> All price update events (consumed by API service, WebSocket broadcast)

Channel: price-alerts
  -> Significant price changes (consumed by notification service)

Channel: provider-health
  -> Health status changes (consumed by admin dashboard)
```

### 为什么用 Pub/Sub 而非直接调用

价格监控和 API 服务是独立的进程（实际上是独立的 Docker 容器）。它们完全通过 Redis 通信——没有直接导入、没有函数调用、没有共享内存。这种隔离意味着：

- 价格监控可以重启而不影响 API。
- API 可以水平扩展（多个实例），所有实例都接收相同的价格更新。
- 价格监控中的错误不会导致 API 服务崩溃。
- 每个服务可以独立部署。

### WebSocket 广播

API 服务订阅 `price-updates` 频道并将更新广播给已连接的 WebSocket 客户端：

```
Price Monitor -> Redis pub/sub -> API Service -> WebSocket -> Client

Latency: ~5ms from provider response to client notification
```

订阅 WebSocket 推送的客户端近乎实时地接收价格更新，实现了实时仪表板和响应式交易界面。

---

## 价格历史存储

每个价格数据点都存储在 PostgreSQL 中用于历史分析。模式捕获完整的价格快照：

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

### 数据量

以 30 秒间隔跨 7 个供应商：

```
7 providers x 2 polls/minute x 60 minutes x 24 hours = 20,160 rows/day
Monthly: ~604,800 rows
Yearly: ~7,257,600 rows
```

每行很小（大约 100 字节），因此年存储量不到 1 GB。配合适当的索引，PostgreSQL 处理这个数据量毫无压力。

### 分析查询

价格历史支持多种有价值的分析：

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

这些查询驱动了 MERX 价格历史 API 端点和管理仪表板。

---

## 边界情况处理

### 供应商返回无效数据

价格监控在缓存前验证每个响应：

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

无效数据被记录并丢弃。前一个有效价格保留在缓存中直到自然过期。

### 供应商 API 变更

供应商 API 偶尔会变更——新字段、弃用端点、修改响应格式。因为每个供应商有自己的适配器，API 变更被隔离在单一文件中。适配器被更新、测试和部署，无需触及系统的任何其他部分。

### 网络分区

如果价格监控失去网络连接，所有供应商轮询同时失败。60 秒的 TTL 确保缓存价格在一分钟内过期，订单执行器停止路由到所有供应商。连接恢复后，下一个轮询周期自动重新填充缓存。

### 时钟漂移

价格监控在单一服务器上运行，因此服务之间的时钟漂移对相对计时来说不是问题。价格历史中的时间戳使用 PostgreSQL 的 `NOW()`，确保一致性。对于 API 响应中的绝对时间戳，服务器运行 NTP。

---

## 监控"监控器"

价格监控本身通过几种机制被监控：

- **健康端点**：服务暴露一个 `/health` 端点，报告每个供应商最后一次成功轮询的时间。
- **告警**：如果 5 分钟内没有成功的价格更新被发布，触发告警。
- **指标**：轮询次数、成功率、平均响应时间和缓存命中率被追踪。
- **日志**：每次轮询结果（成功或失败）都以结构化 JSON 格式记录以供分析。

---

## 性能特征

```
Typical poll cycle (7 providers):
  Total time: 1-3 seconds (parallel HTTP requests)
  Redis writes: 8 (7 provider prices + 1 best price)
  Redis publishes: 7 (one per provider update)
  PostgreSQL inserts: 7 (one per provider)
  Memory usage: < 50 MB
  CPU usage: < 2% average
```

价格监控被设计为刻意轻量。它只做一件事——获取和分发价格——并高效地完成。30 秒的间隔意味着服务 90% 的时间处于空闲状态，为更计算密集的订单执行器和 API 服务留出资源。

---

## 结论

价格监控在概念上很简单——轮询供应商、规范化数据、发布更新。但细节决定成败。适配器模式使添加供应商变得轻而易举。Redis pub/sub 将价格收集与价格消费解耦。60 秒 TTL 自动排除过期数据。健康指标供路由决策使用。价格历史则支持帮助用户做出更好决策的分析。

你在 MERX 上看到的每个价格、每个最优价格推荐、每个自动路由决策，都追溯到价格监控每 30 秒的心跳。它是使聚合成为可能的基础。

在 [https://merx.exchange](https://merx.exchange) 探索实时定价和历史数据。API 访问请参阅 [https://merx.exchange/docs](https://merx.exchange/docs) 的文档。

---

*本文是 MERX 技术系列的一部分。MERX 是首个区块链资源交易所。SDK 源代码：[https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)、[https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)。*

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
