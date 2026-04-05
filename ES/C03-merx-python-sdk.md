# SDK Python de MERX: Cero Dependencias, Potencia Total

El SDK Python de MERX proporciona una interfaz completa al intercambio de energía TRON de MERX utilizando solo la biblioteca estándar de Python - sin requests, sin httpx, sin aiohttp. Este artículo cubre los cuatro módulos del SDK (precios, órdenes, saldo, webhooks), todos los métodos con ejemplos funcionales, el sistema de tipos dataclass, manejo de errores y una comparación con el SDK de JavaScript para equipos que usan ambos lenguajes.

## Por Qué Cero Dependencias

La mayoría de los clientes HTTP de Python traen consigo un árbol de dependencias. La librería `requests` por sí sola trae `urllib3`, `certifi`, `charset-normalizer` e `idna`. Para un SDK ligero que realiza unos pocos llamados a API, ese sobrecargo es innecesario.

El SDK Python de MERX utiliza solo `urllib.request` de la biblioteca estándar. Esto significa:

- Sin conflictos de dependencias en tu entorno virtual
- Sin superficie de ataque de la cadena de suministro de paquetes de terceros
- Funciona en cualquier instalación de Python 3.10+ sin sobrecargo de `pip install`
- Deployable en entornos restringidos donde instalar paquetes es difícil

El compromiso es la ausencia de soporte asincrónico. El SDK utiliza llamadas HTTP sincrónicas. Si necesitas async, la API REST es lo suficientemente simple para llamar directamente con `aiohttp` o `httpx` usando los patrones mostrados en este artículo.

## Instalación

```bash
pip install merx-sdk
```

Eso es todo. No se instalarán dependencias junto a él.

## Inicio Rápido

```python
from merx import MerxClient

client = MerxClient(api_key="sk_live_your_key_here")

# Verificar precios actuales
prices = client.prices.list()
for p in prices:
    if p.energy_prices:
        print(f"{p.provider}: {p.energy_prices[0].price_sun} SUN/energy")

# Comprar energía
order = client.orders.create(
    resource_type="ENERGY",
    amount=65000,
    target_address="TYourTargetAddressHere",
    duration_sec=3600,
)
print(f"Order {order.id}: {order.status}")
```

