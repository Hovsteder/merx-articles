# Webhook Entegrasyonu: Enerji Siparişleri Doldurulduğunda Bildirim Alın

MERX webhook'ları, hesabınızda olaylar meydana geldiğinde—siparişler doldurulduğunda, siparişler başarısız olduğunda, mevduat geldiğinde, çekimler tamamlandığında—gerçek zamanlı HTTP bildirimleri iletir. Bu makale dört olay türünü, yük biçimini, `X-Merx-Signature` başlığı kullanılarak HMAC-SHA256 imza doğrulamasını, üstel geri tepme ile yeniden deneme ilkesini, tekrarlanan başarısızlıklar sonrasında otomatik deaktivasyonu ve imza doğrulama ile Express.js ve Flask'ta tam sunucu uygulamalarını kapsamaktadır.

## Yoklama Yerine Neden Webhook?

Webhook'lara alternatif, bir döngüde `/api/v1/orders/:id` endpoint'ini yoklamak, durumun `PENDING`'den `FILLED`'a değişmesini beklemektir. Bu basit durumlar için çalışsa da açık dezavantajları vardır:

- Boşa harcanmış istekler. Çoğu yoklama değişmemiş aynı durumu döndürür.
- Gecikme. Uygulamanız bir durum değişikliğini yalnızca sonraki yoklama aralığında keşfeder.
- Hız sınırları. Siparişler endpoint'inde 10-istek-per-dakika sınırı ile agresif yoklama hızlı bir şekilde sınıra ulaşır.
- Karmaşıklık. Yoklama mantığı yeniden deneme işleme, zaman aşımı yönetimi ve durum takibi gerektirir.

Webhook'lar modeli ters çevirir. MERX'ten "bir şey değişti mi?" diye sormak yerine, MERX size bir şey olur olmaz söyler. Sunucunuz tam olay yükü ile bir HTTP POST alır, bunu işler ve devam eder. Yoklama döngüleri yok, boşa harcanmış istekler yok, yapay gecikmeler yok.

## Webhook Oluşturma

Webhook'ları REST API veya SDK aracılığıyla oluşturabilirsiniz.

### REST API

```bash
curl -X POST https://merx.exchange/api/v1/webhooks \
  -H "X-API-Key: sk_live_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-server.com/webhooks/merx",
    "events": ["order.filled", "order.failed", "deposit.received", "withdrawal.completed"]
  }'
```

Yanıt:

```json
{
  "data": {
    "id": "wh_abc123",
    "url": "https://your-server.com/webhooks/merx",
    "events": ["order.filled", "order.failed", "deposit.received", "withdrawal.completed"],
    "secret": "a1b2c3d4e5f6...64-hex-characters...9876543210",
    "is_active": true,
    "created_at": "2026-03-30T12:00:00.000Z"
  }
}
```

`secret` alanı, 32 rastgele bayttan oluşturulan 64 karakterli hex dizesidir. Yalnızca oluşturma yanıtında döndürülür. Güvenli bir şekilde saklayın—gelen webhook imzalarını doğrulamak için ihtiyacınız olacak.

### JavaScript SDK

```javascript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY })

const webhook = await merx.webhooks.create({
  url: 'https://your-server.com/webhooks/merx',
  events: ['order.filled', 'order.failed', 'deposit.received'],
})

console.log(`Webhook ID: ${webhook.id}`)
console.log(`Secret: ${webhook.secret}`)  // Store this securely
```

### Python SDK

```python
from merx import MerxClient

client = MerxClient(api_key="sk_live_your_key_here")

webhook = client.webhooks.create(
    url="https://your-server.com/webhooks/merx",
    events=["order.filled", "order.failed", "deposit.received"],
)

print(f"Webhook ID: {webhook.id}")
print(f"Secret: {webhook.secret}")  # Store this securely
```

## Olay Türleri

MERX dört webhook olay türünü destekler. Bir webhook oluştururken hangi olaylara abone olmak istediğinizi seçersiniz. Dördüne de veya yalnızca ihtiyacınız olanlara abone olabilirsiniz.

### order.filled

Bir sipariş tamamen yerine getirildiğinde gönderilir. Tüm sağlayıcı delegasyonları zincir üzerinde doğrulanmıştır.

