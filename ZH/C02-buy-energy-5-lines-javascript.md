# 5行代码购买TRON能量: MERX JavaScript SDK

MERX JavaScript SDK让您通过五行代码即可编程购买TRON能量 - 安装包、初始化客户端、查询价格、创建订单、验证委托。本文涵盖SDK的完整内容,从安装到生产部署,包括TypeScript类型、错误处理,以及四个模块(prices、orders、balance和webhooks)中的所有方法。

## 直接对接供应商API的问题

每个TRON能量供应商都有自己的API。对接一个很简单。但要对接全部七八个供应商以确保最优价格,则是一个工程项目。您需要处理不同的认证方案、响应格式、错误代码、速率限制和故障切换逻辑。

MERX SDK将这一切抽象为一个具有统一接口的客户端。在后台,MERX平台每30秒轮询所有供应商,维护实时价格索引,并将您的订单路由到最便宜的可用来源,同时实现自动故障切换。

## 安装

SDK要求Node.js 18或更高版本。使用原生 `fetch`,零运行时依赖。

```bash
npm install merx-sdk
```

```bash
yarn add merx-sdk
```

```bash
pnpm add merx-sdk
```

该包以ESM格式发布,附带完整的TypeScript声明和源码映射。

## 五行代码购买能量

以下是最精简形式的完整流程:

```typescript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY })
const prices = await merx.prices.list()
const order = await merx.orders.create({
  resource_type: 'ENERGY',
  amount: 65000,
  target_address: 'TYourTargetAddressHere',
  duration_sec: 3600,
})
console.log(`Order ${order.id}: ${order.status}`)
```

逐行解析:

