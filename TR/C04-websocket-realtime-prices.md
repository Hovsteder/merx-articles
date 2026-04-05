# TRON Enerji Fiyatlarını WebSocket ile Gerçek Zamanda İzleyin

MERX, `wss://merx.exchange/ws` adresinde bir WebSocket uç noktası sağlayarak bağlantılı tüm sağlayıcılardan TRON enerji ve bandwidth fiyatlarını gerçek zamanda aktarır. Bu makale bağlantı protokolünü, ileti biçimini, abonelik iletileri aracılığıyla sağlayıcı filtrelemesini, sinyal kalp atışı ve yeniden bağlantı stratejilerini ve JavaScript ve Python'da tam kod örneklerini kapsar; ayrıca enerji belirli bir hedef fiyatın altına düştüğünde sizi bilgilendiren pratik bir fiyat uyarı botu da içerir.

## WebSocket Neden Polling Yerine Kullanılır?

MERX REST API'si `/api/v1/prices` adresinde tüm sağlayıcılardan güncel fiyatları döndürür. Bunu düzenli aralıklarla sorgulanabilir ve dakika başına 300 istek sınırı sayesinde isterse 200 milisaniyede bir yenileyebilirsiniz.

Ancak polling'in doğasında bulunan dezavantajları vardır. Her istek, TLS el sıkışması yükü ile tam bir HTTP gidiş-dönüş yapılır. Bir fiyatın tam olarak ne zaman değiştiğini hiç bilemezsiniz; bunu sadece sonraki anket aralığınızda keşfedersiniz. Ve çok sayıda istemciniz varsa, her biri aynı uç noktaya bağımsız olarak istekte bulunur.

WebSocket bağlantısı tüm bu sorunları çözer. Tek bir kalıcı bağlantı, fiyat güncellemelerini gerçekleşir gerçekleşmez alır. MERX fiyat izleme hizmeti sekiz sağlayıcının tamamını her 30 saniyede sorgulanır ve güncellemeleri Redis pub/sub aracılığıyla yayınlar. WebSocket sunucusu o kanala abone olur ve güncellemeleri bağlı tüm istemcilere anında dağıtır.

Sonuç: daha düşük gecikme süresi, daha düşük bant genişliği kullanımı ve her fiyat değişikliğinin garantili teslimi.

## Bağlantı

WebSocket uç noktasına `wss://merx.exchange/ws` adresinden bağlanın. Kimlik doğrulama gerekli değildir; fiyat verileri herkese açıktır.

### JavaScript (Tarayıcı veya Node.js)

```javascript
const ws = new WebSocket('wss://merx.exchange/ws')

ws.addEventListener('open', () => {
  console.log('MERX WebSocket\'e bağlandı')
})

ws.addEventListener('message', (event) => {
  const msg = JSON.parse(event.data)
  console.log(`[${msg.provider}] Fiyat güncellemesi ${new Date(msg.ts).toISOString()} tarihinde`)
  console.log(msg.data)
})

ws.addEventListener('close', (event) => {
  console.log(`Bağlantı kesildi: code=${event.code} reason=${event.reason}`)
})

ws.addEventListener('error', (error) => {
  console.error('WebSocket hatası:', error)
})
```

### Python (websockets kütüphanesi kullanarak)

```python
import asyncio
import json
import websockets

async def listen():
    uri = "wss://merx.exchange/ws"
    async with websockets.connect(uri) as ws:
        print("MERX WebSocket\'e bağlandı")
        async for raw in ws:
            msg = json.loads(raw)
            print(f"[{msg['provider']}] {msg['data']}")

asyncio.run(listen())
```

websockets kütüphanesini `pip install websockets` ile yükleyin.

## İleti Biçimi

Sunucudan gelen her ileti, üç alan içeren bir JSON nesnesidir:

```json
{
  "type": "price_update",
  "provider": "sohu",
  "data": {
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
  },
  "ts": 1743292800123
}
```

