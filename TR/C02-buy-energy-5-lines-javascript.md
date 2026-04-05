# TRON Enerjisini 5 Satır Kodla Satın Al: MERX JavaScript SDK

MERX JavaScript SDK, beş satır kodla TRON enerjisini programatik olarak satın almanızı sağlar - paketi kurun, istemciyi başlatın, fiyatları kontrol edin, bir sipariş oluşturun ve delegasyonu doğrulayın. Bu makale, kurulumdan üretim dağıtımına kadar tam SDK'yı kapsar; TypeScript türleri, hata işleme ve dört modül (prices, orders, balance ve webhooks) arasında mevcut tüm yöntemleri içerir.

## Doğrudan Sağlayıcı API'lerinin Sorunu

Her TRON energy sağlayıcısının kendi API'si vardır. Bir taneyle entegrasyon basittir. En iyi fiyatı garantilemek için yedi veya sekiz sağlayıcının tümüyle entegrasyon ise bir mühendislik projesidir. Farklı kimlik doğrulama şemaları, yanıt formatları, hata kodları, hız sınırları ve yedekleme mantığını işlemeniz gerekir.

MERX SDK, tüm bunları tutarlı bir arayüze sahip tek bir istemciye soyutlar. Arka planda, MERX platformu her 30 saniyede tüm sağlayıcıları yoklar, gerçek zamanlı bir fiyat indeksi tutar ve siparişlerinizi otomatik yedekleme ile en ucuz mevcut kaynağa yönlendirir.

## Kurulum

SDK, Node.js 18 veya daha yeni bir sürümü gerektirir. Sıfır çalışma zamanı bağımlılığı ile yerli `fetch` kullanır.

```bash
npm install merx-sdk
```

```bash
yarn add merx-sdk
```

```bash
pnpm add merx-sdk
```

Paket, tam TypeScript bildirimleri ve kaynak haritaları ile ESM olarak gönderilir.

## Enerji Satın Almak İçin Beş Satır

İşte minimal biçimdeki tam akış:

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

Satır satır:

1. İstemci sınıfını içe aktarın.
2. API anahtarınızla başlatın. Anahtar `sk_live_` ile başlar ve [merx.exchange](https://merx.exchange) adresindeki MERX panosunda oluşturulur.
3. Bağlı tüm sağlayıcılardan mevcut fiyatları getirin. Bu isteğe bağlıdır ancak siparişten önce pazar durumunu göstermek için yararlıdır.
4. 65.000 energy biriminin hedef adresinize bir saat süreyle devredilmesi için bir pazar siparişi oluşturun. Platform otomatik olarak en ucuz sağlayıcıyı seçer.
5. Sipariş ID'sini ve durumunu kaydedin. Sipariş `PENDING` durumunda başlar ve zincir üstü delegasyon doğrulandıktan sonra `FILLED` durumuna geçer.

Bu tüm entegrasyondur. Sağlayıcı seçimi yok, yedekleme mantığı yok, fiyat karşılaştırma kodu yok.

## İstemci Yapılandırması

```typescript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({
  apiKey: 'sk_live_your_key_here',   // Gerekli
  baseUrl: 'https://merx.exchange',  // İsteğe bağlı, varsayılan olarak üretim
})
```

`apiKey`, tek gerekli seçenektir. `baseUrl`, bir hazırlama ortamına karşı test etmek için geçersiz kılınabilir.

## Prices Modülü

`merx.prices` modülü, gerçek zamanlı pazar verilerini sorgulamak için beş yöntem sağlar.

### prices.list()

Tüm etkin sağlayıcılardan mevcut fiyatlandırmayı döndürür.

```typescript
const prices = await merx.prices.list()

for (const p of prices) {
  const cheapest = p.energy_prices[0]
  if (cheapest) {
    console.log(`${p.provider}: ${cheapest.price_sun} SUN/energy, ${p.available_energy} available`)
  }
}
```

Her `ProviderPrice` nesnesi şunları içerir:
- `provider` - sağlayıcı tanımlayıcısı (örneğin, "sohu", "catfee", "tronsave")
- `is_market` - bu sağlayıcının esnek sürelerle pazar siparişlerini destekleyip desteklemediği
- `energy_prices` - `{ duration_sec, price_sun }` katmanlarının dizisi
- `bandwidth_prices` - bandwidth için aynı yapı
- `available_energy` - energy birimlerinde mevcut kapasite
- `available_bandwidth` - bandwidth birimlerinde mevcut kapasite
- `fetched_at` - son başarılı anketin Unix zaman damgası

### prices.best(resource, amount?)

Verilen bir kaynak türü için tek en ucuz fiyat noktasını döndürür.

```typescript
const best = await merx.prices.best('ENERGY')
console.log(`Cheapest: ${best.price_sun} SUN from ${best.provider}`)

// Minimum miktar filtresi ile
const bestLarge = await merx.prices.best('ENERGY', 500000)
```

İsteğe bağlı `amount` parametresi, siparişinizi yerine getirmek için yeterli kapasitesi olmayan sağlayıcıları filtreler.

### prices.history(params?)

Analiz ve grafik oluşturma için tarihsel fiyat verilerini döndürür.

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

Mevcut dönemler: `'1h'`, `'6h'`, `'24h'`, `'7d'`, `'30d'`. Tüm filtre parametreleri isteğe bağlıdır.

### prices.stats()

Tüm pazar üzerinde toplam istatistikler döndürür.

```typescript
const stats = await merx.prices.stats()
console.log(`Best price: ${stats.best_price_sun} SUN`)
console.log(`Average: ${stats.avg_price_sun} SUN`)
console.log(`Providers online: ${stats.total_providers}`)
console.log(`Cheapest-provider changes (24h): ${stats.cheapest_changes_24h}`)
```

### prices.preview(params)

Bir siparişi yerleştirmeden önce ne kadar tutacağını önizler. En iyi eşleşen sağlayıcıyı ve yedek seçenekleri döndürür.

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

`max_price_sun` parametresi, fiyat tavanınızı aşan sağlayıcıları filtreler.

## Orders Modülü

### orders.create(params)

Bir energy veya bandwidth siparişi oluşturur.

```typescript
const order = await merx.orders.create({
  resource_type: 'ENERGY',
  amount: 65000,
  target_address: 'TYourTargetAddress',
  duration_sec: 3600,
  order_type: 'MARKET',          // İsteğe bağlı, varsayılan olarak 'MARKET'
  max_price_sun: 30,             // İsteğe bağlı, LIMIT siparişleri için gerekli
  idempotency_key: 'unique-id',  // İsteğe bağlı, yinelenen siparişleri engeller
})

console.log(`Order ${order.id} created`)
console.log(`Status: ${order.status}`)
```

Sipariş türleri:
- `MARKET` - mevcut en iyi fiyattan hemen yürütün
- `LIMIT` - yalnızca fiyat `max_price_sun` değerine eşitse veya altındaysa yürütün
- `PERIODIC` - yinelenen sipariş
- `BROADCAST` - önceden imzalanmış bir delegasyon işlemini yayınlayın

`idempotency_key`, üretim sistemleri için kritiktir. Bir ağ hatası oluşursa ve aynı anahtarla isteği yeniden denerseniz, API orijinal siparişi döndürür ve yinelenen bir sipariş oluşturmaz.

### orders.list(limit?, offset?, status?)

Sayfalama ile siparişleri listeler.

```typescript
const { orders, total } = await merx.orders.list(10, 0, 'FILLED')
console.log(`${total} filled orders total`)

for (const o of orders) {
  console.log(`${o.id}: ${o.amount} ${o.resource_type}, cost: ${o.total_cost_sun} SUN`)
}
```

### orders.get(id)

Doldurma ayrıntılarını içeren tek bir siparişi döndürür.

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

`fills` dizisi, siparişin tam olarak nasıl yerine getirildiğini gösterir. Her doldurma, sağlayıcı adını, ayrılan miktarı, birim fiyatı, maliyeti, zincir üstü işlem ID'sini ve delegasyonun zincir üstünde doğrulanıp doğrulanmadığını içerir.

## Balance Modülü

### balance.get()

Mevcut hesap bakiyelerini döndürür.

```typescript
const balance = await merx.balance.get()
console.log(`TRX: ${balance.trx}`)
console.log(`USDT: ${balance.usdt}`)
console.log(`Locked: ${balance.trx_locked}`)
```

`trx_locked` alanı, şu anda bekleyen siparişler için ayrılmış TRX'i gösterir.

### balance.depositInfo()

MERX hesabınıza para yatırma adresi ve notunu döndürür.

```typescript
const info = await merx.balance.depositInfo()
console.log(`Send TRX to: ${info.address}`)
console.log(`Include memo: ${info.memo}`)
console.log(`Minimum deposit: ${info.min_amount_trx} TRX`)
```

Not, otomatik para yatırma kredilendirilmesi için gereklidir. Doğru notlanmadan yapılan para yatırmaları manuel işleme gerektirir.

### balance.withdraw(params)

TRX veya USDT'yi harici bir TRON adresine çeker.

```typescript
const withdrawal = await merx.balance.withdraw({
  address: 'TYourExternalAddress',
  amount: 100,
  currency: 'TRX',
  idempotency_key: 'withdraw-unique-id',
})

console.log(`Withdrawal ${withdrawal.id}: ${withdrawal.status}`)
```

### balance.history(period?) ve balance.summary()

```typescript
const history = await merx.balance.history('7D')
console.log(`${history.length} transactions in last 7 days`)

const summary = await merx.balance.summary()
console.log(`Total orders: ${summary.total_orders}`)
console.log(`Total energy: ${summary.total_energy}`)
console.log(`Average price: ${summary.avg_price_sun} SUN`)
```

## Webhooks Modülü

### webhooks.create(params)

Bir webhook aboneliği oluşturur. `secret` yalnızca oluşturma sırasında döndürülür.

```typescript
const webhook = await merx.webhooks.create({
  url: 'https://your-server.com/merx-webhook',
  events: ['order.filled', 'order.failed', 'deposit.received'],
})

console.log(`Webhook ID: ${webhook.id}`)
console.log(`Secret: ${webhook.secret}`)
```

### webhooks.list() ve webhooks.delete(id)

```typescript
const webhooks = await merx.webhooks.list()
console.log(`${webhooks.filter(w => w.is_active).length} active webhooks`)

await merx.webhooks.delete('wh_abc123')
```

## Hata İşleme

Tüm API hataları, makine tarafından okunabilir bir `code` ve insan tarafından okunabilir bir `message` ile birlikte `MerxError` örnekleri olarak atılır.

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
    // Çıktı: [INVALID_ADDRESS]: Target address is not a valid TRON address
  }
}
```

Yaygın hata kodları:

| Kod                    | Anlamı                                           |
|------------------------|--------------------------------------------------|
| `UNAUTHORIZED`         | Geçersiz veya eksik API anahtarı                |
| `INSUFFICIENT_FUNDS`   | Hesap bakiyesi çok düşük                        |
| `INVALID_ADDRESS`      | Hedef adres geçerli bir TRON adresi değil       |
| `ORDER_NOT_FOUND`      | Sipariş ID'si mevcut değil                      |
| `INVALID_AMOUNT`       | Miktar minimum altında veya sınırları aşıyor    |
| `DUPLICATE_REQUEST`    | Eş güdümlülük anahtarı zaten kullanılmış        |
| `RATE_LIMITED`         | Çok fazla istek                                  |
| `PROVIDER_UNAVAILABLE` | Sağlayıcı mevcut değil                          |

## TypeScript Türleri

Tüm türler paketten dışa aktarılır ve uygulamanız genelinde tür ek açıklamaları için kullanılabilir:

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

SDK, kesin mod uyumludur ve hiçbir `any` türü üretmez. Her yanıt tam olarak yazılmıştır.

## Üretim Örneği: Otomatik USDT Transfer Akışı

Fiyatları kontrol eden, bir sipariş oluşturan ve tamamlanma için yoklayan tam bir üretim deseni şöyledir:

```typescript
import { MerxClient, MerxError } from 'merx-sdk'
import { randomUUID } from 'node:crypto'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY! })

