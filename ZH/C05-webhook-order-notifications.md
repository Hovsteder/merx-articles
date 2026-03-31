# Webhook集成: 能量订单成交实时通知

MERX Webhook在您的账户发生事件时发送实时HTTP通知 - 订单成交、订单失败、充值到账、提现完成。本文涵盖全部四种事件类型、载荷格式、使用 `X-Merx-Signature` 头进行HMAC-SHA256签名验证、带指数退避的重试策略、连续失败后的自动停用机制,以及Express.js和Flask的完整服务端实现(含签名验证)。

## 为什么选择Webhook而非轮询

Webhook的替代方案是在循环中轮询 `/api/v1/orders/:id` 接口,等待状态从 `PENDING` 变为 `FILLED`。这在简单场景下可行,但有明显的缺陷:

- 浪费请求。大多数轮询返回的是未变化的相同状态。
- 延迟。您的应用只能在下一个轮询间隔时发现状态变化。
- 速率限制。订单接口每分钟10次请求的限制下,频繁轮询很快就会触及上限。
- 复杂性。轮询逻辑需要重试处理、超时管理和状态跟踪。

Webhook颠覆了这个模型。不是您去问MERX"有变化吗?",而是MERX在事件发生的瞬间告诉您。您的服务器收到一个包含完整事件载荷的HTTP POST请求,处理它,然后继续。没有轮询循环,没有浪费的请求,没有人为延迟。

## 创建Webhook

您可以通过REST API或SDK创建Webhook。

### REST API

```bash
curl -X POST https://merx.exchange/api/v1/webhooks \
  -H "X-API-Key: sk_live_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-server.com/webhooks/merx",
    "events": ["order.filled", "order.failed", "deposit.received", "withdrawal.completed"]
  }'
```

响应:

```json
{
  "data": {
    "id": "wh_abc123",
    "url": "https://your-server.com/webhooks/merx",
    "events": ["order.filled", "order.failed", "deposit.received", "withdrawal.completed"],
    "secret": "a1b2c3d4e5f6...64-hex-characters...9876543210",
    "is_active": true,
    "created_at": "2026-03-30T12:00:00.000Z"
  }
}
```

`secret` 字段是由32个随机字节生成的64字符十六进制字符串。它仅在创建响应中返回。请安全存储 - 您将需要它来验证传入的Webhook签名。

### JavaScript SDK

```javascript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY })

const webhook = await merx.webhooks.create({
  url: 'https://your-server.com/webhooks/merx',
  events: ['order.filled', 'order.failed', 'deposit.received'],
})

console.log(`Webhook ID: ${webhook.id}`)
console.log(`Secret: ${webhook.secret}`)  // 安全存储
```

### Python SDK

```python
from merx import MerxClient

client = MerxClient(api_key="sk_live_your_key_here")

webhook = client.webhooks.create(
    url="https://your-server.com/webhooks/merx",
    events=["order.filled", "order.failed", "deposit.received"],
)

print(f"Webhook ID: {webhook.id}")
print(f"Secret: {webhook.secret}")  # 安全存储
```

## 事件类型

MERX支持四种Webhook事件类型。创建Webhook时,您选择要订阅哪些事件。可以订阅全部四种,也可以只选择需要的。

### order.filled

当订单完全完成时发送。所有供应商委托已在链上确认。

