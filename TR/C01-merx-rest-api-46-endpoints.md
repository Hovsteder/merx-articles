# MERX REST API: 46 Uç Nokta TRON Enerji Ticareti için

MERX REST API, TRON enerji ve bandwidth ticaretinin tam yaşam döngüsünü kapsayan 46 uç noktası sağlar - sekiz sağlayıcı arasında gerçek zamanlı fiyat keşfinden sipariş yürütme, hesap yönetimi, zincir üstü sorguları ve otomatik kalıcı siparişlere kadar. Bu makale API mimarisini, kimlik doğrulama modelini, uç nokta gruplarını, hız sınırlamalarını, hata yönetimini ve curl, JavaScript ve Python'da pratik kod örneklerini ele almaktadır.

## Birleştirilmiş bir API'nin Neden Önemli Olduğu

TRON enerji piyasası, her birinin kendi API formatı, kimlik doğrulama şeması ve fiyatlandırma modeline sahip birden fazla sağlayıcı arasında parçalanmıştır. Bir USDT transferinde en iyi fiyatı istenen bir geliştirici, her sağlayıcı ile ayrı ayrı entegrasyon yapmak, yedek mantığı işlemek ve fiyatları sürekli izlemek zorundadır.

MERX tüm bunları tek bir REST API'de birleştiriyor. Bir API anahtarı, bir kimlik doğrulama başlığı, bir hata biçimi, bir SDK seti. Platform tüm bağlı sağlayıcıları her 30 saniyede bir yoklar, siparişleri en ucuz kullanılabilir kaynağa yönlendirir ve delegasyonları zincir üstünde doğrular.

API, `/api/v1/` adresinde sürümlendirilmiş ve geriye dönük uyumluluğu koruyacaktır. Tüm uç noktalar tutarlı bir zarf biçiminde JSON döndürür.

## Kimlik Doğrulama

MERX, API anahtarı kimlik doğrulaması kullanır. Her kimliği doğrulanmış istek, `X-API-Key` başlığını içermelidir.

