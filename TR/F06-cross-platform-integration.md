# Çapraz Platform Entegrasyonu: MERX'i Mevcut Yığınınıza Ekleme

Her teknoloji yığını farklıdır. Arka ucunuz Node.js, Go, Python, Ruby ya da Java olabilir. Mimariniz monolitik veya mikroservis olabilir. İletişim kalıplarınız senkron, olay tabanlı veya bunların kombinasyonu olabilir. Soru, MERX'in yığının içine sığıp sığmayacağı değil -- hangi entegrasyon noktasının size en az frikiyon ile en yüksek değeri sağlayacağıdır.

MERX beş entegrasyon yöntemi sunar: REST API, WebSocket, webhook'lar, dil SDK'ları ve yapay zeka ajanları için bir MCP sunucusu. Bu makale her birinin ne zaman kullanılacağını, birbirleriyle nasıl etkileşime girdiklerini ve spesifik mimariniz için doğru yaklaşımı nasıl seçeceğinizi açıklar.

## Entegrasyon Yöntemlerine Genel Bakış

| Yöntem | En İyi Kullanım | Yön | Gecikme | Dil |
|---|---|---|---|---|
| REST API | İstek-yanıt işlemleri | İstemci -> MERX | ~200ms | Herhangi |
| WebSocket | Gerçek zamanlı fiyat akışları | MERX -> İstemci | Gerçek zamanlı | Herhangi |
| Webhook'lar | Asenkron bildirimler | MERX -> İstemci | Olay tabanlı | Herhangi |
| JS SDK | Node.js / tarayıcı uygulamaları | Çift yönlü | ~200ms | JavaScript/TypeScript |
| Python SDK | Python arka uçları, scriptler | Çift yönlü | ~200ms | Python |
| MCP Sunucusu | Yapay zeka ajanı entegrasyonu | Çift yönlü | ~500ms | Herhangi (MCP protokolü aracılığıyla) |

## REST API: Evrensel Entegrasyon

REST API, HTTP desteğine sahip herhangi bir programlama dilinden çalışır. En düşük ortak paydadır -- dil HTTP istekleri yapabilirse, MERX ile konuşabilir.

### REST Ne Zaman Kullanılır?

- Arka ucunuz SDK'sı olmayan bir dilde (Go, Java, Ruby, Rust, PHP)
- Sürekli etkileşim değil, ara sıra enerji satın almanıza ihtiyacınız var
- Mimariniz kalıcı bağlantılardan ziyade açık HTTP çağrılarını tercih ediyor
- Prototip oluşturuyor ve en basit entegrasyonu istiyorsunuz

### Uygulama

API, JSON yükleriyle standart REST kurallarını takip eder:

```bash
# Fiyatları kontrol et
curl -X POST https://merx.exchange/api/v1/prices \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"energy_amount": 65000, "duration": "1h"}'

# Sipariş ver
curl -X POST https://merx.exchange/api/v1/orders \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: unique-request-id-123" \
  -d '{
    "energy_amount": 65000,
    "duration": "1h",
    "target_address": "TYourAddress..."
  }'

# Sipariş durumunu kontrol et
curl https://merx.exchange/api/v1/orders/ORDER_ID \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Go Örneği

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
)

type PriceRequest struct {
    EnergyAmount int    `json:"energy_amount"`
    Duration     string `json:"duration"`
}

type PriceResponse struct {
    Best struct {
        Provider string `json:"provider"`
        PriceSun int    `json:"price_sun"`
    } `json:"best"`
    Providers []struct {
        Provider string `json:"provider"`
        PriceSun int    `json:"price_sun"`
    } `json:"providers"`
}

func getBestPrice(amount int, duration string) (*PriceResponse, error) {
    body, _ := json.Marshal(PriceRequest{
        EnergyAmount: amount,
        Duration:     duration,
    })

    req, _ := http.NewRequest(
        "POST",
        "https://merx.exchange/api/v1/prices",
        bytes.NewBuffer(body),
    )
    req.Header.Set("Authorization", "Bearer "+apiKey)
    req.Header.Set("Content-Type", "application/json")

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var result PriceResponse
    json.NewDecoder(resp.Body).Decode(&result)
    return &result, nil
}
```

### Hata İşleme

Tüm API hataları tutarlı bir biçimi takip eder:

```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Account balance is insufficient for this order",
    "details": {
      "required": 1820000,
      "available": 500000
    }
  }
}
```

