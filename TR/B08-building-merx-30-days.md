# Fikirden Üretime: 30 Günde MERX İnşa Etmek

MERX, konseptten canlı üretim sistemine 30 günde geçti. Bir açılış sayfası değil. Bir prototip değil. Yedi sağlayıcı entegrasyonu, gerçek zamanlı fiyat toplaması, zincir üstü sipariş yürütme, çift girişli muhasebe, kapsamlı belgeler, iki dilde SDK ve AI ajanı entegrasyonu için 55 araçlı bir MCP sunucusu ile tamamen operasyonel bir blockchain kaynağı değişim platformu.

Bu makale, bunun nasıl gerçekleştiğinin teknik hikayesidir - mimari kararlar, çözdüğümüz sorunlar, kasıtlı olarak almadığımız kısayollar ve önemli olan şeylerde taviz vermeden finansal bir platform hızlıyla inşa etmekten çıkardığımız dersler.

---

## Gün 0: Problem İfadesi

TRON enerji pazarı parçalanmıştır. Yedi veya daha fazla sağlayıcı enerji delegasyon hizmetleri sunarak, her birinin kendi API'ı, fiyatlandırması ve güvenilirlik özellikleri vardır. En iyi fiyatı istiyorsanız, hepsinin tümüyle entegre olmanız gerekir. Yedeklemeli sistem istiyorsanız, yönlendirme mantığı inşa etmeniz gerekir. Şeffaflık istiyorsanız, izleme sistemi inşa etmeniz gerekir.

TRON üzerinde USDT gönderen her işletme bu entegrasyon vergisine maruz kalır. Çözüm, bir toplama katmanıdır - çok sağlayıcılı yönlendirmeyi, en iyi fiyat seçimini ve otomatik yedeklemeyi işleyen tek bir API.

Henüz kimse bunu inşa etmemişti. Biz bunu yapmaya karar verdik.

---

## Hafta 1: Temel

### Mimari Öncelikli

Tek satır kod yazmadan önce, mimari üzerine iki gün harcadık. Sonuç, veritabanı şemasından API hata biçimlerine kadar renk onaltılık kodlarına kadar her şeyi kapsayan 40 bölümlü bir mimari belgesi oldu. Bu belge, her uygulama kararı için tek gerçek kaynağı haline geldi.

Bu iki gün içinde alınan anahtar mimari kararlar:

**Karar 1: Birinci günden itibaren mikro hizmetler.**

Mikro hizmetler moda olduğu için değil, ama finansal sistemlerin izolasyona ihtiyacı olduğu için. Hazine imzalayıcı API hizmetinden erişilebilir olmamalıdır. Fiyat monitörünün kullanıcı bakiyelerine yazma erişimi olmamalıdır. Docker konteynerları bu izolasyonu doğal olarak sağlar.

```
services/
  api/              HTTP/WebSocket API
  price-monitor/    Provider price polling
  order-executor/   Order routing and execution
  ledger/           Double-entry accounting
  deposit-monitor/  Incoming payment detection
  treasury-signer/  Transaction signing (isolated)
```

**Karar 2: PostgreSQL + Redis, egzotik veritabanları yok.**

ACID garantisi gerektiren her şey için PostgreSQL (bakiyeler, siparişler, defter girişleri). Hız gerektiren her şey için Redis (fiyat önbelleği, pub/sub, hız sınırlaması). İkisi de savaş yolundan çıkmış, iyi belgelenmiş ve operasyonel olarak basittir.

**Karar 3: Tüm tutarlar SUN cinsinden.**

Her finansal değer SUN'da tamsayı olarak depolanır (1 TRX = 1.000.000 SUN). Finansal yolda hiçbir yerde kayan nokta yoktur. Bu, ilk fonksiyonumuzu yazmadan önce tüm bir hata kategorisini ortadan kaldırdı.

**Karar 4: Hizmetler için Node.js + TypeScript, eşleştirme motoru için Go.**

Sistem büyüklüğünün çoğu için TypeScript - hızlı geliştirme, güçlü yazım, API ve izleme iş yükleri için mükemmel async I/O. Performansın önemli olduğu eşleştirme motoru için Go ayrılmıştır.

### Veritabanı Şeması

Veritabanı geçişleri 3. gün üzerinde yazılmıştır. Her tablo finansal bütünlük göz önünde bulundurularak tasarlanmıştır:

