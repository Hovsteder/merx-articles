# MERX API Kimlik Doğrulaması: Anahtarlar, İzinler ve Hız Sınırları

Her API entegrasyonu kimlik doğrulamayla başlar. Doğru yaparsanız, otomatik enerji ticaretiniz günün 24 saati sorunsuz çalışır. Yanlış yaparsanız, sızdırılan kimlik bilgileri, açıklanamayan 403 hataları veya en kötü zamanımızda üretim sistemlerinizi durduran hız sınırı yasaklamalarıyla uğraşırsınız.

MERX iki kimlik doğrulama yöntemi sağlar; her biri farklı bir kullanım durumu için tasarlanmıştır. Bu makale her ikisini detaylıca kapsar - nasıl çalıştıkları, her birini ne zaman kullanacağınız, izinleri nasıl ayrıntılı olarak yöneteceğiniz ve API yüzeyinde hangi hız sınırlarının geçerli olduğu.

## İki Kimlik Doğrulama Yöntemi

MERX, API isteklerinin kimliğini doğrulamak için iki yolu destekler: API anahtarları ve JWT belirteçleri. Farklı amaçlara hizmet ederler ve birbirinin yerine kullanılamaz.

### API Anahtarı Kimlik Doğrulaması

API anahtarları, sunucudan sunucuya iletişim için tasarlanmış uzun ömürlü kimlik bilgileridir. API aracılığıyla veya yönetici panelinde oluşturursunuz, belirli izinler atarsınız ve her isteğe `X-API-Key` başlığı aracılığıyla dahil edersiniz.

```bash
curl https://merx.exchange/api/v1/prices \
  -H "X-API-Key: merx_live_k7x9m2p4..."
```

API anahtarları şu durumlarda doğru seçimdir:

- Arka uç hizmetiniz MERX'i kullanıcılarınız adına çağırır.
- Siparişler oluşturan veya bakiyeleri kontrol eden planlanmış işler çalıştırırsınız.
- İnce ayarlı izin kontrolü isteyebilirsiniz (para çekemeyen ancak siparişler oluşturabilen bir anahtar).
- Etkileşimli bir oturum açma akışı olmadan çalışan kimlik bilgileri gerekir.

API anahtarları kendi başlarına hiçbir zaman sona ermez. Açıkça iptal edene kadar geçerli kalırlar. Bu, uzun süreli hizmetler için uygun olsa da dikkatli yönetim gerektirir.

### JWT Belirteci Kimlik Doğrulaması

JWT belirteçleri, oturum açtıktan sonra verilen kısa ömürlü kimlik bilgileridir. E-posta ve şifreniz (veya OAuth aracılığıyla) ile kimlik doğrularsınız, JWT alırsınız ve `Bearer` önekiyle `Authorization` başlığı aracılığıyla isteklere dahil edersiniz.

```bash
curl https://merx.exchange/api/v1/balance \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

JWT'ler şu durumlarda doğru seçimdir:

- İnsan kullanıcı, API'yi çağıran bir ön uçla etkileşime girer.
- Otomatik olarak süresi dolan sınırlı süreli erişim istiyorsunuz.
- Oturum açma akışı olan bir web veya mobil uygulama oluşturuyorsunuz.

MERX tarafından verilen JWT belirteçleri 24 saat sonra sona erer. Süresi dolduktan sonra, istemci yeni bir belirteç almak için yeniden kimlik doğrulaması yapmalıdır. Yenileme belirteçleri, kullanıcının yeniden oturum açmasını gerektirmeden oturumları uzatır.

### Hangisini Kullanmalı

Programlı entegrasyonlar (ödeme botları, otomatik ticaret, arka uç hizmetleri) için API anahtarları kullanın. Yönetimi daha basit, ayrıntılı izinleri destekler ve oturum açma akışı gerektirmez.

İnsan oturum açışının olduğu kullanıcıya yönelik uygulamalar için JWT belirteçleri kullanın. Oturum tabanlı erişim sağlarlar ve otomatik olarak sona ererler; bu da istemci taraflı koddan kimlik bilgisi sızıntısı riskini azaltır.

Aynı sistemde her ikisini de kullanabilirsiniz. Yaygın bir desen: Yönetici paneliniz için JWT kimlik doğrulaması (insan operatörleri ayarları yönetmek için oturum açarlar), arka uç hizmetleriniz için API anahtarı kimlik doğrulaması (otomatik sipariş oluşturma, denge izleme).

## API Anahtarları Oluşturma ve Yönetme

### Anahtar Oluşturma

API anahtarlarını API'nin kendisi aracılığıyla veya MERX web panosundan oluşturun. API uç noktası `POST /api/v1/keys`:

```bash
curl -X POST https://merx.exchange/api/v1/keys \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": "production-order-bot",
    "permissions": ["create_orders", "view_orders", "view_balance"]
  }'
