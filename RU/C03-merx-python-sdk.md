# MERX Python SDK: Нулевые зависимости, полная мощь

MERX Python SDK предоставляет полный интерфейс к обмену TRON энергией MERX, используя только стандартную библиотеку Python — без requests, без httpx, без aiohttp. Эта статья охватывает четыре модуля SDK (цены, заказы, баланс, вебхуки), все методы с рабочими примерами, систему типов dataclass, обработку ошибок и сравнение с JavaScript SDK для команд, использующих оба языка.

## Почему нулевые зависимости

Большинство Python HTTP-клиентов подтягивают дерево зависимостей. Сама библиотека `requests` устанавливает `urllib3`, `certifi`, `charset-normalizer` и `idna`. Для лёгкого SDK, который делает несколько API-запросов, эта нагрузка ненужна.

MERX Python SDK использует только `urllib.request` из стандартной библиотеки. Это означает:

- Отсутствие конфликтов зависимостей в вашей виртуальной среде
- Отсутствие поверхности атаки цепи поставок от сторонних пакетов
- Работает на любой установке Python 3.10+ без нагрузки `pip install`
- Развёртывается в ограниченных окружениях, где установка пакетов затруднена

Компромисс — отсутствие асинхронной поддержки. SDK использует синхронные HTTP-вызовы. Если вам нужна асинхронность, REST API достаточно прост для прямого вызова с `aiohttp` или `httpx` по шаблонам, показанным в этой статье.

## Установка

```bash
pip install merx-sdk
```

Это всё. Никаких зависимостей не будет установлено рядом с ним.

## Быстрый старт

```python
from merx import MerxClient

client = MerxClient(api_key="sk_live_your_key_here")

# Проверить текущие цены
prices = client.prices.list()
for p in prices:
    if p.energy_prices:
        print(f"{p.provider}: {p.energy_prices[0].price_sun} SUN/energy")

# Купить энергию
order = client.orders.create(
    resource_type="ENERGY",
    amount=65000,
    target_address="TYourTargetAddressHere",
    duration_sec=3600,
)
print(f"Order {order.id}: {order.status}")
```

