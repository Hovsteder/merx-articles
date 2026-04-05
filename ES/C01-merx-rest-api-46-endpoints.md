# API REST de MERX: 46 Endpoints para Comercio de Energía TRON

La API REST de MERX proporciona 46 endpoints que cubren el ciclo de vida completo del comercio de energía y bandwidth de TRON - desde el descubrimiento de precios en tiempo real a través de ocho proveedores hasta la ejecución de órdenes, gestión de cuentas, consultas en cadena y órdenes permanentes automatizadas. Este artículo recorre la arquitectura de la API, el modelo de autenticación, grupos de endpoints, límites de velocidad, manejo de errores y ejemplos prácticos de código en curl, JavaScript y Python.

## Por qué una API Unificada es Importante

El mercado de energía TRON está fragmentado entre múltiples proveedores, cada uno con su propio formato de API, esquema de autenticación y modelo de precios. Un desarrollador que quiere el mejor precio en una transferencia USDT tiene que integrarse con cada proveedor individualmente, manejar la lógica de conmutación por error y monitorear los precios continuamente.

MERX consolida todo eso en una única API REST. Una clave API, un encabezado de autenticación, un formato de error, un conjunto de SDKs. La plataforma consulta todos los proveedores conectados cada 30 segundos, enruta órdenes a la fuente más barata disponible y verifica delegaciones en cadena.

La API está versionada en `/api/v1/` y mantendrá compatibilidad hacia atrás. Todos los endpoints devuelven JSON en un formato de envolvente consistente.

## Autenticación

MERX utiliza autenticación mediante clave API. Cada solicitud autenticada debe incluir el encabezado `X-API-Key`.