```

Yanıt, tam API anahtarını içerir. Bu, tam anahtarın döndürüldüğü tek seeferdir. Hemen sırlar yöneticinize saklayın.

```json
{
  "id": "key_8f3k2m9x",
  "name": "production-order-bot",
  "key": "merx_live_k7x9m2p4q8r1s5t3u7v2w6x0y4z...",
  "permissions": ["create_orders", "view_orders", "view_balance"],
  "created_at": "2026-03-30T10:00:00Z"
}
```

Not: anahtar oluşturma uç noktası JWT kimlik doğrulaması gerektirir. API anahtarları oluşturmak için önce oturum açmalısınız. Bu kasıtlı bir güvenlik kararıdır - API anahtarları diğer API anahtarları oluşturamaz.

### JavaScript SDK'yı Kullanma

```javascript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
  baseUrl: 'https://merx.exchange/api/v1',
});

// SDK otomatik olarak X-API-Key başlığını dahil eder
const prices = await merx.prices.list();
const balance = await merx.account.getBalance();
```

### Python SDK'yı Kullanma

```python
from merx_sdk import MerxClient

client = MerxClient(
    api_key="merx_live_k7x9m2p4...",
    base_url="https://merx.exchange/api/v1",
)

prices = client.get_prices()
balance = client.get_balance()
```

### Anahtarları Listeleme ve İptal Etme

Hesabınızın tüm etkin anahtarlarını listeleyin:

```bash
curl https://merx.exchange/api/v1/keys \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

Bir anahtarı hemen iptal edin:

```bash
curl -X DELETE https://merx.exchange/api/v1/keys/key_8f3k2m9x \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

İptal etme anlıktır. İptal edilen anahtarı kullanan herhangi bir istek hemen 401 döndürür. Bir uyum süresi yoktur.

## İzin Türleri

MERX API anahtarları ayrıntılı izinleri destekler. Bir anahtar oluştururken, tam olarak hangi işlemleri gerçekleştirebileceğini belirtirsiniz. Gerekli izni olmayan bir anahtar, 403 Yasak yanıtı alır.

### Mevcut İzinler

| İzin | Açıklama | Tipik Kullanım Durumu |
|-----------|-------------|-----------------|
| `view_balance` | Hesap bakiyesini ve işlem geçmişini oku | İzleme panoları, uyarılar |
| `view_orders` | Sipariş durumunu ve sipariş geçmişini oku | Sipariş izleme, raporlama |
| `create_orders` | Yeni enerji ve bandwidth siparişleri oluştur | Otomatik ticaret botları |
| `broadcast` | İmzalı işlemleri yayın için gönder | Özel işlem iş akışları |

### İzin Tasarım İlkeleri

**En az ayrıcalık.** Her anahtara yalnızca gerekli olan izinleri verin. İzleme panosu `create_orders`'a ihtiyaç duymaz. Fiyat görüntüleme widget'ı `view_balance`'a ihtiyaç duymaz.

**Farklı kaygılar için ayrı anahtarlar.** Sipariş botunuz için bir anahtar (`create_orders`, `view_orders`, `view_balance`) ve izleme sisteminiz için farklı bir anahtar (`view_balance`, `view_orders`) kullanın. İzleme anahtarı sızarsa, saldırgan siparişler oluşturamaz.

**API anahtarlarında para çekme izni yok.** Para çekme işlemleri JWT kimlik doğrulaması gerektirir. Bu kasıtlıdır. Tam izinlere sahip bir API anahtarı bile, hesabınızdan para çekemez. Bu, en yüksek riskli işlem için bir koruma katmanı ekler.

### Örnek: Fiyat Widget'ı İçin Minimal Anahtar

Herkese açık bir fiyat widget'ı yalnızca fiyatları getirmeye ihtiyaç duyar. Fiyatlar uç noktası herkese açık olduğundan hiçbir kimlik doğrulamaya ihtiyaç duymaz, ancak kullanımı izlemek isterseniz:

```bash
curl -X POST https://merx.exchange/api/v1/keys \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "website-price-widget",
    "permissions": []
  }'
