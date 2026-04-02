# 通过WebSocket获取实时TRON能量价格

MERX提供 `wss://merx.exchange/ws` 上的WebSocket接口,实时推送所有已连接供应商的TRON能量和带宽价格。本文涵盖连接协议、消息格式、通过订阅消息实现的供应商过滤、心跳和重连策略,以及JavaScript和Python的完整代码示例 - 包括一个实用的价格警报机器人,当能量价格低于目标价格时发送通知。

## 为什么选择WebSocket而非轮询

MERX REST API的 `/api/v1/prices` 接口返回所有供应商的当前价格。您可以定期轮询它,速率限制为每分钟300次请求,如果需要的话,每200毫秒就可以刷新一次。

但轮询有固有的缺陷。每次请求都是完整的HTTP往返,包含TLS握手开销。您永远不知道价格变动的确切时刻 - 只能在下一个轮询间隔时发现。如果有多个客户端,每个客户端都独立地访问同一个接口。

WebSocket连接解决了所有这些问题。一个持久连接在价格变动发生的瞬间接收更新。MERX价格监控服务每30秒轮询所有八个供应商,并通过Redis pub/sub发布更新。WebSocket服务器订阅该频道,并将更新即时分发给所有已连接的客户端。

结果是: 更低的延迟、更低的带宽消耗,以及每次价格变动的可靠送达。

## 连接

连接到 `wss://merx.exchange/ws` 的WebSocket接口。无需认证 - 价格数据是公开的。

### JavaScript(浏览器或Node.js)

```javascript
const ws = new WebSocket('wss://merx.exchange/ws')

ws.addEventListener('open', () => {
  console.log('Connected to MERX WebSocket')
})

ws.addEventListener('message', (event) => {
  const msg = JSON.parse(event.data)
  console.log(`[${msg.provider}] Price update at ${new Date(msg.ts).toISOString()}`)
  console.log(msg.data)
})

ws.addEventListener('close', (event) => {
  console.log(`Disconnected: code=${event.code} reason=${event.reason}`)
})

ws.addEventListener('error', (error) => {
  console.error('WebSocket error:', error)
})
```

### Python(使用websockets库)

```python
import asyncio
import json
import websockets

async def listen():
    uri = "wss://merx.exchange/ws"
    async with websockets.connect(uri) as ws:
        print("Connected to MERX WebSocket")
        async for raw in ws:
            msg = json.loads(raw)
            print(f"[{msg['provider']}] {msg['data']}")

asyncio.run(listen())
```

使用 `pip install websockets` 安装websockets库。

## 消息格式

服务器发送的每条消息都是包含三个字段的JSON对象:

```json
{
  "type": "price_update",
  "provider": "sohu",
  "data": {
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
  },
  "ts": 1743292800123
}
```

| 字段       | 类型   | 描述                                                 |
|------------|--------|------------------------------------------------------|
| `type`     | string | 固定为 `"price_update"`                              |
| `provider` | string | 供应商标识(例如 "sohu"、"catfee"、"itrx")          |
| `data`     | object | 完整的供应商价格数据,格式与REST API相同             |
| `ts`       | number | 服务器时间戳(毫秒)                                 |

`data` 对象的结构与 `/api/v1/prices` REST响应中的单个元素相同。这意味着您可以对实时数据和轮询数据使用相同的解析逻辑。

### 供应商标识

MERX目前监控八个供应商:

| 供应商     | 标识         |
|------------|-------------|
| Sohu       | `sohu`      |
| CatFee     | `catfee`    |
| Netts      | `netts`     |
| TronSave   | `tronsave`  |
| Feee       | `feee`      |
| ITRX       | `itrx`      |
| PowerSun   | `powersun`  |
| TEM        | `tem`       |

每个供应商大约每30秒发送一次更新。八个供应商的情况下,平均每3-4秒可以收到一次更新。

## 订阅特定供应商

默认情况下,新连接接收所有供应商的更新。如需过滤,在连接后发送订阅消息:

```json
{ "subscribe": ["sohu", "catfee", "itrx"] }
```

这告诉服务器仅发送指定供应商的更新。其他所有更新会被静默丢弃。

要重置并接收所有更新,发送空数组:

```json
{ "subscribe": [] }
```

### JavaScript订阅示例

```javascript
const ws = new WebSocket('wss://merx.exchange/ws')

ws.addEventListener('open', () => {
  // 仅接收sohu和catfee的更新
  ws.send(JSON.stringify({
    subscribe: ['sohu', 'catfee']
  }))
})

ws.addEventListener('message', (event) => {
  const msg = JSON.parse(event.data)
  // msg.provider 只会是 "sohu" 或 "catfee"
  console.log(`${msg.provider}: ${msg.data.energy_prices[0]?.price_sun} SUN`)
})
```

### Python订阅示例

