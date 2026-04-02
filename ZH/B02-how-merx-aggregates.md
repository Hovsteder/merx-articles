# MERX 如何将所有能量供应商聚合到一个 API 中

2026 年的 TRON 能量市场存在碎片化问题。至少有七个主要供应商提供能量委托服务，每个都有自己的 API、定价模型和可用性模式。如果你想获得最优价格，你需要与所有供应商集成、持续监控价格、处理各自的差异，并为某个供应商宕机时构建故障转移逻辑。

或者你可以调用 MERX 的一个 API。

本文解释了 MERX 如何将所有主要能量供应商聚合到单一 API 中——价格监控、最优价格路由、自动故障转移背后的架构，以及用一个集成替代七个集成所带来的运营简化。

---

## 供应商生态

TRON 能量市场包含多个独立运营的供应商。截至 2026 年初，主要供应商包括：

- **TronSave** - 最早的能量租赁服务之一
- **Feee** - 具有竞争力的定价，提供 API 访问
- **itrx** - 专注于批量能量订单
- **CatFee** - 中端市场定位
- **Netts** - 定价激进的新进入者
- **SoHu** - 专注中国市场的供应商
- **PowerSun** - 直接质押和委托

每个供应商都有自己的：

- API 格式和认证方式
- 定价结构（有的按能量单位收费，有的按 TRX 收费）
- 最小和最大订单规模
- 可用时长（1 小时、1 天、3 天、7 天等）
- 支持的支付方式
- 状态页面和可用性特征

### 集成税

与单一供应商集成很简单。但要与所有供应商集成以获得最优定价是一项重大工程投入：

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

这就是 MERX 消除的集成税。你只需维护一个 MERX 集成，而不是七个供应商集成。其余的由 MERX 处理。

---

## MERX 架构

MERX 位于你的应用和供应商生态之间。架构有三个核心组件：

### 1. 价格监控

价格监控是一个专用服务，持续轮询每个集成供应商的当前定价。每 30 秒，它查询每个供应商的 API，将响应规范化为标准格式，并将结果发布到 Redis pub/sub 频道。

```
Every 30 seconds:
  For each provider:
    1. Query provider API for current prices
    2. Normalize to standard format (SUN per energy unit)
    3. Validate response (reject outliers, stale data)
    4. Publish to Redis: channel "prices:{provider}"
    5. Store in price history (PostgreSQL)
```

30 秒的间隔是经过深思熟虑的选择。更快的轮询会给供应商 API 造成压力，且几乎不会增加价值（价格很少按秒变化）。更慢的轮询则有服务过时价格的风险。

### 2. Redis 价格缓存

Redis 作为实时价格缓存。价格监控的每次价格更新都存储在 Redis 中，TTL（存活时间）为 60 秒——是轮询间隔的两倍。如果某个供应商的价格数据超过 60 秒，它将自动过期并从路由决策中排除。

```
Redis key structure:
  prices:tronsave     -> { energy: 88, bandwidth: 2, updated: 1711756800 }
  prices:feee         -> { energy: 92, bandwidth: 3, updated: 1711756800 }
  prices:itrx         -> { energy: 85, bandwidth: 2, updated: 1711756800 }
  prices:catfee       -> { energy: 95, bandwidth: 3, updated: 1711756800 }
  ...

  prices:best         -> { provider: "itrx", energy: 85, updated: 1711756800 }
```

`prices:best` 键在每次价格更新时重新计算，使 API 无需扫描所有供应商即可即时访问当前最优价格。

### 3. 订单执行器

当你通过 MERX API 下单时，订单执行器接收并确定最优路由：

```
Order received: 65,000 energy for TBuyerAddress

1. Read prices:best from Redis -> itrx at 85 SUN/unit
2. Check itrx availability for 65,000 energy -> available
3. Submit order to itrx
4. Monitor for on-chain delegation confirmation
5. Verify energy arrived at TBuyerAddress
6. Notify buyer (webhook + WebSocket)
```

如果最便宜的供应商无法完成订单（库存不足、API 错误、超时），执行器会自动切换到下一个最便宜的供应商。

---

## 价格规范化

不同供应商以不同格式报价。有的以每能量单位 SUN 报价。有的以给定能量数量的总 TRX 报价。有的在价格中包含带宽，有的单独收费。

MERX 将所有内容规范化为单一格式：

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