Bu tutarlılık, hata işleme mantığınızın hangi uç noktayı aradığınız veya arka planda hangi sağlayıcıdan yararlanılıyor olursa olsun aynı şekilde çalışacağı anlamına gelir.

## WebSocket: Gerçek Zamanlı Fiyat Akışları

WebSocket bağlantıları, yoklama (polling) yapmadan gerçek zamanlı fiyat güncellemeleri sağlar. Fiyatlar, yedi sağlayıcının tümü arasında değiştikçe uygulamanıza akış halinde gelir.

### WebSocket Ne Zaman Kullanılır?

- Canlı fiyat gösterimlerine (panolar, ticaret arayüzleri) ihtiyacınız var
- Uygulamanız fiyat hareketlerine göre satın alma kararları veriyor
- Fiyatlar eşikleri aştığında eylem tetiklemek istiyorsunuz
- Gerçek zamanlı izleme araçları oluşturuyorsunuz

### Uygulama

```typescript
import WebSocket from 'ws';

const ws = new WebSocket(
  'wss://merx.exchange/ws',
  { headers: { 'Authorization': `Bearer ${API_KEY}` } }
);

ws.on('open', () => {
  // Belirli parametreler için fiyat güncellemelerine abone ol
  ws.send(JSON.stringify({
    type: 'subscribe',
    channel: 'prices',
    params: {
      energy_amount: 65000,
      duration: '1h'
    }
  }));
});

ws.on('message', (data) => {
  const event = JSON.parse(data.toString());

  switch (event.type) {
    case 'price_update':
      console.log(
        `${event.provider}: ${event.price_sun} SUN`
      );
      break;

    case 'order_update':
      console.log(
        `Order ${event.order_id}: ${event.status}`
      );
      break;
  }
});
```

### WebSocket'i REST ile Birleştirme

Yaygın bir desen: fiyat izleme için WebSocket ve sipariş yerleştirme için REST:

```typescript
// WebSocket fiyatları izler
ws.on('message', async (data) => {
  const event = JSON.parse(data.toString());

  if (event.type === 'price_update' &&
      event.price_sun <= targetPrice) {
    // REST API siparişi yerleştirir
    const response = await fetch(
      'https://merx.exchange/api/v1/orders',
      {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${API_KEY}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          energy_amount: 65000,
          duration: '1h',
          target_address: walletAddress
        })
      }
    );
  }
});
```

## Webhook'lar: Asenkron Olay Bildirimleri

Webhook'lar, olaylar oluştuğunda sunucunuza bildirimleri gönderir. WebSocket'ten farklı olarak (kalıcı bir bağlantı gerektiren), webhook'lar HTTP POST isteklerini alabilen herhangi bir sunucuda çalışır.

### Webhook'lar Ne Zaman Kullanılır?

- Mimariniz olay tabanlı (mesaj kuyrukları, sunucusuz işlevler)
- Kalıcı WebSocket bağlantılarını koruyamıyorsunuz
- Yeniden denemelerle güvenilir teslimat istiyorsunuz
- Enerji tedarikini işlem yürütmeden ayırmak istiyorsunuz

### Uygulama

```typescript
import express from 'express';

const app = express();
app.use(express.json());

app.post('/webhooks/merx', (req, res) => {
  const event = req.body;

  switch (event.type) {
    case 'order.filled':
      handleOrderFilled(event.data);
      break;

    case 'order.failed':
      handleOrderFailed(event.data);
      break;

    case 'standing_order.triggered':
      handleStandingOrderTriggered(event.data);
      break;

    case 'auto_energy.purchased':
      handleAutoEnergyPurchased(event.data);
      break;
  }

  // Teslimatı onaylamak için her zaman 200 ile yanıt ver
  res.status(200).json({ received: true });
});

async function handleOrderFilled(
  data: OrderFilledEvent
): Promise<void> {
  // Enerji kullanılabilir -- işleme devam et
  const pendingTx = await db.getPendingTransaction(
    data.order_id
  );
  if (pendingTx) {
    await executeTransaction(pendingTx);
  }
}
```

### Webhook'ların Güvenilirliği

MERX, başarısız webhook teslimatlarını üstel geri alma (exponential backoff) ile yeniden dener. Uç noktanız şunları yapmalıdır:

1. 200 durumuyla hızlı yanıt ver (5 saniye içinde)
2. İşleme zaman alıyorsa olayı asenkron olarak işle
3. Yinelenen teslimatları idempotent olarak işle (yineleme kaldırmak için olay kimliğini kullan)