Las claves API se crean en el panel de MERX en [merx.exchange](https://merx.exchange) o programáticamente a través del endpoint `/api/v1/keys`. Cada clave tiene un conjunto de permisos que controla qué operaciones puede realizar.

```bash
curl -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/balance
```

Las claves siguen el formato `sk_live_` seguido de 64 caracteres hexadecimales. La clave sin procesar se muestra exactamente una vez en el momento de la creación. MERX almacena solo el hash bcrypt, por lo que las claves perdidas no pueden recuperarse - deben revocarse y reemplazarse.

### Permisos de Clave

Al crear una clave API, asigna uno o más permisos:

| Permiso           | Otorga acceso a                         |
|-------------------|-----------------------------------------|
| `create_orders`   | POST /orders, POST /ensure              |
| `view_orders`     | GET /orders, GET /orders/:id            |
| `view_balance`    | GET /balance, GET /history              |
| `broadcast`       | POST /chain/broadcast                   |

Esto permite crear claves de solo lectura para paneles de monitoreo y claves restringidas para sistemas de comercio automatizado.

## Envolvente de Respuesta

Cada respuesta sigue la misma estructura:

```json
{
  "data": { ... }
}
```

En caso de error:

```json
{
  "error": {
    "code": "INSUFFICIENT_FUNDS",
    "message": "Saldo de cuenta muy bajo para la operación solicitada",
    "details": { "required": 150000000, "available": 42000000 }
  }
}
```

Los códigos de error son cadenas legibles por máquina. El campo `details` es opcional y proporciona contexto para depuración.

## Grupos de Endpoints

Los 46 endpoints se organizan en nueve grupos. Aquí está el mapa completo.

### Precios (6 endpoints)

Estos endpoints son públicos - no se requiere clave API.

| Método | Ruta                        | Descripción                              |
|--------|-----------------------------|-----------------------------------------|
| GET    | /api/v1/prices              | Precios actuales de todos los proveedores |
| GET    | /api/v1/prices/best         | Proveedor más barato para un tipo de recurso |
| GET    | /api/v1/prices/history      | Datos de precios históricos              |
| GET    | /api/v1/prices/stats        | Estadísticas agregadas del mercado       |
| GET    | /api/v1/prices/analysis     | Análisis de tendencias y recomendación de compra |
| GET    | /api/v1/orders/preview      | Previsualización de costo antes de colocar una orden |

### Órdenes (3 endpoints)

| Método | Ruta                        | Descripción                              |
|--------|-----------------------------|-----------------------------------------|
| POST   | /api/v1/orders              | Crear una nueva orden de energía o bandwidth |
| GET    | /api/v1/orders              | Listar órdenes con paginación y filtros  |
| GET    | /api/v1/orders/:id          | Obtener detalles de la orden con desglose de ejecuciones |

### Cuenta (7 endpoints)

| Método | Ruta                        | Descripción                              |
|--------|-----------------------------|-----------------------------------------|
| GET    | /api/v1/balance             | Saldos actuales de TRX y USDT            |
| GET    | /api/v1/deposit/info        | Dirección de depósito y memo             |
| POST   | /api/v1/deposit/prepare     | Preparar una transacción de depósito     |
| POST   | /api/v1/deposit/submit      | Enviar comprobante de depósito           |
| POST   | /api/v1/withdraw            | Retirar TRX o USDT                       |
| GET    | /api/v1/history             | Historial de ejecución de órdenes        |
| GET    | /api/v1/history/summary     | Estadísticas agregadas de la cuenta      |

### Claves API (3 endpoints)

| Método | Ruta                        | Descripción                              |
|--------|-----------------------------|-----------------------------------------|
| GET    | /api/v1/keys                | Listar todas las claves API              |
| POST   | /api/v1/keys                | Crear una nueva clave API                |
| DELETE | /api/v1/keys/:id            | Revocar una clave API                    |

### Autenticación (2 endpoints)

| Método | Ruta                        | Descripción                              |
|--------|-----------------------------|-----------------------------------------|
| POST   | /api/v1/auth/register       | Crear una nueva cuenta                   |
| POST   | /api/v1/auth/login          | Autenticarse y recibir un token JWT      |

### Estimación (2 endpoints)

| Método | Ruta                        | Descripción                              |
|--------|-----------------------------|-----------------------------------------|
| POST   | /api/v1/estimate            | Estimar energía y costo para una transacción |
| POST   | /api/v1/ensure              | Asegurar recursos mínimos en una dirección |

### Webhooks (3 endpoints)

| Método | Ruta                        | Descripción                              |
|--------|-----------------------------|-----------------------------------------|
| POST   | /api/v1/webhooks            | Crear una suscripción a webhook          |
| GET    | /api/v1/webhooks            | Listar suscripciones a webhooks          |
| DELETE | /api/v1/webhooks/:id        | Eliminar un webhook                      |

### Órdenes Permanentes y Monitores (7 endpoints)

| Método | Ruta                             | Descripción                           |
|--------|----------------------------------|---------------------------------------|
| POST   | /api/v1/standing-orders          | Crear una orden permanente            |
| GET    | /api/v1/standing-orders          | Listar órdenes permanentes            |
| GET    | /api/v1/standing-orders/:id      | Obtener detalles de la orden permanente |
| DELETE | /api/v1/standing-orders/:id      | Cancelar una orden permanente         |
| POST   | /api/v1/monitors                 | Crear un monitor de recursos          |
| GET    | /api/v1/monitors                 | Listar monitores activos              |
| DELETE | /api/v1/monitors/:id             | Cancelar un monitor                   |

### Proxy de Cadena (10 endpoints)

Estos endpoints hacen proxy de consultas de la red TRON a través de MERX, eliminando la necesidad de que los clientes llamen a TronGrid directamente.

| Método | Ruta                             | Descripción                           |
|--------|----------------------------------|---------------------------------------|
| GET    | /api/v1/chain/account/:address   | Información de cuenta y recursos      |
| GET    | /api/v1/chain/balance/:address   | Saldo de TRX                          |
| GET    | /api/v1/chain/resources/:address | Desglose de energía y bandwidth       |
| GET    | /api/v1/chain/transaction/:txid  | Detalles de la transacción            |
| GET    | /api/v1/chain/block/:number      | Bloque por número (o el más reciente) |
| GET    | /api/v1/chain/parameters         | Parámetros de la cadena               |
| GET    | /api/v1/chain/history/:address   | Historial de transacciones de la dirección |
| POST   | /api/v1/chain/read-contract      | Llamar a una función de contrato constante |
| POST   | /api/v1/chain/broadcast          | Transmitir una transacción firmada    |
| GET    | /api/v1/address/:addr/resources  | Resumen de recursos de la dirección   |

### x402 Pago Por Uso (3 endpoints)

| Método | Ruta                        | Descripción                              |
|--------|-----------------------------|-----------------------------------------|
| POST   | /api/v1/x402/invoice        | Crear una factura de pago                |
| GET    | /api/v1/x402/invoice/:id    | Verificar estado de la factura           |
| POST   | /api/v1/x402/verify         | Verificar pago y ejecutar orden          |

## Endpoints Clave en Detalle

### GET /api/v1/prices

Devuelve precios actuales de todos los proveedores activos. No se requiere autenticación. Este es el endpoint que llama para ver el mercado completo de un vistazo.

```bash
curl https://merx.exchange/api/v1/prices
```

Respuesta (abreviada):

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

Cada entrada de proveedor incluye los niveles de precios (por duración), capacidad disponible y la marca de tiempo del último sondeo exitoso.

### POST /api/v1/orders

Crea una orden de energía o bandwidth. La plataforma compara la orden con los proveedores disponibles y enruta al más barato que pueda cumplirla.

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

Respuesta:

```json
{
  "data": {
    "id": "ord_a1b2c3d4",
    "status": "PENDING",
    "created_at": "2026-03-30T12:00:00.000Z"
  }
}
```

El encabezado `Idempotency-Key` previene órdenes duplicadas si se reintenta la misma solicitud. Si la clave ha sido vista antes, la API devuelve la orden original en lugar de crear una nueva.

Tipos de orden:
- `MARKET` - ejecutar inmediatamente al mejor precio disponible
- `LIMIT` - ejecutar solo si el precio es igual o inferior a `max_price_sun`
- `PERIODIC` - orden recurrente en un horario
- `BROADCAST` - transmitir una transacción de delegación pre-firmada

### GET /api/v1/orders/:id

Devuelve la orden con sus detalles de ejecución - qué proveedores cumplieron la orden, a qué precio y los IDs de transacción en cadena.

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

Estima la energía y bandwidth requeridos para una operación TRON, luego compara el costo de alquiler contra el costo de quema de TRX.

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

Este endpoint es útil para mostrar a los usuarios exactamente cuánto ahorran alquilando energía a través de MERX versus quemar TRX.

## Límites de Velocidad

Los límites de velocidad se aplican por dirección IP usando ventanas deslizantes.

| Grupo de endpoints  | Límite             | Ventana |
|---------------------|--------------------|---------|
| Precios (público)   | 300 solicitudes    | 1 min   |
| Predeterminado (general) | 100 solicitudes | 1 min   |
| Balance             | 60 solicitudes     | 1 min   |
| Historial           | 60 solicitudes     | 1 min   |
| Órdenes             | 10 solicitudes     | 1 min   |
| Retiros             | 5 solicitudes      | 1 min   |
| Broadcast           | 20 solicitudes     | 1 min   |
| Registro            | 5 solicitudes      | 1 hora  |

Cuando se excede un límite de velocidad, la API devuelve HTTP 429 con el formato de error estándar:

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Límite de velocidad excedido"
  }
}
```

Los encabezados de límite de velocidad (`RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`) se incluyen en todas las respuestas siguiendo el estándar de borrador IETF.

## Códigos de Error

| Código                  | HTTP | Descripción                                    |
|-------------------------|------|------------------------------------------------|
| `UNAUTHORIZED`          | 401  | Clave API inválida o faltante                 |
| `RATE_LIMITED`          | 429  | Demasiadas solicitudes                         |
| `VALIDATION_ERROR`     | 400  | Cuerpo o parámetros de solicitud fallaron en validación |
| `INVALID_ADDRESS`      | 400  | No es una dirección TRON válida                |
| `INSUFFICIENT_FUNDS`   | 400  | Saldo de cuenta demasiado bajo                 |
| `BELOW_MINIMUM_ORDER`  | 400  | Monto de orden por debajo del mínimo del proveedor |
| `DUPLICATE_REQUEST`    | 409  | Clave de idempotencia ya utilizada             |
| `ORDER_NOT_FOUND`      | 404  | Orden o recurso no existe                      |
| `PROVIDER_UNAVAILABLE` | 404  | Ningún proveedor puede cumplir la solicitud    |
| `INTERNAL_ERROR`       | 500  | Error del lado del servidor                    |

## Inicio Rápido con SDKs

Si bien la API REST se puede llamar directamente con cualquier cliente HTTP, MERX proporciona SDKs oficiales para JavaScript y Python que manejan autenticación, análisis de errores y seguridad de tipos.

### JavaScript / TypeScript

```bash
npm install merx-sdk
```

```javascript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({ apiKey: 'sk_live_your_key_here' })

