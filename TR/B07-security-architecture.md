# MERX Güvenlik Mimarisi: Fonlarınız Nasıl Korunur

Bir platform finansal işlemleri yönettiğinde, güvenlik bir özellik değildir - her şeyin oturduğu temeldir. Tek bir zafiyet kullanıcı güvenini kalıcı olarak yok edebilir. MERX'te güvenlik hususları, ilk günden itibaren her mimari kararı şekillendirmiştir; sonradan eklenen bir düşünce değil.

Bu makale MERX'in güvenlik mimarisini detaylandırır: fonlar nasıl korunur, veri bütünlüğü nasıl sağlanır, sistem yaygın saldırı vektörlerine nasıl karşı koyar ve platformu esnek kılan tasarım ilkeleri.

---

## İlke 1: Özel Anahtarların Saklanmaması

MERX hiçbir zaman TRON özel anahtarlarınızı tutmaz, depolamaz veya erişim sağlamaz. Bu, tüm bir saldırı sınıfını ortadan kaldıran temel bir tasarım kararıdır.

### Nasıl Çalışır

MERX'i enerji satın almak için kullandığınızda, delegasyon sağlayıcının adresinden sizin adresinize gerçekleşir. MERX bu işlemi yönetir ancak hiçbir zaman özel anahtarınıza erişmesi gerekmez. Özel anahtarınız cihazınızda, donanım cüzdanınızda veya yönettiğiniz başka yerde kalır.

Akış şöyledir:

```
1. MERX'e şunu söylersiniz: "65.000 enerjiyi TMyAddress'e deleget et"
2. MERX sağlayıcıya şunu söyler: "65.000 enerjiyi TMyAddress'e deleget et"
3. Sağlayıcı TProviderAddress'ten TMyAddress'e delegasyon yapar
4. MERX delegasyonu zincirde doğrular
5. Özel anahtarınız hiçbir zaman işin içinde değildir
```

### Neden Bu Önemli

MERX'in güvenliği ihlal edilse, saldırganlar TRX veya jetonlarınızı çalamazlar çünkü MERX'in anahtarlarınız yok. Bunu, tokenleri platform tarafından kontrol edilen bir adrese yatırmanız gereken platformlarla karşılaştırın - bu platformlar anahtarlarınızı (veya fonlarınıza giden anahtarları) tutar, bu da tek bir başarısızlık noktası oluşturur.

### Hazine İstisnası

MERX depozito almak ve para çekme işlemlerini işlemek için kendi hazine adresini yönetir. Hazine özel anahtarı, Docker secret olarak saklanır ve yalnızca `treasury-signer` hizmeti tarafından erişilebilir. API hizmetine, web ön ucuna veya başka herhangi bir bileşene hiçbir zaman açıklanmaz. Bu izolasyon hakkında aşağıda daha fazla bilgi.

---

## İlke 2: Çift Giriş Defteri

MERX'teki her finansal işlem, eşleştirilmiş bir defter girişi oluşturur. Bu, son 700 yılın her bankası ve finansal kurumu tarafından kullanılan aynı muhasebe ilkesidir. Çalışır.

### Nasıl Çalışır

Her işlem iki giriş oluşturur: bir borç ve bir alacak. Tüm borçların toplamı her zaman tüm alacakların toplamına eşittir. Eğer yoksa, bir şey yanlış demektir ve sistem bunu hemen algılar.

```sql
-- Sipariş ödeme örneği
INSERT INTO ledger (account_id, type, amount_sun, direction)
VALUES
  ($user_id, 'ORDER_PAYMENT', 5525000, 'DEBIT'),
  ($provider_settlement, 'ORDER_PAYMENT', 5525000, 'CREDIT');
```

### Değişmezlik

Defter kayıtları hiçbir zaman güncellenmez veya silinmez. Bir işlemin tersine çevrilmesi gerekirse (örn. geri ödeme), ters yönde yeni bir defter girişi oluşturulur:

```sql
-- Geri ödeme: yeni giriş, orijinal giriş kalır
INSERT INTO ledger (account_id, type, amount_sun, direction, reference_id)
VALUES
  ($user_id, 'REFUND', 5525000, 'CREDIT', $original_order_id),
  ($provider_settlement, 'REFUND', 5525000, 'DEBIT', $original_order_id);
```

Orijinal borç girişi hiçbir zaman değiştirilmez. Geri ödeme alacak girişi açıkça orijinal girişe referans verir, tam bir denetim izi oluşturur.

### Neden Bu Önemli