```json
{
  "event": "order.filled",
  "timestamp": "2026-03-30T12:05:00.000Z",
  "data": {
    "order_id": "ord_abc123",
    "resource_type": "ENERGY",
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

这是自动化系统最重要的事件。收到此事件时,能量已完成委托,目标地址可以继续执行TRON交易。

### order.failed

当订单无法完成时发送。可能因为所有供应商不可用、容量耗尽或供应商侧出现错误。

```json
{
  "event": "order.failed",
  "timestamp": "2026-03-30T12:05:00.000Z",
  "data": {
    "order_id": "ord_def456",
    "resource_type": "ENERGY",
    "amount": 500000,
    "target_address": "TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    "reason": "No provider could fulfill the order within the specified parameters",
    "refunded": true,
    "refund_amount_sun": 0
  }
}
```

当订单失败时,任何预留的余额都会被退还。`refunded` 字段确认此操作,`refund_amount_sun` 显示已退还的金额(如果款项已被扣除)。

### deposit.received

当检测到并记入您的MERX账户充值时发送。

```json
{
  "event": "deposit.received",
  "timestamp": "2026-03-30T12:10:00.000Z",
  "data": {
    "deposit_id": "dep_ghi789",
    "amount_sun": 100000000,
    "amount_trx": "100.000000",
    "currency": "TRX",
    "tx_id": "789abc...",
    "new_balance_trx": 250.5
  }
}
```

### withdrawal.completed

当提现请求已处理且链上交易确认时发送。

```json
{
  "event": "withdrawal.completed",
  "timestamp": "2026-03-30T12:15:00.000Z",
  "data": {
    "withdrawal_id": "wdr_jkl012",
    "amount": 50,
    "currency": "TRX",
    "address": "TExternalAddress...",
    "tx_id": "012def...",
    "tronscan_url": "https://tronscan.org/#/transaction/012def..."
  }
}
```

## 签名验证

每次Webhook投递都包含 `X-Merx-Signature` 头,包含使用您的Webhook密钥作为密钥计算的请求体的HMAC-SHA256签名。

验证过程:

1. 读取原始请求体(JSON解析之前)。
2. 使用存储的Webhook密钥计算原始请求体的HMAC-SHA256。
3. 将计算出的签名与 `X-Merx-Signature` 头的值进行比较。
4. 如果匹配,则请求是真实的。如果不匹配,则拒绝。

此机制防御以下攻击:
- 不知道您密钥的第三方伪造请求。
- 传输过程中请求体被篡改。
- 重放攻击(结合时间戳验证)。

比较签名时务必使用常量时间比较函数。标准的字符串相等比较(`===` 或 `==`)容易受到时序攻击。

## Express.js Webhook处理器

以下是一个完整的Express.js服务器,接收并验证MERX Webhook:

```javascript
import express from 'express'
import crypto from 'node:crypto'

const app = express()
const WEBHOOK_SECRET = process.env.MERX_WEBHOOK_SECRET

// 使用原始请求体进行签名验证
app.post('/webhooks/merx', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-merx-signature']
  if (!signature || !WEBHOOK_SECRET) {
    res.status(401).json({ error: 'Missing signature or secret' })
    return
  }

  // 计算期望的签名
  const expected = crypto
    .createHmac('sha256', WEBHOOK_SECRET)
    .update(req.body)
    .digest('hex')

  // 常量时间比较
  if (!crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected))) {
    console.warn('Webhook signature mismatch')
    res.status(401).json({ error: 'Invalid signature' })
    return
  }

  // 签名已验证 - 解析并处理事件
  const event = JSON.parse(req.body.toString())
  console.log(`Received event: ${event.event}`)

  switch (event.event) {
    case 'order.filled':
      handleOrderFilled(event.data)
      break
    case 'order.failed':
      handleOrderFailed(event.data)
      break
    case 'deposit.received':
      handleDepositReceived(event.data)
      break
    case 'withdrawal.completed':
      handleWithdrawalCompleted(event.data)
      break
    default:
      console.warn(`Unknown event type: ${event.event}`)
  }

  // 始终快速返回200以确认接收
  res.status(200).json({ received: true })
})

function handleOrderFilled(data) {
  console.log(`Order ${data.order_id} filled`)
  console.log(`  Amount: ${data.amount} ${data.resource_type}`)
  console.log(`  Cost: ${(data.total_cost_sun / 1_000_000).toFixed(3)} TRX`)
  console.log(`  Fills: ${data.fills.length}`)

  for (const fill of data.fills) {
    console.log(`  ${fill.provider}: ${fill.amount} at ${fill.price_sun} SUN`)
    if (fill.tronscan_url) {
      console.log(`    TX: ${fill.tronscan_url}`)
    }
  }

  // 您的业务逻辑:
  // - 标记交易已准备就绪
  // - 触发USDT转账
  // - 更新数据库
}

