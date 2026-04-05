# Integración de Webhooks: Recibe Notificaciones Cuando tus Órdenes se Completen

Los webhooks de MERX entregan notificaciones HTTP en tiempo real cuando ocurren eventos en tu cuenta - órdenes completadas, órdenes fallidas, depósitos recibidos, retiros completados. Este artículo cubre los cuatro tipos de eventos, el formato de carga útil, verificación de firmas HMAC-SHA256 usando el encabezado `X-Merx-Signature`, la política de reintentos con retroceso exponencial, auto-desactivación después de fallos repetidos, e implementaciones completas de servidores en Express.js y Flask con verificación de firmas.

## Por Qué Webhooks en Lugar de Polling

La alternativa a los webhooks es hacer polling del endpoint `/api/v1/orders/:id` en un bucle, esperando a que el estado cambie de `PENDING` a `FILLED`. Esto funciona para casos simples pero tiene desventajas claras:

- Solicitudes desperdiciadas. La mayoría de los polls devuelven el mismo estado sin cambios.
- Latencia. Tu aplicación solo descubre un cambio de estado en el siguiente intervalo de polling.
- Límites de velocidad. Con un límite de 10 solicitudes por minuto en el endpoint de órdenes, el polling agresivo rápidamente alcanza el límite.
- Complejidad. La lógica de polling necesita manejo de reintentos, gestión de tiempos de espera y seguimiento de estado.

Los webhooks invierten el modelo. En lugar de preguntarle a MERX "¿ha cambiado algo?", MERX te lo comunica en el momento en que sucede algo. Tu servidor recibe un POST HTTP con la carga útil completa del evento, la procesa y continúa. Sin bucles de polling, sin solicitudes desperdiciadas, sin retrasos artificiales.

## Creando un Webhook

Puedes crear webhooks a través de la REST API o el SDK.

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

Respuesta:

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

El campo `secret` es una cadena hexadecimal de 64 caracteres generada a partir de 32 bytes aleatorios. Se devuelve solo en la respuesta de creación. Guárdalo de forma segura - lo necesitarás para verificar las firmas de los webhooks entrantes.

### JavaScript SDK

```javascript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY })

const webhook = await merx.webhooks.create({
  url: 'https://your-server.com/webhooks/merx',
  events: ['order.filled', 'order.failed', 'deposit.received'],
})

console.log(`Webhook ID: ${webhook.id}`)
console.log(`Secret: ${webhook.secret}`)  // Store this securely
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
print(f"Secret: {webhook.secret}")  # Store this securely
```

## Tipos de Eventos

MERX soporta cuatro tipos de eventos de webhooks. Cuando creas un webhook, eliges a cuáles eventos suscribirse. Puedes suscribirte a los cuatro o solo a los que necesites.

### order.filled

Se envía cuando una orden ha sido completamente cumplida. Todas las delegaciones de proveedores han sido confirmadas en la cadena.

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

Este es el evento más importante para sistemas automatizados. Cuando lo recibas, la energía ha sido delegada y la dirección de destino puede proceder con sus transacciones TRON.

### order.failed

Se envía cuando una orden no pudo ser cumplida. Esto puede ocurrir cuando todos los proveedores no están disponibles, la capacidad se ha agotado, o ocurre un error del lado del proveedor.

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

Cuando una orden falla, cualquier saldo reservado es reembolsado. El campo `refunded` confirma esto, y `refund_amount_sun` muestra la cantidad devuelta si el pago ya había sido deducido.

### deposit.received

Se envía cuando se ha detectado un depósito en tu cuenta de MERX y ha sido acreditado.

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

Se envía cuando una solicitud de retiro ha sido procesada y la transacción en la cadena ha sido confirmada.

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

## Verificación de Firmas

Cada entrega de webhook incluye un encabezado `X-Merx-Signature` que contiene una firma HMAC-SHA256 del cuerpo de la solicitud, calculada usando tu secreto de webhook como clave.

El proceso de verificación:

1. Lee el cuerpo de la solicitud sin procesar (antes del análisis JSON).
2. Calcula HMAC-SHA256 del cuerpo sin procesar usando tu secreto de webhook almacenado.
3. Compara la firma calculada con el valor del encabezado `X-Merx-Signature`.
4. Si coinciden, la solicitud es auténtica. Si no, recházala.

Esto te protege contra:
- Solicitudes falsificadas de terceros que no conocen tu secreto.
- Cargas útiles manipuladas donde el cuerpo ha sido modificado en tránsito.
- Ataques de repetición (combinado con validación de marca de tiempo).

Siempre usa una función de comparación de tiempo constante al verificar firmas. La igualdad de cadenas estándar (`===` o `==`) es vulnerable a ataques de temporización.

## Controlador de Webhooks en Express.js

Aquí hay un servidor Express.js completo que recibe y verifica webhooks de MERX:

```javascript
import express from 'express'
import crypto from 'node:crypto'

const app = express()
const WEBHOOK_SECRET = process.env.MERX_WEBHOOK_SECRET

// Use raw body for signature verification
app.post('/webhooks/merx', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-merx-signature']
  if (!signature || !WEBHOOK_SECRET) {
    res.status(401).json({ error: 'Missing signature or secret' })
    return
  }

  // Compute expected signature
  const expected = crypto
    .createHmac('sha256', WEBHOOK_SECRET)
    .update(req.body)
    .digest('hex')

  // Constant-time comparison
  if (!crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected))) {
    console.warn('Webhook signature mismatch')
    res.status(401).json({ error: 'Invalid signature' })
    return
  }

  // Signature verified - parse and handle the event
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

  // Always respond with 200 quickly to acknowledge receipt
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

  // Your business logic here:
  // - Mark the transaction as ready to proceed
  // - Trigger the USDT transfer
  // - Update your database
}

function handleOrderFailed(data) {
  console.log(`Order ${data.order_id} failed: ${data.reason}`)
  if (data.refunded) {
    console.log(`  Refunded: ${data.refund_amount_sun} SUN`)
  }
  // Alert your operations team
}

function handleDepositReceived(data) {
  console.log(`Deposit received: ${data.amount_trx} ${data.currency}`)
  console.log(`  New balance: ${data.new_balance_trx} TRX`)
  // Update local balance cache
}

function handleWithdrawalCompleted(data) {
  console.log(`Withdrawal ${data.withdrawal_id} completed`)
  console.log(`  ${data.amount} ${data.currency} to ${data.address}`)
  // Update withdrawal status in your system
}

app.listen(3000, () => {
  console.log('Webhook server listening on port 3000')
})
```

Detalles clave de la implementación:

- Usa `express.raw({ type: 'application/json' })` en lugar de `express.json()` para la ruta del webhook. Necesitas los bytes sin procesar para el cálculo de la firma.
- Usa `crypto.timingSafeEqual()` para comparación de firmas de tiempo constante.
- Responde con HTTP 200 lo antes posible. Realiza el procesamiento pesado de forma asincrónica después de confirmar la recepción.

## Controlador de Webhooks en Flask

Aquí está la implementación equivalente en Python usando Flask:

```python
import hashlib
import hmac
import json
import os

from flask import Flask, request, jsonify

app = Flask(__name__)
WEBHOOK_SECRET = os.environ.get("MERX_WEBHOOK_SECRET", "")


def verify_signature(payload: bytes, signature: str, secret: str) -> bool:
    """Verify HMAC-SHA256 signature using constant-time comparison."""
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

    # Your business logic here


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

Detalles específicos de Python:

- Usa `request.get_data()` para obtener el cuerpo de la solicitud sin procesar como bytes.
- Usa `hmac.compare_digest()` para comparación de cadenas de tiempo constante. El operador `==` de Python no es de tiempo constante.
- Usa `hmac.new()` con `hashlib.sha256` para calcular el HMAC.

## Política de Reintentos

MERX reintenta entregas fallidas de webhooks usando un esquema de retroceso exponencial:

| Intento | Retraso después del fallo |
|---------|--------------------------|
| 1       | Inmediato                 |
| 2       | 30 segundos               |
| 3       | 5 minutos                 |

Una entrega se considera fallida si:
- Tu servidor no responde dentro de 10 segundos.
- Tu servidor devuelve un código de estado HTTP fuera del rango 2xx.
- La conexión no se puede establecer (fallo de DNS, conexión rechazada, error de TLS).

Después de 3 intentos fallidos para un único evento, el evento se descarta. No se hacen más reintentos para esa entrega específica.

Tu controlador de webhooks debe:
- Responder con HTTP 200 dentro de algunos segundos. Realiza el procesamiento pesado de forma asincrónica.
- Ser idempotente. El mismo evento puede ser entregado más de una vez en casos extremos.
- Registrar la carga útil del evento para depuración si el procesamiento falla.

## Auto-Desactivación

Si un endpoint de webhook falla consistentemente, MERX lo desactiva automáticamente para evitar desperdiciar recursos en un endpoint muerto.

El umbral de desactivación se basa en fallos consecutivos en múltiples eventos. Si tu endpoint falla repetidamente en aceptar entregas, la bandera `is_active` del webhook se establece en `false`.

Cuando un webhook es desactivado:
- No se envían más eventos a esa URL.
- El webhook sigue apareciendo en tu lista con `is_active: false`.
- Puedes corregir el problema del endpoint y crear un nuevo webhook.

Monitorea el estado de tus webhooks periódicamente:

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

## Gestionando Webhooks

### Listando Webhooks

```bash
curl -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/webhooks
```

### Eliminando un Webhook

```bash
curl -X DELETE -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/webhooks/wh_abc123
```

Ten en cuenta que el secreto del webhook no se puede recuperar después de la creación. Si pierdes el secreto, elimina el webhook y crea uno nuevo.

## Probando Webhooks Localmente

Durante el desarrollo, tu URL de webhook debe ser accesible públicamente. Herramientas como ngrok pueden exponer un servidor local:

```bash
# Terminal 1: Start your webhook server
node webhook-server.js

# Terminal 2: Expose it publicly
ngrok http 3000
```

Usa la URL de ngrok (por ejemplo, `https://abc123.ngrok.io/webhooks/merx`) cuando crees el webhook. Una vez que verifiques que todo funciona, reemplázala con tu URL de producción.

## Mejores Prácticas

1. Siempre verifica firmas. Nunca confíes en cargas útiles de webhooks sin verificar el encabezado `X-Merx-Signature`.

2. Responde rápidamente. Devuelve HTTP 200 dentro de 2-3 segundos. Coloca el procesamiento pesado en la cola para trabajadores en segundo plano.

3. Sé idempotente. Usa `order_id` o `deposit_id` como clave de deduplicación. Si recibes el mismo evento dos veces, el segundo procesamiento debe ser una no-operación.

4. Almacena la carga útil sin procesar. Registra el cuerpo JSON completo antes de procesar. Si tu controlador tiene un error, puedes reproducir los eventos desde los registros.

5. Monitorea la salud del webhook. Verifica regularmente el estado `is_active`. Configura alertas si los webhooks son desactivados.

6. Usa HTTPS. MERX requiere que las URLs de webhooks usen HTTPS. Los certificados autofirmados no se aceptan.

7. Suscríbete selectivamente. Solo suscríbete a los eventos que realmente manejas. Los eventos innecesarios desperdician ancho de banda y tiempo de procesamiento.

## Recursos

- Plataforma: [merx.exchange](https://merx.exchange)
- Documentación: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- Python SDK: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- MCP Server: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)


## Pruébalo Ahora con IA

Añade MERX a Claude Desktop o cualquier cliente compatible con MCP -- sin instalación, sin clave API necesaria para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregúntale a tu agente de IA: "¿Cuál es la energía TRON más barata ahora mismo?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)