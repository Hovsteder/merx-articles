# Цены энергии TRON в реальном времени через WebSocket

MERX предоставляет endpoint WebSocket по адресу `wss://merx.exchange/ws`, который транслирует цены энергии и bandwidth TRON в реальном времени от всех подключённых провайдеров. В этой статье рассматриваются протокол подключения, формат сообщений, фильтрация провайдеров через сообщения подписки, стратегии heartbeat и переподключения, а также полные примеры кода на JavaScript и Python — включая практический бот для оповещения о цене, который уведомляет вас при падении цены энергии ниже целевого значения.

## Почему WebSocket вместо опроса

REST API MERX по адресу `/api/v1/prices` возвращает текущие цены от всех провайдеров. Вы можете опрашивать его через регулярные интервалы, и при ограничении в 300 запросов в минуту вы можете обновлять данные каждые 200 миллисекунд, если хотите.

Но опрос имеет встроенные недостатки. Каждый запрос — это полный HTTP цикл с накладными расходами на TLS handshake. Вы никогда не знаете точный момент изменения цены — вы узнаёте об этом только при следующем интервале опроса. И если у вас много клиентов, каждый из них независимо обрушивается на один и тот же endpoint.

Подключение WebSocket решает все эти проблемы. Одно постоянное соединение получает обновления цен в момент их возникновения. Сервис мониторинга цен MERX опрашивает всех восьми провайдеров каждые 30 секунд и публикует обновления через Redis pub/sub. WebSocket сервер подписывается на этот канал и мгновенно распространяет обновления всем подключённым клиентам.

В результате: более низкая задержка, более низкое потребление полосы пропускания и гарантированная доставка каждого изменения цены.

## Подключение

Подключитесь к endpoint WebSocket по адресу `wss://merx.exchange/ws`. Аутентификация не требуется — данные о ценах являются публичными.

### JavaScript (браузер или Node.js)

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

### Python (с использованием библиотеки websockets)

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

Установите библиотеку websockets с помощью `pip install websockets`.

## Формат сообщений

Каждое сообщение с сервера — это объект JSON с тремя полями:

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

| Поле       | Тип    | Описание                                       |
|------------|--------|------------------------------------------------|
| `type`     | string | Всегда `"price_update"`                        |
| `provider` | string | Идентификатор провайдера (например, "sohu", "catfee", "itrx") |
| `data`     | object | Полные данные о ценах провайдера, тот же формат, что и REST API |
| `ts`       | number | Временная метка сервера в миллисекундах       |

Объект `data` имеет ту же структуру, что и один элемент в ответе REST API `/api/v1/prices`. Это значит, что вы можете использовать одну и ту же логику парсинга для данных в реальном времени и для опрашиваемых данных.

### Идентификаторы провайдеров

MERX в настоящее время мониторит восьми провайдеров:

| Провайдер  | Идентификатор |
|------------|----------------|
| Sohu       | `sohu`         |
| CatFee     | `catfee`       |
| Netts      | `netts`        |
| TronSave   | `tronsave`     |
| Feee       | `feee`         |
| ITRX       | `itrx`         |
| PowerSun   | `powersun`     |
| TEM        | `tem`          |

Каждый провайдер отправляет обновление примерно каждые 30 секунд. При восьми провайдерах вы можете рассчитывать на примерно одно обновление каждые 3-4 секунды в среднем.

## Подписка на конкретных провайдеров

По умолчанию новое соединение получает обновления от всех провайдеров. Для фильтрации отправьте сообщение подписки после подключения:

```json
{ "subscribe": ["sohu", "catfee", "itrx"] }
```

Это указывает серверу отправлять обновления только от указанных провайдеров. Все остальные обновления для вашего соединения молча отбрасываются.

Чтобы сбросить настройки и получать все обновления снова, отправьте пустой массив:

```json
{ "subscribe": [] }
```

### Пример подписки на JavaScript

```javascript
const ws = new WebSocket('wss://merx.exchange/ws')

ws.addEventListener('open', () => {
  // Получать обновления только от sohu и catfee
  ws.send(JSON.stringify({
    subscribe: ['sohu', 'catfee']
  }))
})

ws.addEventListener('message', (event) => {
  const msg = JSON.parse(event.data)
  // msg.provider будет только "sohu" или "catfee"
  console.log(`${msg.provider}: ${msg.data.energy_prices[0]?.price_sun} SUN`)
})
```

