# Precios de Energía TRON en Tiempo Real vía WebSocket

MERX proporciona un endpoint WebSocket en `wss://merx.exchange/ws` que transmite en tiempo real los precios de energía y bandwidth de TRON de todos los proveedores conectados. Este artículo cubre el protocolo de conexión, el formato de mensajes, el filtrado de proveedores mediante mensajes de suscripción, las estrategias de latido del corazón y reconexión, y ejemplos de código completos en JavaScript y Python, incluyendo un bot práctico de alerta de precios que te notifica cuando la energía cae por debajo de un precio objetivo.

## Por Qué WebSocket en Lugar de Polling

La API REST de MERX en `/api/v1/prices` devuelve los precios actuales de todos los proveedores. Puedes consultarla a intervalos regulares, y con un límite de velocidad de 300 solicitudes por minuto, puedes actualizar cada 200 milisegundos si lo deseas.

Pero el polling tiene inconvenientes inherentes. Cada solicitud es un recorrido completo de HTTP con sobrecarga de handshake TLS. Nunca sabes el momento exacto en que cambia un precio, solo lo descubres en tu próximo intervalo de consulta. Y si tienes muchos clientes, cada uno bombardea el mismo endpoint de forma independiente.

La conexión WebSocket resuelve todo esto. Una única conexión persistente recibe actualizaciones de precios en el momento en que suceden. El servicio de monitoreo de precios de MERX consulta a los ocho proveedores cada 30 segundos y publica las actualizaciones vía Redis pub/sub. El servidor WebSocket se suscribe a ese canal y distribuye las actualizaciones a todos los clientes conectados instantáneamente.

El resultado: menor latencia, menor uso de bandwidth, y entrega garantizada de cada cambio de precio.

## Conexión

Conéctate al endpoint WebSocket en `wss://merx.exchange/ws`. No se requiere autenticación, los datos de precios son públicos.

### JavaScript (Navegador o Node.js)

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

### Python (usando la librería websockets)

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

Instala la librería websockets con `pip install websockets`.

## Formato de Mensajes

Cada mensaje del servidor es un objeto JSON con tres campos:

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

| Campo      | Tipo   | Descripción                                          |
|------------|--------|------------------------------------------------------|
| `type`     | string | Siempre `"price_update"`                             |
| `provider` | string | Identificador del proveedor (ej: "sohu", "catfee", "itrx") |
| `data`     | object | Datos de precios completos del proveedor, mismo formato que la API REST |
| `ts`       | number | Marca de tiempo del servidor en milisegundos         |

El objeto `data` tiene la misma estructura que un elemento único en la respuesta de la API REST `/api/v1/prices`. Esto significa que puedes usar la misma lógica de análisis para datos en tiempo real y consultados.

### Identificadores de Proveedores

MERX monitorea actualmente ocho proveedores:

| Proveedor  | Identificador |
|------------|-------------|
| Sohu       | `sohu`      |
| CatFee     | `catfee`    |
| Netts      | `netts`     |
| TronSave   | `tronsave`  |
| Feee       | `feee`      |
| ITRX       | `itrx`      |
| PowerSun   | `powersun`  |
| TEM        | `tem`       |

Cada proveedor envía una actualización aproximadamente cada 30 segundos. Con ocho proveedores, puedes esperar aproximadamente una actualización cada 3-4 segundos en promedio.

## Suscribirse a Proveedores Específicos

Por defecto, una nueva conexión recibe actualizaciones de todos los proveedores. Para filtrar, envía un mensaje de suscripción después de conectarte:

```json
{ "subscribe": ["sohu", "catfee", "itrx"] }
```

Esto le indica al servidor que envíe actualizaciones solo de los proveedores especificados. Todas las demás actualizaciones se descartan silenciosamente para tu conexión.

Para restablecer y recibir todas las actualizaciones nuevamente, envía un array vacío:

```json
{ "subscribe": [] }
```

### Ejemplo de Suscripción en JavaScript

```javascript
const ws = new WebSocket('wss://merx.exchange/ws')

ws.addEventListener('open', () => {
  // Solo recibir actualizaciones de sohu y catfee
  ws.send(JSON.stringify({
    subscribe: ['sohu', 'catfee']
  }))
})

ws.addEventListener('message', (event) => {
  const msg = JSON.parse(event.data)
  // msg.provider solo será "sohu" o "catfee"
  console.log(`${msg.provider}: ${msg.data.energy_prices[0]?.price_sun} SUN`)
})
```

### Ejemplo de Suscripción en Python