```typescript
const processedEvents = new Set<string>();

app.post('/webhooks/merx', async (req, res) => {
  const event = req.body;

  // İdempotentlik kontrolü
  if (processedEvents.has(event.id)) {
    res.status(200).json({ received: true });
    return;
  }

  processedEvents.add(event.id);

  // Hemen onayla
  res.status(200).json({ received: true });

  // Asenkron olarak işle
  processEvent(event).catch(console.error);
});
```

## JavaScript SDK

JavaScript/TypeScript SDK, REST API ve WebSocket bağlantılarını yazılı arayüzler ve kolaylık yöntemleriyle sarmalanır.

### JS SDK Ne Zaman Kullanılır?

- Arka ucunuz Node.js veya ön ucunuz tarayıcı tabanlı
- TypeScript türleri ve IDE otomatik tamamlamayı istiyorsunuz
- Ham HTTP isteklerinden ziyade yöntem çağrılarını tercih ediyorsunuz
- Bir pakette hem REST hem WebSocket'e ihtiyacınız var

### Uygulama

```typescript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY!
});

// Fiyatlar
const prices = await merx.getPrices({
  energy_amount: 65000,
  duration: '1h'
});

// Siparişler
const order = await merx.createOrder({
  energy_amount: 65000,
  duration: '1h',
  target_address: 'TAddress...'
});

// Sabit siparişler
const standing = await merx.createStandingOrder({
  energy_amount: 65000,
  max_price_sun: 25,
  duration: '1h',
  repeat: true
});

// Enerji tahmini
const estimate = await merx.estimateEnergy({
  contract_address: 'TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t',
  function_selector: 'transfer(address,uint256)',
  parameter: [recipient, amount],
  owner_address: sender
});

// WebSocket (SDK'ya yerleştirilmiş)
const ws = merx.connectWebSocket();
ws.on('price_update', (data) => {
  console.log(`${data.provider}: ${data.price_sun} SUN`);
});
```

SDK kimlik doğrulama, istek serileştirme, hata ayrıştırma ve tür doğrulamasını otomatik olarak işler.

## Python SDK

Python SDK, Python arka uçları, veri analizi işlem hatları ve otomasyon scriptleri için aynı işlevselliği sağlar.

### Python SDK Ne Zaman Kullanılır?

- Arka ucunuz Python (Django, Flask, FastAPI)
- Veri analizi veya raporlama araçları oluşturuyorsunuz
- DevOps scriptleriniz Python'da
- Python tabanlı ticaret botları ile entegrasyonunuz var

### Uygulama

```python
from merx import MerxClient

merx = MerxClient(api_key="your-api-key")

# Fiyatları al
prices = merx.get_prices(
    energy_amount=65000,
    duration="1h"
)
print(f"En iyi: {prices.best.price_sun} SUN "
      f"via {prices.best.provider}")

# Sipariş ver
order = merx.create_order(
    energy_amount=65000,
    duration="1h",
    target_address="TAddress..."
)

# Bir kontrat çağrısı için enerji tahmin et
estimate = merx.estimate_energy(
    contract_address="TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    function_selector="transfer(address,uint256)",
    parameter=[recipient, amount],
    owner_address=sender
)
print(f"Gerekli enerji: {estimate.energy_required}")
```

### Veri Analizi Örneği

```python
import pandas as pd
from merx import MerxClient

merx = MerxClient(api_key="your-api-key")

# Analiz için fiyat tarihçesini çek
history = merx.get_price_history(
    energy_amount=65000,
    duration="1h",
    period="30d"
)

df = pd.DataFrame(history.prices)
print(f"Ortalama fiyat: {df['price_sun'].mean():.1f} SUN")
print(f"Minimum fiyat: {df['price_sun'].min()} SUN")
print(f"Maksimum fiyat: {df['price_sun'].max()} SUN")
print(f"Standart sapma: {df['price_sun'].std():.1f} SUN")
```

## MCP Sunucusu: Yapay Zeka Ajanı Entegrasyonu

MERX MCP (Model Context Protocol) sunucusu, yapay zeka ajanlarının TRON enerji piyasası ile doğrudan etkileşim kurmasını sağlar. Bu yeni entegrasyon noktası ve temelde farklı bir etkileşim modelini sağlar.

### MCP Ne Zaman Kullanılır?

- TRON işlemlerini yöneten yapay zeka ajanları oluşturuyorsunuz
- Konuşsal enerji yönetimi istiyorsunuz
- Claude, ChatGPT veya araç kullanımı özellikleriyle diğer LLM'ler kullanıyorsunuz
- Doğal dil aracılığıyla enerji stratejilerini hızlı bir şekilde prototiplemeniz gerekiyor