// Obtener todos los precios
const prices = await merx.prices.list()

// Crear una orden
const order = await merx.orders.create({
  resource_type: 'ENERGY',
  amount: 65000,
  target_address: 'TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb',
  duration_sec: 3600,
})

// Verificar estado de la orden
const details = await merx.orders.get(order.id)
```

### Python

```bash
pip install merx-sdk
```

```python
from merx import MerxClient

client = MerxClient(api_key="sk_live_your_key_here")

# Obtener todos los precios
prices = client.prices.list()

# Crear una orden
order = client.orders.create(
    resource_type="ENERGY",
    amount=65000,
    target_address="TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    duration_sec=3600,
)

# Verificar estado de la orden
details = client.orders.get(order.id)
```

## WebSocket para Datos en Tiempo Real

Además de la API REST, MERX proporciona un endpoint WebSocket en `wss://merx.exchange/ws` para actualizaciones de precios en tiempo real. Los cambios de precio se empujan a los clientes conectados a medida que suceden, con actualizaciones llegando cada 30 segundos por proveedor.

La conexión WebSocket admite filtrado de proveedores - suscribirse solo a los proveedores que te importan e ignorar el resto.

## Órdenes Permanentes

Las órdenes permanentes automatizan compras de energía basadas en disparadores. Puede establecer un umbral de precio, un horario o una condición de saldo, y la plataforma ejecuta órdenes automáticamente dentro de su presupuesto especificado.

