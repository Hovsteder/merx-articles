# Цены на energy TRON в реальном времени через WebSocket

MERX provides a WebSocket endpoint at `wss://merx.exchange/ws` that streams real-time TRON energy and bandwidth prices from all connected providers. This article covers the connection protocol, message format, provider filtering via subscription messages, heartbeat and reconnection strategies, and complete code examples in JavaScript and Python - including a practical price alert bot that notifies you when energy drops below a target price.

## Why WebSocket Instead of Polling

The MERX REST API at `/api/v1/prices` returns current prices from all providers. You can poll it at regular intervals, and with a rate limit of 300 requests per minute, you can refresh every 200 milliseconds if you want.

But polling has inherent drawbacks. Each request is a full HTTP roundtrip with TLS handshake overhead. You never know the exact moment a price changes - you only discover it at your next poll interval. And if you have many clients, each one hammers the same endpoint independently.

The WebSocket connection solves all of these. A single persistent connection receives price updates the moment they happen. The MERX price monitor service polls all eight providers every 30 seconds and publishes updates via Redis pub/sub. The WebSocket server subscribes to that channel and fans out updates to all connected clients instantly.

The result: lower latency, lower bandwidth usage, and guaranteed delivery of every price change.

## Connection

Connect to the WebSocket endpoint at `wss://merx.exchange/ws`. No authentication is required - price data is public.

### JavaScript (Browser or Node.js)

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

### Python (using websockets library)

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

Install the websockets library with `pip install websockets`.

## Message Format

Every message from the server is a JSON object with three fields:

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

| Field      | Type   | Description                                          |
|------------|--------|------------------------------------------------------|
| `type`     | string | Always `"price_update"`                              |
| `provider` | string | Provider identifier (e.g., "sohu", "catfee", "itrx") |
| `data`     | object | Full provider price data, same format as REST API    |
| `ts`       | number | Server timestamp in milliseconds                     |

The `data` object has the same structure as a single element in the `/api/v1/prices` REST response. This means you can use the same parsing logic for both real-time and polled data.

### Provider Identifiers

MERX currently monitors eight providers:

| Provider   | Identifier   |
|------------|-------------|
| Sohu       | `sohu`      |
| CatFee     | `catfee`    |
| Netts      | `netts`     |
| TronSave   | `tronsave`  |
| Feee       | `feee`      |
| ITRX       | `itrx`      |
| PowerSun   | `powersun`  |
| TEM        | `tem`       |

Each provider sends an update approximately every 30 seconds. With eight providers, you can expect roughly one update every 3-4 seconds on average.

## Subscribing to Specific Providers

By default, a new connection receives updates from all providers. To filter, send a subscribe message after connecting:

```json
{ "subscribe": ["sohu", "catfee", "itrx"] }
```

This tells the server to send updates only from the specified providers. All other updates are silently dropped for your connection.

To reset and receive all updates again, send an empty array:

```json
{ "subscribe": [] }
```

### JavaScript Subscribe Example

```javascript
const ws = new WebSocket('wss://merx.exchange/ws')

ws.addEventListener('open', () => {
  // Only receive updates from sohu and catfee
  ws.send(JSON.stringify({
    subscribe: ['sohu', 'catfee']
  }))
})

ws.addEventListener('message', (event) => {
  const msg = JSON.parse(event.data)
  // msg.provider will only be "sohu" or "catfee"
  console.log(`${msg.provider}: ${msg.data.energy_prices[0]?.price_sun} SUN`)
})
```

### Python Subscribe Example

```python
import asyncio
import json
import websockets

async def listen_filtered():
    async with websockets.connect("wss://merx.exchange/ws") as ws:
        # Subscribe to specific providers
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

You can change your subscription at any time by sending a new subscribe message. The server replaces the previous filter with the new one.

## Heartbeat and Connection Health

The MERX WebSocket server sends ping frames every 30 seconds. Clients must respond with pong frames to keep the connection alive. If a client fails to respond to a ping, the server terminates the connection on the next heartbeat cycle (approximately 30 seconds later).

Most WebSocket libraries handle ping/pong automatically at the protocol level. The `websockets` Python library and browser `WebSocket` API both respond to pings without any additional code.

For Node.js using the `ws` library, pong responses are also automatic. However, if you are using a library that does not handle pings automatically, you need to listen for the ping event:

```javascript
// Node.js with 'ws' library (handles pong automatically)
import WebSocket from 'ws'

