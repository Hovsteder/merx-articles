# İdempotency Anahtarları: TRON Energy Siparişleri için Güvenli Yeniden Denemeler

Ağ istekleri başarısız olur. Bağlantılar zaman aşımına uğrar. Yük dengeleyiciler paketleri düşürür. Mobil istemciler isteğin ortasında sinyal kaybeder. Herhangi bir dağıtılmış sistemde, soru başarısızlığın olup olmayacağı değil, olduğunda bunu nasıl ele alacağınızdır.

Çoğu API çağrısı için, cevap basittir: isteği yeniden deneyin. Ancak istek para içerdiğinde - bakiyenizi borçlandıran bir sipariş oluşturmak veya fonları blok zincirinde hareket ettiren bir çekimi başlatmak - naif bir yeniden deneme yıkıcı olabilir. İsteği gönderirsiniz, sunucu onu işler, ancak yanıt size asla ulaşmaz. Yeniden denersiniz. Sunucu onu tekrar işler. Aynı sipariş için iki kez ödeme yaptınız, hatta daha kötüsü, iki tane blok zinciri çekişini tetiklediniz.

Bu, idempotency anahtarlarının çözdüğü problemdir. MERX, her finansal uç noktada idempotency uygulayarak, herhangi bir isteği yinelenen yürütme riski olmadan yeniden denemeyi güvenli hale getirir.

## İdempotency Pratikte Ne Anlama Gelir

Bir işlem idempotent'tir eğer birden fazla kez gerçekleştirilmesi, bir kez gerçekleştirilmişle aynı sonucu üretirse. HTTP GET doğal olarak idempotent'tir - aynı URL'yi on kez getirmek aynı veriyi döndürür. HTTP DELETE kural gereği idempotent'tir - zaten silinmiş bir kaynağı silmek işlem yapmamaktır.

HTTP POST idempotent değildir. `/api/v1/orders` adresine üç kez POST göndermek üç sipariş oluşturur, üç kez ödeme yaparsınız.

İdempotency anahtarları POST isteklerini idempotent davranması sağlar. İstemci benzersiz bir tanımlayıcı oluşturur ve isteğin yanına gönderir. Sunucu bu anahtarı kullanarak çoğaltmalar tespit eder. Aynı anahtar tekrar gelirse, sunucu isteği yeniden işlemek yerine ilk yürütmeden elde edilen sonucu döndürür.

Ayrım önemlidir: sunucu çoğaltmaları reddetmez. Orijinal yanıtı döndürür. İstemci perspektifinden, yeniden deneme tam olarak başarılı bir ilk girişim gibi davranır. Bu otomasyon için kritiktir - kodunuz "ilk başarılı çağrı" ile "başarılı yeniden deneme" arasında ayrım yapmak zorunda değildir.

## TRON Energy Ticareti için Neden Bu Önemlidir

TRON energy siparişleri gerçek para ve gerçek sistemler içerir. MERX aracılığıyla bir sipariş oluşturduğunuzda, sırayla birkaç şey olur:

1. Hesap bakiyeniz borçlandırılır.
2. Sipariş en ucuz mevcut sağlayıcıya yönlendirilir.
3. Sağlayıcı hedef adresinize blok zincirde energy delegesi verir.
4. Sipariş durumu güncellenir ve web kancaları çalışır.

Bağlantı 1. adımdan sonra ancak onay almadan önce koparsa, siparişin oluşturulup oluşturulmadığını bilmenin hiçbir yolu yoktur. İdempotency olmadan, seçenekleriniz kötüdür:

- **Yeniden denemeyin.** Hakkında bilgi sahibi olmadığınız bekleyen bir siparişiniz olabilir veya sunucuya hiç ulaşmayan bir isteği kaybetmiş olabilirsiniz. Bunu öğrenmek için sipariş listesini yoklamanız gerekir, bu da karmaşıklık ekler.
- **Körlü bir şekilde yeniden deneyin.** İlk istek gerçekten gittiyse, artık iki siparişiniz ve iki bakiye borçlandırmanız var. Günde yüzlerce sipariş işleyen otomatik bir sistem için, bu hızla birikir.
- **İdempotency anahtarıyla yeniden deneyin.** Sunucu çoğaltmayı tanır, mevcut siparişi döndürür ve kodunuz normal devam eder. Çift harcama yok. El ile mutabakat yok.

