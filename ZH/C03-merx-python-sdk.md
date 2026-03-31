# MERX Python SDK: 零依赖,全功能

MERX Python SDK仅使用Python标准库提供完整的MERX TRON能量交易接口 - 无需requests、httpx或aiohttp。本文涵盖四个SDK模块(prices、orders、balance、webhooks)、每个方法的可运行示例、数据类类型系统、错误处理,以及与JavaScript SDK的对比,方便同时使用两种语言的团队参考。

## 为什么选择零依赖

大多数Python HTTP客户端会引入依赖树。仅 `requests` 库就会带入 `urllib3`、`certifi`、`charset-normalizer` 和 `idna`。对于一个只需要少量API调用的轻量级SDK来说,这种开销完全没有必要。

MERX Python SDK仅使用标准库中的 `urllib.request`。这意味着:

- 虚拟环境中不会有依赖冲突
- 没有第三方包带来的供应链攻击面
- 在任何Python 3.10+环境中无需额外 `pip install` 即可运行
- 可部署到安装包受限的环境中

代价是缺少异步支持。SDK使用同步HTTP调用。如果您需要异步,REST API足够简单,可以按照本文展示的模式直接使用 `aiohttp` 或 `httpx` 调用。

## 安装

```bash
pip install merx-sdk
```

仅此而已。不会安装任何附加依赖。

## 快速入门

```python
from merx import MerxClient

client = MerxClient(api_key="sk_live_your_key_here")

# 查询当前价格
prices = client.prices.list()
for p in prices:
    if p.energy_prices:
        print(f"{p.provider}: {p.energy_prices[0].price_sun} SUN/energy")

# 购买能量
order = client.orders.create(
    resource_type="ENERGY",
    amount=65000,
    target_address="TYourTargetAddressHere",
    duration_sec=3600,
)
print(f"Order {order.id}: {order.status}")
```

