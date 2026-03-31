# MERX Python SDK: Zero Dependencies, Full Power

The MERX Python SDK provides a complete interface to the MERX TRON energy exchange using only the Python standard library - no requests, no httpx, no aiohttp. This article covers the four SDK modules (prices, orders, balance, webhooks), every method with working examples, the dataclass type system, error handling, and a comparison with the JavaScript SDK for teams that use both languages.

## Why Zero Dependencies

Most Python HTTP clients pull in a dependency tree. The `requests` library alone brings in `urllib3`, `certifi`, `charset-normalizer`, and `idna`. For a lightweight SDK that makes a handful of API calls, that overhead is unnecessary.

The MERX Python SDK uses only `urllib.request` from the standard library. This means:

- No dependency conflicts in your virtual environment
- No supply chain attack surface from third-party packages
- Works on any Python 3.10+ installation without `pip install` overhead
- Deployable to restricted environments where installing packages is difficult

The tradeoff is the absence of async support. The SDK uses synchronous HTTP calls. If you need async, the REST API is simple enough to call directly with `aiohttp` or `httpx` using the patterns shown in this article.

## Installation

```bash
pip install merx-sdk
```

That is it. No dependencies will be installed alongside it.

## Quick Start

```python
from merx import MerxClient

client = MerxClient(api_key="sk_live_your_key_here")

# Check current prices
prices = client.prices.list()
for p in prices:
    if p.energy_prices:
        print(f"{p.provider}: {p.energy_prices[0].price_sun} SUN/energy")

# Buy energy
order = client.orders.create(
    resource_type="ENERGY",
    amount=65000,
    target_address="TYourTargetAddressHere",
    duration_sec=3600,
)
print(f"Order {order.id}: {order.status}")
```