Aynı mantık çekişlere de uygulanır. Yinelenen bir çekim isteği fonları blok zincirde iki kez hareket ettirebilir. İdempotency anahtarlarıyla, yeniden deneme mevcut çekim kaydını döndürür.

## MERX İdempotency Uygulamasını Nasıl Yapıyor

MERX, `Idempotency-Key` başlığını iki uç noktada destekler:

- `POST /api/v1/orders` - energy ve bandwidth sipariş oluşturma
- `POST /api/v1/withdraw` - bakiye çekilişleri

Uygulama, basit bir yaşam döngüsü izler.

### Anahtar Format ve Oluşturma

İdempotency anahtarı, istemci tarafından oluşturulan, 64 karaktere kadar bir dizedir. MERX belirli bir biçim uygulamaz, ancak en iyi uygulama UUIDler (v4) veya bağlam kodlayan yapılandırılmış tanımlayıcılar kullanmaktır:

```
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

Veya işletme bağlamı ile:

```
Idempotency-Key: order-user42-2026-03-30-batch7-item3
```

Anahtar, her ayrı işlem için benzersiz olmalıdır. Farklı istek parametreleriyle bir anahtarı yeniden kullanmak (farklı miktar, farklı hedef adres) yeni parametreleri sessizce görmezden gelmek yerine bir hata döndürür.

### Sunucu Tarafı Davranışı

MERX bir `Idempotency-Key` başlığı içeren bir istek aldığında, aşağıdaki mantık çalışır:

1. **Bu anahtarla ilk istek.** Sunucu isteği normal şekilde işler - parametreleri doğrular, bakiye borçlandırır, sipariş oluşturur. Anahtarı, istek karmasını ve yanıtı saklar. HTTP 201 ile yanıt döndürür.

2. **Aynı anahtar ve aynı parametrelerle yinelenen istek.** Sunucu tüm işlemeyi atlar ve ilk yürütmeden saklanan yanıtı döndürür. Yanıt aynıdır, aynı sipariş ID'si, durum ve zaman damgaları dahil olmak üzere. HTTP 200 döndürür (201 değil), böylece istemci gerekirse ayırt edebilir.

3. **Aynı anahtar ancak farklı parametrelerle yinelenen istek.** Sunucu HTTP 409 Çatışma döndürür. Bu, bir anahtar çarpışmasının ilgisiz bir siparişin döndürülmesine neden olduğu hafif bir hatayı önler.

4. **İlk hala işleniyorken istek.** Sunucu HTTP 409 ile orijinal isteğin hala devam etmekte olduğunu gösteren bir mesaj döndürür. Bu, yeniden denemenin ilk istek bitmeden önce geldiği yarış durumunu ele alır.

### Anahtar Süresi Dolması

İdempotency anahtarları 24 saat boyunca saklanır. Bundan sonra, aynı anahtar yeni bir istek için yeniden kullanılabilir. Pratikte, yeniden denemeler saniye veya dakika içinde olur, gün içinde değil. 24 saatlik pencere, herhangi bir gerçekçi yeniden deneme senaryosunu kapsayacak kadar cömerttir, ancak sınırsız depolama büyümesini önler.

## Kod Örnekleri

### JavaScript SDK - Yeniden Deneme Güvenli Sipariş Oluşturma

MERX JavaScript SDK, `idempotencyKey` seçeneğini geçtiğinizde idempotency anahtarlarını otomatik olarak ele alır. Yeniden deneme mantığıyla tam bir örnek aşağıdadır:

```javascript
import { MerxClient } from 'merx-sdk';
import { randomUUID } from 'crypto';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
  baseUrl: 'https://merx.exchange/api/v1',
});

async function createOrderSafe(params, maxRetries = 3) {
  const idempotencyKey = randomUUID();

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const order = await merx.orders.create(
        {
          energy_amount: params.energyAmount,
          target_address: params.targetAddress,
          duration_hours: params.durationHours,
        },
        {
          idempotencyKey,
        }
      );

      console.log(`Order created: ${order.id}, status: ${order.status}`);
      return order;
    } catch (err) {
      if (err.status === 409 && err.code === 'IDEMPOTENCY_CONFLICT') {
        // Same key with different params - this is a bug in our code
        throw new Error('Idempotency key conflict - parameters mismatch');
      }

      if (attempt === maxRetries) {
        throw err;
      }

      const delay = Math.min(1000 * Math.pow(2, attempt - 1), 10000);
      console.log(`Attempt ${attempt} failed, retrying in ${delay}ms...`);
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }
}