```json
{
  "event": "order.filled",
  "timestamp": "2026-03-30T12:05:00.000Z",
  "data": {
    "order_id": "ord_abc123",
    "resource_type": "ENERGY",
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

Bu, otomatikleştirilmiş sistemler için en önemli olaydır. Bunu aldığınızda, enerji delege edilmiş ve hedef adres TRON işlemleriyle ilerlemeye başlayabilir.

### order.failed

Bir sipariş yerine getirilemediğinde gönderilir. Bu, tüm sağlayıcılar kullanılamadığında, kapasite tükenmişse veya sağlayıcı tarafında bir hata oluştuğunda meydana gelebilir.

```json
{
  "event": "order.failed",
  "timestamp": "2026-03-30T12:05:00.000Z",
  "data": {
    "order_id": "ord_def456",
    "resource_type": "ENERGY",
    "amount": 500000,
    "target_address": "TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    "reason": "No provider could fulfill the order within the specified parameters",
    "refunded": true,
    "refund_amount_sun": 0
  }
}
```

Bir sipariş başarısız olduğunda, ayrılan herhangi bir bakiye iade edilir. `refunded` alanı bunu doğrular ve `refund_amount_sun` ödeme zaten yapılmışsa geri döndürülen tutarı gösterir.

### deposit.received

MERX hesabınıza yapılan bir mevduat tespit edilerek kredilendiğinde gönderilir.

```json
{
  "event": "deposit.received",
  "timestamp": "2026-03-30T12:10:00.000Z",
  "data": {
    "deposit_id": "dep_ghi789",
    "amount_sun": 100000000,
    "amount_trx": "100.000000",
    "currency": "TRX",
    "tx_id": "789abc...",
    "new_balance_trx": 250.5
  }
}
```

### withdrawal.completed

Bir çekim isteği işlenip zincir üzerinde işlem doğrulandığında gönderilir.

```json
{
  "event": "withdrawal.completed",
  "timestamp": "2026-03-30T12:15:00.000Z",
  "data": {
    "withdrawal_id": "wdr_jkl012",
    "amount": 50,
    "currency": "TRX",
    "address": "TExternalAddress...",
    "tx_id": "012def...",
    "tronscan_url": "https://tronscan.org/#/transaction/012def..."
  }
}
```

## İmza Doğrulaması

Her webhook teslimati, webhook sırrınız kullanılarak hesaplanan istek gövdesinin HMAC-SHA256 imzasını içeren `X-Merx-Signature` başlığını içerir.

Doğrulama süreci:

1. JSON ayrıştırmasından önce ham istek gövdesini okuyun.
2. Depolanan webhook sırrınızı anahtar olarak kullanarak ham gövdenin HMAC-SHA256'sını hesaplayın.
3. Hesaplanan imzayı `X-Merx-Signature` başlığı değeri ile karşılaştırın.
4. Eşleşirse, istek orijinaldir. Değilse, reddedin.

Bu korumalı:
- Sırrınızı bilmeyen üçüncü taraflardan gelen sahte istekler.
- İletim sırasında gövde değiştirilmiş, tahrif edilmiş yükler.
- Zaman damgası doğrulaması ile birleştirilmiş yeniden oynatma saldırıları.

İmzaları kontrol ederken her zaman sabit-zaman karşılaştırma işlevi kullanın. Standart dize eşitliği (`===` veya `==`) zamanlama saldırılarına karşı savunmasızdır.

## Express.js Webhook İşleyicisi

MERX webhook'larını alan ve doğrulayan eksiksiz bir Express.js sunucusu:

```javascript
import express from 'express'
import crypto from 'node:crypto'

const app = express()
const WEBHOOK_SECRET = process.env.MERX_WEBHOOK_SECRET

// Use raw body for signature verification
app.post('/webhooks/merx', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-merx-signature']
  if (!signature || !WEBHOOK_SECRET) {
    res.status(401).json({ error: 'Missing signature or secret' })
    return
  }

  // Compute expected signature
  const expected = crypto
    .createHmac('sha256', WEBHOOK_SECRET)
    .update(req.body)
    .digest('hex')

  // Constant-time comparison
  if (!crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected))) {
    console.warn('Webhook signature mismatch')
    res.status(401).json({ error: 'Invalid signature' })
    return
  }

  // Signature verified - parse and handle the event
  const event = JSON.parse(req.body.toString())
  console.log(`Received event: ${event.event}`)

  switch (event.event) {
    case 'order.filled':
      handleOrderFilled(event.data)
      break
    case 'order.failed':
      handleOrderFailed(event.data)
      break
    case 'deposit.received':
      handleDepositReceived(event.data)
      break
    case 'withdrawal.completed':
      handleWithdrawalCompleted(event.data)
      break
    default:
      console.warn(`Unknown event type: ${event.event}`)
  }

  // Always respond with 200 quickly to acknowledge receipt
  res.status(200).json({ received: true })
})

function handleOrderFilled(data) {
  console.log(`Order ${data.order_id} filled`)
  console.log(`  Amount: ${data.amount} ${data.resource_type}`)
  console.log(`  Cost: ${(data.total_cost_sun / 1_000_000).toFixed(3)} TRX`)
  console.log(`  Fills: ${data.fills.length}`)

  for (const fill of data.fills) {
    console.log(`  ${fill.provider}: ${fill.amount} at ${fill.price_sun} SUN`)
    if (fill.tronscan_url) {
      console.log(`    TX: ${fill.tronscan_url}`)
    }
  }

  // Your business logic here:
  // - Mark the transaction as ready to proceed
  // - Trigger the USDT transfer
  // - Update your database
}