El cliente se inicializa con tu clave API del panel de MERX en [merx.exchange](https://merx.exchange). El parámetro opcional `base_url` tiene como valor predeterminado `https://merx.exchange`.

```python
client = MerxClient(
    api_key="sk_live_your_key_here",
    base_url="https://merx.exchange",  # Opcional
)
```

## El Sistema de Tipos

Todas las respuestas de API se analizan en un dataclass de Python. Sin diccionarios sin procesar, sin datos sin tipo. Tu IDE obtiene autocompletado completo y tu verificador de tipos detecta errores antes del tiempo de ejecución.

El SDK exporta 16 tipos de dataclass:

```python
from merx import (
    Balance,
    DepositInfo,
    Fill,
    HistoryEntry,
    HistorySummary,
    Order,
    OrderPreview,
    OrderWithFills,
    PreviewMatch,
    PriceHistoryEntry,
    PricePoint,
    PriceStats,
    ProviderPrice,
    Webhook,
    Withdrawal,
    MerxError,
)
```

### Definiciones de Tipos Clave

```python
@dataclass
class PricePoint:
    duration_sec: int
    price_sun: int

@dataclass
class ProviderPrice:
    provider: str
    is_market: bool
    energy_prices: list[PricePoint]
    bandwidth_prices: list[PricePoint]
    available_energy: int
    available_bandwidth: int
    fetched_at: int

@dataclass
class Order:
    id: str
    resource_type: str       # "ENERGY" o "BANDWIDTH"
    order_type: str          # "MARKET", "LIMIT", "PERIODIC", "BROADCAST"
    status: str              # "PENDING", "EXECUTING", "FILLED", "FAILED", "CANCELLED"
    amount: int
    target_address: str
    duration_sec: int
    total_cost_sun: Optional[int] = None
    total_fee_sun: Optional[int] = None
    created_at: str = ""
    filled_at: Optional[str] = None
    expires_at: Optional[str] = None

@dataclass
class Fill:
    provider: str
    amount: int
    price_sun: int
    cost_sun: int
    tx_id: Optional[str] = None
    status: str = ""
    delegation_tx: Optional[str] = None
    verified: bool = False
    tronscan_url: Optional[str] = None
```

Todos los dataclasses utilizan indicaciones de tipo de Python estándar. Los campos con valores predeterminados son campos de respuesta API opcionales que pueden no estar presentes en cada respuesta.

## Módulo 1: Precios

El módulo de precios proporciona cinco métodos para consultar datos de precios en tiempo real e históricos. Todos los puntos finales de precios son públicos y no requieren autenticación, pero el SDK siempre envía el encabezado de clave API para mantener coherencia.

### prices.list()

Devuelve precios actuales de todos los proveedores activos.

```python
prices = client.prices.list()

for p in prices:
    print(f"\n{p.provider} (market={p.is_market}):")
    print(f"  Available energy: {p.available_energy:,}")
    for tier in p.energy_prices:
        hours = tier.duration_sec // 3600
        print(f"  {hours}h: {tier.price_sun} SUN/unit")
```

Devuelve `list[ProviderPrice]`. Cada entrada incluye el nombre del proveedor, si admite órdenes de mercado con duraciones flexibles, niveles de precios para energía y bandwidth, y capacidad disponible actual.

### prices.best(resource, amount=None)

Devuelve el precio disponible más barato para un tipo de recurso.

```python
best = client.prices.best("ENERGY")
print(f"Cheapest: {best.price_sun} SUN from {best.provider}")

# Solo proveedores con al menos 200,000 disponibles
best_large = client.prices.best("ENERGY", amount=200000)
```

Devuelve `PriceHistoryEntry`. El parámetro `amount` filtra proveedores que carecen de capacidad suficiente.

### prices.history(provider=None, resource=None, period="24h")

Devuelve instantáneas de precios históricos para gráficos y análisis.

```python
history = client.prices.history(
    provider="sohu",
    resource="ENERGY",
    period="7d",
)

for entry in history:
    print(f"{entry.polled_at}: {entry.price_sun} SUN ({entry.available:,} available)")
```

Devuelve `list[PriceHistoryEntry]`. Períodos disponibles: `"1h"`, `"6h"`, `"24h"`, `"7d"`, `"30d"`.

### prices.stats()

Devuelve estadísticas agregadas del mercado.

```python
stats = client.prices.stats()
print(f"Best price: {stats.best_price_sun} SUN")
print(f"Average price: {stats.avg_price_sun} SUN")
print(f"Providers online: {stats.total_providers}")
print(f"Cheapest-provider changes (24h): {stats.cheapest_changes_24h}")
```

Devuelve `PriceStats`.

### prices.preview(resource, amount, duration, max_price_sun=None)

Obtiene una vista previa de lo que costaría una orden antes de colocarla.

```python
preview = client.prices.preview(
    resource="ENERGY",
    amount=100000,
    duration=86400,       # 24 horas
    max_price_sun=35,     # Techo opcional
)

if preview.best:
    print(f"Best: {preview.best.provider}")
    print(f"Cost: {preview.best.cost_trx} TRX")
    print(f"Price: {preview.best.price_sun} SUN/unit")

for fb in preview.fallbacks:
    print(f"Alternative: {fb.provider} at {fb.cost_trx} TRX")

if preview.no_providers:
    print("No providers available for these parameters")
```

Devuelve `OrderPreview` con una coincidencia `best` (o `None`), una lista de `fallbacks`, y un booleano `no_providers`.

## Módulo 2: Órdenes

### orders.create(...)

Crea una orden de energía o bandwidth.

```python
order = client.orders.create(
    resource_type="ENERGY",
    amount=65000,
    target_address="TYourTargetAddress",
    duration_sec=3600,
    order_type="MARKET",           # Predeterminado
    max_price_sun=None,            # Requerido para órdenes LIMIT
    idempotency_key="unique-id",   # Previene órdenes duplicadas
)

print(f"Order {order.id}: {order.status}")
```

Devuelve `Order`. Nota que a diferencia del SDK de JavaScript donde los parámetros se pasan como un diccionario, el SDK de Python utiliza argumentos de palabra clave explícitos para mayor claridad.

El parámetro `idempotency_key` es crítico para sistemas en producción. Si un error de red causa un reintento, la API devuelve la orden original en lugar de crear una duplicada.

### orders.list(limit=30, offset=0, status=None)

Lista órdenes con paginación.

```python
orders, total = client.orders.list(limit=10, offset=0, status="FILLED")
print(f"{total} filled orders total")

for o in orders:
    cost_trx = (o.total_cost_sun or 0) / 1_000_000
    print(f"  {o.id}: {o.amount:,} {o.resource_type}, {cost_trx:.3f} TRX")
```

Devuelve `tuple[list[Order], int]` - la lista de órdenes y el recuento total para paginación.

### orders.get(order_id)

Devuelve una sola orden con su desglose de rellenos.

```python
order = client.orders.get("ord_abc123")

print(f"Status: {order.status}")
print(f"Total cost: {(order.total_cost_sun or 0) / 1_000_000:.3f} TRX")

for fill in order.fills:
    print(f"  {fill.provider}: {fill.amount:,} at {fill.price_sun} SUN")
    print(f"  Verified: {fill.verified}")
    if fill.tronscan_url:
        print(f"  TX: {fill.tronscan_url}")
```

Devuelve `OrderWithFills`, que extiende `Order` con una lista `fills` de objetos `Fill`.

## Módulo 3: Saldo

### balance.get()

Devuelve los saldos actuales de la cuenta.

```python
bal = client.balance.get()
print(f"TRX: {bal.trx}")
print(f"USDT: {bal.usdt}")
print(f"Locked: {bal.trx_locked}")
```

Devuelve `Balance`. El campo `trx_locked` muestra TRX reservado para órdenes pendientes.

### balance.deposit_info()

Devuelve la dirección de depósito y el memo para financiar tu cuenta.

```python
info = client.balance.deposit_info()
print(f"Send TRX to: {info.address}")
print(f"Memo: {info.memo}")
print(f"Minimum: {info.min_amount_trx} TRX / {info.min_amount_usdt} USDT")
```

Devuelve `DepositInfo`. Siempre incluye el memo en tu transacción de depósito para acreditación automática.

### balance.withdraw(address, amount, currency="TRX", idempotency_key=None)

Retira fondos a una dirección TRON externa.

```python
withdrawal = client.balance.withdraw(
    address="TExternalAddress",
    amount=100,
    currency="TRX",
    idempotency_key="withdraw-unique-id",
)
print(f"Withdrawal {withdrawal.id}: {withdrawal.status}")
```

Devuelve `Withdrawal`.

### balance.history(period="30D") y balance.summary()

```python
history = client.balance.history("7D")
print(f"{len(history)} transactions in last 7 days")

summary = client.balance.summary()
print(f"Total orders: {summary.total_orders}")
print(f"Total energy: {summary.total_energy:,}")
print(f"Average price: {summary.avg_price_sun} SUN")
print(f"Total spent: {summary.total_spent_sun / 1_000_000:.3f} TRX")
```

## Módulo 4: Webhooks

### webhooks.create(url, events)

Crea una suscripción a webhook.

```python
webhook = client.webhooks.create(
    url="https://your-server.com/merx-webhook",
    events=["order.filled", "order.failed", "deposit.received"],
)
print(f"Webhook ID: {webhook.id}")
print(f"Secret: {webhook.secret}")  # Se muestra solo una vez
```

Devuelve `Webhook`. El campo `secret` solo se incluye en la respuesta de creación y se usa para verificación de firma HMAC-SHA256.

Tipos de eventos disponibles: `order.filled`, `order.failed`, `deposit.received`, `withdrawal.completed`.

### webhooks.list() y webhooks.delete(webhook_id)

```python
webhooks = client.webhooks.list()
active = [w for w in webhooks if w.is_active]
print(f"{len(active)} active webhooks")

deleted = client.webhooks.delete("wh_abc123")
print(f"Deleted: {deleted}")  # True o False
```

## Manejo de Errores

Todos los errores de API lanzan `MerxError` con un atributo `code` y un mensaje legible por humanos.

```python
from merx import MerxClient, MerxError

client = MerxClient(api_key="sk_live_your_key_here")

try:
    order = client.orders.create(
        resource_type="ENERGY",
        amount=65000,
        target_address="TInvalidAddress",
        duration_sec=3600,
    )
except MerxError as e:
    print(f"Error [{e.code}]: {e}")
    # Output: Error [INVALID_ADDRESS]: Target address is not a valid TRON address
```

La clase `MerxError` extiende `Exception`, así que se integra naturalmente con el manejo de excepciones de Python. El atributo `code` proporciona clasificación legible por máquina:

| Código                 | Significado                                  |
|------------------------|----------------------------------------------|
| `UNAUTHORIZED`         | Clave API inválida o faltante                |
| `INSUFFICIENT_FUNDS`   | Saldo de cuenta insuficiente                 |
| `INVALID_ADDRESS`      | No es una dirección TRON válida              |
| `ORDER_NOT_FOUND`      | El ID de orden no existe                     |
| `DUPLICATE_REQUEST`    | Clave de idempotencia ya utilizada           |
| `RATE_LIMITED`         | Demasiadas solicitudes, retroceder           |
| `PROVIDER_UNAVAILABLE` | Ningún proveedor puede cumplir la solicitud  |
| `VALIDATION_ERROR`     | Cuerpo de solicitud o parámetros no válidos  |

## Comparación con el SDK de JavaScript

Ambos SDKs cubren los mismos cuatro módulos con los mismos métodos. Las diferencias clave están en los idiomas de programación:

| Aspecto             | SDK Python                       | SDK JavaScript                     |
|---------------------|----------------------------------|------------------------------------|
| Cliente HTTP        | `urllib.request` (stdlib)        | Native `fetch`                     |
| Dependencias        | Cero                             | Cero                               |
| Soporte async       | No (solo sincrónico)             | Sí (todos los métodos devuelven Promises)  |
| Tipos               | Dataclasses                      | Interfaces TypeScript              |
| Creación de orden   | Argumentos de palabra clave      | Parámetro objeto                   |
| Retorno de lista de órdenes | `tuple[list, int]`        | `{ orders: [], total: number }`    |
| Eliminación de webhook | Devuelve `bool`              | Devuelve `{ deleted: boolean }`    |
| Tiempo de ejecución mínimo | Python 3.10+               | Node.js 18+                        |
| Sistema de módulos  | Importaciones estándar           | Solo ESM                           |

Los dos SDKs están diseñados para sentirse naturales en sus respectivos lenguajes mientras mantienen paridad funcional. Un equipo que usa Python y JavaScript puede esperar un comportamiento idéntico en cada uno.

## Patrón de Producción: Colocación de Órdenes Consciente de Precios

Aquí hay un flujo de producción completo que verifica precios actuales, valida el costo y crea una orden con idempotencia:

```python
import uuid
from merx import MerxClient, MerxError

client = MerxClient(api_key="sk_live_your_key_here")

def buy_energy(target: str, amount: int = 65000, max_cost_trx: float = 5.0):
    """Compra energía con validación de costo e idempotencia."""

    # Obtener vista previa del costo primero
    preview = client.prices.preview(
        resource="ENERGY",
        amount=amount,
        duration=3600,
    )

    if preview.no_providers:
        raise RuntimeError("No energy providers available")

    cost = float(preview.best.cost_trx)
    if cost > max_cost_trx:
        raise RuntimeError(
            f"Cost {cost:.3f} TRX exceeds limit {max_cost_trx:.3f} TRX"
        )

    # Colocar la orden
    order = client.orders.create(
        resource_type="ENERGY",
        amount=amount,
        target_address=target,
        duration_sec=3600,
        idempotency_key=str(uuid.uuid4()),
    )

    print(f"Order {order.id} created at {preview.best.provider}")
    print(f"Expected cost: {cost:.3f} TRX")
    return order


try:
    order = buy_energy("TYourTargetAddress", amount=65000, max_cost_trx=3.0)
except MerxError as e:
    print(f"API error: [{e.code}] {e}")
except RuntimeError as e:
    print(f"Business logic error: {e}")
```

## Verificación de Instalación

Después de instalar, verifica que el SDK esté funcionando:

```python
from merx import MerxClient, __version__

print(f"merx-sdk version: {__version__}")

# Punto final público, no requiere autenticación
client = MerxClient(api_key="test")
try:
    prices = client.prices.list()
    print(f"Providers online: {len(prices)}")
    for p in prices:
        if p.energy_prices:
            print(f"  {p.provider}: {p.energy_prices[0].price_sun} SUN")
except Exception as e:
    print(f"Connection error: {e}")
```

El punto final `prices.list()` es público, por lo que incluso una clave API de marcador de posición funcionará para probar conectividad.

## Recursos

- Plataforma: [merx.exchange](https://merx.exchange)
- Documentación: [merx.exchange/docs](https://merx.exchange/docs)
- PyPI: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- SDK JavaScript: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- Servidor MCP para agentes IA: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp) | [npm](https://www.npmjs.com/package/merx-mcp)

## Pruébalo Ahora con IA

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

Pregunta a tu agente IA: "¿Cuál es la energía TRON más barata en este momento?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)