API anahtarları, MERX panosunda [merx.exchange](https://merx.exchange) adresinde veya `/api/v1/keys` uç noktası aracılığıyla programlı olarak oluşturulur. Her anahtarın, hangi işlemleri gerçekleştirebileceğini kontrol eden bir izin seti vardır.

```bash
curl -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/balance
```

Anahtarlar `sk_live_` formatını takip eder ve bunu 64 onaltılı karakter izler. Ham anahtar, oluşturma sırasında tam olarak bir kez gösterilir. MERX yalnızca bcrypt karmasını depolar, bu nedenle kaybolan anahtarlar kurtarılamaz - bunları iptal etmek ve değiştirmek gerekir.

### Anahtar İzinleri

Bir API anahtarı oluştururken, bir veya daha fazla izin atarsınız:

| İzin             | Erişim sağlar                    |
|------------------|----------------------------------|
| `create_orders`  | POST /orders, POST /ensure       |
| `view_orders`    | GET /orders, GET /orders/:id     |
| `view_balance`   | GET /balance, GET /history       |
| `broadcast`      | POST /chain/broadcast            |

Bu, izleme panolarında yalnızca okunabilir anahtarlar ve otomatik ticaret sistemleri için kısıtlı anahtarlar oluşturmanıza olanak sağlar.

## Yanıt Zarfı

Her yanıt aynı yapıyı izler:

```json
{
  "data": { ... }
}
```

Hata durumunda:

```json
{
  "error": {
    "code": "INSUFFICIENT_FUNDS",
    "message": "Account balance too low for the requested operation",
    "details": { "required": 150000000, "available": 42000000 }
  }
}
```

Hata kodları makine tarafından okunabilir dizelerdir. `details` alanı isteğe bağlı olup hata ayıklama için bağlam sağlar.

## Uç Nokta Grupları

46 uç noktası dokuz gruba ayrılmıştır. İşte tam harita.

### Fiyatlar (6 Uç Nokta)

Bu uç noktalar herkese açıktır - API anahtarı gerekmez.

| Method | Yol                         | Açıklama                                 |
|--------|-----------------------------|-----------------------------------------|
| GET    | /api/v1/prices              | Tüm sağlayıcılardan güncel fiyatlar     |
| GET    | /api/v1/prices/best         | Kaynak türü için en ucuz sağlayıcı      |
| GET    | /api/v1/prices/history      | Geçmiş fiyat verileri                   |
| GET    | /api/v1/prices/stats        | Toplam pazar istatistikleri             |
| GET    | /api/v1/prices/analysis     | Trend analizi ve satın alma tavsiyesi   |
| GET    | /api/v1/orders/preview      | Sipariş vermeden önce maliyet önizlemesi|

### Siparişler (3 Uç Nokta)

| Method | Yol                      | Açıklama                                 |
|--------|--------------------------|------------------------------------------|
| POST   | /api/v1/orders           | Yeni bir enerji veya bandwidth siparişi |
| GET    | /api/v1/orders           | Sayfalama ve filtrelerle siparişleri listele |
| GET    | /api/v1/orders/:id       | Doldurma dökümü ile sipariş detayları   |

### Hesap (7 Uç Nokta)

| Method | Yol                        | Açıklama                               |
|--------|----------------------------|----------------------------------------|
| GET    | /api/v1/balance            | Güncel TRX ve USDT bakiyeleri          |
| GET    | /api/v1/deposit/info       | Yatırım adresi ve memo                 |
| POST   | /api/v1/deposit/prepare    | Yatırım işlemini hazırla               |
| POST   | /api/v1/deposit/submit     | Yatırım kanıtı gönder                  |
| POST   | /api/v1/withdraw           | TRX veya USDT çek                      |
| GET    | /api/v1/history            | Sipariş yürütme geçmişi                |
| GET    | /api/v1/history/summary    | Toplam hesap istatistikleri            |

### API Anahtarları (3 Uç Nokta)

| Method | Yol              | Açıklama                   |
|--------|------------------|----------------------------|
| GET    | /api/v1/keys     | Tüm API anahtarlarını listele |
| POST   | /api/v1/keys     | Yeni API anahtarı oluştur  |
| DELETE | /api/v1/keys/:id | Bir API anahtarını iptal et |

### Kimlik Doğrulama (2 Uç Nokta)

| Method | Yol                   | Açıklama                          |
|--------|----------------------|-----------------------------------|
| POST   | /api/v1/auth/register | Yeni hesap oluştur                |
| POST   | /api/v1/auth/login    | Kimlik doğrulaması yap ve JWT al  |

### Tahmin (2 Uç Nokta)

| Method | Yol           | Açıklama                              |
|--------|---------------|---------------------------------------|
| POST   | /api/v1/estimate | İşlem için enerji ve maliyeti tahmin et |
| POST   | /api/v1/ensure   | Bir adreste minimum kaynakları sağla  |

### Web Kancaları (3 Uç Nokta)

| Method | Yol              | Açıklama                       |
|--------|------------------|--------------------------------|
| POST   | /api/v1/webhooks | Web kancası aboneliği oluştur  |
| GET    | /api/v1/webhooks | Web kancası aboneliklerini listele |
| DELETE | /api/v1/webhooks/:id | Web kancasını sil           |

### Kalıcı Siparişler ve İzleyiciler (7 Uç Nokta)

| Method | Yol                              | Açıklama                      |
|--------|----------------------------------|-------------------------------|
| POST   | /api/v1/standing-orders          | Kalıcı sipariş oluştur        |
| GET    | /api/v1/standing-orders          | Kalıcı siparişleri listele    |
| GET    | /api/v1/standing-orders/:id      | Kalıcı sipariş detaylarını al |
| DELETE | /api/v1/standing-orders/:id      | Kalıcı siparişi iptal et      |
| POST   | /api/v1/monitors                 | Kaynak izleyicisi oluştur     |
| GET    | /api/v1/monitors                 | Aktif izleyicileri listele    |
| DELETE | /api/v1/monitors/:id             | İzleyiciyi iptal et           |

### Zincir Vekili (10 Uç Nokta)

Bu uç noktalar TRON ağı sorgularını MERX üzerinden vekil eder ve istemcilerin TronGrid'i doğrudan aramasına gerek kalmaz.

| Method | Yol                              | Açıklama                      |
|--------|----------------------------------|-------------------------------|
| GET    | /api/v1/chain/account/:address   | Hesap bilgisi ve kaynakları   |
| GET    | /api/v1/chain/balance/:address   | TRX bakiyesi                  |
| GET    | /api/v1/chain/resources/:address | Energy ve bandwidth dökümü    |
| GET    | /api/v1/chain/transaction/:txid  | İşlem detayları               |
| GET    | /api/v1/chain/block/:number      | Blok numarası veya son blok   |
| GET    | /api/v1/chain/parameters         | Zincir parametreleri          |
| GET    | /api/v1/chain/history/:address   | Adres işlem geçmişi           |
| POST   | /api/v1/chain/read-contract      | Sabit kontrat işlevini çağır  |
| POST   | /api/v1/chain/broadcast          | İmzalı işlemi yayınla         |
| GET    | /api/v1/address/:addr/resources  | Adres kaynakları özeti        |

### x402 Kullanım Başına Ödeme (3 Uç Nokta)

| Method | Yol                     | Açıklama                   |
|--------|-------------------------|----------------------------|
| POST   | /api/v1/x402/invoice    | Ödeme faturası oluştur     |
| GET    | /api/v1/x402/invoice/:id | Fatura durumunu kontrol et |
| POST   | /api/v1/x402/verify     | Ödemeyi doğrula ve yürüt   |

## Temel Uç Noktalar Detaylı

### GET /api/v1/prices

Tüm etkin sağlayıcılardan güncel fiyatlandırma döndürür. Kimlik doğrulama gerekmez. Piyasanın tamamını bir bakışta görmek için çağırdığınız uç noktadır.

```bash
curl https://merx.exchange/api/v1/prices
```

Yanıt (kısaltılmış):

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

Her sağlayıcı girdisi fiyat seviyelerini (süreye göre), kullanılabilir kapasiteyi ve son başarılı yoklama zaman damgasını içerir.

### POST /api/v1/orders

Enerji veya bandwidth siparişi oluşturur. Platform siparişi mevcut sağlayıcılara karşı eşleştirir ve bunu yerine getirebilecek en ucuz sağlayıcıya yönlendirir.

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

Yanıt:

```json
{
  "data": {
    "id": "ord_a1b2c3d4",
    "status": "PENDING",
    "created_at": "2026-03-30T12:00:00.000Z"
  }
}
```

`Idempotency-Key` başlığı, aynı istek yeniden denenirse yinelenen siparişleri önler. Anahtar daha önce görülmüşse, API yeni bir tane oluşturmak yerine orijinal siparişi döndürür.

Sipariş türleri:
- `MARKET` - en iyi mevcut fiyattan hemen yürüt
- `LIMIT` - yalnızca fiyat `max_price_sun` veya daha azsa yürüt
- `PERIODIC` - bir zamanlamaya göre yinelenen sipariş
- `BROADCAST` - önceden imzalanmış bir delegasyon işlemini yayınla

### GET /api/v1/orders/:id

Doldurma detayları ile siparişi döndürür - hangi sağlayıcılar siparişi yerine getirdi, hangi fiyattan ve zincir üstü işlem kimlikleri.

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

TRON işlemi için gereken enerji ve bandwidth'i tahmin eder, daha sonra kiralama maliyetini TRX yakma maliyeti ile karşılaştırır.

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

Bu uç nokta, kullanıcılara MERX aracılığıyla enerji kiralayarak TRX yakmaya karşı tam olarak ne kadar tasarruf ettiklerini göstermek için yararlıdır.

## Hız Sınırlamaları

Hız sınırlamaları, kayan pencereler kullanan IP adresi başına uygulanır.

| Uç nokta grubu    | Sınır           | Pencere |
|-------------------|-----------------|--------|
| Fiyatlar (herkese açık) | 300 istek | 1 dak |
| Varsayılan (genel) | 100 istek | 1 dak |
| Bakiye            | 60 istek        | 1 dak  |
| Geçmiş            | 60 istek        | 1 dak  |
| Siparişler        | 10 istek        | 1 dak  |
| Çekilişler        | 5 istek         | 1 dak  |
| Yayınlama         | 20 istek        | 1 dak  |
| Kayıt             | 5 istek         | 1 saat |

Hız sınırlaması aşıldığında, API HTTP 429'u standart hata biçimi ile döndürür:

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded"
  }
}
```

Hız sınırı başlıkları (`RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`) IETF taslak standardını takip ederek tüm yanıtlara dahil edilir.

## Hata Kodları

| Kod                    | HTTP | Açıklama                                  |
|------------------------|------|-------------------------------------------|
| `UNAUTHORIZED`         | 401  | Geçersiz veya eksik API anahtarı          |
| `RATE_LIMITED`         | 429  | Çok fazla istek                           |
| `VALIDATION_ERROR`    | 400  | İstek gövdesi veya parametreleri doğrulama başarısız |
| `INVALID_ADDRESS`     | 400  | Geçerli olmayan TRON adresi               |
| `INSUFFICIENT_FUNDS`  | 400  | Hesap bakiyesi çok düşük                  |
| `BELOW_MINIMUM_ORDER` | 400  | Sipariş miktarı sağlayıcı minimumunun altında |
| `DUPLICATE_REQUEST`   | 409  | İdempotency anahtarı zaten kullanıldı     |
| `ORDER_NOT_FOUND`     | 404  | Sipariş veya kaynak bulunamadı            |
| `PROVIDER_UNAVAILABLE`| 404  | Hiçbir sağlayıcı isteği yerine getiremez  |
| `INTERNAL_ERROR`      | 500  | Sunucu tarafı hatası                      |

## SDK'larla Hızlı Başlangıç

REST API, herhangi bir HTTP istemcisi ile doğrudan çağrılabilse de, MERX kimlik doğrulama, hata ayrıştırma ve tip güvenliğini işleyen JavaScript ve Python için resmi SDK'lar sağlar.

### JavaScript / TypeScript

```bash
npm install merx-sdk
```

```javascript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({ apiKey: 'sk_live_your_key_here' })