// Usage
const order = await createOrderSafe({
  energyAmount: 65000,
  targetAddress: 'TTargetAddressHere',
  durationHours: 1,
});
```

Bu uygulamadaki önemli noktalar:

- İdempotency anahtarı bir kez, yeniden deneme döngüsünden önce oluşturulur. Her yeniden deneme aynı anahtarı gönderir.
- Katlanarak geri alma, geçici arızalar sırasında sunucuya vurmayı önler.
- Parametre uyumsuzluğu üzerinde 409 çatışması, bir ağ sorunu değil bir mantık hatası gösterdiği için yeniden denenemez bir hata olarak ele alınır.

### Python SDK - Yeniden Deneme Güvenli Çekme

Python SDK aynı deseni izler. İşte çekimler için bir örnek:

```python
import uuid
import time
from merx_sdk import MerxClient, MerxAPIError

client = MerxClient(
    api_key="your_api_key",
    base_url="https://merx.exchange/api/v1",
)


def withdraw_safe(address: str, amount_sun: int, max_retries: int = 3):
    idempotency_key = str(uuid.uuid4())

    for attempt in range(1, max_retries + 1):
        try:
            result = client.withdraw(
                address=address,
                amount=amount_sun,
                idempotency_key=idempotency_key,
            )
            print(f"Withdrawal initiated: {result['id']}")
            return result

        except MerxAPIError as e:
            if e.status_code == 409:
                raise ValueError(
                    "Idempotency conflict - check parameters"
                ) from e

            if attempt == max_retries:
                raise

            delay = min(2 ** (attempt - 1), 10)
            print(f"Attempt {attempt} failed: {e}. Retrying in {delay}s...")
            time.sleep(delay)

    raise RuntimeError("All retry attempts exhausted")


# Usage
result = withdraw_safe(
    address="TWithdrawalAddressHere",
    amount_sun=100_000_000,  # 100 TRX
)
```

### Ham HTTP - curl ile Doğrudan API Çağrısı

SDK kullanmıyorsanız, `Idempotency-Key` başlığı standart bir HTTP başlığıdır:

```bash
IDEMPOTENCY_KEY=$(uuidgen)

curl -X POST https://merx.exchange/api/v1/orders \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $IDEMPOTENCY_KEY" \
  -d '{
    "energy_amount": 65000,
    "target_address": "TTargetAddressHere",
    "duration_hours": 1
  }'
```

Tamamen aynı curl komutunu (`Idempotency-Key` değeriyle) yeniden deneyin ve yeni bir tane yerine orijinal siparişi geri alacaksınız.

## Üretim Sistemleri için En İyi Uygulamalar

### Anahtarları Doğru Seviyede Oluşturun

İdempotency anahtarı, HTTP isteğini değil, işletme amacını temsil etmelidir. Bir kullanıcı bir kez "Energy Satın Al" seçeneğini tıklarsa, bu bir işletme işlemidir - kaç HTTP yeniden denemesi alacağından bağımsız olarak.

Her yeniden denemede yeni bir anahtar üretmeyin. Bu amacı tamamen bozar.

### Gönderilmeden Önce Anahtarları Saklayın

Bir kuyrukta sipariş işleyen bir sistemde, API çağrısını yapmadan önce idempotency anahtarını veritabanınıza yazın. İşleminiz kilitlense ve yeniden başlarsa, veritabanından aynı anahtarı alır ve güvenli bir şekilde yeniden dener.

```javascript
// Write the intent to your database first
const intent = await db.orderIntents.create({
  idempotencyKey: randomUUID(),
  energyAmount: 65000,
  targetAddress: 'TTargetAddressHere',
  status: 'pending',
});

// Now make the API call with the stored key
const order = await merx.orders.create(
  { energy_amount: intent.energyAmount, target_address: intent.targetAddress },
  { idempotencyKey: intent.idempotencyKey }
);