```sql
-- Temel ilke: her bakiye mutasyonu bir defter girişi oluşturur
CREATE TABLE ledger (
  id BIGSERIAL PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(id),
  type VARCHAR(50) NOT NULL,
  amount_sun BIGINT NOT NULL,
  direction VARCHAR(6) NOT NULL CHECK (direction IN ('DEBIT', 'CREDIT')),
  reference_type VARCHAR(50),
  reference_id UUID,
  balance_before BIGINT NOT NULL,
  balance_after BIGINT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- UPDATE veya DELETE tetiklemesi yok - defter sadece eklemeye özgüdür
```

Geçiş başına bir dosya, `YYYYMMDD_description.sql` şeklinde adlandırılmıştır. 30 günün sonunda 14 geçiş dosyası vardı, her biri toplamsal, yok edici değildir.

### Sağlayıcı Arayüzü

`IEnergyProvider` arayüzü 4. gün tanımlanmıştır. Bu, her sağlayıcı adaptörünün uygulanacağı sözleşmedir:

```typescript
interface IEnergyProvider {
  name: string;
  getPrices(): Promise<ProviderPriceResponse>;
  createOrder(params: OrderParams): Promise<OrderResult>;
  getOrderStatus(orderId: string): Promise<OrderStatus>;
  healthCheck(): Promise<boolean>;
}
```

Bu arayüz hiçbir zaman değişmedi. Aşağıdaki haftalarda yedi sağlayıcı, her biri kendi dosyasında, hiçbiri çekirdek sisteme değişiklik gerektirmeden bu arayüze karşı entegre edildi.

---

## Hafta 2: Temel Hizmetler

### Fiyat Monitörü

Fiyat monitörü canlı gitmesi gereken ilk hizmet oldu. Her sağlayıcıyı 30 saniyede bir inceler, fiyatları normalleştirir, Redis'e yayımlar ve geçmişi PostgreSQL'de saklar. Uygulama üç dosya genelinde kabaca 180 satır TypeScript'tir.

En zor kısım yoklama mantığı değil - normalleştirmedir. Her sağlayıcı fiyatları biraz farklı biçimlerde döndürür:

- Sağlayıcı A: Enerji birimi başına SUN
- Sağlayıcı B: Sabit enerji tutarı için toplam TRX
- Sağlayıcı C: Enerji birimi başına SUN, ancak farklı minimum sipariş ile
- Sağlayıcı D: Hacim tabanlı kademeli fiyatlandırma

Her adaptör, sağlayıcısının biçimini standart `ProviderPriceResponse` içine çevirir. Fiyat monitörü sağlayıcı tuhaftlıkları umursamaz; yalnızca normalleştirilmiş veriler görür.

### Sipariş Yürütücüsü

Sipariş yürütücüsü en karmaşık hizmettir. Redis'ten fiyatları okur, optimal yönlendirmeyi belirler, siparişleri sağlayıcılara gönderir, zincir üstü doğrulama için izler ve kapatma etkinlikleri yayımlar.

Yedeklemeli sistem zinciri kritik tasarım öğesiydi. Sağlayıcı A başarısız olursa, Sağlayıcı B'yi deneyin. B başarısız olursa, C'yi deneyin. Alıcının API çağrısı, herhangi bir sağlayıcı operasyonel olduğu sürece başarılı olur.

```
Order received -> Read prices -> Select cheapest
  -> Execute at Provider A
    -> Success? Verify on-chain -> Settle
    -> Failure? Try Provider B
      -> Success? Verify on-chain -> Settle
      -> Failure? Try Provider C
        -> ... and so on
```

### Defter Hizmeti

Defter hizmeti çift girişli kısıtlamayı uygular. Her bakiye mutasyonu eşleştirilmiş girişler oluşturur. Hizmet saatte bir kez mutabakat kontrolü çalıştırır:

```sql
SELECT SUM(CASE direction
  WHEN 'DEBIT' THEN amount_sun
  WHEN 'CREDIT' THEN -amount_sun
END) FROM ledger;
-- Must be 0. If not: alert immediately.
```

30 gün geliştirme ve test döneminde, bu kontrol hiçbir zaman çalışmadı. Kısıtlama hiçbir zaman ihlal edilmedi çünkü mimari ihlalleri sadece olası değil, yapısal olarak imkansız kıldı.

---

## Hafta 3: API, Ön Uç ve Zincir Üstü Doğrulama

### API Tasarımı

API, katı sürümlemeyle (`/api/v1/...`) REST kurallarına uyar. Her uç nokta, uygulamadan önce tasarlandı:

```
GET    /api/v1/prices          Current prices from all providers
GET    /api/v1/prices/best     Best current price
POST   /api/v1/orders          Create a new order
GET    /api/v1/orders/:id      Get order status
GET    /api/v1/balance         Get account balance
POST   /api/v1/deposit         Get deposit address
POST   /api/v1/withdraw        Request withdrawal
```

Hata yanıtları tutarlı bir biçim kullanır:

```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Account balance (5.2 TRX) is insufficient for this order (8.1 TRX)",
    "details": {
      "balance": 5200000,
      "required": 8100000
    }
  }
}
```

Hiçbir uç nokta, tüm girdiler üzerinde Zod doğrulaması olmadan yayımlanmadı.

### Ön Uç

Ön uç, katı bir tasarım sistemi olan bir Next.js uygulamasıdır: yalnızca koyu tema, 2px üzerinde köşe yok, gradyan yok, gölge yok, başlıklar için Cormorant Garamond, her şey diğer için IBM Plex Mono. Görsel kimlik mimari belgede tanımlanmış ve sadık bir şekilde uygulanmıştır.

### Zincir Üstü Doğrulama

Her sipariş TRON blokzincirinde doğrulanır. Doğrulama hizmeti delegasyon işlemlerini izler ve enerjinin hedef adrese ulaştığını doğrular. Bu, blokzincir doğrulama süreleri değişken ve sağlayıcı işlem biçimleri farklı olduğu için en zorlayıcı entegrasyon oldu.

Test aşamasında sekiz ana ağ işlemi doğrulandı, uçtan uca akış - API çağrısından zincir üstü delegasyona - gerçek TRX ve gerçek sağlayıcılarla doğru bir şekilde çalıştığını doğruladı.

---

## Hafta 4: SDK'lar, MCP Sunucusu ve Belgeler

### JavaScript SDK

JavaScript SDK, Node.js ve tarayıcı ortamları için inşa edilmiştir:

```typescript
import { MerxClient } from 'merx-sdk';

const client = new MerxClient({ apiKey: 'your-key' });
const prices = await client.getPrices({ energy: 65000 });
const order = await client.createOrder({
  energy: 65000,
  targetAddress: 'TAddress...',
  duration: '1h'
});
```

Kaynak: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)

### Python SDK

Python SDK, JavaScript SDK'sının API yüzeyini yansıtır:

```python
from merx import MerxClient

client = MerxClient(api_key='your-key')
prices = client.get_prices(energy=65000)
order = client.create_order(
    energy=65000,
    target_address='TAddress...',
    duration='1h'
)
```

Kaynak: [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)

### MCP Sunucusu: 52 Araç

MCP (Model Context Protocol) sunucusu belki de en ileriye dönük bileşen oldu. MERX işlevselliğini AI ajanlarının doğrudan kullanabileceği araçlar olarak ortaya koymaktadır.

MCP sunucusu başlangıç sürümünde 7 araçtan 30 günün sonunda 55 araçla büyümüştür:

```
Account management:    create_account, login, get_balance, get_deposit_info
Price data:            get_prices, get_best_price, compare_providers, analyze_prices
Order management:      create_order, get_order, list_orders, create_standing_order
Resource monitoring:   check_address_resources, estimate_transaction_cost
TRON utilities:        validate_address, convert_address, get_trx_balance
On-chain operations:   transfer_trx, transfer_trc20, approve_trc20
Analytics:             calculate_savings, get_price_history, suggest_duration
... and 30 more
```

Kaynak: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)

### Belgeler