// Tüm fiyatları al
const prices = await merx.prices.list()

// Sipariş oluştur
const order = await merx.orders.create({
  resource_type: 'ENERGY',
  amount: 65000,
  target_address: 'TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb',
  duration_sec: 3600,
})

// Sipariş durumunu kontrol et
const details = await merx.orders.get(order.id)
```

### Python

```bash
pip install merx-sdk
```

```python
from merx import MerxClient

client = MerxClient(api_key="sk_live_your_key_here")

# Tüm fiyatları al
prices = client.prices.list()

# Sipariş oluştur
order = client.orders.create(
    resource_type="ENERGY",
    amount=65000,
    target_address="TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    duration_sec=3600,
)

# Sipariş durumunu kontrol et
details = client.orders.get(order.id)
```

## Gerçek Zamanlı Veriler için WebSocket

REST API'ye ek olarak, MERX `wss://merx.exchange/ws` adresinde gerçek zamanlı fiyat güncellemeleri için bir WebSocket uç noktası sağlar. Fiyat değişiklikleri gerçekleştikçe bağlı istemcilere gönderilir ve güncellemeler her sağlayıcı için 30 saniyede bir gelir.

WebSocket bağlantısı sağlayıcı filtrelemesini destekler - yalnızca önemsediğiniz sağlayıcılara abone olun ve geri kalanını yok sayın.