| Alan       | Tür    | Açıklama                                                  |
|------------|--------|-----------------------------------------------------------|
| `type`     | string | Her zaman `"price_update"`                                |
| `provider` | string | Sağlayıcı tanımlayıcısı (ör. "sohu", "catfee", "itrx")    |
| `data`     | object | Tam sağlayıcı fiyat verisi, REST API ile aynı biçim      |
| `ts`       | number | Sunucu zaman damgası (milisaniye cinsinden)              |

`data` nesnesi, `/api/v1/prices` REST yanıtındaki tek bir öğe ile aynı yapıya sahiptir. Bu, gerçek zamanlı ve sorgulanmış veriler için aynı ayrıştırma mantığını kullanabileceğiniz anlamına gelir.

### Sağlayıcı Tanımlayıcıları

MERX şu anda sekiz sağlayıcıyı izlemektedir:

| Sağlayıcı   | Tanımlayıcı  |
|------------|-------------|
| Sohu       | `sohu`      |
| CatFee     | `catfee`    |
| Netts      | `netts`     |
| TronSave   | `tronsave`  |
| Feee       | `feee`      |
| ITRX       | `itrx`      |
| PowerSun   | `powersun`  |
| TEM        | `tem`       |

Her sağlayıcı yaklaşık olarak her 30 saniyede bir güncelleme gönderir. Sekiz sağlayıcı ile ortalama olarak yaklaşık her 3-4 saniyede bir güncelleme alacaksınız.

## Belirli Sağlayıcılara Abone Olma

Varsayılan olarak, yeni bir bağlantı tüm sağlayıcılardan güncellemeler alır. Filtrelemek için bağlandıktan sonra bir abone iletisi gönderin:

```json
{ "subscribe": ["sohu", "catfee", "itrx"] }
```

Bu sunucuya yalnızca belirtilen sağlayıcılardan güncellemeler göndermesini söyler. Diğer tüm güncellemeler bağlantınız için sessizce atılır.

Sıfırlamak ve tüm güncellemeleri almaya başlamak için boş bir dizi gönderin:

```json
{ "subscribe": [] }
```

### JavaScript Abone Olma Örneği

```javascript
const ws = new WebSocket('wss://merx.exchange/ws')

ws.addEventListener('open', () => {
  // Sadece sohu ve catfee'den güncellemeler alın
  ws.send(JSON.stringify({
    subscribe: ['sohu', 'catfee']
  }))
})

ws.addEventListener('message', (event) => {
  const msg = JSON.parse(event.data)
  // msg.provider sadece "sohu" veya "catfee" olacak
  console.log(`${msg.provider}: ${msg.data.energy_prices[0]?.price_sun} SUN`)
})
```

### Python Abone Olma Örneği

```python
import asyncio
import json
import websockets

async def listen_filtered():
    async with websockets.connect("wss://merx.exchange/ws") as ws:
        # Belirli sağlayıcılara abone olun
        await ws.send(json.dumps({
            "subscribe": ["sohu", "catfee"]
        }))
        print("sohu ve catfee'ye abone olundu")

        async for raw in ws:
            msg = json.loads(raw)
            prices = msg["data"].get("energy_prices", [])
            if prices:
                print(f"{msg['provider']}: {prices[0]['price_sun']} SUN")

asyncio.run(listen_filtered())
```

Herhangi bir zamanda yeni bir abone iletisi göndererek aboneliğinizi değiştirebilirsiniz. Sunucu önceki filtreyi yenisiyle değiştirir.

## Sinyal Kalp Atışı ve Bağlantı Sağlığı

MERX WebSocket sunucusu her 30 saniyede ping frame'leri gönderir. İstemciler bağlantıyı canlı tutmak için pong frame'leriyle yanıt vermelidir. Bir istemci ping'e yanıt vermezse, sunucu bir sonraki sinyal kalp atışı döngüsünde bağlantıyı sonlandırır (yaklaşık 30 saniye sonra).