```python
import asyncio
import json
import websockets

async def listen_filtered():
    async with websockets.connect("wss://merx.exchange/ws") as ws:
        # 订阅特定供应商
        await ws.send(json.dumps({
            "subscribe": ["sohu", "catfee"]
        }))
        print("Subscribed to sohu and catfee")

        async for raw in ws:
            msg = json.loads(raw)
            prices = msg["data"].get("energy_prices", [])
            if prices:
                print(f"{msg['provider']}: {prices[0]['price_sun']} SUN")

asyncio.run(listen_filtered())
```

您可以随时通过发送新的订阅消息来更改订阅。服务器将用新的过滤器替换之前的过滤器。

## 心跳和连接健康

MERX WebSocket服务器每30秒发送一次ping帧。客户端必须响应pong帧以保持连接活跃。如果客户端未能响应ping,服务器将在下一个心跳周期(大约30秒后)断开连接。

大多数WebSocket库在协议层面自动处理ping/pong。Python的 `websockets` 库和浏览器的 `WebSocket` API都会自动响应ping,无需额外代码。

对于使用 `ws` 库的Node.js,pong响应也是自动的。但如果您使用的库不自动处理ping,需要监听ping事件:

```javascript
// Node.js使用'ws'库(自动处理pong)
import WebSocket from 'ws'

const ws = new WebSocket('wss://merx.exchange/ws')

ws.on('ping', () => {
  // ws库自动发送pong
  // 此处理函数仅用于日志记录
  console.log('Received ping, pong sent')
})
```

## 重连策略

WebSocket连接可能因多种原因断开: 网络中断、服务器部署、负载均衡器超时。生产客户端必须优雅地处理重连。

推荐的策略是带随机抖动的指数退避:

1. 断开连接后,等待1秒再重连。
2. 如果重连失败,将等待时间翻倍: 2秒、4秒、8秒,最长30秒。
3. 添加随机抖动(0-1秒)以防止所有客户端同时重连。
4. 重连成功后,将退避时间重置为1秒。
5. 重连后重新发送订阅消息。

### JavaScript重连

```javascript
function createConnection(providers = []) {
  let backoff = 1000
  const maxBackoff = 30000

  function connect() {
    const ws = new WebSocket('wss://merx.exchange/ws')

    ws.addEventListener('open', () => {
      console.log('Connected')
      backoff = 1000  // 成功后重置退避

      if (providers.length > 0) {
        ws.send(JSON.stringify({ subscribe: providers }))
      }
    })

    ws.addEventListener('message', (event) => {
      const msg = JSON.parse(event.data)
      handlePriceUpdate(msg)
    })

    ws.addEventListener('close', () => {
      const jitter = Math.random() * 1000
      const delay = Math.min(backoff + jitter, maxBackoff)
      console.log(`Disconnected. Reconnecting in ${Math.round(delay)}ms`)
      setTimeout(connect, delay)
      backoff = Math.min(backoff * 2, maxBackoff)
    })

    ws.addEventListener('error', () => {
      ws.close()  // 触发close处理函数进行重连
    })

    return ws
  }

  return connect()
}

function handlePriceUpdate(msg) {
  const prices = msg.data.energy_prices || []
  if (prices.length > 0) {
    console.log(`${msg.provider}: ${prices[0].price_sun} SUN/energy`)
  }
}

// 连接并订阅两个供应商
createConnection(['sohu', 'catfee'])
```

### Python重连

```python
import asyncio
import json
import websockets

async def listen_with_reconnect(providers=None):
    backoff = 1.0
    max_backoff = 30.0

    while True:
        try:
            async with websockets.connect("wss://merx.exchange/ws") as ws:
                print("Connected")
                backoff = 1.0  # 成功后重置

                if providers:
                    await ws.send(json.dumps({"subscribe": providers}))

                async for raw in ws:
                    msg = json.loads(raw)
                    handle_price_update(msg)

        except (websockets.ConnectionClosed, ConnectionError, OSError) as e:
            jitter = asyncio.get_event_loop().time() % 1
            delay = min(backoff + jitter, max_backoff)
            print(f"Disconnected ({e}). Reconnecting in {delay:.1f}s")
            await asyncio.sleep(delay)
            backoff = min(backoff * 2, max_backoff)

def handle_price_update(msg):
    prices = msg["data"].get("energy_prices", [])
    if prices:
        print(f"{msg['provider']}: {prices[0]['price_sun']} SUN/energy")

asyncio.run(listen_with_reconnect(["sohu", "catfee"]))
```

## 构建价格警报机器人

以下是一个实际使用场景: 一个监控实时价格的机器人,当能量价格低于目标价格时发送通知。此示例使用简单的控制台提醒,但您可以将其替换为Telegram消息、Slack Webhook或电子邮件。

### JavaScript价格警报机器人