Клиент инициализируется с вашим API-ключом из панели управления MERX на [merx.exchange](https://merx.exchange). Необязательный параметр `base_url` по умолчанию установлен на `https://merx.exchange`.

```python
client = MerxClient(
    api_key="sk_live_your_key_here",
    base_url="https://merx.exchange",  # Опционально
)
```

## Система типов

Каждый ответ API преобразуется в Python dataclass. Никаких сырых словарей, никаких нетипизированных данных. Ваша IDE получает полное автодополнение, а ваш проверяющий тип ловит ошибки до выполнения.

SDK экспортирует 16 типов dataclass:

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

### Ключевые определения типов

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
    resource_type: str       # "ENERGY" or "BANDWIDTH"
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

Все dataclasses используют стандартные подсказки типов Python. Поля со значениями по умолчанию — это необязательные поля ответа API, которые могут отсутствовать в каждом ответе.

## Модуль 1: Цены

Модуль цен предоставляет пять методов для запроса данных о ценах в реальном времени и исторических данных. Все конечные точки цен являются открытыми и не требуют аутентификации, но SDK всегда отправляет заголовок API-ключа для согласованности.

### prices.list()

Возвращает текущие цены от всех активных поставщиков.

```python
prices = client.prices.list()

for p in prices:
    print(f"\n{p.provider} (market={p.is_market}):")
    print(f"  Available energy: {p.available_energy:,}")
    for tier in p.energy_prices:
        hours = tier.duration_sec // 3600
        print(f"  {hours}h: {tier.price_sun} SUN/unit")
```

Возвращает `list[ProviderPrice]`. Каждая запись включает имя поставщика, поддерживает ли он рыночные заказы с гибкой длительностью, ценовые уровни для энергии и пропускной способности, а также текущую доступную ёмкость.

### prices.best(resource, amount=None)

Возвращает самую дешевую доступную цену для типа ресурса.

```python
best = client.prices.best("ENERGY")
print(f"Cheapest: {best.price_sun} SUN from {best.provider}")

# Только поставщики с не менее 200 000 доступных
best_large = client.prices.best("ENERGY", amount=200000)
```

Возвращает `PriceHistoryEntry`. Параметр `amount` отфильтровывает поставщиков, у которых недостаточно ёмкости.

### prices.history(provider=None, resource=None, period="24h")

Возвращает исторические снимки цен для графиков и анализа.

```python
history = client.prices.history(
    provider="sohu",
    resource="ENERGY",
    period="7d",
)

for entry in history:
    print(f"{entry.polled_at}: {entry.price_sun} SUN ({entry.available:,} available)")
```

Возвращает `list[PriceHistoryEntry]`. Доступные периоды: `"1h"`, `"6h"`, `"24h"`, `"7d"`, `"30d"`.

### prices.stats()

Возвращает сводную статистику рынка.

```python
stats = client.prices.stats()
print(f"Best price: {stats.best_price_sun} SUN")
print(f"Average price: {stats.avg_price_sun} SUN")
print(f"Providers online: {stats.total_providers}")
print(f"Cheapest-provider changes (24h): {stats.cheapest_changes_24h}")
```

Возвращает `PriceStats`.

### prices.preview(resource, amount, duration, max_price_sun=None)

Предпросмотр, что будет стоить заказ, перед его размещением.

```python
preview = client.prices.preview(
    resource="ENERGY",
    amount=100000,
    duration=86400,       # 24 часа
    max_price_sun=35,     # Опциональное ограничение сверху
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

Возвращает `OrderPreview` с совпадением `best` (или `None`), списком `fallbacks` и булевым значением `no_providers`.

## Модуль 2: Заказы

### orders.create(...)

Создаёт заказ энергии или пропускной способности.

```python
order = client.orders.create(
    resource_type="ENERGY",
    amount=65000,
    target_address="TYourTargetAddress",
    duration_sec=3600,
    order_type="MARKET",           # По умолчанию
    max_price_sun=None,            # Требуется для заказов LIMIT
    idempotency_key="unique-id",   # Предотвращает дублирующиеся заказы
)

print(f"Order {order.id}: {order.status}")
```

Возвращает `Order`. Обратите внимание, что в отличие от JavaScript SDK, где параметры передаются как словарь, Python SDK использует явные ключевые аргументы для ясности.

Параметр `idempotency_key` критичен для рабочих систем. Если сетевая ошибка вызывает повтор, API возвращает исходный заказ вместо создания дубликата.

### orders.list(limit=30, offset=0, status=None)

Список заказов с пагинацией.

```python
orders, total = client.orders.list(limit=10, offset=0, status="FILLED")
print(f"{total} filled orders total")

for o in orders:
    cost_trx = (o.total_cost_sun or 0) / 1_000_000
    print(f"  {o.id}: {o.amount:,} {o.resource_type}, {cost_trx:.3f} TRX")
```

Возвращает `tuple[list[Order], int]` — список заказов и общее количество для пагинации.

### orders.get(order_id)

Возвращает один заказ с разбивкой по заполнениям.

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

Возвращает `OrderWithFills`, который расширяет `Order` списком `fills` объектов `Fill`.

## Модуль 3: Баланс

### balance.get()

Возвращает текущие остатки счёта.

```python
bal = client.balance.get()
print(f"TRX: {bal.trx}")
print(f"USDT: {bal.usdt}")
print(f"Locked: {bal.trx_locked}")
```

Возвращает `Balance`. Поле `trx_locked` показывает TRX, зарезервированный для ожидающих заказов.

### balance.deposit_info()

Возвращает адрес депозита и memo для пополнения вашего счёта.

```python
info = client.balance.deposit_info()
print(f"Send TRX to: {info.address}")
print(f"Memo: {info.memo}")
print(f"Minimum: {info.min_amount_trx} TRX / {info.min_amount_usdt} USDT")
```

Возвращает `DepositInfo`. Всегда включайте memo в вашу транзакцию депозита для автоматического зачисления.

### balance.withdraw(address, amount, currency="TRX", idempotency_key=None)

Выводит средства на внешний адрес TRON.

```python
withdrawal = client.balance.withdraw(
    address="TExternalAddress",
    amount=100,
    currency="TRX",
    idempotency_key="withdraw-unique-id",
)
print(f"Withdrawal {withdrawal.id}: {withdrawal.status}")
```

Возвращает `Withdrawal`.

### balance.history(period="30D") и balance.summary()

```python
history = client.balance.history("7D")
print(f"{len(history)} transactions in last 7 days")

summary = client.balance.summary()
print(f"Total orders: {summary.total_orders}")
print(f"Total energy: {summary.total_energy:,}")
print(f"Average price: {summary.avg_price_sun} SUN")
print(f"Total spent: {summary.total_spent_sun / 1_000_000:.3f} TRX")
```

## Модуль 4: Вебхуки

### webhooks.create(url, events)

Создаёт подписку вебхука.

```python
webhook = client.webhooks.create(
    url="https://your-server.com/merx-webhook",
    events=["order.filled", "order.failed", "deposit.received"],
)
print(f"Webhook ID: {webhook.id}")
print(f"Secret: {webhook.secret}")  # Показывается только один раз
```

Возвращает `Webhook`. Поле `secret` включается только в ответ создания и используется для проверки подписи HMAC-SHA256.

Доступные типы событий: `order.filled`, `order.failed`, `deposit.received`, `withdrawal.completed`.

### webhooks.list() и webhooks.delete(webhook_id)

```python
webhooks = client.webhooks.list()
active = [w for w in webhooks if w.is_active]
print(f"{len(active)} active webhooks")

deleted = client.webhooks.delete("wh_abc123")
print(f"Deleted: {deleted}")  # True или False
```

## Обработка ошибок

Все ошибки API вызывают `MerxError` с атрибутом `code` и понятным для человека сообщением.

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

Класс `MerxError` расширяет `Exception`, поэтому он естественно интегрируется с обработкой исключений Python. Атрибут `code` обеспечивает машиночитаемую классификацию:

| Код                    | Значение                                         |
|------------------------|--------------------------------------------------|
| `UNAUTHORIZED`         | Недействительный или отсутствующий API-ключ     |
| `INSUFFICIENT_FUNDS`   | Баланс счёта слишком низкий                      |
| `INVALID_ADDRESS`      | Не действительный адрес TRON                     |
| `ORDER_NOT_FOUND`      | ID заказа не существует                          |
| `DUPLICATE_REQUEST`    | Ключ идемпотентности уже использован             |
| `RATE_LIMITED`         | Слишком много запросов, отступитесь              |
| `PROVIDER_UNAVAILABLE` | Ни один поставщик не может выполнить запрос      |
| `VALIDATION_ERROR`     | Корпус запроса или параметры не прошли валидацию |

## Сравнение с JavaScript SDK

Оба SDK охватывают одинаковые четыре модуля с одинаковыми методами. Ключевые различия заключаются в языковых идиомах:

| Аспект              | Python SDK                       | JavaScript SDK                     |
|---------------------|----------------------------------|------------------------------------|
| HTTP-клиент         | `urllib.request` (stdlib)        | Native `fetch`                     |
| Зависимости         | Нулевые                          | Нулевые                            |
| Асинхронная поддержка| Нет (только синхронные)          | Да (все методы возвращают Promises)|
| Типы                | Dataclasses                      | TypeScript interfaces              |
| Создание заказа     | Ключевые аргументы               | Параметр объекта                   |
| Возврат списка заказов| `tuple[list, int]`              | `{ orders: [], total: number }`    |
| Удаление вебхука    | Возвращает `bool`                | Возвращает `{ deleted: boolean }`  |
| Минимальная среда выполнения| Python 3.10+                | Node.js 18+                        |
| Система модулей     | Стандартные импорты              | Только ESM                         |

Оба SDK разработаны таким образом, чтобы выглядеть естественно в своих соответствующих языках, при этом сохраняя функциональную паритет. Команда, использующая Python и JavaScript, может ожидать идентичного поведения от каждого.

## Производственный паттерн: размещение заказа с учётом цены

Вот полный производственный поток, который проверяет текущие цены, проверяет стоимость и создаёт заказ с идемпотентностью:

```python
import uuid
from merx import MerxClient, MerxError

client = MerxClient(api_key="sk_live_your_key_here")

def buy_energy(target: str, amount: int = 65000, max_cost_trx: float = 5.0):
    """Купить энергию с проверкой стоимости и идемпотентностью."""

    # Сначала предпросмотреть стоимость
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

    # Разместить заказ
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

## Проверка установки

После установки проверьте, что SDK работает:

```python
from merx import MerxClient, __version__

print(f"merx-sdk version: {__version__}")

# Открытая конечная точка, аутентификация не требуется
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

Конечная точка `prices.list()` открыта, поэтому даже вспомогательный API-ключ будет работать для проверки подключения.

## Ресурсы

- Платформа: [merx.exchange](https://merx.exchange)
- Документация: [merx.exchange/docs](https://merx.exchange/docs)
- PyPI: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- MCP Server для AI-агентов: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp) | [npm](https://www.npmjs.com/package/merx-mcp)


## Попробуйте прямо сейчас с AI

Добавьте MERX в Claude Desktop или любой MCP-совместимый клиент — никакой установки, никакого API-ключа не требуется для инструментов только для чтения:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Спросите у вашего AI-агента: "What is the cheapest TRON energy right now?" и получите актуальные цены от всех подключённых поставщиков.

Полная документация MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)