Çoğu WebSocket kütüphanesi ping/pong'u protokol seviyesinde otomatik olarak işler. Python `websockets` kütüphanesi ve tarayıcı `WebSocket` API'si her ikisi de herhangi bir ek kod olmadan ping'lere yanıt verir.

Node.js'de `ws` kütüphanesi kullanıyorsanız, pong yanıtları da otomatiktir. Ancak ping'leri otomatik olarak işlemeyen bir kütüphane kullanıyorsanız, ping olayını dinlemeniz gerekir:

```javascript
// Node.js 'ws' kütüphanesi ile (pong'u otomatik olarak işler)
import WebSocket from 'ws'

const ws = new WebSocket('wss://merx.exchange/ws')

ws.on('ping', () => {
  // ws kütüphanesi pong'u otomatik olarak gönderir
  // Bu handler sadece günlüğe kaydetmek içindir
  console.log('Ping alındı, pong gönderildi')
})
```

## Yeniden Bağlantı Stratejisi

WebSocket bağlantıları birçok nedenle kesilir: ağ kesintileri, sunucu dağıtımları, yük dengeleyici zaman aşımları. Bir üretim istemcisi yeniden bağlantıyı zarif bir şekilde işlemelidir.

Önerilen strateji, jitter ile üstel geri çekilmedir:

1. Bağlantı kesildiğinde, yeniden bağlanmadan önce 1 saniye bekleyin.
2. Yeniden bağlantı başarısız olursa, bekleme süresini iki katına çıkarın: 2s, 4s, 8s, maksimum 30 saniyeye kadar.
3. Tüm istemcilerin aynı anda yeniden bağlanmasını önlemek için rastgele jitter ekleyin (0-1 saniye).
4. Başarılı yeniden bağlantıda, geri çekilmeyi 1 saniyeye sıfırlayın.
5. Yeniden bağlandıktan sonra abonelik iletinizi tekrar gönderin.

### JavaScript Yeniden Bağlantı

```javascript
function createConnection(providers = []) {
  let backoff = 1000
  const maxBackoff = 30000

  function connect() {
    const ws = new WebSocket('wss://merx.exchange/ws')

    ws.addEventListener('open', () => {
      console.log('Bağlantı kuruldu')
      backoff = 1000  // Başarıda geri çekilmeyi sıfırla

      if (providers.length > 0) {
        ws.send(JSON.stringify({ subscribe: providers }))
      }
    })

    ws.addEventListener('message', (event) => {
      const msg = JSON.parse(event.data)
      handlePriceUpdate(msg)
    })

    ws.addEventListener('close', () => {
      const jitter = Math.random() * 1000
      const delay = Math.min(backoff + jitter, maxBackoff)
      console.log(`Bağlantı kesildi. ${Math.round(delay)}ms içinde yeniden bağlanılacak`)
      setTimeout(connect, delay)
      backoff = Math.min(backoff * 2, maxBackoff)
    })

    ws.addEventListener('error', () => {
      ws.close()  // Yeniden bağlantı için close handler'ı tetikler
    })

    return ws
  }

  return connect()
}

function handlePriceUpdate(msg) {
  const prices = msg.data.energy_prices || []
  if (prices.length > 0) {
    console.log(`${msg.provider}: ${prices[0].price_sun} SUN/energy`)
  }
}

// İki sağlayıcıya bağlanın ve abone olun
createConnection(['sohu', 'catfee'])
```

### Python Yeniden Bağlantı