### Пример подписки на Python

```python
import asyncio
import json
import websockets

async def listen_filtered():
    async with websockets.connect("wss://merx.exchange/ws") as ws:
        # Подписаться на конкретных провайдеров
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

Вы можете изменить подписку в любой момент, отправив новое сообщение подписки. Сервер заменит предыдущий фильтр новым.

## Heartbeat и здоровье соединения

WebSocket сервер MERX отправляет ping-фреймы каждые 30 секунд. Клиенты должны отвечать pong-фреймами, чтобы поддерживать соединение в живом состоянии. Если клиент не ответит на ping, сервер прервёт соединение на следующем цикле heartbeat (примерно через 30 секунд).

Большинство библиотек WebSocket обрабатывают ping/pong автоматически на уровне протокола. Библиотека `websockets` для Python и браузерный API `WebSocket` оба отвечают на ping-сигналы без дополнительного кода.

Для Node.js с использованием библиотеки `ws` ответы pong также автоматические. Однако, если вы используете библиотеку, которая не обрабатывает ping-сигналы автоматически, вам нужно слушать событие ping:

```javascript
// Node.js с библиотекой 'ws' (автоматически обрабатывает pong)
import WebSocket from 'ws'

const ws = new WebSocket('wss://merx.exchange/ws')

ws.on('ping', () => {
  // ws библиотека отправляет pong автоматически
  // Этот обработчик только для логирования
  console.log('Received ping, pong sent')
})
```

## Стратегия переподключения

Соединения WebSocket могут разорваться по многим причинам: сбои сети, развёртывание сервера, истечение таймаута load balancer'а. Продакшен клиент должен обрабатывать переподключение корректно.

Рекомендуемая стратегия — экспоненциальная задержка с jitter:

1. При разрыве соединения подождите 1 секунду перед переподключением.
2. Если переподключение не удаётся, удвойте время ожидания: 2s, 4s, 8s, вплоть до максимума 30 секунд.
3. Добавьте случайный jitter (0-1 секунда), чтобы предотвратить одновременное переподключение всех клиентов.
4. При успешном переподключении сбросьте задержку на 1 секунду.
5. После переподключения повторно отправьте сообщение подписки.

### Переподключение на JavaScript

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

### Переподключение на Python

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

## Создание бота для оповещения о цене

Вот практический вариант использования: бот, который мониторит цены в реальном времени и отправляет уведомление, когда цена энергии падает ниже целевой цены. Этот пример использует простое консольное оповещение, но вы можете заменить его на сообщение Telegram, webhook Slack или электронное письмо.

### Бот для оповещения о цене на JavaScript

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

### Бот для оповещения о цене на Python

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

## Ведение локального состояния

Для приложений, которым требуется отображать живое представление рынка, ведите локальный словарь последних цен от каждого провайдера и обновляйте его при каждом сообщении:

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

## Комбинирование WebSocket с REST API

Типичный паттерн — использовать REST API для начальной загрузки состояния и WebSocket для добавочных обновлений:

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

Это гарантирует, что у вас будет полное представление рынка с первого рендера, с обновлениями нулевой задержки с этого момента.

## Рассмотрения производительности

Endpoint WebSocket разработан для высокопроизводительного потребления. Несколько заметок для развёртываний в production:

- Каждое обновление провайдера занимает примерно 200-500 байт JSON. При восьми провайдерах, обновляющихся каждые 30 секунд, общая полоса пропускания составляет менее 1 КБ/с.
- Используйте фильтр подписки, чтобы уменьшить трафик, если вам нужны только конкретные провайдеры.
- Интервал heartbeat на стороне сервера составляет 30 секунд. Если ваше приложение имеет firewall или proxy, закрывающий неактивные соединения раньше, отправляйте приложение-уровень keepalive сообщения.
- Путь WebSocket — `/ws`. Убедитесь, что ваш reverse proxy или load balancer настроен на обновление HTTP соединений на этом пути.

## Ресурсы

- Платформа: [merx.exchange](https://merx.exchange)
- Документация: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- Python SDK: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- MCP Server: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)


## Попробуйте прямо сейчас с AI

Добавьте MERX в Claude Desktop или любого клиента, совместимого с MCP — без установки, без API ключа для инструментов только для чтения:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Спросите вашего AI агента: "What is the cheapest TRON energy right now?" и получите живые цены от всех подключённых провайдеров.

Полная документация MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)