// Update the intent with the result
await db.orderIntents.update(intent.id, {
  status: 'completed',
  orderId: order.id,
});
```

### Tüm Yanıt Kodlarını İşleyin

Yeniden deneme mantığınız bu durumları ele almalıdır:

| HTTP Durum | Anlam | İşlem |
|------------|-------|-------|
| 201 | İlk başarılı oluşturma | Sonuç saklayın, devam edin |
| 200 | Yinelenen anahtar, aynı parametreler | Başarı olarak ele alın (aynı sonuç) |
| 409 | Anahtar çatışması veya hala işlenme | Yeniden denemeyin - araştırın |
| 429 | Hız sınırlı | Gecikmeden sonra yeniden deneyin |
| 500+ | Sunucu hatası | Geri almayla yeniden deneyin |

### Hata Ayıklama için Yapılandırılmış Anahtarlar Kullanın

UUID'ler mükemmel şekilde çalışırken, yapılandırılmış anahtarlar hata ayıklamayı kolaylaştırır:

```
order-{userId}-{date}-{sequenceNumber}
withdraw-{userId}-{timestamp}-{nonce}
```

Bir destek durumunu araştırırken, yapılandırılmış bir anahtar size hemen kim tarafından ve ne zaman işlem başlatıldığını söyler.

### Makul Yeniden Deneme Sınırları Belirleyin

Üç ile beş yeniden deneme, katlanarak geri alma ile geçici arızaların büyük çoğunluğunu kapsar. Sunucu gerçekten düştüyse, 50 kez yeniden deneme yardımcı olmaz ve sadece günlüklerde gürültü üretir.

Makul bir üst sınır: 3 kez yeniden deneyin, gecikmeler 1 saniye, 2 saniye ve 4 saniye. Hepsi başarısız olursa, hatayı sınırsız yeniden denemek yerine bir izleme sistemine sunun.

## Yaygın Hatalar

**Her yeniden denemede yeni bir anahtar oluşturma.** Bu en yaygın hatadır. Her yeniden denemenin yeni bir anahtarı varsa, sunucu her birini benzersiz bir istek olarak ele alır. Yinelenen siparişler alırsınız.

**Anahtarları farklı işlemler arasında paylaşma.** Her ayrı işletme işleminin kendi anahtarı gerekir. Aynı anahtarı iki farklı sipariş için kullanırsanız (farklı miktarlar, farklı adresler), ikinci bir 409 ile başarısız olur.

**200 vs 201 ayrımını işlememek.** Her ikisi de başarıyı gösterirken, durum kodu bunun ilk yürütme veya tekrar oynatma olup olmadığını söyler. Bu, günlüğe kaydetme ve ölçümler için yararlı olabilir - yeniden denemelerin duplikatlara ne sıklıkla çarptığını bilmek, ağ güvenilirliğiniz hakkında bir şeyler söyler.

**Web kancaları için idempotency görmezden gelme.** MERX, sipariş durumu değişikliklerinde web kancaları gönderir. Web kanca işleyiciniz de idempotent olmalıdır - aynı `order.filled` etkinliğini iki kez alırsanız, onu iki kez işlemek sorun yaratmamalıdır. Kendi işlemeniz için idempotency anahtarı olarak sipariş ID'sini kullanın.

## Sonuç

İdempotency anahtarları, API çağrılarınıza küçük bir eklentidir - bir başlık, bir UUID - ancak tüm finansal hata sınıfını ortadan kaldırırlar. TRON energy siparişleri oluşturan veya programlı olarak çekişleri başlatan herhangi bir sistem için, isteğe bağlı değildirler. Ağ arızalarını zarif bir şekilde işleyen bir sistem ile bağlantı koparıldığında her seferinde destek biletleri oluşturan bir sistem arasındaki fark budur.

MERX tüm finansal uç noktalarda idempotency anahtarlarını destekler. JavaScript SDK ve Python SDK ikisi de yerleşik destek sağlar. Birinci günden itibaren bunları kullanmaya başlayın - idempotency'yi zaten yinelenen siparişleri yaşamış bir üretim sistemine geriye dönük olarak eklemek, başlangıçtan itibaren inşa etmekten önemli ölçüde daha acılıdır.

- MERX platformu: [merx.exchange](https://merx.exchange)
- API belgeleri: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- Python SDK: [github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)


## Yapay Zeka ile Şimdi Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin - kurulum, salt okunur araçlar için API anahtarı gerekli değildir:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Yapay zeka aracınıza sorun: "Şu anda en ucuz TRON energy nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)