Belgeler 5 sayfadan 36 sayfaya yeniden inşa edildi, tam API referansı, SDK kılavuzları, TRON kavramları ve entegrasyon öğreticilerini kapsayan. Belgeler [https://merx.exchange/docs](https://merx.exchange/docs) adresinde bulunmaktadır.

Ek olarak, 4 SEO kılavuz sayfası ve 7 sağlayıcı karşılaştırma sayfası yayımlandı, sitemap'i 53 URL'ye getirdi.

---

## Taviz Vermediğimiz Şeyler

Hız, köşe kesme baskısı oluşturur. İşte kasıtlı olarak kesmediğimiz köşeler:

### Para için Kayan Nokta Yok

Tüm finansal değerler için tamsayılar (SUN) kullanmak, ekran biçimlendirmesinde karmaşıklık ekledi ama yuvarlama hatalarını tamamen ortadan kaldırdı. Her test durumu tam olarak beklenen değerleri eşleştirdi.

### SQL için String Birleştirme Yok

Her veritabanı sorgusu parametreli deyimler kullanır. Bu birinci günden beri tartışılmaz bir kuraldı. SQL enjeksiyonu çözülen bir problemdir ve biz onu çözüldü tutmaya devam ettik.

### Sabit Sırlar Yok

Birinci günden itibaren ortam değişkenleri. Hazine anahtarı için Docker gizlilikleri. `.gitignore` ilk commit'ten önce ayarlanmıştır.

### Hizmetler Doğrudan Durumu Paylaşmıyor

Hizmetler Redis pub/sub veya REST API çağrıları aracılığıyla iletişim kurar. Hizmetler arasında doğrudan içe aktarma yok. Bu bağımsız dağıtımı mümkün kıldı ve çağlı hataları önledi.

### Defter Mutasyonları Yok

İlk göçten itibaren sadece eklemeli defter. Defter tablolarında UPDATE veya DELETE yok. Düzeltmeler yeni girişler oluşturur, modifikasyonlar değil.

---

## Neler Öğrendik

### Ders 1: Mimari Belgeler Kendileri İçin Ödeme Yapıyor

Mimari üzerine harcanan iki gün haftaların yeniden yapılandırmasını kaydetti. Her geliştirici sorusu belgede cevaplandı. Her tasarım anlaşmazlığı spesifikasyona başvurarak çözüldü. 40 bölüm bürokratik havale değildi; bunlar sorunları bug haline gelmeden önce düşünmeye zorlayan bir mekanizmaydı.

### Ders 2: Sağlayıcı API'leri Güvenilmez

Entegre edilen yedi sağlayıcıdan, en az ikisi 30 günlük yapı döneminde kapalı kalma yaşadı. Yedeklemeli sistem teorik bir zarafet değildi - test döneminin ilk haftasında uygulandı.

### Ders 3: Adaptör Deseni Demirbaş Değişikliğine Değer

Aynı arayüzü uygulayan yedi adaptör yazmak tekrarlı göründü. Ama Sağlayıcı C 22. gün API yanıt biçimini değiştirdiğinde, bir dosya güncelledik ve başka hiçbir şey değişmedi. Adaptörü güncelleme süresi 10 dakika versus her çağrı sitesini güncelleme süresi gün, desenin değerini açıkça yaptı.

### Ders 4: MCP, Hizmet Entegrasyonunun Geleceğidir

MCP sunucusu başlangıçta bir deneydi. Ama AI ajanlarının enerji tedarikini özerk şekilde yönetmek için MERX araçlarını kullanmasını izlemek bir açıklama oldu. Bu, hizmetlerin gelecekte nasıl tüketileceğidir - insan geliştiriciler entegrasyon kodu yazmayarak değil, AI ajanları doğrudan araç API'lerini çağırarak.

### Ders 5: 200 Satır Dosya Sınırı Bir Özellik

Proje genelinde katı bir 200 satır dosya sınırı uygulandı. Bu sürekli ayrıştırmaya zorladı. Fonksiyonlar küçük kaldı. Sorumluluklar net kaldı. Bir dosya 200 satıra yaklaştığında, bölünmenin zamanı oldu ve bölünme her zaman açıklığı iyileştirdi.

---

## Rakamlar Bazında

```
Architecture document:     40 sections
Services:                  9 Docker containers
Provider integrations:     7
Database migrations:       14
API endpoints:             20+
MCP tools:                 52 (from initial 7)
SDK languages:             2 (JavaScript, Python)
Documentation pages:       36 (from initial 5)
Sitemap URLs:              53
Mainnet transactions:      8 verified
Commission rate:           0%
Days to production:        30
```

---

## Bundan Sonra Ne Geliyor

Platform [https://merx.exchange](https://merx.exchange) adresinde canlıdır. Yakın odak, test, optimizasyon ve ilk üretim kullanıcılarını eklemeye odaklandır. Temel katı - mimari yatay ölçeklendirmeyi destekler, yeni sağlayıcılar saatler içinde eklenebilir ve sıfır komisyon modeli benimseme uyuşmazlığını kaldırır.

TRON üzerindeki enerji toplama pazarı bunu basit kılan bir platform için bekliyor. MERX bu platformdur.

---

*MERX ilk blockchain kaynağı değişimidir. Platform'u [https://merx.exchange](https://merx.exchange) adresinde keşfedin. Belgeler [https://merx.exchange/docs](https://merx.exchange/docs) adresinde. GitHub'da açık kaynak SDK'lar ve MCP sunucusu.*


## Şimdi AI ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin -- sıfır kurulum, salt okunur araçlar için API anahtarına gerek yok:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI ajanınızdan şunu sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)