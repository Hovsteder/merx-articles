# Compra energía TRON con 5 líneas de código: MERX JavaScript SDK

El SDK de JavaScript de MERX te permite comprar energía TRON de forma programática en cinco líneas de código: instala el paquete, inicializa el cliente, verifica los precios, crea una orden y verifica la delegación. Este artículo cubre el SDK completo desde la instalación hasta el despliegue en producción, incluyendo tipos TypeScript, manejo de errores y todos los métodos disponibles en los cuatro módulos: precios, órdenes, saldo y webhooks.

## El problema con las APIs de proveedores directos

Cada proveedor de energía TRON tiene su propia API. Integrarse con uno es sencillo. Integrarse con los siete u ocho proveedores para garantizar el mejor precio es un proyecto de ingeniería. Necesitas manejar esquemas de autenticación diferentes, formatos de respuesta, códigos de error, límites de velocidad y lógica de conmutación por error.

El SDK de MERX abstrae todo esto en un cliente único con una interfaz consistente. Detrás de las escenas, la plataforma de MERX consulta todos los proveedores cada 30 segundos, mantiene un índice de precios en tiempo real y enruta tus órdenes a la fuente más barata disponible con conmutación por error automática.

## Instalación

El SDK requiere Node.js 18 o posterior. Utiliza `fetch` nativo sin dependencias de tiempo de ejecución.

```bash
npm install merx-sdk
```

```bash
yarn add merx-sdk
```

```bash
pnpm add merx-sdk
```

El paquete se distribuye como ESM con declaraciones TypeScript completas y mapas de origen.

## Cinco líneas para comprar energía

Aquí está el flujo completo en su forma mínima:

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

Línea por línea:

1. Importa la clase del cliente.
2. Inicializa con tu clave API. La clave comienza con `sk_live_` y se crea en el panel de MERX en [merx.exchange](https://merx.exchange).
3. Obtén los precios actuales de todos los proveedores conectados. Esto es opcional pero útil para mostrar el estado del mercado antes de realizar la orden.
4. Crea una orden de mercado para 65.000 unidades de energía delegadas a tu dirección objetivo durante una hora. La plataforma selecciona automáticamente el proveedor más barato.
5. Registra el ID de la orden y el estado. La orden comienza en estado `PENDING` y transiciona a `FILLED` una vez que la delegación en cadena se confirma.

Esta es toda la integración. Sin selección de proveedor, sin lógica de conmutación por error, sin código de comparación de precios.

## Configuración del cliente

```typescript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({
  apiKey: 'sk_live_your_key_here',   // Requerido
  baseUrl: 'https://merx.exchange',  // Opcional, por defecto a producción
})
```

El `apiKey` es la única opción requerida. El `baseUrl` puede anularse para realizar pruebas en un entorno de prueba.

## El módulo de precios

El módulo `merx.prices` proporciona cinco métodos para consultar datos de mercado en tiempo real.

### prices.list()

Devuelve los precios actuales de todos los proveedores activos.

```typescript
const prices = await merx.prices.list()

for (const p of prices) {
  const cheapest = p.energy_prices[0]
  if (cheapest) {
    console.log(`${p.provider}: ${cheapest.price_sun} SUN/energy, ${p.available_energy} available`)
  }
}
```

Cada objeto `ProviderPrice` incluye:
- `provider` - identificador del proveedor (ej: "sohu", "catfee", "tronsave")
- `is_market` - si este proveedor admite órdenes de mercado con duraciones flexibles
- `energy_prices` - array de niveles `{ duration_sec, price_sun }`
- `bandwidth_prices` - la misma estructura para bandwidth
- `available_energy` - capacidad actual en unidades de energía
- `available_bandwidth` - capacidad actual en unidades de bandwidth
- `fetched_at` - marca de tiempo Unix de la última consulta exitosa

### prices.best(resource, amount?)

Devuelve el punto de precio único más barato para un tipo de recurso determinado.

```typescript
const best = await merx.prices.best('ENERGY')
console.log(`Cheapest: ${best.price_sun} SUN from ${best.provider}`)

// Con filtro de cantidad mínima
const bestLarge = await merx.prices.best('ENERGY', 500000)
```

El parámetro opcional `amount` filtra los proveedores que no tienen suficiente capacidad para cumplir tu orden.

### prices.history(params?)

Devuelve datos de precios históricos para análisis y gráficos.

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

Períodos disponibles: `'1h'`, `'6h'`, `'24h'`, `'7d'`, `'30d'`. Todos los parámetros de filtro son opcionales.

### prices.stats()

Devuelve estadísticas agregadas en todo el mercado.

```typescript
const stats = await merx.prices.stats()
console.log(`Best price: ${stats.best_price_sun} SUN`)
console.log(`Average: ${stats.avg_price_sun} SUN`)
console.log(`Providers online: ${stats.total_providers}`)
console.log(`Cheapest-provider changes (24h): ${stats.cheapest_changes_24h}`)
```

### prices.preview(params)

Obtiene una vista previa del costo de una orden antes de realizarla. Devuelve el proveedor con mejor coincidencia y opciones de respaldo.

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

El parámetro `max_price_sun` filtra los proveedores que superan tu límite de precio.

## El módulo de órdenes

### orders.create(params)

Crea una orden de energía o bandwidth.

```typescript
const order = await merx.orders.create({
  resource_type: 'ENERGY',
  amount: 65000,
  target_address: 'TYourTargetAddress',
  duration_sec: 3600,
  order_type: 'MARKET',          // Opcional, por defecto 'MARKET'
  max_price_sun: 30,             // Opcional, requerido para órdenes LIMIT
  idempotency_key: 'unique-id',  // Opcional, previene órdenes duplicadas
})

console.log(`Order ${order.id} created`)
console.log(`Status: ${order.status}`)
```

Tipos de orden:
- `MARKET` - ejecutar inmediatamente al mejor precio disponible
- `LIMIT` - ejecutar solo si el precio está en o por debajo de `max_price_sun`
- `PERIODIC` - orden recurrente
- `BROADCAST` - transmitir una transacción de delegación prefirmada

El `idempotency_key` es crítico para sistemas en producción. Si ocurre un error de red y reintentas la solicitud con la misma clave, la API devuelve la orden original en lugar de crear una duplicada.

### orders.list(limit?, offset?, status?)

Lista órdenes con paginación.

```typescript
const { orders, total } = await merx.orders.list(10, 0, 'FILLED')
console.log(`${total} filled orders total`)

for (const o of orders) {
  console.log(`${o.id}: ${o.amount} ${o.resource_type}, cost: ${o.total_cost_sun} SUN`)
}
```

### orders.get(id)

Devuelve una única orden con detalles de cumplimiento.

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

El array `fills` muestra exactamente cómo se cumplió la orden. Cada cumplimiento incluye el nombre del proveedor, la cantidad asignada, el precio por unidad, el costo, el ID de la transacción en cadena y si la delegación ha sido verificada en cadena.

## El módulo de saldo

### balance.get()

Devuelve los saldos actuales de la cuenta.

```typescript
const balance = await merx.balance.get()
console.log(`TRX: ${balance.trx}`)
console.log(`USDT: ${balance.usdt}`)
console.log(`Locked: ${balance.trx_locked}`)
```

El campo `trx_locked` muestra TRX que está actualmente reservado para órdenes pendientes.

### balance.depositInfo()

Devuelve la dirección de depósito y memo para financiar tu cuenta de MERX.

```typescript
const info = await merx.balance.depositInfo()
console.log(`Send TRX to: ${info.address}`)
console.log(`Include memo: ${info.memo}`)
console.log(`Minimum deposit: ${info.min_amount_trx} TRX`)
```

El memo es requerido para el acreditamiento automatizado de depósitos. Los depósitos sin el memo correcto requieren procesamiento manual.

### balance.withdraw(params)

Retira TRX o USDT a una dirección TRON externa.

```typescript
const withdrawal = await merx.balance.withdraw({
  address: 'TYourExternalAddress',
  amount: 100,
  currency: 'TRX',
  idempotency_key: 'withdraw-unique-id',
})

console.log(`Withdrawal ${withdrawal.id}: ${withdrawal.status}`)
```

### balance.history(period?) y balance.summary()

```typescript
const history = await merx.balance.history('7D')
console.log(`${history.length} transactions in last 7 days`)

const summary = await merx.balance.summary()
console.log(`Total orders: ${summary.total_orders}`)
console.log(`Total energy: ${summary.total_energy}`)
console.log(`Average price: ${summary.avg_price_sun} SUN`)
```

## El módulo de webhooks

### webhooks.create(params)

Crea una suscripción de webhook. El `secret` se devuelve solo en el momento de la creación.

```typescript
const webhook = await merx.webhooks.create({
  url: 'https://your-server.com/merx-webhook',
  events: ['order.filled', 'order.failed', 'deposit.received'],
})

console.log(`Webhook ID: ${webhook.id}`)
console.log(`Secret: ${webhook.secret}`)
```

### webhooks.list() y webhooks.delete(id)

```typescript
const webhooks = await merx.webhooks.list()
console.log(`${webhooks.filter(w => w.is_active).length} active webhooks`)

await merx.webhooks.delete('wh_abc123')
```

## Manejo de errores

Todos los errores de API se lanzan como instancias de `MerxError` con un `code` legible por máquina y un `message` legible por humanos.

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
    // Output: [INVALID_ADDRESS]: Target address is not a valid TRON address
  }
}
```

Códigos de error comunes:

| Código                 | Significado                                      |
|------------------------|--------------------------------------------------|
| `UNAUTHORIZED`         | Clave API inválida o faltante                   |
| `INSUFFICIENT_FUNDS`   | Saldo de cuenta muy bajo                        |
| `INVALID_ADDRESS`      | La dirección objetivo no es una dirección TRON válida |
| `ORDER_NOT_FOUND`      | El ID de orden no existe                        |
| `INVALID_AMOUNT`       | Cantidad por debajo del mínimo o excede límites |
| `DUPLICATE_REQUEST`    | Clave de idempotencia ya utilizada             |
| `RATE_LIMITED`         | Demasiadas solicitudes                          |
| `PROVIDER_UNAVAILABLE` | Ningún proveedor disponible                     |

## Tipos TypeScript

Todos los tipos se exportan desde el paquete y se pueden usar para anotaciones de tipo en toda tu aplicación:

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

El SDK es compatible con modo estricto y no produce tipos `any`. Cada respuesta está completamente tipada.

## Ejemplo en producción: flujo de transferencia automatizada de USDT

Aquí hay un patrón completo de producción que verifica precios, crea una orden y consulta su finalización:

```typescript
import { MerxClient, MerxError } from 'merx-sdk'
import { randomUUID } from 'node:crypto'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY! })