### Nasıl Çalışır?

[github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp) adresindeki MCP sunucusu MERX yeteneklerini yapay zeka ajanlarının çağırabileceği araçlar olarak sunar:

- `get_prices` -- Geçerli enerji fiyatlarını kontrol et
- `create_order` -- Enerji satın al
- `analyze_prices` -- Fiyat istatistiklerini al
- `estimate_energy` -- İşlem enerji gereksinimlerini simüle et
- `check_resources` -- Cüzdan enerji bakiyesini kontrol et

Bir yapay zeka ajanı bu araçları "Şu anda mevcut olan en ucuz enerji nedir?" gibi soruları yanıtlamak veya "65.000 enerji satın al en iyi fiyattan cüzdanım için" gibi komutları yürütmek için kullanabilir.

### Entegrasyon

```json
{
  "mcpServers": {
    "merx": {
      "command": "npx",
      "args": ["merx-mcp"],
      "env": {
        "MERX_API_KEY": "your-api-key"
      }
    }
  }
}
```

## Doğru Entegrasyonu Seçmek

### Karar Matrisi

| Durumunuz | Önerilen Entegrasyon |
|---|---|
| Hızlı prototip, herhangi bir dil | REST API |
| Node.js arka ucu | JS SDK |
| Python arka ucu | Python SDK |
| Gerçek zamanlı fiyat gösterimi | WebSocket |
| Olay tabanlı mimari | Webhook'lar |
| Sunucusuz (Lambda, Cloud Functions) | REST API + Webhook'lar |
| Yapay zeka ajanı oluşturma | MCP Sunucusu |
| Go / Java / Ruby arka ucu | REST API |
| Tam özellikli uygulama | JS/Python SDK + Webhook'lar |

### Entegrasyon Noktalarını Birleştirme

Çoğu üretim sistemi birden fazla entegrasyon yöntemini kullanır:

- **SDK + Webhook'lar**: Giden istekler için SDK'yı (fiyatlar, siparişler) ve gelen bildirimler için webhook'ları (sipariş dolduruldu, fiyat uyarıları) kullan
- **WebSocket + REST**: İzleme için WebSocket ve eylemler için REST
- **REST + Webhook'lar**: Dil agnostik tam özellikli yığın
- **MCP + SDK**: Strateji için yapay zeka ajanı, yürütme için SDK

## Geçiş Yolu

Şu anda tek bir sağlayıcının API'sini kullanıyorsanız, MERX'e geçiş doğrudandır:

1. **MERX SDK'sını** projenize ekle
2. **Fiyat kontrolleri**, `merx.getPrices()` ile değiştir -- aynı veriler, daha fazla sağlayıcı
3. **Sipariş yerleştirmeyi**, `merx.createOrder()` ile değiştir -- aynı akış, en iyi fiyat yönlendirmesi
4. **Webhook'ları** sipariş bildirimleri için ekle (zaten olay tabanlı değilse)
5. **Eski sağlayıcı kodunu** MERX entegrasyonu doğrulandıktan sonra kaldır

Geçiş artımlı olarak yapılabilir. Testler sırasında her iki sistemi de çalıştır, tam olarak geçişten önce fiyatları ve sipariş sonuçlarını karşılaştır.

## Sonuç

MERX, mimarinizle eşleşen entegrasyon yöntemi aracılığıyla herhangi bir teknoloji yığınına sığacak şekilde tasarlanmıştır. Evrensellik için REST, gerçek zamanlı için WebSocket, olaylar için webhook'lar, geliştirici deneyimi için SDK'lar ve yapay zeka ajanları için MCP.

Seçim münhasır değildir -- ihtiyaçlarınıza uyacak şekilde entegrasyon noktalarını birleştir. Acil sorununuzu çözen en basit seçenek ile başla ve gereksinimler büyüdükçe genişlet.

[https://merx.exchange/docs](https://merx.exchange/docs) adresinde API belgeleri. [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp) adresinde MCP sunucusu. Platform: [https://merx.exchange](https://merx.exchange).


## Şimdi Yapay Zeka ile Deneyin

MERX'i Claude Desktop veya herhangi bir MCP uyumlu istemciye ekle -- yükleme yok, yalnızca okunur araçlar için API anahtarı gerekli değil:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Yapay zeka ajanınıza şunu sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)