```python
import asyncio
import json
import websockets

async def listen_with_reconnect(providers=None):
    backoff = 1.0
    max_backoff = 30.0

    while True:
        try:
            async with websockets.connect("wss://merx.exchange/ws") as ws:
                print("Bağlantı kuruldu")
                backoff = 1.0  # Başarıda sıfırla

                if providers:
                    await ws.send(json.dumps({"subscribe": providers}))

                async for raw in ws:
                    msg = json.loads(raw)
                    handle_price_update(msg)

        except (websockets.ConnectionClosed, ConnectionError, OSError) as e:
            jitter = asyncio.get_event_loop().time() % 1
            delay = min(backoff + jitter, max_backoff)
            print(f"Bağlantı kesildi ({e}). {delay:.1f}s içinde yeniden bağlanılacak")
            await asyncio.sleep(delay)
            backoff = min(backoff * 2, max_backoff)

def handle_price_update(msg):
    prices = msg["data"].get("energy_prices", [])
    if prices:
        print(f"{msg['provider']}: {prices[0]['price_sun']} SUN/energy")

asyncio.run(listen_with_reconnect(["sohu", "catfee"]))
```

## Fiyat Uyarı Botu Oluşturma

İşte pratik bir kullanım örneği: gerçek zamanlı fiyatları izleyen ve enerji belirli bir hedef fiyatın altına düştüğünde bildirim gönderen bir bot. Bu örnek basit bir konsol uyarısı kullanır, ancak bunu Telegram iletisi, Slack webhook'u veya e-posta ile değiştirebilirsiniz.

### JavaScript Fiyat Uyarı Botu

```javascript
const TARGET_PRICE_SUN = 25
const COOLDOWN_MS = 300000  // Uyarılar arasında 5 dakika

let lastAlertTime = 0

function createAlertBot() {
  let backoff = 1000

  function connect() {
    const ws = new WebSocket('wss://merx.exchange/ws')

    ws.addEventListener('open', () => {
      console.log('Fiyat uyarı botu bağlandı')
      backoff = 1000
    })

    ws.addEventListener('message', (event) => {
      const msg = JSON.parse(event.data)
      checkPriceAlert(msg)
    })

    ws.addEventListener('close', () => {
      const delay = Math.min(backoff + Math.random() * 1000, 30000)
      setTimeout(connect, delay)
      backoff = Math.min(backoff * 2, 30000)
    })

    ws.addEventListener('error', () => ws.close())
  }

  connect()
}

function checkPriceAlert(msg) {
  const prices = msg.data.energy_prices || []
  if (prices.length === 0) return

  const cheapest = prices[0].price_sun
  const now = Date.now()

  if (cheapest <= TARGET_PRICE_SUN && now - lastAlertTime > COOLDOWN_MS) {
    lastAlertTime = now
    console.log(`UYARI: ${msg.provider} enerjisi ${cheapest} SUN'da (hedef: ${TARGET_PRICE_SUN})`)
    console.log(`  Mevcut: ${msg.data.available_energy.toLocaleString()} birim`)
    console.log(`  Zaman: ${new Date(msg.ts).toISOString()}`)
    // Bildirim mantığınız ile değiştirin:
    // sendTelegramMessage(...)
    // postToSlack(...)
  }
}

createAlertBot()
```

### Python Fiyat Uyarı Botu

```python
import asyncio
import json
import time
import websockets

TARGET_PRICE_SUN = 25
COOLDOWN_SEC = 300  # Uyarılar arasında 5 dakika
last_alert_time = 0

async def alert_bot():
    global last_alert_time
    backoff = 1.0

    while True:
        try:
            async with websockets.connect("wss://merx.exchange/ws") as ws:
                print("Fiyat uyarı botu bağlandı")
                backoff = 1.0

                async for raw in ws:
                    msg = json.loads(raw)
                    check_alert(msg)

        except (websockets.ConnectionClosed, ConnectionError, OSError):
            delay = min(backoff + (time.time() % 1), 30.0)
            print(f"{delay:.1f}s içinde yeniden bağlanılacak")
            await asyncio.sleep(delay)
            backoff = min(backoff * 2, 30.0)