function handleOrderFailed(data) {
  console.log(`Order ${data.order_id} failed: ${data.reason}`)
  if (data.refunded) {
    console.log(`  Refunded: ${data.refund_amount_sun} SUN`)
  }
  // Alert your operations team
}

function handleDepositReceived(data) {
  console.log(`Deposit received: ${data.amount_trx} ${data.currency}`)
  console.log(`  New balance: ${data.new_balance_trx} TRX`)
  // Update local balance cache
}

function handleWithdrawalCompleted(data) {
  console.log(`Withdrawal ${data.withdrawal_id} completed`)
  console.log(`  ${data.amount} ${data.currency} to ${data.address}`)
  // Update withdrawal status in your system
}

app.listen(3000, () => {
  console.log('Webhook server listening on port 3000')
})
```

Anahtar uygulama detayları:

- Webhook rotası için `express.json()` yerine `express.raw({ type: 'application/json' })` kullanın. İmza hesaplaması için ham baytlara ihtiyacınız var.
- Sabit-zaman imza karşılaştırması için `crypto.timingSafeEqual()` kullanın.
- HTTP 200 ile mümkün olduğunca hızlı yanıt verin. Ağır işlemeyi alındısını onayladıktan sonra asenkron olarak yapın.

## Flask Webhook İşleyicisi

Python'da Flask kullanan eşdeğer uygulama:

```python
import hashlib
import hmac
import json
import os

from flask import Flask, request, jsonify

app = Flask(__name__)
WEBHOOK_SECRET = os.environ.get("MERX_WEBHOOK_SECRET", "")


def verify_signature(payload: bytes, signature: str, secret: str) -> bool:
    """Verify HMAC-SHA256 signature using constant-time comparison."""
    expected = hmac.new(
        secret.encode("utf-8"),
        payload,
        hashlib.sha256,
    ).hexdigest()
    return hmac.compare_digest(signature, expected)


@app.route("/webhooks/merx", methods=["POST"])
def handle_webhook():
    signature = request.headers.get("X-Merx-Signature", "")
    raw_body = request.get_data()

    if not signature or not WEBHOOK_SECRET:
        return jsonify({"error": "Missing signature or secret"}), 401

    if not verify_signature(raw_body, signature, WEBHOOK_SECRET):
        app.logger.warning("Webhook signature mismatch")
        return jsonify({"error": "Invalid signature"}), 401

    event = json.loads(raw_body)
    event_type = event.get("event", "")
    data = event.get("data", {})

    app.logger.info(f"Received event: {event_type}")

    if event_type == "order.filled":
        handle_order_filled(data)
    elif event_type == "order.failed":
        handle_order_failed(data)
    elif event_type == "deposit.received":
        handle_deposit_received(data)
    elif event_type == "withdrawal.completed":
        handle_withdrawal_completed(data)
    else:
        app.logger.warning(f"Unknown event type: {event_type}")

    return jsonify({"received": True}), 200


def handle_order_filled(data):
    order_id = data["order_id"]
    amount = data["amount"]
    resource = data["resource_type"]
    cost_trx = data["total_cost_sun"] / 1_000_000

    app.logger.info(f"Order {order_id} filled: {amount} {resource}, {cost_trx:.3f} TRX")

    for fill in data.get("fills", []):
        app.logger.info(
            f"  {fill['provider']}: {fill['amount']} at {fill['price_sun']} SUN"
        )

    # Your business logic here


def handle_order_failed(data):
    app.logger.warning(
        f"Order {data['order_id']} failed: {data.get('reason', 'unknown')}"
    )


def handle_deposit_received(data):
    app.logger.info(
        f"Deposit: {data['amount_trx']} {data['currency']}, "
        f"new balance: {data.get('new_balance_trx', 'N/A')} TRX"
    )


def handle_withdrawal_completed(data):
    app.logger.info(
        f"Withdrawal {data['withdrawal_id']}: "
        f"{data['amount']} {data['currency']} to {data['address']}"
    )


if __name__ == "__main__":
    app.run(port=3000)