```

Boş izinler dizisine sahip bir anahtar, herkese açık uç noktalara (fiyatlar, sağlayıcı listesi) erişebilirken, istek hacmini izlemenizi ve anahtar başına hız sınırlarını uygulamanızı sağlar.

### Örnek: Tam Ticaret Botu Anahtarı

```bash
curl -X POST https://merx.exchange/api/v1/keys \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "trading-bot-prod",
    "permissions": ["create_orders", "view_orders", "view_balance"]
  }'
```

## Hız Sınırları

MERX, anahtar başına değil, uç nokta kategorisi başına hız sınırları uygular. Tüm hız sınırları, aynı kimliğini doğrulanmış (API anahtarı veya JWT oturumu) kimlik tarafından dakika başına istekler cinsinden ölçülür.

### Hız Sınırı Tablosu

| Uç Nokta Kategorisi | Hız Sınırı | Örnekler |
|------------------|-----------|---------|
| Fiyat verileri | 300/dak | `GET /prices`, `GET /prices/history` |
| Sipariş oluşturma | 10/dak | `POST /orders` |
| Sipariş sorguları | 60/dak | `GET /orders`, `GET /orders/:id` |
| Para çekme | 5/dak | `POST /withdraw` |
| Hesap verileri | 60/dak | `GET /balance`, `GET /keys` |
| Anahtar yönetimi | 10/dak | `POST /keys`, `DELETE /keys/:id` |

### Hız Sınırı Başlıkları

Her yanıt, HTTP başlıklarında hız sınırı bilgisini içerir:

```
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 287
X-RateLimit-Reset: 1711785660
```

- `X-RateLimit-Limit` - geçerli pencerede izin verilen maksimum istek.
- `X-RateLimit-Remaining` - sınıra ulaşılmadan önce kalan istek.
- `X-RateLimit-Reset` - pencere sıfırlandığında Unix zaman damgası.

### Sınıra Ulaştığınızda

Hız sınırını aşmak, beklenecek saniye sayısını belirten `Retry-After` başlığı ile HTTP 429 Çok Fazla İstek döndürür:

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Retry after 12 seconds.",
    "details": {
      "limit": 10,
      "window": "1m",
      "retry_after": 12
    }
  }
}
```

### Koddaki Hız Sınırlarını İşleme

SDK'lar, yapılandırılabilir yeniden deneme davranışı ile hız sınırlarını otomatik olarak işler:

```javascript
const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
  maxRetries: 3,
  retryOnRateLimit: true, // 429'de otomatik olarak bekle ve yeniden dene
});
```

Ham HTTP istemcileri için, `Retry-After` başlığına dayalı geri alma uygulayın:

```python
import requests
import time

def merx_request(method, path, **kwargs):
    url = f"https://merx.exchange/api/v1{path}"
    headers = {"X-API-Key": API_KEY}

    for attempt in range(3):
        response = requests.request(method, url, headers=headers, **kwargs)

        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", 10))
            print(f"Rate limited. Waiting {retry_after}s...")
            time.sleep(retry_after)
            continue

        response.raise_for_status()
        return response.json()

    raise Exception("Rate limit retries exhausted")
```

## Güvenlik En İyi Uygulamaları

### Anahtarları Ortam Değişkenlerinde Depolayın