def check_alert(msg):
    global last_alert_time
    prices = msg["data"].get("energy_prices", [])
    if not prices:
        return

    cheapest = prices[0]["price_sun"]
    now = time.time()

    if cheapest <= TARGET_PRICE_SUN and now - last_alert_time > COOLDOWN_SEC:
        last_alert_time = now
        provider = msg["provider"]
        available = msg["data"].get("available_energy", 0)
        print(f"UYARI: {provider} enerjisi {cheapest} SUN'da (hedef: {TARGET_PRICE_SUN})")
        print(f"  Mevcut: {available:,} birim")
        # Bildirim mantığınız ile değiştirin

asyncio.run(alert_bot())
```

## Yerel Durumu Koruma

Canlı bir pazarı görüntülemesi gereken uygulamalar için, her sağlayıcıdan en son fiyatların yerel bir sözlüğünü tutun ve her iletide güncelleyin:

```javascript
const latestPrices = new Map()

function handlePriceUpdate(msg) {
  latestPrices.set(msg.provider, {
    data: msg.data,
    receivedAt: msg.ts,
  })

  // Tüm sağlayıcılar arasında en ucuz olanı bulun
  let cheapest = null
  for (const [provider, entry] of latestPrices) {
    const prices = entry.data.energy_prices || []
    if (prices.length === 0) continue
    const price = prices[0].price_sun
    if (cheapest === null || price < cheapest.price) {
      cheapest = { provider, price, available: entry.data.available_energy }
    }
  }

  if (cheapest) {
    console.log(`Pazarın en iyisi: ${cheapest.provider} ${cheapest.price} SUN'da`)
  }
}
```

## WebSocket'i REST API ile Birleştirme

Yaygın bir model, ilk durum yüklü için REST API'sını ve artımlı güncellemeler için WebSocket'i kullanmaktır:

```javascript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY })

// REST API'sından ilk durumu yükleyin
const initialPrices = await merx.prices.list()
const priceMap = new Map()
for (const p of initialPrices) {
  priceMap.set(p.provider, p)
}
console.log(`REST API'sından ${priceMap.size} sağlayıcı yüklendi`)

// Gerçek zamanlı güncellemeler için WebSocket'e geçin
const ws = new WebSocket('wss://merx.exchange/ws')
ws.addEventListener('message', (event) => {
  const msg = JSON.parse(event.data)
  priceMap.set(msg.provider, msg.data)
  // İlk renderdan itibaren UI veya mantığınız sürekli güncellenmiş bir fiyat haritasına sahip
})
```

Bu, ilk render'dan bir tam pazarı görünümüne sahip olmanızı, o noktadan ileri sıfır gecikme süresiyle güncellemeler almanızı sağlar.

## Performans Dikkatleri

WebSocket uç noktası yüksek aktarım hızı tüketimine yönelik tasarlanmıştır. Üretim dağıtımları için birkaç not:

- Her sağlayıcı güncellemesi yaklaşık 200-500 bayt JSON'dır. Sekiz sağlayıcı her 30 saniyede güncellendiğinde, toplam bant genişliği saniyede 1 KB'ın altındadır.
- Sadece belirli sağlayıcılara ihtiyacınız varsa trafiği azaltmak için abonelik filtresini kullanın.
- Sunucu tarafı sinyal kalp atışı aralığı 30 saniyedir. Uygulamanızda boşta bağlantıları daha erken kapatan bir güvenlik duvarı veya proxy varsa, uygulama düzeyinde canlı tutma iletileri gönderin.
- WebSocket yolu `/ws` dir. Ters proxy veya yük dengeleyicinizin bu yoldaki HTTP bağlantılarını yükseltmek için yapılandırıldığından emin olun.

## Kaynaklar

- Platform: [merx.exchange](https://merx.exchange)
- Belgelendirme: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- Python SDK: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- MCP Sunucusu: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)


## Şimdi AI ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin -- kurulum yok, salt okunur araçlar için API anahtarı gerekmez:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI ajanınıza sorun: "TRON enerjisinin şu anda en ucuz fiyatı ne?" ve bağlantılı tüm sağlayıcılardan canlı fiyatları alın.

Tam MCP belgelendirmesi: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)