async function ensureEnergyAndTransfer(targetAddress: string) {
  // Preview the cost
  const preview = await merx.prices.preview({
    resource: 'ENERGY',
    amount: 65000,
    duration: 3600,
  })

  if (preview.no_providers) {
    throw new Error('No energy providers available')
  }

  console.log(`Best price: ${preview.best!.cost_trx} TRX from ${preview.best!.provider}`)

  // Create the order with idempotency
  const order = await merx.orders.create({
    resource_type: 'ENERGY',
    amount: 65000,
    target_address: targetAddress,
    duration_sec: 3600,
    idempotency_key: randomUUID(),
  })

  // Poll until filled (in production, use webhooks instead)
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

  // Energy is now available on the target address
  // Proceed with your USDT transfer
}
```

En producción, reemplaza el bucle de consulta con un escuchador de webhook. Crea una suscripción de webhook para el evento `order.filled` y tu servidor será notificado en el momento en que la delegación se confirme en cadena.

## Requisitos y compatibilidad

- Node.js 18+ (utiliza `fetch` nativo)
- Cero dependencias de tiempo de ejecución
- Solo ESM (se distribuye como módulos ES)
- Soporte completo de TypeScript 5.4+
- Funciona con Bun y Deno (cualquier tiempo de ejecución que admita `fetch`)

## Recursos

El SDK es de código abierto y está disponible en GitHub. Las contribuciones y reportes de errores son bienvenidos.

- Plataforma: [merx.exchange](https://merx.exchange)
- Documentación: [merx.exchange/docs](https://merx.exchange/docs)
- GitHub: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- npm: [npmjs.com/package/merx-sdk](https://www.npmjs.com/package/merx-sdk)
- SDK de Python: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- Servidor MCP para agentes de IA: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)


## Pruébalo ahora con IA

Añade MERX a Claude Desktop o a cualquier cliente compatible con MCP -- sin instalación, sin necesidad de clave API para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregunta a tu agente de IA: "¿Cuál es la energía TRON más barata en este momento?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)