# MERX Python SDK: Sıfır Bağımlılık, Tam Güç

MERX Python SDK, yalnızca Python standart kütüphanesini kullanarak MERX TRON enerji borsasına tam bir arayüz sağlar - ne requests, ne httpx, ne de aiohttp. Bu makale dört SDK modülünü (prices, orders, balance, webhooks), çalışan örnekleriyle her metodu, dataclass tip sistemini, hata işlemesini ve her iki dili kullanan takımlar için JavaScript SDK ile karşılaştırmayı kapsar.

## Neden Sıfır Bağımlılık

Çoğu Python HTTP istemcisi bir bağımlılık ağacını beraberinde getirir. Tek başına `requests` kütüphanesi `urllib3`, `certifi`, `charset-normalizer` ve `idna`'yı içerir. Yalnızca birkaç API çağrısı yapan hafif bir SDK için bu ek yük gereksizdir.

MERX Python SDK yalnızca standart kütüphanedeki `urllib.request`'i kullanır. Bu şu anlama gelir:

- Sanal ortamınızda bağımlılık çatışması yok
- Üçüncü taraf paketlerden kaynaklanan tedarik zinciri saldırı yüzeyi yok
- Herhangi bir Python 3.10+ kurulumunda `pip install` ek yükü olmadan çalışır
- Paket kurulumunun zor olduğu kısıtlı ortamlara dağıtılabilir

Takas, eşzamanlı (async) desteğinin olmamasıdır. SDK eşzamanlı HTTP çağrıları kullanır. Eşzamanlı desteğe ihtiyacınız varsa, REST API bu makalede gösterilen desenler kullanılarak `aiohttp` veya `httpx` ile doğrudan çağrı yapacak kadar basittir.

## Kurulum

```bash
pip install merx-sdk
```

Hepsi bu. Yanına hiçbir bağımlılık kurulmayacak.

## Hızlı Başlangıç

```python
from merx import MerxClient

client = MerxClient(api_key="sk_live_your_key_here")

# Güncel fiyatları kontrol et
prices = client.prices.list()
for p in prices:
    if p.energy_prices:
        print(f"{p.provider}: {p.energy_prices[0].price_sun} SUN/energy")

# Enerji satın al
order = client.orders.create(
    resource_type="ENERGY",
    amount=65000,
    target_address="TYourTargetAddressHere",
    duration_sec=3600,
)
print(f"Order {order.id}: {order.status}")
```