Saldırganlar uygulama katmanının güvenliğini ihlal eder ve kullanıcının bakiyesini artırmaya çalışırsa, defter girişleri dengelenmiş olmayacaktır. Düzenli mutabakat kontrolleri bunu hemen algılar:

```sql
-- Mutabakat sorgusu: her zaman 0 döndürmelidir
SELECT SUM(CASE direction
  WHEN 'DEBIT' THEN amount_sun
  WHEN 'CREDIT' THEN -amount_sun
END) as imbalance
FROM ledger;
```

Sıfır olmayan herhangi bir sonuç anında bir uyarı ve araştırma tetikler.

---

## İlke 3: Atomik Bakiye İşlemleri

Her bakiye değişikliği, yarış koşullarını önlemek için `SELECT FOR UPDATE` kullanır. Bu isteğe bağlı değildir - veritabanı düzeyinde uygulanır.

### Yarış Koşulu Sorunu

Uygun kilitlemek olmadan, 10 TRX bakiyesi olan bir kullanıcı 8 TRX'lik iki eş zamanlı sipariş sunabilir:

```
Thread 1: SELECT balance WHERE user_id = 1    -> 10 TRX
Thread 2: SELECT balance WHERE user_id = 1    -> 10 TRX
Thread 1: balance (10) >= order (8)? EVET     -> devam et
Thread 2: balance (10) >= order (8)? EVET     -> devam et
Thread 1: UPDATE balance = 10 - 8 = 2 TRX
Thread 2: UPDATE balance = 10 - 8 = 2 TRX

Sonuç: Kullanıcı 10 TRX'le 16 TRX harcadı
```

### Çözüm

```sql
BEGIN;

-- Satırı kilitle - ikinci işlem burada bekler
SELECT balance_sun FROM accounts
WHERE user_id = $1
FOR UPDATE;

-- Bakiyeyi kontrol et
-- Yetersiz: ROLLBACK yap
-- Yeterli: devam et

UPDATE accounts
SET balance_sun = balance_sun - $order_amount
WHERE user_id = $1
  AND balance_sun >= $order_amount;  -- UPDATE'de tekrar kontrol

COMMIT;
```