```python
import asyncio
import json
import websockets

async def listen_filtered():
    async with websockets.connect("wss://merx.exchange/ws") as ws:
        # Suscribirse a proveedores específicos
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

Puedes cambiar tu suscripción en cualquier momento enviando un nuevo mensaje de suscripción. El servidor reemplaza el filtro anterior con el nuevo.

## Latido del Corazón y Salud de la Conexión

El servidor WebSocket de MERX envía frames de ping cada 30 segundos. Los clientes deben responder con frames de pong para mantener la conexión activa. Si un cliente no responde a un ping, el servidor termina la conexión en el siguiente ciclo de latido (aproximadamente 30 segundos después).

La mayoría de las librerías WebSocket manejan ping/pong automáticamente a nivel de protocolo. La librería `websockets` de Python y la API `WebSocket` del navegador ambas responden a pings sin código adicional.

Para Node.js usando la librería `ws`, las respuestas de pong también son automáticas. Sin embargo, si usas una librería que no maneja pings automáticamente, necesitas escuchar el evento de ping:

```javascript
// Node.js con la librería 'ws' (maneja pong automáticamente)
import WebSocket from 'ws'

const ws = new WebSocket('wss://merx.exchange/ws')

ws.on('ping', () => {
  // la librería ws envía pong automáticamente
  // Este manejador es solo para registro
  console.log('Received ping, pong sent')
})
```

## Estrategia de Reconexión

Las conexiones WebSocket pueden caerse por muchas razones: interrupciones de red, despliegues de servidor, timeouts de balanceador de carga. Un cliente de producción debe manejar la reconexión correctamente.

La estrategia recomendada es backoff exponencial con jitter:

1. Al desconectar, espera 1 segundo antes de reconectar.
2. Si la reconexión falla, duplica el tiempo de espera: 2s, 4s, 8s, hasta un máximo de 30 segundos.
3. Añade jitter aleatorio (0-1 segundo) para evitar que todos los clientes se reconecten simultáneamente.
4. Al reconectar exitosamente, restablece el backoff a 1 segundo.
5. Reenvía tu mensaje de suscripción después de reconectar.

### Reconexión en JavaScript

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

// Conectar y suscribirse a dos proveedores
createConnection(['sohu', 'catfee'])
```

### Reconexión en Python

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

## Construir un Bot de Alerta de Precios

Aquí hay un caso de uso práctico: un bot que monitorea precios en tiempo real y envía una notificación cuando la energía cae por debajo de un precio objetivo. Este ejemplo usa una alerta simple en consola, pero puedes reemplazarla con un mensaje de Telegram, un webhook de Slack, o un correo electrónico.

### Bot de Alerta de Precios en JavaScript

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
    // Reemplaza con tu lógica de notificación:
    // sendTelegramMessage(...)
    // postToSlack(...)
  }
}

createAlertBot()
```

### Bot de Alerta de Precios en Python

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
        # Reemplaza con tu lógica de notificación

asyncio.run(alert_bot())
```

## Mantener Estado Local

Para aplicaciones que necesitan mostrar una vista de mercado en vivo, mantén un diccionario local de los últimos precios de cada proveedor y actualízalo en cada mensaje:

```javascript
const latestPrices = new Map()

function handlePriceUpdate(msg) {
  latestPrices.set(msg.provider, {
    data: msg.data,
    receivedAt: msg.ts,
  })

  // Encuentra el más barato actual entre todos los proveedores
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

## Combinando WebSocket con API REST

Un patrón común es usar la API REST para la carga de estado inicial y WebSocket para actualizaciones incrementales:

```javascript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY })

// Cargar estado inicial desde la API REST
const initialPrices = await merx.prices.list()
const priceMap = new Map()
for (const p of initialPrices) {
  priceMap.set(p.provider, p)
}
console.log(`Loaded ${priceMap.size} providers from REST API`)

// Cambiar a WebSocket para actualizaciones en tiempo real
const ws = new WebSocket('wss://merx.exchange/ws')
ws.addEventListener('message', (event) => {
  const msg = JSON.parse(event.data)
  priceMap.set(msg.provider, msg.data)
  // Tu UI o lógica ahora tiene un mapa de precios continuamente actualizado
})
```

Esto asegura que tengas una vista de mercado completa desde el primer renderizado, con actualizaciones de latencia cero desde ese punto en adelante.

## Consideraciones de Rendimiento

El endpoint WebSocket está diseñado para consumo de alto rendimiento. Algunas notas para despliegues de producción:

- Cada actualización de proveedor es aproximadamente 200-500 bytes de JSON. Con ocho proveedores actualizando cada 30 segundos, el bandwidth total es menor a 1 KB/s.
- Usa el filtro de suscripción para reducir tráfico si solo necesitas proveedores específicos.
- El intervalo de latido del lado del servidor es 30 segundos. Si tu aplicación tiene un firewall o proxy que cierra conexiones inactivas antes, envía mensajes de keepalive a nivel de aplicación.
- La ruta WebSocket es `/ws`. Asegúrate de que tu proxy inverso o balanceador de carga esté configurado para actualizar conexiones HTTP en esta ruta.

## Recursos

- Plataforma: [merx.exchange](https://merx.exchange)
- Documentación: [merx.exchange/docs](https://merx.exchange/docs)
- SDK de JavaScript: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- SDK de Python: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- Servidor MCP: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)


## Pruébalo Ahora con IA

Añade MERX a Claude Desktop o cualquier cliente compatible con MCP, sin instalación, sin necesidad de API key para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregunta a tu agente de IA: "¿Cuál es la energía TRON más barata ahora mismo?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)