İstemci, [merx.exchange](https://merx.exchange) adresindeki MERX panosundan alınan API anahtarınızla başlatılır. İsteğe bağlı `base_url` parametresi varsayılan olarak `https://merx.exchange` değerine ayarlanır.

```python
client = MerxClient(
    api_key="sk_live_your_key_here",
    base_url="https://merx.exchange",  # İsteğe bağlı
)
```

## Tip Sistemi

Her API yanıtı bir Python dataclass'ına ayrıştırılır. Ham sözlük yok, türsüz veri yok. IDE'niz tam otomatik tamamlama alır ve tür denetleyiciniz hataları çalışma zamanından önce yakalar.

SDK 16 dataclass türü dışa aktarır:

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

### Temel Tip Tanımları

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
    resource_type: str       # "ENERGY" veya "BANDWIDTH"
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

Tüm dataclass'lar standart Python tip ipuçlarını kullanır. Varsayılan değeri olan alanlar, her yanıtta mevcut olmayabilecek isteğe bağlı API yanıt alanlarıdır.

## Modül 1: Fiyatlar

Fiyatlar modülü gerçek zamanlı ve geçmiş fiyat verileri sorgulamak için beş yöntem sağlar. Tüm fiyat uç noktaları herkese açıktır ve kimlik doğrulaması gerektirmez, ancak SDK tutarlılık için her zaman API anahtarı başlığını gönderir.

### prices.list()

Tüm aktif sağlayıcılardan güncel fiyatlandırmayı döndürür.

```python
prices = client.prices.list()

for p in prices:
    print(f"\n{p.provider} (market={p.is_market}):")
    print(f"  Available energy: {p.available_energy:,}")
    for tier in p.energy_prices:
        hours = tier.duration_sec // 3600
        print(f"  {hours}h: {tier.price_sun} SUN/unit")
```

`list[ProviderPrice]` döndürür. Her giriş sağlayıcı adını, esnek süreli pazar siparişlerini destekleyip desteklemediğini, enerji ve bant genişliği için fiyat katmanlarını ve güncel kullanılabilir kapasiteyi içerir.

### prices.best(resource, amount=None)

Bir kaynak türü için en ucuz kullanılabilir fiyatı döndürür.

```python
best = client.prices.best("ENERGY")
print(f"Cheapest: {best.price_sun} SUN from {best.provider}")

# En az 200.000 kullanılabilir olanlar
best_large = client.prices.best("ENERGY", amount=200000)
```

`PriceHistoryEntry` döndürür. `amount` parametresi yeterli kapasiteye sahip olmayan sağlayıcıları filtreler.

### prices.history(provider=None, resource=None, period="24h")

Grafik oluşturma ve analiz için geçmiş fiyat anlık görüntülerini döndürür.

```python
history = client.prices.history(
    provider="sohu",
    resource="ENERGY",
    period="7d",
)

for entry in history:
    print(f"{entry.polled_at}: {entry.price_sun} SUN ({entry.available:,} available)")
```

`list[PriceHistoryEntry]` döndürür. Kullanılabilir dönemler: `"1h"`, `"6h"`, `"24h"`, `"7d"`, `"30d"`.

### prices.stats()

Toplam pazar istatistiklerini döndürür.

```python
stats = client.prices.stats()
print(f"Best price: {stats.best_price_sun} SUN")
print(f"Average price: {stats.avg_price_sun} SUN")
print(f"Providers online: {stats.total_providers}")
print(f"Cheapest-provider changes (24h): {stats.cheapest_changes_24h}")
```

`PriceStats` döndürür.

### prices.preview(resource, amount, duration, max_price_sun=None)

Bir sipariş yerleştirmeden önce ne kadar tutacağını gösterir.

```python
preview = client.prices.preview(
    resource="ENERGY",
    amount=100000,
    duration=86400,       # 24 saat
    max_price_sun=35,     # İsteğe bağlı üst sınır
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

`best` eşleşmesi (veya `None`), `fallbacks` listesi ve `no_providers` boole değeri ile `OrderPreview` döndürür.

## Modül 2: Siparişler

### orders.create(...)

Bir enerji veya bant genişliği siparişi oluşturur.

```python
order = client.orders.create(
    resource_type="ENERGY",
    amount=65000,
    target_address="TYourTargetAddress",
    duration_sec=3600,
    order_type="MARKET",           # Varsayılan
    max_price_sun=None,            # LIMIT siparişleri için gerekli
    idempotency_key="unique-id",   # Yinelenen siparişleri engeller
)

print(f"Order {order.id}: {order.status}")
```

`Order` döndürür. JavaScript SDK'sında parametrelerin sözlük olarak geçirildiği aksine, Python SDK netlik için açık anahtar sözcük bağımsız değişkenleri kullanır.

`idempotency_key` parametresi üretim sistemleri için kritiktir. Bir ağ hatası yeniden denemesine neden olursa, API yinelenen bir sipariş oluşturmak yerine orijinal siparişi döndürür.

### orders.list(limit=30, offset=0, status=None)

Sayfalandırmayla siparişleri listeler.

```python
orders, total = client.orders.list(limit=10, offset=0, status="FILLED")
print(f"{total} filled orders total")

for o in orders:
    cost_trx = (o.total_cost_sun or 0) / 1_000_000
    print(f"  {o.id}: {o.amount:,} {o.resource_type}, {cost_trx:.3f} TRX")
```

`tuple[list[Order], int]` döndürür - siparişlerin listesi ve sayfalandırma için toplam sayı.

### orders.get(order_id)

Doldurma dökümü ile tek bir siparişi döndürür.

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

`OrderWithFills` döndürür, bu da `Order`'ı `Fill` nesnelerinin `fills` listesiyle genişletir.

## Modül 3: Bakiye

### balance.get()

Güncel hesap bakiyelerini döndürür.

```python
bal = client.balance.get()
print(f"TRX: {bal.trx}")
print(f"USDT: {bal.usdt}")
print(f"Locked: {bal.trx_locked}")
```

`Balance` döndürür. `trx_locked` alanı bekleyen siparişler için ayrılan TRX'i gösterir.

### balance.deposit_info()

Hesabınızı finanse etmek için depozito adresini ve memo'yu döndürür.

```python
info = client.balance.deposit_info()
print(f"Send TRX to: {info.address}")
print(f"Memo: {info.memo}")
print(f"Minimum: {info.min_amount_trx} TRX / {info.min_amount_usdt} USDT")
```

`DepositInfo` döndürür. Otomatik kredi için her zaman depozito işleminize memo ekleyin.

### balance.withdraw(address, amount, currency="TRX", idempotency_key=None)

Fonları harici bir TRON adresine çeker.

```python
withdrawal = client.balance.withdraw(
    address="TExternalAddress",
    amount=100,
    currency="TRX",
    idempotency_key="withdraw-unique-id",
)
print(f"Withdrawal {withdrawal.id}: {withdrawal.status}")
```

`Withdrawal` döndürür.

### balance.history(period="30D") ve balance.summary()

```python
history = client.balance.history("7D")
print(f"{len(history)} transactions in last 7 days")

summary = client.balance.summary()
print(f"Total orders: {summary.total_orders}")
print(f"Total energy: {summary.total_energy:,}")
print(f"Average price: {summary.avg_price_sun} SUN")
print(f"Total spent: {summary.total_spent_sun / 1_000_000:.3f} TRX")
```

## Modül 4: Web Kancaları

### webhooks.create(url, events)

Bir webhook aboneliği oluşturur.

```python
webhook = client.webhooks.create(
    url="https://your-server.com/merx-webhook",
    events=["order.filled", "order.failed", "deposit.received"],
)
print(f"Webhook ID: {webhook.id}")
print(f"Secret: {webhook.secret}")  # Yalnızca bir kez gösterilir
```

`Webhook` döndürür. `secret` alanı yalnızca oluşturma yanıtına dahil edilir ve HMAC-SHA256 imza doğrulaması için kullanılır.

Kullanılabilir olay türleri: `order.filled`, `order.failed`, `deposit.received`, `withdrawal.completed`.

### webhooks.list() ve webhooks.delete(webhook_id)

```python
webhooks = client.webhooks.list()
active = [w for w in webhooks if w.is_active]
print(f"{len(active)} active webhooks")

deleted = client.webhooks.delete("wh_abc123")
print(f"Deleted: {deleted}")  # True veya False
```

## Hata İşlemesi

Tüm API hataları `code` özniteliği ve insanlar tarafından okunabilir bir mesajla `MerxError` harekete geçirir.

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

`MerxError` sınıfı `Exception`'ı genişletir, böylece Python istisnai işlemesiyle doğal olarak bütünleşir. `code` özniteliği makine tarafından okunabilir sınıflandırma sağlar:

| Kod                    | Anlamı                                           |
|------------------------|--------------------------------------------------|
| `UNAUTHORIZED`         | Geçersiz veya eksik API anahtarı                |
| `INSUFFICIENT_FUNDS`   | Hesap bakiyesi çok düşük                        |
| `INVALID_ADDRESS`      | Geçerli bir TRON adresi değil                   |
| `ORDER_NOT_FOUND`      | Sipariş kimliği mevcut değil                    |
| `DUPLICATE_REQUEST`    | Eşzamanlılık anahtarı zaten kullanılmış         |
| `RATE_LIMITED`         | Çok fazla istek, geri çekin                     |
| `PROVIDER_UNAVAILABLE` | Hiçbir sağlayıcı isteği yerine getiremez       |
| `VALIDATION_ERROR`     | İstek gövdesi veya parametreler doğrulama başarısız|

## JavaScript SDK ile Karşılaştırma

Her iki SDK da aynı dört modülü aynı yöntemlerle kapsar. Ana farklar dil deyişleridir:

| Yön                 | Python SDK                       | JavaScript SDK                     |
|---------------------|----------------------------------|------------------------------------|
| HTTP istemcisi      | `urllib.request` (stdlib)        | Yerli `fetch`                      |
| Bağımlılıklar       | Sıfır                            | Sıfır                              |
| Eşzamanlı destek    | Hayır (yalnızca eşzamanlı)       | Evet (tüm yöntemler Promise döndürür)|
| Türler              | Dataclass'lar                    | TypeScript arabirimleri            |
| Sipariş oluşturma   | Anahtar sözcük bağımsız değişkenleri| Nesne parametresi                 |
| Siparişlerin listesi| `tuple[list, int]`               | `{ orders: [], total: number }`    |
| Webhook silme       | `bool` döndürür                  | `{ deleted: boolean }` döndürür    |
| En az çalışma zamanı| Python 3.10+                     | Node.js 18+                        |
| Modül sistemi       | Standart içe aktarmalar          | Yalnızca ESM                       |

İki SDK, her bir dilde doğal hissetmek için tasarlanmıştır ve işlevsel parite sağlarken. Hem Python hem de JavaScript kullanan bir takım, her birinden aynı davranışı bekleyebilir.

## Üretim Deseni: Fiyat Farkında Sipariş Yerleştirme

Güncel fiyatları kontrol eden, maliyeti doğrulayan ve idempotentlik ile bir sipariş oluşturan tam bir üretim akışı:

```python
import uuid
from merx import MerxClient, MerxError

client = MerxClient(api_key="sk_live_your_key_here")

def buy_energy(target: str, amount: int = 65000, max_cost_trx: float = 5.0):
    """Maliyet doğrulaması ve idempotentlik ile enerji satın al."""

    # Önce maliyeti önizle
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

    # Siparişi yerleştir
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

## Kurulum Doğrulaması

Kurulduktan sonra SDK'nın çalıştığını doğrulayın:

```python
from merx import MerxClient, __version__

print(f"merx-sdk version: {__version__}")

# Genel uç nokta, kimlik doğrulaması gerekmez
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

`prices.list()` uç noktası herkese açıktır, bu nedenle bağlantı testleri için yer tutucu bir API anahtarı bile işe yarayacak.

## Kaynaklar

- Platform: [merx.exchange](https://merx.exchange)
- Belgelendirme: [merx.exchange/docs](https://merx.exchange/docs)
- PyPI: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- MCP Server for AI agents: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp) | [npm](https://www.npmjs.com/package/merx-mcp)

## Yapay Zeka ile Hemen Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemcisine ekleyin -- sıfır kurulum, salt okunur araçlar için API anahtarı gerekmez:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Yapay zeka aracınızdan şu soruyu sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgelendirmesi: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)