Los tipos de disparador incluyen `price_below`, `price_above`, `schedule`, `balance_below` y `provider_available`. Los tipos de acción incluyen `buy_resource`, `ensure_resources`, `deposit_trx` y `notify_only`.

Esto hace que MERX sea adecuado para gestión de infraestructura completamente automatizada - establezca sus reglas una vez, y la plataforma maneja la ejecución.

## Lo Que Viene Después

La API de MERX está diseñada para desarrolladores y empresas que necesitan acceso confiable y rentable a los recursos de la red TRON. Ya sea que esté construyendo un procesador de pagos, una aplicación DeFi o un intercambio, la API proporciona los componentes básicos para gestionar energía y bandwidth programáticamente.

La documentación completa de la API está disponible en [merx.exchange/docs](https://merx.exchange/docs). El SDK de JavaScript está en [GitHub](https://github.com/Hovsteder/merx-sdk-js) y [npm](https://www.npmjs.com/package/merx-sdk). El SDK de Python está en [PyPI](https://pypi.org/project/merx-sdk/).

---

**Enlaces:**
- Plataforma: [merx.exchange](https://merx.exchange)
- Documentación: [merx.exchange/docs](https://merx.exchange/docs)
- SDK de JavaScript: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- SDK de Python: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- Servidor MCP: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp) | [npm](https://www.npmjs.com/package/merx-mcp)


## Pruébalo Ahora con IA

Agregue MERX a Claude Desktop o cualquier cliente compatible con MCP -- sin instalación, sin clave API necesaria para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregúntale a tu agente de IA: "¿Cuál es la energía TRON más barata en este momento?" y obtén precios actuales de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)