```

Python'a özgü anahtar detaylar:

- Ham istek gövdesini bayt olarak almak için `request.get_data()` kullanın.
- Sabit-zaman dize karşılaştırması için `hmac.compare_digest()` kullanın. Python'un `==` operatörü sabit-zaman değildir.
- HMAC hesaplamak için `hashlib.sha256` ile `hmac.new()` kullanın.

## Yeniden Deneme İlkesi

MERX başarısız webhook teslimatlarını üstel geri tepme programı kullanarak yeniden dener:

| Deneme | Başarısızlıktan Sonra Gecikme |
|--------|------------------------------|
| 1      | Hemen                        |
| 2      | 30 saniye                    |
| 3      | 5 dakika                     |

Bir teslim şu durumlarda başarısız sayılır:
- Sunucunuz 10 saniye içinde yanıt vermez.
- Sunucunuz 2xx aralığı dışında bir HTTP durum kodu döndürür.
- Bağlantı kurulamaz (DNS hatası, bağlantı reddedildi, TLS hatası).

Bir olaya yönelik 3 başarısız denemeden sonra, olay bırakılır. O belirli teslim için başka yeniden deneme yapılmaz.

Webhook işleyiciniz:
- HTTP 200 ile birkaç saniye içinde yanıt verin. Ağır işlemeyi asenkron olarak yapın.
- İdempotent olun. Aynı olay kenar durumlarda birden fazla kez teslim edilebilir.
- İşlem başarısız olursa hata ayıklama için olay yükünü günlüğe kaydedin.

## Otomatik Deaktivizyon

Bir webhook endpoint'i sürekli başarısız olursa, MERX ölü bir endpoint'e kaynak harcamamak için bunu otomatik olarak deaktive eder.

Deaktivizyon eşiği birden çok olay arasında ardışık başarısızlıklara dayanır. Endpoint'iniz teslimatları kabul etmeyi sürekli başarısız olursa, webhook'un `is_active` bayrağı `false` olarak ayarlanır.

Bir webhook deaktive edildiğinde:
- O URL'ye başka olay gönderilmez.
- Webhook listenizde `is_active: false` ile görünmeye devam eder.
- Endpoint sorununu düzelterek yeni bir webhook oluşturabilirsiniz.

Webhook durumunu düzenli olarak izleyin:

```javascript
const webhooks = await merx.webhooks.list()
for (const wh of webhooks) {
  if (!wh.is_active) {
    console.warn(`Webhook ${wh.id} (${wh.url}) is deactivated`)
  }
}
```

```python
webhooks = client.webhooks.list()
for wh in webhooks:
    if not wh.is_active:
        print(f"Webhook {wh.id} ({wh.url}) is deactivated")
```

## Webhook'ları Yönetme

### Webhook'ları Listeleme

```bash
curl -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/webhooks
```

### Webhook Silme

```bash
curl -X DELETE -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/webhooks/wh_abc123
```

Webhook sırrının oluşturulduktan sonra alınamayacağını unutmayın. Sırrı kaybederseniz, webhook'u silin ve yeni bir tane oluşturun.

## Webhook'ları Yerel Olarak Test Etme

Geliştirme sırasında webhook URL'niz genel olarak erişilebilir olması gerekir. ngrok gibi araçlar yerel bir sunucuyu açığa çıkarabilir:

```bash
# Terminal 1: Start your webhook server
node webhook-server.js

# Terminal 2: Expose it publicly
ngrok http 3000
```

Webhook oluştururken ngrok URL'sini kullanın (örn. `https://abc123.ngrok.io/webhooks/merx`). Her şeyin çalıştığını doğruladıktan sonra, onu üretim URL'niz ile değiştirin.

## En İyi Uygulamalar

1. Her zaman imzaları doğrulayın. `X-Merx-Signature` başlığını kontrol etmeden webhook yüklerine asla güvenmeyin.

2. Hızlı yanıt verin. HTTP 200 ile 2-3 saniye içinde döndürün. Ağır işlemeyi arka plan işçileri için kuyruğa alın.

3. İdempotent olun. `order_id` veya `deposit_id` öğesini çoğaltma kaldırma anahtarı olarak kullanın. Aynı olayı iki kez alırsanız, ikinci işlem bir işlem olmamalıdır.

4. Ham yükü saklayın. İşlemeden önce tam JSON gövdesini günlüğe kaydedin. İşleyicinizin bir hatası varsa, olayları günlüklerden yeniden oynatabilirsiniz.

5. Webhook sağlığını izleyin. `is_active` durumunu düzenli olarak kontrol edin. Webhook'lar deaktive edilirse uyarılar ayarlayın.

6. HTTPS kullanın. MERX webhook URL'lerinin HTTPS kullanmasını gerektirir. Kendi imzalı sertifikalar kabul edilmez.

7. Seçici olarak abone olun. Yalnızca gerçekten işlediğiniz olaylara abone olun. Gereksiz olaylar bant genişliğini ve işlem süresini boşa harcar.

## Kaynaklar

- Platform: [merx.exchange](https://merx.exchange)
- Dokümantasyon: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- Python SDK: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- MCP Server: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)


## Şimdi AI ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin—kurulum yok, salt okunur araçlar için API anahtarı gerekli değil:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI ajanınıza sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve bağlı tüm sağlayıcılardan canlı fiyatlar alın.

Tam MCP dokümantasyonu: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)