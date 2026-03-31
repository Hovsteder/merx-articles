# MERX REST API: 46个TRON能量交易接口

MERX REST API 提供46个接口,覆盖TRON能量和带宽交易的完整生命周期 - 从跨八家供应商的实时价格发现到订单执行、账户管理、链上查询以及自动化常备订单。本文将详细介绍API架构、认证模型、接口分组、速率限制、错误处理,以及curl、JavaScript和Python的实用代码示例。

## 为什么统一API至关重要

TRON能量市场分散在多个供应商之间,每家都有自己的API格式、认证方案和定价模型。如果开发者想以最优价格完成一笔USDT转账,就必须逐一对接每个供应商,处理故障切换逻辑,并持续监控价格。

MERX将这一切整合到一个REST API中。一个API密钥、一个认证头、一种错误格式、一套SDK。平台每30秒轮询所有已连接的供应商,将订单路由到最便宜的可用来源,并在链上验证委托。

API版本路径为 `/api/v1/`,并将保持向后兼容。所有接口返回统一信封格式的JSON。

## 认证

MERX使用API密钥认证。每个需要认证的请求必须包含 `X-API-Key` 头。

API密钥可通过 [merx.exchange](https://merx.exchange) 的MERX控制面板创建,也可通过 `/api/v1/keys` 接口以编程方式创建。每个密钥拥有一组权限,控制其可执行的操作。

```bash
curl -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/balance
```

密钥格式为 `sk_live_` 后跟64个十六进制字符。原始密钥仅在创建时显示一次。MERX仅存储bcrypt哈希值,因此丢失的密钥无法恢复 - 只能撤销并重新生成。

### 密钥权限

创建API密钥时,您可以分配一个或多个权限:

| 权限             | 授权访问的接口                          |
|------------------|-----------------------------------------|
| `create_orders`  | POST /orders, POST /ensure              |
| `view_orders`    | GET /orders, GET /orders/:id            |
| `view_balance`   | GET /balance, GET /history              |
| `broadcast`      | POST /chain/broadcast                   |

这允许您为监控面板创建只读密钥,为自动化交易系统创建受限密钥。

## 响应信封

每个响应遵循相同的结构:

```json
{
  "data": { ... }
}
```

错误响应:

```json
{
  "error": {
    "code": "INSUFFICIENT_FUNDS",
    "message": "Account balance too low for the requested operation",
    "details": { "required": 150000000, "available": 42000000 }
  }
}
```

错误代码为机器可读的字符串。`details` 字段为可选项,提供调试上下文信息。

## 接口分组

46个接口分为九组。以下是完整的接口映射。

### 价格 (6个接口)

这些接口为公开接口 - 无需API密钥。

| 方法   | 路径                        | 描述                                     |
|--------|-----------------------------|------------------------------------------|
| GET    | /api/v1/prices              | 所有供应商的当前价格                     |
| GET    | /api/v1/prices/best         | 指定资源类型的最低价供应商               |
| GET    | /api/v1/prices/history      | 历史价格数据                             |
| GET    | /api/v1/prices/stats        | 市场汇总统计                             |
| GET    | /api/v1/prices/analysis     | 趋势分析和购买建议                       |
| GET    | /api/v1/orders/preview      | 下单前的成本预览                         |

### 订单 (3个接口)

| 方法   | 路径                        | 描述                                     |
|--------|-----------------------------|------------------------------------------|
| POST   | /api/v1/orders              | 创建能量或带宽订单                       |
| GET    | /api/v1/orders              | 带分页和过滤的订单列表                   |
| GET    | /api/v1/orders/:id          | 获取订单详情及成交明细                   |

### 账户 (7个接口)

| 方法   | 路径                        | 描述                                     |
|--------|-----------------------------|------------------------------------------|
| GET    | /api/v1/balance             | 当前TRX和USDT余额                       |
| GET    | /api/v1/deposit/info        | 充值地址和备注                           |
| POST   | /api/v1/deposit/prepare     | 准备充值交易                             |
| POST   | /api/v1/deposit/submit      | 提交充值证明                             |
| POST   | /api/v1/withdraw            | 提取TRX或USDT                            |
| GET    | /api/v1/history             | 订单执行历史                             |
| GET    | /api/v1/history/summary     | 账户汇总统计                             |

### API密钥 (3个接口)

| 方法   | 路径                        | 描述                                     |
|--------|-----------------------------|------------------------------------------|
| GET    | /api/v1/keys                | 列出所有API密钥                          |
| POST   | /api/v1/keys                | 创建新的API密钥                          |
| DELETE | /api/v1/keys/:id            | 撤销API密钥                              |

### 认证 (2个接口)

| 方法   | 路径                        | 描述                                     |
|--------|-----------------------------|------------------------------------------|
| POST   | /api/v1/auth/register       | 创建新账户                               |
| POST   | /api/v1/auth/login          | 认证并获取JWT令牌                        |

### 估算 (2个接口)

| 方法   | 路径                        | 描述                                     |
|--------|-----------------------------|------------------------------------------|
| POST   | /api/v1/estimate            | 估算交易所需能量和成本                   |
| POST   | /api/v1/ensure              | 确保地址上有最低资源                     |

### Webhook (3个接口)

| 方法   | 路径                        | 描述                                     |
|--------|-----------------------------|------------------------------------------|
| POST   | /api/v1/webhooks            | 创建Webhook订阅                          |
| GET    | /api/v1/webhooks            | 列出Webhook订阅                          |
| DELETE | /api/v1/webhooks/:id        | 删除Webhook                              |

### 常备订单和监控 (7个接口)

| 方法   | 路径                             | 描述                                  |
|--------|----------------------------------|---------------------------------------|
| POST   | /api/v1/standing-orders          | 创建常备订单                          |
| GET    | /api/v1/standing-orders          | 列出常备订单                          |
| GET    | /api/v1/standing-orders/:id      | 获取常备订单详情                      |
| DELETE | /api/v1/standing-orders/:id      | 取消常备订单                          |
| POST   | /api/v1/monitors                 | 创建资源监控                          |
| GET    | /api/v1/monitors                 | 列出活跃监控                          |
| DELETE | /api/v1/monitors/:id             | 取消监控                              |

### 链上代理 (10个接口)

这些接口通过MERX代理TRON网络查询,无需客户端直接调用TronGrid。

| 方法   | 路径                             | 描述                                  |
|--------|----------------------------------|---------------------------------------|
| GET    | /api/v1/chain/account/:address   | 账户信息和资源                        |
| GET    | /api/v1/chain/balance/:address   | TRX余额                              |
| GET    | /api/v1/chain/resources/:address | 能量和带宽详细分解                    |
| GET    | /api/v1/chain/transaction/:txid  | 交易详情                              |
| GET    | /api/v1/chain/block/:number      | 按编号查询区块(或最新区块)          |
| GET    | /api/v1/chain/parameters         | 链参数                                |
| GET    | /api/v1/chain/history/:address   | 地址交易历史                          |
| POST   | /api/v1/chain/read-contract      | 调用只读合约函数                      |
| POST   | /api/v1/chain/broadcast          | 广播已签名交易                        |
| GET    | /api/v1/address/:addr/resources  | 地址资源汇总                          |

### x402按次付费 (3个接口)

| 方法   | 路径                        | 描述                                     |
|--------|-----------------------------|------------------------------------------|
| POST   | /api/v1/x402/invoice        | 创建支付账单                             |
| GET    | /api/v1/x402/invoice/:id    | 查询账单状态                             |
| POST   | /api/v1/x402/verify         | 验证支付并执行订单                       |

## 核心接口详解

### GET /api/v1/prices

返回所有活跃供应商的当前定价。无需认证。这是您一览市场全貌时调用的接口。

```bash
curl https://merx.exchange/api/v1/prices
```

响应(简略版):

```json
{
  "data": [
    {
      "provider": "sohu",
      "is_market": false,
      "energy_prices": [
        { "duration_sec": 3600, "price_sun": 24 },
        { "duration_sec": 86400, "price_sun": 30 }
      ],
      "bandwidth_prices": [],
      "available_energy": 5000000,
      "available_bandwidth": 0,
      "fetched_at": 1743292800
    }
  ]
}
```

每个供应商条目包含价格层级(按时长分类)、可用容量以及最后一次成功轮询的时间戳。

### POST /api/v1/orders

创建能量或带宽订单。平台根据可用供应商进行匹配,路由到能够履行订单的最低价供应商。

```bash
curl -X POST https://merx.exchange/api/v1/orders \
  -H "X-API-Key: sk_live_your_key_here" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: unique-request-id-123" \
  -d '{
    "resource_type": "ENERGY",
    "order_type": "MARKET",
    "amount": 65000,
    "target_address": "TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    "duration_sec": 3600
  }'
```

响应:

```json
{
  "data": {
    "id": "ord_a1b2c3d4",
    "status": "PENDING",
    "created_at": "2026-03-30T12:00:00.000Z"
  }
}
```

`Idempotency-Key` 头用于防止相同请求重试时产生重复订单。如果该密钥之前已使用过,API将返回原始订单而非创建新订单。

订单类型:
- `MARKET` - 以最优可用价格立即执行
- `LIMIT` - 仅当价格等于或低于 `max_price_sun` 时执行
- `PERIODIC` - 按计划周期性执行的订单
- `BROADCAST` - 广播预签名的委托交易

### GET /api/v1/orders/:id

返回订单及其成交详情 - 哪些供应商完成了订单、以何种价格成交,以及链上交易ID。

```bash
curl -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/orders/ord_a1b2c3d4
```

```json
{
  "data": {
    "id": "ord_a1b2c3d4",
    "resource_type": "ENERGY",
    "order_type": "MARKET",
    "status": "FILLED",
    "amount": 65000,
    "target_address": "TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    "duration_sec": 3600,
    "total_cost_sun": 1560000,
    "fills": [
      {
        "provider": "sohu",
        "amount": 65000,
        "price_sun": 24,
        "cost_sun": 1560000,
        "delegation_tx": "abc123def456...",
        "verified": true,
        "tronscan_url": "https://tronscan.org/#/transaction/abc123def456..."
      }
    ]
  }
}
```

### POST /api/v1/estimate

估算TRON操作所需的能量和带宽,然后比较租赁成本与TRX燃烧成本。

```bash
curl -X POST https://merx.exchange/api/v1/estimate \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "trc20_transfer",
    "to_address": "TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    "amount": "1000000"
  }'
```

```json
{
  "data": {
    "energy_required": 65000,
    "bandwidth_required": 345,
    "rental_cost": {
      "energy": {
        "best_price_sun": 24,
        "best_provider": "sohu",
        "cost_trx": "1.560"
      }
    },
    "total_rental_trx": "1.905",
    "total_burn_trx": "27.645",
    "savings_percent": 93.1
  }
}
```

此接口用于向用户准确展示通过MERX租赁能量相比直接燃烧TRX可以节省多少成本。

## 速率限制

速率限制按IP地址使用滑动窗口机制应用。

| 接口组别            | 限制               | 窗口    |
|---------------------|--------------------|---------|
| 价格(公开)        | 300次请求          | 1分钟   |
| 默认(通用)        | 100次请求          | 1分钟   |
| 余额                | 60次请求           | 1分钟   |
| 历史记录            | 60次请求           | 1分钟   |
| 订单                | 10次请求           | 1分钟   |
| 提现                | 5次请求            | 1分钟   |
| 广播                | 20次请求           | 1分钟   |
| 注册                | 5次请求            | 1小时   |

超出速率限制时,API返回HTTP 429,使用标准错误格式:

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded"
  }
}
```

速率限制头(`RateLimit-Limit`、`RateLimit-Remaining`、`RateLimit-Reset`)按照IETF草案标准包含在所有响应中。

## 错误代码

| 代码                   | HTTP | 描述                                               |
|------------------------|------|----------------------------------------------------|
| `UNAUTHORIZED`         | 401  | 无效或缺少API密钥                                 |
| `RATE_LIMITED`         | 429  | 请求过多                                           |
| `VALIDATION_ERROR`    | 400  | 请求体或参数验证失败                               |
| `INVALID_ADDRESS`     | 400  | 不是有效的TRON地址                                 |
| `INSUFFICIENT_FUNDS`  | 400  | 账户余额不足                                       |
| `BELOW_MINIMUM_ORDER` | 400  | 订单金额低于供应商最低限额                         |
| `DUPLICATE_REQUEST`   | 409  | 幂等性密钥已被使用                                 |
| `ORDER_NOT_FOUND`     | 404  | 订单或资源不存在                                   |
| `PROVIDER_UNAVAILABLE`| 404  | 没有供应商可以完成该请求                           |
| `INTERNAL_ERROR`      | 500  | 服务端错误                                         |

## SDK快速入门

虽然REST API可以通过任何HTTP客户端直接调用,但MERX提供了JavaScript和Python的官方SDK,处理认证、错误解析和类型安全。

### JavaScript / TypeScript

```bash
npm install merx-sdk
```

```javascript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({ apiKey: 'sk_live_your_key_here' })