## Kalıcı Siparişler

Kalıcı siparişler, tetikleyicilere dayalı enerji satın almalarını otomatikleştirir. Bir fiyat eşiği, bir zamanlama veya bir bakiye koşulu belirleyebilir ve platform belirtilen bütçe dahilinde siparişleri otomatik olarak yürütür.

Tetikleme türleri `price_below`, `price_above`, `schedule`, `balance_below` ve `provider_available` içerir. İşlem türleri `buy_resource`, `ensure_resources`, `deposit_trx` ve `notify_only` içerir.

Bu, MERX'i tam otomatik altyapı yönetimi için uygun hale getirir - kurallarınızı bir kez belirleyin ve platform yürütmeyi işleyin.

## Sırada Ne Var

MERX API, TRON ağ kaynaklarına güvenilir, uygun maliyetli erişim gereken geliştiriciler ve işletmeler için tasarlanmıştır. Bir ödeme işlemcisi, bir DeFi uygulaması veya bir exchange oluşturuyor olsanız da, API enerji ve bandwidth'i programlı olarak yönetmek için yapı taşları sağlar.

Tam API belgeleri [merx.exchange/docs](https://merx.exchange/docs) adresinde mevcuttur. JavaScript SDK [GitHub](https://github.com/Hovsteder/merx-sdk-js) ve [npm](https://www.npmjs.com/package/merx-sdk) üzerindedir. Python SDK [PyPI](https://pypi.org/project/merx-sdk/) üzerindedir.

---

**Bağlantılar:**
- Platform: [merx.exchange](https://merx.exchange)
- Belgelendirme: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- Python SDK: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- MCP Sunucusu: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp) | [npm](https://www.npmjs.com/package/merx-mcp)


## Yapay Zeka ile Hemen Deneyin

MERX'i Claude Desktop veya herhangi bir MCP uyumlu istemcisine ekleyin -- kurulum yok, yalnızca okunan araçlar için API anahtarı yok:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Yapay zeka ajanınıza sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)