API anahtarlarını asla kaynak kodunda değil, ortam değişkenlerinde veya bir sırlar yöneticisinde saklayın:

```bash
# .env dosyası (bunu asla işlemek yok)
MERX_API_KEY=merx_live_k7x9m2p4q8r1s5t3u7v2w6x0y4z...
```

```javascript
// Ortamdan yükleyin
const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
});
```

### Anahtarları Düzenli Olarak Döndürün

Yeni bir anahtar oluşturun, hizmetlerinizi onu kullanacak şekilde güncelleyin, ardından eski anahtarı iptal edin. MERX aynı anda birden fazla etkin anahtarı destekler, böylece kapalı kalma süresi olmadan döndürebilirsiniz:

1. Aynı izinlere sahip yeni bir anahtar oluşturun.
2. Yeni anahtarı hizmetlerinize dağıtın.
3. Yeni anahtarın üretimde çalıştığını doğrulayın.
4. Eski anahtarı iptal edin.

### Anahtar Kullanımını İzleyin

API anahtar listenizi periyodik olarak inceleyin. Artık kullanılmayan anahtarları iptal edin. Her anahtarın bir `last_used_at` zaman damgası vardır - eğer bir anahtar aylar boyunca kullanılmadıysa, iptal edilmesi için bir adayıdır.

### Anahtarları Asla İstemci Taraflı Kodda Açıklamayın

API anahtarları hiçbir zaman bir tarayıcıda çalışan JavaScript'te, mobil uygulama paketlerinde veya son kullanıcıların inceleyebileceği herhangi bir kodda görünmemelidir. Bir ön uçtan MERX'i çağırmanız gerekiyorsa, istekleri API anahtarını sunucu taraflında tutan arka uçunuz aracılığıyla iletin.

```
Tarayıcı -> Arka Uçunuz (API anahtarını tutar) -> MERX API
```

### Farklı Ortamlar İçin Ayrı Anahtarlar Kullanın

Geliştirme, hazırlık ve üretim için ayrı anahtarlar koruyun. Bir geliştirme anahtarı sızarsa, üretimi etkileyemez. MERX şu anda ortam kapsamlı anahtarları desteklemiyor, ancak adlandırma kuralları yardımcı olur:

```
dev-price-monitor
staging-order-bot
prod-order-bot
prod-balance-alerter
```

### Şüphe Durumunda Denetim Yapın

Bir anahtarın tehlikede olduğunu düşünürseniz, hemen iptal edin ve bir değişim oluşturun. Son sipariş ve para çekme geçmişinizi yetkisiz etkinlik açısından kontrol edin. MERX, tüm API anahtarı kullanımını IP adresleriyle günlüğe kaydeder; bu, yetkisiz erişim kaynağını belirlemeye yardımcı olabilir.

## Hepsini Bir Araya Getirmek

Üretim sistemi için tipik bir tam kimlik doğrulama kurulumu şöyle görünür:

1. Web panosu aracılığıyla oturum açarak JWT oturumu alın.
2. Her hizmet için ayrı API anahtarları oluşturun: sipariş botu, izleme, raporlama.
3. Her anahtara minimum izinler atayın.
4. Anahtarları sırlar yöneticinizde veya ortam değişkenlerinde saklayın.
5. Otomatik yeniden deneme ile hız sınırı işlemeyi uygulayın.
6. Düzenli bir programa göre anahtar döndürme ayarlayın (üç aylık makuldür).
7. Anahtar kullanımını izleyin ve kullanılmayan anahtarları iptal edin.

Kimlik doğrulama, her MERX entegrasyonunun temeldir. Bunu doğru yapmak için bir saat harcamak, hata ayıklamada ve güvenlik olaylarında günler kaydeder.

- Platform ve pano: [merx.exchange](https://merx.exchange)
- Tam API referansı: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- Python SDK: [github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)


## Şimdi AI ile Deneyin

Claude Desktop'a veya herhangi bir MCP uyumlu istemciye MERX ekleyin -- kurulum yok, yalnızca okunur araçlar için API anahtarı gerekli değildir:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI aracınıza sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve bağlı tüm sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgelendirmesi: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)