async function ensureEnergyAndTransfer(targetAddress: string) {
  // Maliyeti önizle
  const preview = await merx.prices.preview({
    resource: 'ENERGY',
    amount: 65000,
    duration: 3600,
  })

  if (preview.no_providers) {
    throw new Error('No energy providers available')
  }

  console.log(`Best price: ${preview.best!.cost_trx} TRX from ${preview.best!.provider}`)

  // Eş güdümlülük ile siparişi oluştur
  const order = await merx.orders.create({
    resource_type: 'ENERGY',
    amount: 65000,
    target_address: targetAddress,
    duration_sec: 3600,
    idempotency_key: randomUUID(),
  })

  // Doldurulana kadar yokla (üretimde bunun yerine webhook kullan)
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

  // Energy artık hedef adres üzerinde mevcut
  // USDT transferine devam et
}
```

Üretimde, yoklama döngüsünü bir webhook dinleyici ile değiştirin. `order.filled` etkinliği için bir webhook aboneliği oluşturun ve sunucunuz delegasyon zincir üstünde doğrulandığı anda bilgilendirilecektir.

## Gereksinimler ve Uyumluluk

- Node.js 18+ (yerli `fetch` kullanır)
- Sıfır çalışma zamanı bağımlılığı
- Yalnızca ESM (ES modülleri olarak gönderilir)
- Tam TypeScript 5.4+ desteği
- Bun ve Deno ile çalışır (`fetch` destekleyen herhangi bir çalışma zamanı)

## Kaynaklar

SDK açık kaynaktır ve GitHub'da mevcuttur. Katkılar ve hata raporları hoş karşılanır.

- Platform: [merx.exchange](https://merx.exchange)
- Belgeler: [merx.exchange/docs](https://merx.exchange/docs)
- GitHub: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- npm: [npmjs.com/package/merx-sdk](https://www.npmjs.com/package/merx-sdk)
- Python SDK: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- AI ajanları için MCP Sunucusu: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)


## Şimdi AI ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin - sıfır kurulum, salt okunur araçlar için API anahtarı gerekli değil:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI ajanınıza şunu sorun: "TRON enerjisinin en ucuz hali şu anda nedir?" ve bağlı tüm sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)