The client is initialized with your API key from the MERX dashboard at [merx.exchange](https://merx.exchange). The optional `base_url` parameter defaults to `https://merx.exchange`.

```python
client = MerxClient(
    api_key="sk_live_your_key_here",
    base_url="https://merx.exchange",  # Optional
)
```

## The Type System

Every API response is parsed into a Python dataclass. No raw dictionaries, no untyped data. Your IDE gets full autocompletion and your type checker catches errors before runtime.

The SDK exports 16 dataclass types:

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

### Key Type Definitions

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

All dataclasses use standard Python type hints. Fields with defaults are optional API response fields that may not be present in every response.

## Module 1: Prices

The prices module provides five methods for querying real-time and historical price data. All price endpoints are public and do not require authentication, but the SDK always sends the API key header for consistency.

### prices.list()

Returns current pricing from all active providers.

```python
prices = client.prices.list()

for p in prices:
    print(f"\n{p.provider} (market={p.is_market}):")
    print(f"  Available energy: {p.available_energy:,}")
    for tier in p.energy_prices:
        hours = tier.duration_sec // 3600
        print(f"  {hours}h: {tier.price_sun} SUN/unit")
```

Returns `list[ProviderPrice]`. Each entry includes the provider name, whether it supports market orders with flexible durations, price tiers for energy and bandwidth, and current available capacity.

### prices.best(resource, amount=None)

Returns the cheapest available price for a resource type.

```python
best = client.prices.best("ENERGY")
print(f"Cheapest: {best.price_sun} SUN from {best.provider}")

# Only providers with at least 200,000 available
best_large = client.prices.best("ENERGY", amount=200000)
```

Returns `PriceHistoryEntry`. The `amount` parameter filters out providers that lack sufficient capacity.

### prices.history(provider=None, resource=None, period="24h")

Returns historical price snapshots for charting and analysis.

```python
history = client.prices.history(
    provider="sohu",
    resource="ENERGY",
    period="7d",
)

for entry in history:
    print(f"{entry.polled_at}: {entry.price_sun} SUN ({entry.available:,} available)")
```

Returns `list[PriceHistoryEntry]`. Available periods: `"1h"`, `"6h"`, `"24h"`, `"7d"`, `"30d"`.

### prices.stats()

Returns aggregate market statistics.

```python
stats = client.prices.stats()
print(f"Best price: {stats.best_price_sun} SUN")
print(f"Average price: {stats.avg_price_sun} SUN")
print(f"Providers online: {stats.total_providers}")
print(f"Cheapest-provider changes (24h): {stats.cheapest_changes_24h}")
```

Returns `PriceStats`.

### prices.preview(resource, amount, duration, max_price_sun=None)

Previews what an order would cost before placing it.

```python
preview = client.prices.preview(
    resource="ENERGY",
    amount=100000,
    duration=86400,       # 24 hours
    max_price_sun=35,     # Optional ceiling
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

Returns `OrderPreview` with a `best` match (or `None`), a list of `fallbacks`, and a `no_providers` boolean.

## Module 2: Orders

### orders.create(...)

Creates an energy or bandwidth order.

```python
order = client.orders.create(
    resource_type="ENERGY",
    amount=65000,
    target_address="TYourTargetAddress",
    duration_sec=3600,
    order_type="MARKET",           # Default
    max_price_sun=None,            # Required for LIMIT orders
    idempotency_key="unique-id",   # Prevents duplicate orders
)

print(f"Order {order.id}: {order.status}")
```

Returns `Order`. Note that unlike the JavaScript SDK where parameters are passed as a dictionary, the Python SDK uses explicit keyword arguments for clarity.

The `idempotency_key` parameter is critical for production systems. If a network error causes a retry, the API returns the original order instead of creating a duplicate.

### orders.list(limit=30, offset=0, status=None)

Lists orders with pagination.

```python
orders, total = client.orders.list(limit=10, offset=0, status="FILLED")
print(f"{total} filled orders total")

for o in orders:
    cost_trx = (o.total_cost_sun or 0) / 1_000_000
    print(f"  {o.id}: {o.amount:,} {o.resource_type}, {cost_trx:.3f} TRX")
```

Returns `tuple[list[Order], int]` - the list of orders and the total count for pagination.

### orders.get(order_id)

Returns a single order with its fill breakdown.

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

Returns `OrderWithFills`, which extends `Order` with a `fills` list of `Fill` objects.

## Module 3: Balance

### balance.get()

Returns current account balances.

```python
bal = client.balance.get()
print(f"TRX: {bal.trx}")
print(f"USDT: {bal.usdt}")
print(f"Locked: {bal.trx_locked}")
```

Returns `Balance`. The `trx_locked` field shows TRX reserved for pending orders.

### balance.deposit_info()

Returns the deposit address and memo for funding your account.

```python
info = client.balance.deposit_info()
print(f"Send TRX to: {info.address}")
print(f"Memo: {info.memo}")
print(f"Minimum: {info.min_amount_trx} TRX / {info.min_amount_usdt} USDT")
```

Returns `DepositInfo`. Always include the memo in your deposit transaction for automatic crediting.

### balance.withdraw(address, amount, currency="TRX", idempotency_key=None)

Withdraws funds to an external TRON address.

```python
withdrawal = client.balance.withdraw(
    address="TExternalAddress",
    amount=100,
    currency="TRX",
    idempotency_key="withdraw-unique-id",
)
print(f"Withdrawal {withdrawal.id}: {withdrawal.status}")
```

Returns `Withdrawal`.

### balance.history(period="30D") and balance.summary()

```python
history = client.balance.history("7D")
print(f"{len(history)} transactions in last 7 days")

summary = client.balance.summary()
print(f"Total orders: {summary.total_orders}")
print(f"Total energy: {summary.total_energy:,}")
print(f"Average price: {summary.avg_price_sun} SUN")
print(f"Total spent: {summary.total_spent_sun / 1_000_000:.3f} TRX")
```

## Module 4: Webhooks

### webhooks.create(url, events)

Creates a webhook subscription.

```python
webhook = client.webhooks.create(
    url="https://your-server.com/merx-webhook",
    events=["order.filled", "order.failed", "deposit.received"],
)
print(f"Webhook ID: {webhook.id}")
print(f"Secret: {webhook.secret}")  # Shown only once
```

Returns `Webhook`. The `secret` field is only included in the creation response and is used for HMAC-SHA256 signature verification.

Available event types: `order.filled`, `order.failed`, `deposit.received`, `withdrawal.completed`.

### webhooks.list() and webhooks.delete(webhook_id)

```python
webhooks = client.webhooks.list()
active = [w for w in webhooks if w.is_active]
print(f"{len(active)} active webhooks")

deleted = client.webhooks.delete("wh_abc123")
print(f"Deleted: {deleted}")  # True or False
```

## Error Handling

All API errors raise `MerxError` with a `code` attribute and a human-readable message.

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

The `MerxError` class extends `Exception`, so it integrates naturally with Python exception handling. The `code` attribute provides machine-readable classification:

| Code                   | Meaning                                          |
|------------------------|--------------------------------------------------|
| `UNAUTHORIZED`         | Invalid or missing API key                       |
| `INSUFFICIENT_FUNDS`   | Account balance too low                          |
| `INVALID_ADDRESS`      | Not a valid TRON address                         |
| `ORDER_NOT_FOUND`      | Order ID does not exist                          |
| `DUPLICATE_REQUEST`    | Idempotency key already used                     |
| `RATE_LIMITED`         | Too many requests, back off                      |
| `PROVIDER_UNAVAILABLE` | No providers can fulfill the request             |
| `VALIDATION_ERROR`     | Request body or parameters failed validation     |

## Comparison with the JavaScript SDK

Both SDKs cover the same four modules with the same methods. The key differences are in language idioms:

| Aspect              | Python SDK                       | JavaScript SDK                     |
|---------------------|----------------------------------|------------------------------------|
| HTTP client         | `urllib.request` (stdlib)        | Native `fetch`                     |
| Dependencies        | Zero                             | Zero                               |
| Async support       | No (synchronous only)            | Yes (all methods return Promises)  |
| Types               | Dataclasses                      | TypeScript interfaces              |
| Order creation      | Keyword arguments                | Object parameter                   |
| Orders list return  | `tuple[list, int]`               | `{ orders: [], total: number }`    |
| Webhook delete      | Returns `bool`                   | Returns `{ deleted: boolean }`     |
| Min runtime         | Python 3.10+                     | Node.js 18+                        |
| Module system       | Standard imports                 | ESM only                           |

The two SDKs are designed to feel natural in their respective languages while maintaining functional parity. A team using both Python and JavaScript can expect identical behavior from each.

## Production Pattern: Price-Aware Order Placement

Here is a complete production flow that checks current prices, validates the cost, and creates an order with idempotency:

```python
import uuid
from merx import MerxClient, MerxError

client = MerxClient(api_key="sk_live_your_key_here")

def buy_energy(target: str, amount: int = 65000, max_cost_trx: float = 5.0):
    """Buy energy with cost validation and idempotency."""

    # Preview the cost first
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

    # Place the order
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

## Installation Verification

After installing, verify the SDK is working:

```python
from merx import MerxClient, __version__

print(f"merx-sdk version: {__version__}")

# Public endpoint, no auth required
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

The `prices.list()` endpoint is public, so even a placeholder API key will work for testing connectivity.

## Resources

- Platform: [merx.exchange](https://merx.exchange)
- Documentation: [merx.exchange/docs](https://merx.exchange/docs)
- PyPI: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- MCP Server for AI agents: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp) | [npm](https://www.npmjs.com/package/merx-mcp)