1. 导入客户端类。
2. 使用API密钥初始化。密钥以 `sk_live_` 开头,在 [merx.exchange](https://merx.exchange) 的MERX控制面板中创建。
3. 获取所有已连接供应商的当前价格。此步骤为可选,但在下单前展示市场状态很有用。
4. 创建市价订单,购买65,000单位能量,委托至目标地址,时长一小时。平台自动选择最便宜的供应商。
5. 输出订单ID和状态。订单以 `PENDING` 状态开始,链上委托确认后转为 `FILLED`。

这就是整个集成过程。无需选择供应商,无需故障切换逻辑,无需价格比较代码。

## 客户端配置

```typescript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({
  apiKey: 'sk_live_your_key_here',   // 必填
  baseUrl: 'https://merx.exchange',  // 可选,默认为生产环境
})
```

`apiKey` 是唯一必填选项。`baseUrl` 可以被覆盖以用于测试环境。

## Prices模块

`merx.prices` 模块提供五个方法用于查询实时市场数据。

### prices.list()

返回所有活跃供应商的当前定价。

```typescript
const prices = await merx.prices.list()

for (const p of prices) {
  const cheapest = p.energy_prices[0]
  if (cheapest) {
    console.log(`${p.provider}: ${cheapest.price_sun} SUN/energy, ${p.available_energy} available`)
  }
}
```

每个 `ProviderPrice` 对象包含:
- `provider` - 供应商标识(例如 "sohu"、"catfee"、"tronsave")
- `is_market` - 该供应商是否支持灵活时长的市价订单
- `energy_prices` - `{ duration_sec, price_sun }` 层级数组
- `bandwidth_prices` - 带宽的相同结构
- `available_energy` - 当前可用能量容量(能量单位)
- `available_bandwidth` - 当前可用带宽容量(带宽单位)
- `fetched_at` - 最后一次成功轮询的Unix时间戳

### prices.best(resource, amount?)

返回指定资源类型的单个最低价格点。

```typescript
const best = await merx.prices.best('ENERGY')
console.log(`Cheapest: ${best.price_sun} SUN from ${best.provider}`)

// 带最低数量过滤
const bestLarge = await merx.prices.best('ENERGY', 500000)
```

可选的 `amount` 参数会过滤掉容量不足以完成您订单的供应商。

### prices.history(params?)

返回用于分析和图表的历史价格数据。

```typescript
const history = await merx.prices.history({
  provider: 'sohu',
  resource: 'ENERGY',
  period: '24h',
})

for (const entry of history) {
  console.log(`${entry.polled_at}: ${entry.price_sun} SUN, ${entry.available} available`)
}
```

可用周期: `'1h'`、`'6h'`、`'24h'`、`'7d'`、`'30d'`。所有过滤参数均为可选。

### prices.stats()

返回整个市场的汇总统计数据。

```typescript
const stats = await merx.prices.stats()
console.log(`Best price: ${stats.best_price_sun} SUN`)
console.log(`Average: ${stats.avg_price_sun} SUN`)
console.log(`Providers online: ${stats.total_providers}`)
console.log(`Cheapest-provider changes (24h): ${stats.cheapest_changes_24h}`)
```

### prices.preview(params)

在下单前预览订单成本。返回最佳匹配供应商和备选方案。

```typescript
const preview = await merx.prices.preview({
  resource: 'ENERGY',
  amount: 100000,
  duration: 86400,
  max_price_sun: 35,
})

if (preview.best) {
  console.log(`Best: ${preview.best.provider} at ${preview.best.cost_trx} TRX`)
  console.log(`Price: ${preview.best.price_sun} SUN/unit`)
}

for (const fb of preview.fallbacks) {
  console.log(`Fallback: ${fb.provider} at ${fb.cost_trx} TRX`)
}

if (preview.no_providers) {
  console.log('No providers available for this configuration')
}
```

`max_price_sun` 参数会过滤掉超过您价格上限的供应商。

## Orders模块

### orders.create(params)

创建能量或带宽订单。

```typescript
const order = await merx.orders.create({
  resource_type: 'ENERGY',
  amount: 65000,
  target_address: 'TYourTargetAddress',
  duration_sec: 3600,
  order_type: 'MARKET',          // 可选,默认为 'MARKET'
  max_price_sun: 30,             // 可选,LIMIT订单必填
  idempotency_key: 'unique-id',  // 可选,防止重复订单
})

console.log(`Order ${order.id} created`)
console.log(`Status: ${order.status}`)
```

订单类型:
- `MARKET` - 以最优可用价格立即执行
- `LIMIT` - 仅当价格等于或低于 `max_price_sun` 时执行
- `PERIODIC` - 周期性订单
- `BROADCAST` - 广播预签名的委托交易

`idempotency_key` 对生产系统至关重要。如果发生网络错误导致重试,API将返回原始订单而非创建重复订单。

### orders.list(limit?, offset?, status?)

带分页的订单列表。

```typescript
const { orders, total } = await merx.orders.list(10, 0, 'FILLED')
console.log(`${total} filled orders total`)

for (const o of orders) {
  console.log(`${o.id}: ${o.amount} ${o.resource_type}, cost: ${o.total_cost_sun} SUN`)
}
```

### orders.get(id)

返回单个订单及其成交详情。

```typescript
const order = await merx.orders.get('ord_abc123')

console.log(`Status: ${order.status}`)
console.log(`Fills: ${order.fills.length}`)

for (const fill of order.fills) {
  console.log(`  ${fill.provider}: ${fill.amount} units at ${fill.price_sun} SUN`)
  console.log(`  Verified: ${fill.verified}`)
  if (fill.tronscan_url) {
    console.log(`  TX: ${fill.tronscan_url}`)
  }
}
```

`fills` 数组精确展示订单是如何被完成的。每个成交条目包含供应商名称、分配数量、单位价格、成本、链上交易ID,以及委托是否已在链上验证。

## Balance模块

### balance.get()

返回当前账户余额。

```typescript
const balance = await merx.balance.get()
console.log(`TRX: ${balance.trx}`)
console.log(`USDT: ${balance.usdt}`)
console.log(`Locked: ${balance.trx_locked}`)
```

`trx_locked` 字段显示当前为待处理订单预留的TRX。

### balance.depositInfo()

返回为MERX账户充值的充值地址和备注。

```typescript
const info = await merx.balance.depositInfo()
console.log(`Send TRX to: ${info.address}`)
console.log(`Include memo: ${info.memo}`)
console.log(`Minimum deposit: ${info.min_amount_trx} TRX`)
```

备注是自动到账的必要条件。未包含正确备注的充值需要人工处理。

### balance.withdraw(params)

将TRX或USDT提现至外部TRON地址。

```typescript
const withdrawal = await merx.balance.withdraw({
  address: 'TYourExternalAddress',
  amount: 100,
  currency: 'TRX',
  idempotency_key: 'withdraw-unique-id',
})

console.log(`Withdrawal ${withdrawal.id}: ${withdrawal.status}`)
```

### balance.history(period?) 和 balance.summary()

```typescript
const history = await merx.balance.history('7D')
console.log(`${history.length} transactions in last 7 days`)

const summary = await merx.balance.summary()
console.log(`Total orders: ${summary.total_orders}`)
console.log(`Total energy: ${summary.total_energy}`)
console.log(`Average price: ${summary.avg_price_sun} SUN`)
```

## Webhooks模块

### webhooks.create(params)

创建Webhook订阅。`secret` 仅在创建时返回。

```typescript
const webhook = await merx.webhooks.create({
  url: 'https://your-server.com/merx-webhook',
  events: ['order.filled', 'order.failed', 'deposit.received'],
})

console.log(`Webhook ID: ${webhook.id}`)
console.log(`Secret: ${webhook.secret}`)
```

### webhooks.list() 和 webhooks.delete(id)

```typescript
const webhooks = await merx.webhooks.list()
console.log(`${webhooks.filter(w => w.is_active).length} active webhooks`)

await merx.webhooks.delete('wh_abc123')
```

## 错误处理

所有API错误都作为 `MerxError` 实例抛出,包含机器可读的 `code` 和人类可读的 `message`。

```typescript
import { MerxClient, MerxError } from 'merx-sdk'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY })

try {
  await merx.orders.create({
    resource_type: 'ENERGY',
    amount: 65000,
    target_address: 'TInvalidAddress',
    duration_sec: 3600,
  })
} catch (err) {
  if (err instanceof MerxError) {
    console.error(`[${err.code}]: ${err.message}`)
    // 输出: [INVALID_ADDRESS]: Target address is not a valid TRON address
  }
}
```

常见错误代码:

| 代码                   | 含义                                             |
|------------------------|--------------------------------------------------|
| `UNAUTHORIZED`         | 无效或缺少API密钥                               |
| `INSUFFICIENT_FUNDS`   | 账户余额不足                                     |
| `INVALID_ADDRESS`      | 目标地址不是有效的TRON地址                       |
| `ORDER_NOT_FOUND`      | 订单ID不存在                                     |
| `INVALID_AMOUNT`       | 数量低于最低限额或超出限制                       |
| `DUPLICATE_REQUEST`    | 幂等性密钥已被使用                               |
| `RATE_LIMITED`         | 请求过多                                         |
| `PROVIDER_UNAVAILABLE` | 没有可用的供应商                                 |

## TypeScript类型

所有类型均从包中导出,可在整个应用中用于类型注解:

```typescript
import type {
  ProviderPrice,
  PricePoint,
  PriceHistoryEntry,
  PriceStats,
  OrderPreview,
  PreviewMatch,
  Order,
  OrderWithFills,
  Fill,
  CreateOrderParams,
  OrderType,
  OrderStatus,
  ResourceType,
  Balance,
  DepositInfo,
  Withdrawal,
  Webhook,
} from 'merx-sdk'
```

SDK与严格模式兼容,不产生任何 `any` 类型。每个响应都有完整的类型定义。

## 生产示例: 自动化USDT转账流程

以下是一个完整的生产模式,包含价格查询、创建订单和轮询完成状态:

```typescript
import { MerxClient, MerxError } from 'merx-sdk'
import { randomUUID } from 'node:crypto'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY! })

async function ensureEnergyAndTransfer(targetAddress: string) {
  // 预览成本
  const preview = await merx.prices.preview({
    resource: 'ENERGY',
    amount: 65000,
    duration: 3600,
  })

  if (preview.no_providers) {
    throw new Error('No energy providers available')
  }

  console.log(`Best price: ${preview.best!.cost_trx} TRX from ${preview.best!.provider}`)

  // 使用幂等性密钥创建订单
  const order = await merx.orders.create({
    resource_type: 'ENERGY',
    amount: 65000,
    target_address: targetAddress,
    duration_sec: 3600,
    idempotency_key: randomUUID(),
  })

  // 轮询直到完成(生产环境建议使用Webhook替代)
  let status = order.status
  while (status === 'PENDING' || status === 'EXECUTING') {
    await new Promise(r => setTimeout(r, 2000))
    const updated = await merx.orders.get(order.id)
    status = updated.status

    if (status === 'FILLED') {
      console.log(`Energy delegated. Fills:`)
      for (const fill of updated.fills) {
        console.log(`  ${fill.provider}: ${fill.amount} units, TX: ${fill.tronscan_url}`)
      }
    }
  }

  if (status === 'FAILED') {
    throw new Error(`Order ${order.id} failed`)
  }

  // 能量现已可用于目标地址
  // 继续执行您的USDT转账
}
```

在生产环境中,建议使用Webhook监听器替代轮询循环。为 `order.filled` 事件创建Webhook订阅,委托在链上确认的瞬间您的服务器就会收到通知。

## 环境要求和兼容性

- Node.js 18+(使用原生 `fetch`)
- 零运行时依赖
- 仅ESM(以ES模块形式发布)
- 完整支持TypeScript 5.4+
- 兼容Bun和Deno(任何支持 `fetch` 的运行时)

## 资源

SDK为开源项目,托管在GitHub上。欢迎贡献代码和提交问题报告。

- 平台: [merx.exchange](https://merx.exchange)
- 文档: [merx.exchange/docs](https://merx.exchange/docs)
- GitHub: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- npm: [npmjs.com/package/merx-sdk](https://www.npmjs.com/package/merx-sdk)
- Python SDK: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- MCP Server(AI代理专用): [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)

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