```javascript
const TARGET_PRICE_SUN = 25
const COOLDOWN_MS = 300000  // 两次警报之间间隔5分钟

let lastAlertTime = 0

function createAlertBot() {
  let backoff = 1000

  function connect() {
    const ws = new WebSocket('wss://merx.exchange/ws')

    ws.addEventListener('open', () => {
      console.log('Price alert bot connected')
      backoff = 1000
    })

    ws.addEventListener('message', (event) => {
      const msg = JSON.parse(event.data)
      checkPriceAlert(msg)
    })

    ws.addEventListener('close', () => {
      const delay = Math.min(backoff + Math.random() * 1000, 30000)
      setTimeout(connect, delay)
      backoff = Math.min(backoff * 2, 30000)
    })

    ws.addEventListener('error', () => ws.close())
  }

  connect()
}

function checkPriceAlert(msg) {
  const prices = msg.data.energy_prices || []
  if (prices.length === 0) return

  const cheapest = prices[0].price_sun
  const now = Date.now()

  if (cheapest <= TARGET_PRICE_SUN && now - lastAlertTime > COOLDOWN_MS) {
    lastAlertTime = now
    console.log(`ALERT: ${msg.provider} energy at ${cheapest} SUN (target: ${TARGET_PRICE_SUN})`)
    console.log(`  Available: ${msg.data.available_energy.toLocaleString()} units`)
    console.log(`  Time: ${new Date(msg.ts).toISOString()}`)
    // 替换为您的通知逻辑:
    // sendTelegramMessage(...)
    // postToSlack(...)
  }
}

createAlertBot()
```

### Python价格警报机器人

```python
import asyncio
import json
import time
import websockets

TARGET_PRICE_SUN = 25
COOLDOWN_SEC = 300  # 两次警报之间间隔5分钟
last_alert_time = 0

async def alert_bot():
    global last_alert_time
    backoff = 1.0

    while True:
        try:
            async with websockets.connect("wss://merx.exchange/ws") as ws:
                print("Price alert bot connected")
                backoff = 1.0

                async for raw in ws:
                    msg = json.loads(raw)
                    check_alert(msg)

        except (websockets.ConnectionClosed, ConnectionError, OSError):
            delay = min(backoff + (time.time() % 1), 30.0)
            print(f"Reconnecting in {delay:.1f}s")
            await asyncio.sleep(delay)
            backoff = min(backoff * 2, 30.0)

def check_alert(msg):
    global last_alert_time
    prices = msg["data"].get("energy_prices", [])
    if not prices:
        return

    cheapest = prices[0]["price_sun"]
    now = time.time()

    if cheapest <= TARGET_PRICE_SUN and now - last_alert_time > COOLDOWN_SEC:
        last_alert_time = now
        provider = msg["provider"]
        available = msg["data"].get("available_energy", 0)
        print(f"ALERT: {provider} energy at {cheapest} SUN (target: {TARGET_PRICE_SUN})")
        print(f"  Available: {available:,} units")
        # 替换为您的通知逻辑

asyncio.run(alert_bot())
```

## 维护本地状态

对于需要展示实时市场视图的应用,可以维护一个最新价格的本地字典,并在每条消息到达时更新:

```javascript
const latestPrices = new Map()

function handlePriceUpdate(msg) {
  latestPrices.set(msg.provider, {
    data: msg.data,
    receivedAt: msg.ts,
  })

  // 在所有供应商中查找当前最低价
  let cheapest = null
  for (const [provider, entry] of latestPrices) {
    const prices = entry.data.energy_prices || []
    if (prices.length === 0) continue
    const price = prices[0].price_sun
    if (cheapest === null || price < cheapest.price) {
      cheapest = { provider, price, available: entry.data.available_energy }
    }
  }

  if (cheapest) {
    console.log(`Market best: ${cheapest.provider} at ${cheapest.price} SUN`)
  }
}
```

## 结合WebSocket与REST API

一种常见模式是使用REST API加载初始状态,然后使用WebSocket进行增量更新:

```javascript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY })

// 从REST API加载初始状态
const initialPrices = await merx.prices.list()
const priceMap = new Map()
for (const p of initialPrices) {
  priceMap.set(p.provider, p)
}
console.log(`Loaded ${priceMap.size} providers from REST API`)

// 切换到WebSocket进行实时更新
const ws = new WebSocket('wss://merx.exchange/ws')
ws.addEventListener('message', (event) => {
  const msg = JSON.parse(event.data)
  priceMap.set(msg.provider, msg.data)
  // 您的UI或逻辑现在拥有一个持续更新的价格映射
})
```

这确保您从首次渲染起就拥有完整的市场视图,此后的更新延迟为零。

## 性能注意事项

WebSocket接口专为高吞吐量消费而设计。以下是生产部署的几点建议:

- 每个供应商更新大约200-500字节的JSON。八个供应商每30秒更新一次,总带宽低于1 KB/s。
- 使用订阅过滤器减少流量(如果您只需要特定供应商)。
- 服务端心跳间隔为30秒。如果您的应用有防火墙或代理在更短时间内关闭空闲连接,请发送应用层级的保活消息。
- WebSocket路径为 `/ws`。请确保您的反向代理或负载均衡器已配置为在此路径上升级HTTP连接。

## 资源

- 平台: [merx.exchange](https://merx.exchange)
- 文档: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- Python SDK: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- MCP Server: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)

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