这种规范化至关重要。没有它，跨供应商比较价格需要使用者理解每个供应商的定价模型。有了它，价格比较就是简单的数值排序。

---

## 最优价格路由详解

路由算法不仅仅是"选最便宜的"。多个因素影响路由决策：

### 因素 1：价格

主要因素。其他条件相同时，最便宜的供应商胜出。

### 因素 2：可用性

一个报价 80 SUN 但只有 10,000 能量可用的供应商无法完成 65,000 能量的订单。路由器必须检查可用库存。

### 因素 3：可靠性

MERX 追踪每个供应商的历史完成率、响应时间和失败率。完成率 95% 的供应商相对于 99% 完成率的供应商会被降权，即使 95% 的供应商报价略低。

```
Effective price = quoted_price / fill_rate

Provider A: 85 SUN, 99% fill rate -> 85.86 effective
Provider B: 82 SUN, 94% fill rate -> 87.23 effective
Winner: Provider A despite higher quoted price
```

### 因素 4：时长支持

并非所有供应商都支持所有时长。如果你需要 1 小时委托，只提供每日最低时长的供应商会被排除。

### 订单拆分

对于超出任何单一供应商容量的大额订单，路由器会将订单拆分到多个供应商：

```
Order: 500,000 energy

Provider A: 200,000 available at 85 SUN -> fill 200,000
Provider B: 180,000 available at 87 SUN -> fill 180,000
Provider C: 300,000 available at 92 SUN -> fill 120,000

Total filled: 500,000 energy
Blended rate: 87.28 SUN/unit
```

买家看到的是一个混合费率的单一订单。多供应商执行的复杂性完全被隐藏。

---

## 自动故障转移

故障转移是聚合真正体现价值的地方。当你直接与供应商集成且他们宕机时，你的应用停止运行。有了 MERX，供应商故障被透明处理。

### 故障转移链

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

### 健康追踪

价格监控为每个供应商维护一个健康评分：

```
Health score components:
  - Last successful price fetch: must be within 60s
  - API response time: penalize > 2 seconds
  - Recent order fill rate: penalize < 95%
  - Recent error rate: penalize > 5%
```

不健康的供应商在恢复前被排除在路由之外。当价格监控成功从它们获取价格时，恢复会被自动检测。

### 买家零宕机

从买家的角度来看，供应商故障是不可见的。只要至少有一个供应商在运行，API 调用就会成功。实际上，拥有七个或更多供应商意味着全市场中断基本不可能——所有供应商同时宕机的概率可以忽略不计。

---

## 一个 API 替代多个

以下是使用 MERX 与直接供应商集成的对比：

### 不使用 MERX

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

### 使用 MERX

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

七个集成变成一个。数百行路由和故障转移代码变成四行。供应商 API 变更的持续维护降为零。

---

## 实时价格推送

对于需要展示实时价格或做出实时路由决策的应用，MERX 提供 WebSocket 价格推送：

```typescript
const client = new MerxClient({ apiKey: 'your-key' });

client.onPriceUpdate((update) => {
  console.log(`${update.provider}: ${update.energyPrice} SUN/unit`);
  console.log(`Best price: ${update.bestPrice} SUN/unit`);
});
```

WebSocket 推送发布价格监控的每次价格更新——每个供应商大约每 30 秒一次。这使应用无需轮询即可显示实时定价。

---

## 供应商透明度

MERX 不会隐藏哪个供应商完成了你的订单。每个订单响应都包含供应商名称、支付的价格和链上委托交易哈希：

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

你始终知道能量来自哪里、支付了多少，并且可以在链上独立验证委托。

---

## 快速上手

MERX 聚合可通过 REST API、JavaScript SDK、Python SDK 和供 AI 代理使用的 MCP 服务器访问：

- **API 文档**：[https://merx.exchange/docs](https://merx.exchange/docs)
- **JavaScript SDK**：[https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- **Python SDK**：[https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)
- **MCP 服务器**：[https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)

在 [https://merx.exchange](https://merx.exchange) 创建账户，获取 API 密钥，通过单一 API 调用开始将能量订单路由到最优可用价格。

---

*本文是 MERX 技术系列的一部分。MERX 是首个区块链资源交易所，将所有主要 TRON 能量供应商聚合到单一 API 中，提供最优价格路由和自动故障转移。*

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