客户端使用 [merx.exchange](https://merx.exchange) 的MERX控制面板中的API密钥初始化。可选的 `base_url` 参数默认为 `https://merx.exchange`。

```python
client = MerxClient(
    api_key="sk_live_your_key_here",
    base_url="https://merx.exchange",  # 可选
)
```

## 类型系统

每个API响应都被解析为Python数据类。没有原始字典,没有无类型数据。您的IDE可以提供完整的自动补全,类型检查器可以在运行前捕获错误。

SDK导出16个数据类类型:

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

### 核心类型定义

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
    resource_type: str       # "ENERGY" 或 "BANDWIDTH"
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

所有数据类使用标准Python类型提示。带有默认值的字段是可选的API响应字段,可能不会出现在每个响应中。

## 模块一: Prices

Prices模块提供五个方法用于查询实时和历史价格数据。所有价格接口均为公开接口,无需认证,但SDK始终发送API密钥头以保持一致性。

### prices.list()

返回所有活跃供应商的当前定价。

```python
prices = client.prices.list()

for p in prices:
    print(f"\n{p.provider} (market={p.is_market}):")
    print(f"  Available energy: {p.available_energy:,}")
    for tier in p.energy_prices:
        hours = tier.duration_sec // 3600
        print(f"  {hours}h: {tier.price_sun} SUN/unit")
```

返回 `list[ProviderPrice]`。每个条目包含供应商名称、是否支持灵活时长的市价订单、能量和带宽的价格层级,以及当前可用容量。

### prices.best(resource, amount=None)

返回指定资源类型的最低可用价格。

```python
best = client.prices.best("ENERGY")
print(f"Cheapest: {best.price_sun} SUN from {best.provider}")

# 仅显示至少有200,000可用量的供应商
best_large = client.prices.best("ENERGY", amount=200000)
```

返回 `PriceHistoryEntry`。`amount` 参数会过滤掉容量不足的供应商。

### prices.history(provider=None, resource=None, period="24h")

返回用于图表和分析的历史价格快照。

```python
history = client.prices.history(
    provider="sohu",
    resource="ENERGY",
    period="7d",
)

for entry in history:
    print(f"{entry.polled_at}: {entry.price_sun} SUN ({entry.available:,} available)")
```

返回 `list[PriceHistoryEntry]`。可用周期: `"1h"`、`"6h"`、`"24h"`、`"7d"`、`"30d"`。

### prices.stats()

返回市场汇总统计数据。

```python
stats = client.prices.stats()
print(f"Best price: {stats.best_price_sun} SUN")
print(f"Average price: {stats.avg_price_sun} SUN")
print(f"Providers online: {stats.total_providers}")
print(f"Cheapest-provider changes (24h): {stats.cheapest_changes_24h}")
```

返回 `PriceStats`。

### prices.preview(resource, amount, duration, max_price_sun=None)

在下单前预览订单成本。

```python
preview = client.prices.preview(
    resource="ENERGY",
    amount=100000,
    duration=86400,       # 24小时
    max_price_sun=35,     # 可选价格上限
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

返回 `OrderPreview`,包含 `best` 匹配(或 `None`)、`fallbacks` 列表和 `no_providers` 布尔值。

## 模块二: Orders

### orders.create(...)

创建能量或带宽订单。

```python
order = client.orders.create(
    resource_type="ENERGY",
    amount=65000,
    target_address="TYourTargetAddress",
    duration_sec=3600,
    order_type="MARKET",           # 默认值
    max_price_sun=None,            # LIMIT订单必填
    idempotency_key="unique-id",   # 防止重复订单
)

print(f"Order {order.id}: {order.status}")
```

返回 `Order`。请注意,与JavaScript SDK使用字典传递参数不同,Python SDK使用显式关键字参数以提高清晰度。

`idempotency_key` 参数对生产系统至关重要。如果网络错误导致重试,API将返回原始订单而非创建重复订单。

### orders.list(limit=30, offset=0, status=None)

带分页的订单列表。

```python
orders, total = client.orders.list(limit=10, offset=0, status="FILLED")
print(f"{total} filled orders total")

for o in orders:
    cost_trx = (o.total_cost_sun or 0) / 1_000_000
    print(f"  {o.id}: {o.amount:,} {o.resource_type}, {cost_trx:.3f} TRX")
```

返回 `tuple[list[Order], int]` - 订单列表和用于分页的总数。

### orders.get(order_id)

返回单个订单及其成交明细。

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

返回 `OrderWithFills`,它扩展了 `Order` 并包含 `Fill` 对象的 `fills` 列表。

## 模块三: Balance

### balance.get()

返回当前账户余额。

```python
bal = client.balance.get()
print(f"TRX: {bal.trx}")
print(f"USDT: {bal.usdt}")
print(f"Locked: {bal.trx_locked}")
```

返回 `Balance`。`trx_locked` 字段显示为待处理订单预留的TRX。

### balance.deposit_info()

返回为账户充值的充值地址和备注。

```python
info = client.balance.deposit_info()
print(f"Send TRX to: {info.address}")
print(f"Memo: {info.memo}")
print(f"Minimum: {info.min_amount_trx} TRX / {info.min_amount_usdt} USDT")
```

返回 `DepositInfo`。请始终在充值交易中包含备注以实现自动到账。

### balance.withdraw(address, amount, currency="TRX", idempotency_key=None)

将资金提现至外部TRON地址。

```python
withdrawal = client.balance.withdraw(
    address="TExternalAddress",
    amount=100,
    currency="TRX",
    idempotency_key="withdraw-unique-id",
)
print(f"Withdrawal {withdrawal.id}: {withdrawal.status}")
```

返回 `Withdrawal`。

### balance.history(period="30D") 和 balance.summary()

```python
history = client.balance.history("7D")
print(f"{len(history)} transactions in last 7 days")

summary = client.balance.summary()
print(f"Total orders: {summary.total_orders}")
print(f"Total energy: {summary.total_energy:,}")
print(f"Average price: {summary.avg_price_sun} SUN")
print(f"Total spent: {summary.total_spent_sun / 1_000_000:.3f} TRX")
```

## 模块四: Webhooks

### webhooks.create(url, events)

创建Webhook订阅。

```python
webhook = client.webhooks.create(
    url="https://your-server.com/merx-webhook",
    events=["order.filled", "order.failed", "deposit.received"],
)
print(f"Webhook ID: {webhook.id}")
print(f"Secret: {webhook.secret}")  # 仅显示一次
```

返回 `Webhook`。`secret` 字段仅在创建响应中包含,用于HMAC-SHA256签名验证。

可用事件类型: `order.filled`、`order.failed`、`deposit.received`、`withdrawal.completed`。

### webhooks.list() 和 webhooks.delete(webhook_id)

```python
webhooks = client.webhooks.list()
active = [w for w in webhooks if w.is_active]
print(f"{len(active)} active webhooks")

deleted = client.webhooks.delete("wh_abc123")
print(f"Deleted: {deleted}")  # True 或 False
```

## 错误处理

所有API错误都抛出 `MerxError`,包含 `code` 属性和人类可读的消息。

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
    # 输出: Error [INVALID_ADDRESS]: Target address is not a valid TRON address
```

`MerxError` 类继承自 `Exception`,因此可以自然地融入Python异常处理机制。`code` 属性提供机器可读的分类:

| 代码                   | 含义                                             |
|------------------------|--------------------------------------------------|
| `UNAUTHORIZED`         | 无效或缺少API密钥                               |
| `INSUFFICIENT_FUNDS`   | 账户余额不足                                     |
| `INVALID_ADDRESS`      | 不是有效的TRON地址                               |
| `ORDER_NOT_FOUND`      | 订单ID不存在                                     |
| `DUPLICATE_REQUEST`    | 幂等性密钥已被使用                               |
| `RATE_LIMITED`         | 请求过多,请稍后重试                             |
| `PROVIDER_UNAVAILABLE` | 没有供应商可以完成该请求                         |
| `VALIDATION_ERROR`     | 请求体或参数验证失败                             |

## 与JavaScript SDK的对比

两个SDK涵盖相同的四个模块和相同的方法。主要差异在于语言惯用法:

| 方面                | Python SDK                       | JavaScript SDK                     |
|---------------------|----------------------------------|------------------------------------|
| HTTP客户端          | `urllib.request`(标准库)        | 原生 `fetch`                       |
| 依赖                | 零                               | 零                                 |
| 异步支持            | 否(仅同步)                     | 是(所有方法返回Promise)          |
| 类型                | 数据类                           | TypeScript接口                     |
| 创建订单            | 关键字参数                       | 对象参数                           |
| 订单列表返回值      | `tuple[list, int]`               | `{ orders: [], total: number }`    |
| 删除Webhook         | 返回 `bool`                      | 返回 `{ deleted: boolean }`        |
| 最低运行时          | Python 3.10+                     | Node.js 18+                        |
| 模块系统            | 标准导入                         | 仅ESM                              |

两个SDK的设计理念是在各自的语言中提供自然的使用体验,同时保持功能一致。同时使用Python和JavaScript的团队可以期望每个SDK具有相同的行为。

## 生产模式: 价格感知的订单下达

以下是一个完整的生产流程,包含查询当前价格、验证成本,以及使用幂等性密钥创建订单:

```python
import uuid
from merx import MerxClient, MerxError

client = MerxClient(api_key="sk_live_your_key_here")

def buy_energy(target: str, amount: int = 65000, max_cost_trx: float = 5.0):
    """购买能量,带成本验证和幂等性保护。"""

    # 先预览成本
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

    # 下单
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

## 安装验证

安装后,验证SDK是否正常工作:

```python
from merx import MerxClient, __version__

print(f"merx-sdk version: {__version__}")

# 公开接口,无需认证
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

`prices.list()` 接口为公开接口,因此即使使用占位API密钥也可以测试连通性。

## 资源

- 平台: [merx.exchange](https://merx.exchange)
- 文档: [merx.exchange/docs](https://merx.exchange/docs)
- PyPI: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- MCP Server(AI代理专用): [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp) | [npm](https://www.npmjs.com/package/merx-mcp)