`FOR UPDATE` satır düzeyinde bir kilit alır. İkinci işlem, ilk işlem işlenene veya geri alınana kadar bekler. İlk işlem tamamlandıktan sonra (bakiye 2 TRX'ye düşer), ikinci işlem güncellenmiş bakiyeyi (2 TRX) okur ve yetersiz fon siparişini doğru şekilde reddeder.

---

## İlke 4: Girdi Doğrulaması

Tüm girdiler işlenmeden önce Zod şemaları ile doğrulanır. Buna API istekleri, webhook yükleri, sağlayıcı yanıtları ve iç hizmet iletileri dahildir.

### API Girdi Doğrulaması

```typescript
const CreateOrderSchema = z.object({
  energy: z.number()
    .int('Enerji bir tamsayı olmalıdır')
    .min(10000, 'Minimum sipariş 10.000 enerjidir')
    .max(100000000, 'Maksimum sipariş 100.000.000 enerjidir'),

  targetAddress: z.string()
    .regex(/^T[1-9A-HJ-NP-Za-km-z]{33}$/, 'Geçersiz TRON adresi formatı')
    .refine(isValidTronAddress, 'Geçersiz TRON adresi sağlama toplamı'),

  duration: z.enum(['1h', '1d', '3d', '7d', '14d', '30d']),

  maxPrice: z.number()
    .positive()
    .optional(),

  idempotencyKey: z.string()
    .max(255)
    .optional()
});
```

Her alan türünü belirtilmiş, sınırlandırılmış ve doğrulanmıştır. Ham kullanıcı girdisi hiçbir zaman iş mantığına veya veritabanı sorgularına ulaşmaz.

### SQL Enjeksiyonu Önleme

Tüm veritabanı sorguları parametreli ifadeleri kullanır. Dize birleştirme hiçbir zaman SQL oluşturmak için kullanılmaz:

```typescript
// Hiçbir zaman bunu yapma:
const query = `SELECT * FROM users WHERE id = '${userId}'`;  // SQL enjeksiyonu

// Her zaman bunu yap:
const query = 'SELECT * FROM users WHERE id = $1';
const result = await db.query(query, [userId]);
```

Bu kod incelemesi ve linting kuralları tarafından uygulanır. Kodda ham SQL dize interpolasyonu için hiçbir mekanizma yoktur.

### TRON Adresi Doğrulaması

TRON adresleri birden fazla seviyede doğrulanır:

1. **Format kontrolü**: TRON adresi regex'i ile eşleşmelidir (T ile başlar, 34 karakter, base58).
2. **Sağlama toplamı doğrulaması**: adres yazım hatalarını tespit eden bir sağlama toplamı içerir.
3. **Zincir üstü doğrulama** (isteğe bağlı): adresi doğrulayın ve etkinleştirilip etkinleştirilmediğini onaylayın.

Enerjiyi geçersiz bir adrese göndermek kaynakları boşa harcatır ve zincirde tersine çevrilemez. Kesin doğrulama bunu önler.

---

## İlke 5: Hizmet İzolasyonu

MERX, her biri minimal izinler ve gereksiz erişim olmadan izole edilmiş Docker kapsayıcıları seti olarak çalışır.

### Kapsayıcı Mimarisi

```
Docker ağı:
  |
  |-- api           (port 3000, herkese açık)
  |-- price-monitor (harici port yok)
  |-- order-executor (harici port yok)
  |-- ledger        (harici port yok)
  |-- treasury-signer (harici port yok, Docker secret erişimi)
  |-- deposit-monitor (harici port yok)
  |-- withdrawal-executor (harici port yok)
  |
  |-- postgresql    (port 5432, sadece dahili)
  |-- redis         (port 6379, sadece dahili)
```

### Temel İzolasyon Özellikleri

- **API hizmeti hazine özel anahtarına erişemez.** Yalnızca `treasury-signer` kapsayıcısı anahtarı içeren Docker secret'ı okuyabilir.
- **Fiyat monitörü bakiyeleri değiştiremez.** Yalnızca sağlayıcı API'lerine okuma erişimi ve Redis fiyat kanallarına yazma erişimi vardır.
- **Sipariş yürütücüsü defteri doğrudan değiştiremez.** Kapatma olaylarını Redis'e yayınlar, bu da defter hizmeti tarafından tüketilir.
- **PostgreSQL ve Redis harici olarak açıklanmamıştır.** Yalnızca Docker ağı içinden erişilebilir.

### Neden Bu Önemli

Saldırganlar API hizmetinin (en açık bileşen) güvenliğini ihlal etseler de, şunları yapamaz:
- Hazine özel anahtarına erişim (farklı kapsayıcı, Docker secret).
- Defter giriş doğrudan değiştir (farklı hizmet, defter tablolarına yazma erişimi yok).
- Bakiye kontrollerini atla (veritabanı düzeyinde SELECT FOR UPDATE ile uygulanır).

Herhangi bir hizmet düzeyinde ihlal yapmanın hasar alanı tasarım gereği sınırlıdır.

---

## İlke 6: Hız Sınırlandırması ve Suistimal Önleme

### API Hız Sınırları

Her API uç noktasının amacına uygun hız sınırları vardır:

```
Genel uç noktalar (fiyatlar, sağlık):     100 istek/dakika
Doğrulanmış okumalar (siparişler, bakiye): 60 istek/dakika
Doğrulanmış yazımlar (sipariş oluştur):   30 istek/dakika
Para çekme:                                5 istek/dakika
```

Hız sınırları API anahtarı başına uygulanır, Redis'te kayan pencereler ile izlenir.

### Para Çekme Güvenlik Önlemleri

Para çekme en yüksek riskli işlemdir (gerçek varlıkları platformdan çıkarmak). Ek güvenlik önlemleri şunları içerir:

- **Hız sınırlandırması**: dakikada maksimum 5 çekme isteği.
- **Miktar sınırları**: hesap başına günlük çekme sınırları.
- **Doğrulama gecikmesi**: büyük çekmeler bir bekleme süresini tetikler.
- **Bakiye doğrulaması**: `SELECT FOR UPDATE` yeterli bakiyeyi sağlar.
- **İdempotanslık**: yinelenen çekme istekleri (aynı idempotency anahtarı) orijinal sonucu döndürür.

---

## İlke 7: Webhook Güvenliği

MERX sipariş durumu güncellemeleri, depozitolar ve diğer olaylar için webhook bildirimleri gönderir. Webhook'lar sahteciliği önlemek için HMAC-SHA256 ile imzalanır.

### HMAC Webhook'ları Nasıl Çalışır

```
1. MERX hesaplar: HMAC-SHA256(webhook_body, your_webhook_secret)
2. MERX imzayı X-Merx-Signature başlığına ekler
3. Sunucunuz HMAC'ı aynı secret ile yeniden hesaplar
4. İmzalar eşleşirse: orijinal webhook. Yoksa: sahte, atla.
```

### Kod'da Doğrulama

```typescript
import crypto from 'crypto';

function verifyWebhook(body: string, signature: string, secret: string): boolean {
  const computed = crypto
    .createHmac('sha256', secret)
    .update(body)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(computed)
  );
}
```

Zamanlama saldırılarını önlemek için `timingSafeEqual` kullanımına dikkat edin. Saf bir dize karşılaştırması (`===`) yanıt süresi varyasyonları aracılığıyla doğru imza hakkında bilgi sızıtacaktır.

---

## İlke 8: Gizli Bilgiler Yönetimi

Hiçbir secret hiçbir zaman kaynak kodunda sabit kodlanmaz. Tüm hassas değerler ortam değişkenleri ve Docker secret'ları aracılığıyla yönetilir.

### Ortam Değişkenleri

```
# .env (hiçbir zaman git'e işlenmez)
DATABASE_URL=postgresql://...
REDIS_URL=redis://...
API_JWT_SECRET=...
WEBHOOK_SIGNING_SECRET=...
TRON_API_KEY=...
```

### Docker Secret'ları (Yüksek Hassasiyetli Değerler İçin)

Hazine özel anahtarı ortam değişkenleri için çok hassastır (günlük alınabilir veya işlem incelemesi aracılığıyla sızabilir). Docker secret olarak saklanır:

```yaml
# docker-compose.yml
services:
  treasury-signer:
    secrets:
      - treasury_private_key

secrets:
  treasury_private_key:
    file: /run/secrets/treasury_key
```

Docker secret'ları kapsayıcı içinde dosya olarak monte edilir, yalnızca hizmet süreci tarafından okunabilir. Ortam değişkeni listelerinde, kapsayıcı inceleme çıktısında veya günlüklerde görünmez.

### Git Koruması

`.gitignore` dosyası tüm hassas dosyaları hariç tutar:

```
.env
*.key
*.pem
secrets/
```

Bu ilk commit'ten önce kurulur, değil sonra.

---

## İzleme ve Olaylara Yanıt

### Otomatik Uyarılar

Aşağıdaki koşullar anında uyarı tetikler:

- Defter dengesizliği algılandı (borç-alacak uyuşmazlığı).
- Hazine bakiyesi eşik altına düşer.
- Başarısız kimlik doğrulama denemeleri eşik aşarsa (10/dakika IP başına).
- Sağlayıcı API beklenmeyen hatalar döndürür.
- Sipariş yürütme başarısızlık oranı %5'i aşarsa.

### Denetim Günlüğü

Her güvenlikle ilgili işlem yapılandırılmış veriler ile günlüğe kaydedilir:

```json
{
  "event": "withdrawal_requested",
  "user_id": "usr_abc123",
  "amount_sun": 10000000,
  "destination": "TAddress...",
  "ip": "203.0.113.45",
  "timestamp": "2026-03-30T12:00:00Z"
}
```

Günlükler adli analiz ve uyum için saklanır. Bunlar yalnızca ekleme ve uygulama verilerinden ayrı olarak saklanır.

---

## Sonuç

MERX'teki güvenlik, tek bir özellik değildir, ancak birbirine bağlı ilkeler kümesidir: anahtar saklama değil, çift giriş muhasebesi, atomik bakiye işlemleri, katı girdi doğrulaması, hizmet izolasyonu, hız sınırlandırması, imzalı webhook'lar ve uygun secret yönetimi. Her ilke belirli bir tehdit vektörünü ele alır ve birlikte, herhangi bir bileşenin güvenliğini ihlal etmenin sistemi tehlikeye atmadığı derinlemesine savunma mimarisi yaratırlar.

Hiçbir sistem, tam korunaklı değildir. Fakat güvenliği mimari içine entegre ederek tasarlararak - daha sonra düzeltmek yerine - MERX saldırı yüzeyini en aza indirir ve saldırganın zarar vermesi için ödeyeceği maliyeti arttırır.

Açık kaynak bileşenleri inceleyin: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js), [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python), [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp).

MERX'i [https://merx.exchange](https://merx.exchange)'de kullanmaya başlayın.

---

*Bu makale MERX teknik serisinin bir parçasıdır. MERX, güvenliği temel bir gereksinim olarak tasarlanan ilk blokzincir kaynakları değişimidir, yan düşünce değil.*


## Şimdi AI ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin -- sıfır kurulum, salt-okunur araçlar için API anahtarı gerekmez:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI aracınızdan şunu sorun: "TRON enerjisinin en ucuz fiyatı şu anda nedir?" ve bağlantılı tüm sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)