// 获取所有价格
const prices = await merx.prices.list()

// 创建订单
const order = await merx.orders.create({
  resource_type: 'ENERGY',
  amount: 65000,
  target_address: 'TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb',
  duration_sec: 3600,
})

// 查询订单状态
const details = await merx.orders.get(order.id)
```

### Python

```bash
pip install merx-sdk
```

```python
from merx import MerxClient

client = MerxClient(api_key="sk_live_your_key_here")

# 获取所有价格
prices = client.prices.list()

# 创建订单
order = client.orders.create(
    resource_type="ENERGY",
    amount=65000,
    target_address="TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    duration_sec=3600,
)

# 查询订单状态
details = client.orders.get(order.id)
```

## WebSocket实时数据

除REST API外,MERX还提供 `wss://merx.exchange/ws` 上的WebSocket接口用于实时价格更新。价格变动会在发生时即刻推送给已连接的客户端,每个供应商每30秒更新一次。

WebSocket连接支持供应商过滤 - 仅订阅您关注的供应商,忽略其余供应商。

## 常备订单

常备订单基于触发条件自动执行能量采购。您可以设置价格阈值、时间计划或余额条件,平台在您指定的预算内自动执行订单。

触发类型包括 `price_below`、`price_above`、`schedule`、`balance_below` 和 `provider_available`。操作类型包括 `buy_resource`、`ensure_resources`、`deposit_trx` 和 `notify_only`。

这使得MERX适用于全自动化的基础设施管理 - 设定规则一次,平台负责执行。

## 后续发展

MERX API专为需要可靠、低成本访问TRON网络资源的开发者和企业而设计。无论您是在构建支付处理器、DeFi应用还是交易所,该API都提供了以编程方式管理能量和带宽的基础模块。

完整的API文档请访问 [merx.exchange/docs](https://merx.exchange/docs)。JavaScript SDK发布在 [GitHub](https://github.com/Hovsteder/merx-sdk-js) 和 [npm](https://www.npmjs.com/package/merx-sdk)。Python SDK发布在 [PyPI](https://pypi.org/project/merx-sdk/)。

---

**链接:**
- 平台: [merx.exchange](https://merx.exchange)
- 文档: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- Python SDK: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- MCP Server: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp) | [npm](https://www.npmjs.com/package/merx-mcp)