const ws = new WebSocket('wss://merx.exchange/ws')

ws.on('ping', () => {
  // ws library sends pong automatically
  // This handler is just for logging
  console.log('Received ping, pong sent')
})
```

## Reconnection Strategy

WebSocket connections can drop for many reasons: network interruptions, server deployments, load balancer timeouts. A production client must handle reconnection gracefully.

The recommended strategy is exponential backoff with jitter:

1. On disconnect, wait 1 second before reconnecting.
2. If the reconnection fails, double the wait time: 2s, 4s, 8s, up to a maximum of 30 seconds.
3. Add random jitter (0-1 second) to prevent all clients from reconnecting simultaneously.
4. On successful reconnection, reset the backoff to 1 second.
5. Re-send your subscription message after reconnecting.

### JavaScript Reconnection

```javascript
function createConnection(providers = []) {
  let backoff = 1000
  const maxBackoff = 30000

  function connect() {
    const ws = new WebSocket('wss://merx.exchange/ws')

    ws.addEventListener('open', () => {
      console.log('Connected')
      backoff = 1000  // Reset backoff on success

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
      ws.close()  // Triggers the close handler for reconnection
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

// Connect and subscribe to two providers
createConnection(['sohu', 'catfee'])
```

### Python Reconnection

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
                backoff = 1.0  # Reset on success

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

## Building a Price Alert Bot

Here is a practical use case: a bot that monitors real-time prices and sends a notification when energy drops below a target price. This example uses a simple console alert, but you can replace it with a Telegram message, a Slack webhook, or an email.

### JavaScript Price Alert Bot

```javascript
const TARGET_PRICE_SUN = 25
const COOLDOWN_MS = 300000  // 5 minutes between alerts

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
    // Replace with your notification logic:
    // sendTelegramMessage(...)
    // postToSlack(...)
  }
}

createAlertBot()
```

### Python Price Alert Bot

```python
import asyncio
import json
import time
import websockets

TARGET_PRICE_SUN = 25
COOLDOWN_SEC = 300  # 5 minutes between alerts
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
        # Replace with your notification logic

asyncio.run(alert_bot())
```

## Maintaining Local State

For applications that need to display a live market view, maintain a local dictionary of the latest prices from each provider and update it on every message:

```javascript
const latestPrices = new Map()

function handlePriceUpdate(msg) {
  latestPrices.set(msg.provider, {
    data: msg.data,
    receivedAt: msg.ts,
  })

  // Find current cheapest across all providers
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

## Combining WebSocket with REST API

A common pattern is to use the REST API for the initial state load and the WebSocket for incremental updates:

```javascript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY })

// Load initial state from REST API
const initialPrices = await merx.prices.list()
const priceMap = new Map()
for (const p of initialPrices) {
  priceMap.set(p.provider, p)
}
console.log(`Loaded ${priceMap.size} providers from REST API`)

// Switch to WebSocket for real-time updates
const ws = new WebSocket('wss://merx.exchange/ws')
ws.addEventListener('message', (event) => {
  const msg = JSON.parse(event.data)
  priceMap.set(msg.provider, msg.data)
  // Your UI or logic now has a continuously updated price map
})
```

This ensures you have a complete market view from the first render, with zero-latency updates from that point forward.

## Производительность Considerations

The WebSocket endpoint is designed for high-throughput consumption. A few notes for production deployments:

- Each provider update is approximately 200-500 bytes of JSON. With eight providers updating every 30 seconds, total bandwidth is under 1 KB/s.
- Use the subscription filter to reduce traffic if you only need specific providers.
- The server-side heartbeat interval is 30 seconds. If your application has a firewall or proxy that closes idle connections sooner, send application-level keepalive messages.
- The WebSocket path is `/ws`. Make sure your reverse proxy or load balancer is configured to upgrade HTTP connections on this path.

## Ресурсы

- Платформа: [merx.exchange](https://merx.exchange)
- Документация: [merx.exchange/docs](https://merx.exchange/docs)
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