function handleOrderFailed(data) {
  console.log(`Order ${data.order_id} failed: ${data.reason}`)
  if (data.refunded) {
    console.log(`  Refunded: ${data.refund_amount_sun} SUN`)
  }
  // 通知运维团队
}

function handleDepositReceived(data) {
  console.log(`Deposit received: ${data.amount_trx} ${data.currency}`)
  console.log(`  New balance: ${data.new_balance_trx} TRX`)
  // 更新本地余额缓存
}

function handleWithdrawalCompleted(data) {
  console.log(`Withdrawal ${data.withdrawal_id} completed`)
  console.log(`  ${data.amount} ${data.currency} to ${data.address}`)
  // 更新系统中的提现状态
}

app.listen(3000, () => {
  console.log('Webhook server listening on port 3000')
})
```

关键实现细节:

- Webhook路由使用 `express.raw({ type: 'application/json' })` 而非 `express.json()`。您需要原始字节来计算签名。
- 使用 `crypto.timingSafeEqual()` 进行常量时间签名比较。
- 尽快返回HTTP 200。在确认接收后异步执行耗时的处理逻辑。

## Flask Webhook处理器

以下是使用Flask的等效Python实现:

```python
import hashlib
import hmac
import json
import os

from flask import Flask, request, jsonify

app = Flask(__name__)
WEBHOOK_SECRET = os.environ.get("MERX_WEBHOOK_SECRET", "")


def verify_signature(payload: bytes, signature: str, secret: str) -> bool:
    """使用常量时间比较验证HMAC-SHA256签名。"""
    expected = hmac.new(
        secret.encode("utf-8"),
        payload,
        hashlib.sha256,
    ).hexdigest()
    return hmac.compare_digest(signature, expected)


@app.route("/webhooks/merx", methods=["POST"])
def handle_webhook():
    signature = request.headers.get("X-Merx-Signature", "")
    raw_body = request.get_data()

    if not signature or not WEBHOOK_SECRET:
        return jsonify({"error": "Missing signature or secret"}), 401

    if not verify_signature(raw_body, signature, WEBHOOK_SECRET):
        app.logger.warning("Webhook signature mismatch")
        return jsonify({"error": "Invalid signature"}), 401

    event = json.loads(raw_body)
    event_type = event.get("event", "")
    data = event.get("data", {})

    app.logger.info(f"Received event: {event_type}")

    if event_type == "order.filled":
        handle_order_filled(data)
    elif event_type == "order.failed":
        handle_order_failed(data)
    elif event_type == "deposit.received":
        handle_deposit_received(data)
    elif event_type == "withdrawal.completed":
        handle_withdrawal_completed(data)
    else:
        app.logger.warning(f"Unknown event type: {event_type}")

    return jsonify({"received": True}), 200


def handle_order_filled(data):
    order_id = data["order_id"]
    amount = data["amount"]
    resource = data["resource_type"]
    cost_trx = data["total_cost_sun"] / 1_000_000

    app.logger.info(f"Order {order_id} filled: {amount} {resource}, {cost_trx:.3f} TRX")

    for fill in data.get("fills", []):
        app.logger.info(
            f"  {fill['provider']}: {fill['amount']} at {fill['price_sun']} SUN"
        )

    # 您的业务逻辑


def handle_order_failed(data):
    app.logger.warning(
        f"Order {data['order_id']} failed: {data.get('reason', 'unknown')}"
    )


def handle_deposit_received(data):
    app.logger.info(
        f"Deposit: {data['amount_trx']} {data['currency']}, "
        f"new balance: {data.get('new_balance_trx', 'N/A')} TRX"
    )


