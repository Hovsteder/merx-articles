# MERX Panosu: Kod Yazmadan Enerji Alışverişi

Her enerji alıcısı bir geliştirici değildir. Her geliştirici tek seferlik bir satın alma için kod yazmak istemez. Ve her kuruluşun TRON işlem maliyetlerinden tasarruf etmeye başlamadan önce bir API'yi entegre etmek için mühendislik kapasitesi yoktur.

[merx.exchange](https://merx.exchange) adresindeki MERX panosu, enerji satın almak, sağlayıcı fiyatlarını karşılaştırmak, bakiyenizi yönetmek, siparişleri izlemek ve API anahtarları oluşturmak için -- tek bir kod satırı yazılmadan -- tam bir web arayüzü sağlar. Bu makale panonun tüm özelliklerini ele almakta ve etkili kullanımını göstermektedir.

## Başlarken

[merx.exchange](https://merx.exchange) adresine gidin ve bir hesap oluşturun. Kayıt bir e-posta adresi ve şifre gerektirir. KYC yok. Kimlik doğrulaması yok. Bekleme süresi yok. Hesabınız hemen aktif olur.

Oturum açtıktan sonra ana pano görünümüne ulaşırsınız. Arayüz koyu bir tema izler ve yüksek kontrastlı tipografi kullanır -- görsel gürültü yok, gereksiz dekorasyon yok, sadece satın alma kararlarını vermek için ihtiyacınız olan bilgiler.

## Fiyat Paneli

Panoda ilk gördüğünüz şey canlı fiyat panelidir. Bu, tümleştirilmiş yedi sağlayıcıdan gelen güncel enerji fiyatlarını gösterir ve her 30 saniyede güncellenir.

```
Sağlayıcı     1s Fiyat    1g Fiyat    Kullanılabilir
---------------------------------------------------
Feee          28 SUN      22 SUN      Evet
Netts         31 SUN      24 SUN      Evet
itrx          32 SUN      23 SUN      Evet
CatFee        30 SUN      25 SUN      Evet
PowerSun      33 SUN      26 SUN      Evet
TronSave      35 SUN      28 SUN      Evet
SoHu          34 SUN      27 SUN      Evet
```

Panel her süre katmanı için en ucuz sağlayıcıyı vurgular. Fiyatlar SUN cinsinden gösterilir (enerji birimi başına) -- sağlayıcıların oranlarını dahili olarak nasıl belirttiğinden bağımsız olarak doğrudan karşılaştırma yapmanızı sağlayan standart birim.

### Fiyatların Anlamı

Her fiyat, belirtilen kiralama süresi için birim enerji başına maliyeti temsil eder. Satın almanızın toplam maliyetini hesaplamak için:

```
Toplam maliyet = enerji_miktarı * birim_fiyat

Örnek:
  65.000 enerji * 28 SUN/birim = 1.820.000 SUN = 1,82 TRX
```

Pano bir sipariş girdiğinizde bu hesaplamayı otomatik olarak gerçekleştirir. Onaylamadan önce toplam maliyeti hem SUN hem de TRX cinsinden görsünüz.

### Fiyat Geçmişi

Canlı fiyatların altında, geçmiş 24 saat, 7 gün veya 30 günde fiyatların nasıl hareket ettiğini gösteren geçmiş bir grafik vardır. Bu, örüntüleri belirlemenize yardımcı olur -- fiyatlar genellikle yoğun olmayan saatlerde (00:00-08:00 UTC) daha düşük ve yoğun işlem dönemlerinde daha yüksektir.

Grafik sadece bilgi amaçlı değildir. Büyük bir sipariş veriyorsanız ve zaman esnekliği varsa, bir fiyat düşüşü için birkaç saat beklemek anlamlı bir yüzde tasarrufu yapabilir.

## Sipariş Oluşturma

Sipariş oluşturma formu panonun temel işlevidir. İşlem şu şekildedir:

### Adım 1: Sipariş Parametrelerini Girin

Üç alanı doldurun:

- **Enerji miktarı**: İhtiyacınız olan enerji birimi sayısı. Emin değilseniz, pano yaygın miktarlar için hızlı seçim düğmeleri sağlar:
  - 32.000 (minimal USDT transferi)
  - 65.000 (standart USDT transferi)
  - 200.000 (DEX swapı)
  - 500.000 (karmaşık kontrat etkileşimi)
  - Özel miktar

- **Hedef adres**: Enerji delegasyonunu alacak TRON adresi. Bu, enerji gerektiren işlemi yürütecek adresidir. Pano ilerlemeden önce adres biçimini doğrular.

- **Süre**: Enerjiye ne kadar süre ihtiyacınız olduğu. Seçenekler tipik olarak 1 saat, 1 gün, 3 gün, 7 gün, 14 gün ve 30 günü içerir.

### Adım 2: Teklifi İnceleyin

Parametrelerinizi girdikten sonra, pano detaylı bir teklif görüntüler:

```
Sipariş Özeti
-------------------------------------------------
Enerji:           65.000 birim
Süre:             1 saat
Hedef:            TYourAddress...
En iyi sağlayıcı: Feee
Birim başına fiyat: 28 SUN
Toplam maliyet:   1.820.000 SUN (1,82 TRX)
Hesap bakiyesi:   50,00 TRX
Sonraki bakiye:   48,18 TRX
```

Teklif, tam olarak hangi sağlayıcının siparişi yerine getireceğini, birim başına fiyatı, toplam maliyeti ve hesap bakiyeniz üzerindeki etkisini gösterir. Gizli ücret veya marj yoktur.

### Adım 3: Onaylayın ve Gerçekleştirin

Onay düğmesine tıklayın. Sipariş MERX arka ucuna gönderilir; bu da onu seçilen sağlayıcıya yönlendirir. Delegasyon tipik olarak saniyeler içinde tamamlanır.

Sipariş yerine getirildikten sonra, pano sipariş durumunu, zincir üstü delegasyon işlem hash'ini ve kiralama üzerindeki kalan zamanı göstermek için güncellenir.

## Bakiyenizi Yönetme

### Fon Yatırma

Enerji satın almadan önce, MERX'te bir TRX bakiyesine ihtiyacınız vardır. Para yatırma işlemi:

1. Bakiye bölümüne gidin
2. "Para Yatır"a tıklayın
3. Pano benzersiz para yatırma adresinizi görüntüler
4. Herhangi bir TRON cüzdanından bu adrese TRX gönderin
5. Para yatırma, zincir üstü onaylamadan sonra otomatik olarak kredi verilir

Para yatırma monitör servisi gelen işlemleri sürekli olarak izler. Krediler tipik olarak işlem zincir üstünde onaylandıktan 1-2 dakika içinde görünür.

### Bakiye Geçmişini Görüntüleme

Bakiye bölümü tüm bakiye değişikliklerinin tam geçmişini gösterir:

```
Tarih/Saat           Tür           Miktar        Bakiye
-----------------------------------------------------------
2026-03-28 14:30     Para Yatır    +100,00 TRX   100,00 TRX
2026-03-28 14:35     Sipariş #1247 -1,82 TRX     98,18 TRX
2026-03-28 16:20     Sipariş #1248 -1,95 TRX     96,23 TRX
2026-03-29 09:00     Para Yatır    +50,00 TRX    146,23 TRX
2026-03-29 09:15     Sipariş #1249 -5,40 TRX     140,83 TRX
```

Her giriş MERX muhasebe sistemindeki bir çift yönlü defter kaydına karşılık gelir. Tüm girdilerin toplamı her zaman mevcut bakiyenizle uzlaşır -- bu doğrulanabilir ve denetlenebilirdir.

### Para Çekme

Eğer MERX hesabınızdan TRX çekmek istiyorsanız:

1. Bakiye bölümüne gidin
2. "Para Çek"e tıklayın
3. Hedef TRON adresini girin
4. Çekilecek miktarı girin
5. Para çekme işlemini onaylayın

Para çekme işlemleri, hazine-imzalayıcı servisi tarafından işlenir ve TRON ağına yayınlanır. İşleme süresi ağ koşullarına bağlıdır ancak tipik olarak birkaç dakika içinde tamamlanır.

## Sipariş Geçmişi

Sipariş Geçmişi görünümü, tüm geçmiş siparişlerinizin aranabilir, filtrelenebilir bir listesini sağlar. Her giriş şunları gösterir:

- **Sipariş ID**: Referans için benzersiz tanımlayıcı
- **Tarih/Saat**: Siparişin ne zaman verildiği
- **Enerji Miktarı**: Kaç enerji birimi satın alındığı
- **Süre**: Kiralama dönemi
- **Sağlayıcı**: Siparişi hangi sağlayıcının yerine getirdiği
- **Fiyat**: SUN cinsinden birim başına fiyat
- **Toplam Maliyet**: TRX cinsinden tahsil edilen toplam tutar
- **Durum**: Beklemede, Aktif, Tamamlandı veya Başarısız
- **Hedef Adres**: Enerji nereye delege edildiği

### Filtreleme ve Arama

Siparişleri şu ölçütlere göre filtreleyebilirsiniz:

- Tarih aralığı
- Durum (aktif, tamamlandı, tümü)
- Sağlayıcı
- Hedef adres

Bu, özellikle birden çok adres için enerji satın alan ve cüzdan veya uygulama başına maliyetleri izlemesi gereken kuruluşlar için faydalıdır.

### Sipariş Ayrıntıları

Herhangi bir siparişe tıklamak, tam bilgisinin yer aldığı bir ayrıntı görünümünü açar:

```
Sipariş #1247
-------------------------------------------------
Durum:              Tamamlandı
Oluşturuldu:        2026-03-28 14:35:22 UTC
Enerji:             65.000 birim
Süre:               1 saat (süresi doldu)
Sağlayıcı:          Feee
Fiyat:              28 SUN/birim
Toplam:             1,82 TRX
Hedef:              TYourAddress...
TX Hash:            7f3a2b...
Delegasyon Başlangıcı: 2026-03-28 14:35:30 UTC
Delegasyon Sonu:      2026-03-28 15:35:30 UTC
```

İşlem hash'i zincir üstü delegasyon kaydına bir bağlantıdır; herhangi bir TRON blok tarayıcısı aracılığıyla doğrulanabilir.

## API Anahtarları Oluşturma

MERX'i uygulamanıza programlama yoluyla entegre etmeye hazır olduğunuzda, pano desteğe ihtiyaç duymadan API anahtarları oluşturmanıza olanak tanır.

1. Ayarlar veya API bölümüne gidin
2. "API Anahtarı Oluştur"a tıklayın
3. Anahtar için bir etiket sağlayın (örn. "Üretim sunucusu", "Hazırlama ortamı")
4. Anahtar bir kez görüntülenir -- hemen kopyalayın
5. Anahtar sunucuda şifreli olarak depolanır; tekrar alınamaz

Her biri kendi etiketi olan birden çok API anahtarını yönetebilirsiniz. Bir anahtar tehlikeye girerse, panelden iptal edin ve yeni bir tane oluşturun. Bir anahtarı iptal etmek anında gerçekleşir -- iptal edilen anahtarı kullanan tüm istekler bir kimlik doğrulaması hatası ile başarısız olur.

### API Anahtarı Güvenliği

API anahtarları MERX REST API, WebSocket akışları ve SDK'lara yönelik isteklerinizi doğrular. Onlara parolalar gibi davranın:

- API anahtarlarını asla sürüm denetimine işlemeyin
- Onları ortam değişkenlerinde veya gizli yöneticilerde saklayın
- Geliştirme, hazırlama ve üretim için ayrı anahtarlar kullanın
- Anahtarları periyodik olarak döndürün

Pano her anahtar için son kullanılan zaman damgasını gösterir; bu sayede kullanılmayan anahtarları belirlemeyi ve iptal etmeyi kolaylaştırır.

## Aktif Delegasyonları İzleme

Pano, aktif enerji delegasyonlarınızın gerçek zamanlı görünümünü içerir:

```
Aktif Delegasyonlar
-------------------------------------------------
Hedef            Enerji     Süresi Doluyor      Kalan
TAddr1...        65.000     2026-03-30 15:00    2s 30d
TAddr2...        200.000    2026-03-31 09:00    20s 30d
TAddr3...        500.000    2026-04-02 12:00    3g 2s 30d
```

Her aktif delegasyon için şunları yapabilirsiniz:

- Tam bitiş zamanını görmek
- Kalan zamanı görmek
- Ne kadar enerji bulunduğunu görmek
- Aynı adres için yeni bir sipariş vererek delegasyonu uzatmak

Sistem şu anda aktif bir delegasyonu erken iptal etmeyi desteklemez (TRON'daki enerji delegasyonu, belirtilen süresi boyunca çalışan bir zincir üstü işlemdir), ancak aynı hedef adres için ek enerji satın alarak her zaman mevcut bir delegasyonu uzatabilirsiniz.

## Enerji Tahmini Aracı

Pano, yerleşik bir enerji tahmini aracı içerir. İşleminizin ne kadar enerji gerekeceğinden emin değilseniz, bunu doğrudan panodan simüle edebilirsiniz:

1. Kontrat adresini girin (örn. USDT kontratı)
2. İşlevi seçin (örn. transfer)
3. Parametreleri girin (alıcı adresi, miktar)
4. "Tahmin Et"e tıklayın

Araç, arka planda `triggerConstantContract` çağrısı yapar ve geçerli kontrat durumuna karşı spesifik işleminiz için gereken tam enerjiyi döndürür. Bu, tahmin etme ihtiyacını ortadan kaldırır ve aşırı veya yetersiz enerji satın almayı engeller.

## Pano Kimin İçin?

### İşletme Operatörleri

USDT ödemeleri gönderen bir işletme yönetiyorsanız -- bordro, satıcı ödemeleri, remitanslar -- bir gelişticiye API entegrasyonu için ihtiyacınız yoktur. Panoyu açın, TRX yatırın ve her transfer partisinden önce enerji satın almaya başlayın. Maliyet tasarrufu hemen ve önemlidir.

### MERX'i Değerlendiren Geliştiriciler

Bir API entegrasyonuna taahhüt etmeden önce, hizmeti test etmek için panoyu kullanın. Birkaç sipariş verin, fiyatlandırmayı gözlemleyin, delegasyonların zincir üstünde beklendiği gibi gelip gelmediğini doğrulayın. İkna olduktan sonra, bir API anahtarı oluşturun ve programlama erişimine geçin.

### Finans Ekipleri

Sipariş geçmişi ve bakiye görünümleri, finans ekiplerinin ihtiyaç duyduğu raporlamayı sağlar: ne harcandı, ne zaman, ne için, hangi sağlayıcıdan. Bu verileri iç muhasebe sistemleriyle uzlaştırma için dışa aktarın.

### Ara Sıra Kullanıcılar

TRON işlemleri ara sıra yapıyorsanız -- haftada veya ayda birkaç işlem -- pano muhtemelen ihtiyacınız olan herşeydir. Entegrasyon yok, kod yok, bakım yok. Sadece oturum açın, enerji satın alın ve işlem ücretlerinde %90 tasarruf edin.

## Panodan API'ye

Pano ve API aynı arka ucu paylaşır. Panoda gerçekleştirdiğiniz her işlem -- fiyatları kontrol etmek, sipariş vermek, geçmişi görmek -- doğrudan bir API uç noktasına eşlenir. Otomasyona hazır olduğunuzda:

```
Pano işlemi               API eşdeğeri
-----------------------------------------------------------
Fiyatları görüntüle       GET /api/v1/prices
Sipariş oluştur           POST /api/v1/orders
Siparişi görüntüle        GET /api/v1/orders/:id
Bakiyeyi görüntüle        GET /api/v1/balance
Enerji tahmin et          POST /api/v1/estimate
```

Pano kullanıcısından API kullanıcısına geçiş sorunsuzdur. Hesabınız, bakiye ve sipariş geçmişi aktarılır. Tek ek, panoda oluşturduğunuz API anahtarıdır.

Tam dokümantasyon: [https://merx.exchange/docs](https://merx.exchange/docs)
Platform: [https://merx.exchange](https://merx.exchange)


## Şimdi AI ile Deneyin

MERX'i Claude Desktop veya herhangi bir MCP uyumlu istemciye ekleyin -- sıfır kurulum, salt okunur araçlar için API anahtarı gerekmez:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI aracınıza şunları sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatlar alın.

Tam MCP dokümantasyonu: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)