def handle_withdrawal_completed(data):
    app.logger.info(
        f"Withdrawal {data['withdrawal_id']}: "
        f"{data['amount']} {data['currency']} to {data['address']}"
    )


if __name__ == "__main__":
    app.run(port=3000)
```

Python特定的关键细节:

- 使用 `request.get_data()` 获取原始请求体的字节数据。
- 使用 `hmac.compare_digest()` 进行常量时间字符串比较。Python的 `==` 运算符不是常量时间的。
- 使用 `hmac.new()` 配合 `hashlib.sha256` 计算HMAC。

## 重试策略

MERX使用指数退避计划重试失败的Webhook投递:

| 尝试次数 | 失败后延迟    |
|----------|---------------|
| 1        | 立即          |
| 2        | 30秒          |
| 3        | 5分钟         |

以下情况被视为投递失败:
- 您的服务器在10秒内没有响应。
- 您的服务器返回2xx范围之外的HTTP状态码。
- 无法建立连接(DNS解析失败、连接被拒绝、TLS错误)。

单个事件在3次尝试失败后将被丢弃。不会对该特定投递进行更多重试。

您的Webhook处理器应当:
- 在几秒内返回HTTP 200。将耗时的处理逻辑放到后台异步执行。
- 保持幂等性。极端情况下同一事件可能被投递多次。
- 如果处理失败,记录事件载荷以供调试。

## 自动停用

如果Webhook端点持续失败,MERX会自动停用它,以避免在无效端点上浪费资源。

停用阈值基于跨多个事件的连续失败次数。如果您的端点反复无法接受投递,Webhook的 `is_active` 标志将被设置为 `false`。

当Webhook被停用时:
- 不会再向该URL发送更多事件。
- 该Webhook仍会以 `is_active: false` 状态出现在列表中。
- 您可以修复端点问题并创建新的Webhook。

定期监控您的Webhook状态:

```javascript
const webhooks = await merx.webhooks.list()
for (const wh of webhooks) {
  if (!wh.is_active) {
    console.warn(`Webhook ${wh.id} (${wh.url}) is deactivated`)
  }
}
```

```python
webhooks = client.webhooks.list()
for wh in webhooks:
    if not wh.is_active:
        print(f"Webhook {wh.id} ({wh.url}) is deactivated")
```

## 管理Webhook

### 列出Webhook

```bash
curl -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/webhooks
```

### 删除Webhook

```bash
curl -X DELETE -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/webhooks/wh_abc123
```

请注意,Webhook密钥在创建后无法再次获取。如果您丢失了密钥,请删除该Webhook并创建新的。

## 本地测试Webhook

在开发阶段,您的Webhook URL需要是公开可访问的。ngrok等工具可以将本地服务器暴露到公网:

```bash
# 终端1: 启动您的Webhook服务器
node webhook-server.js

# 终端2: 公开暴露
ngrok http 3000
```

创建Webhook时使用ngrok URL(例如 `https://abc123.ngrok.io/webhooks/merx`)。确认一切正常后,替换为您的生产URL。

## 最佳实践

1. 始终验证签名。不要在未检查 `X-Merx-Signature` 头的情况下信任Webhook载荷。

2. 快速响应。在2-3秒内返回HTTP 200。将耗时的处理逻辑排入后台工作队列。

3. 保持幂等性。使用 `order_id` 或 `deposit_id` 作为去重键。如果两次收到相同的事件,第二次处理应为空操作。

4. 存储原始载荷。在处理之前记录完整的JSON请求体。如果处理器有bug,您可以从日志中重放事件。

5. 监控Webhook健康状态。定期检查 `is_active` 状态。设置Webhook被停用时的告警。

6. 使用HTTPS。MERX要求Webhook URL使用HTTPS。不接受自签名证书。

7. 选择性订阅。仅订阅您实际处理的事件。不必要的事件会浪费带宽和处理时间。

## 资源

- 平台: [merx.exchange](https://merx.exchange)
- 文档: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- Python SDK: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- MCP Server: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
