# MERX Tüm Enerji Sağlayıcılarını Tek Bir API'de Nasıl Toplar

2026 yılında TRON enerji pazarında bir parçalanma sorunu vardır. En az yedi büyük sağlayıcı enerji delegasyon hizmetleri sunmakta, her birinin kendi API'si, fiyatlandırma modeli ve kullanılabilirlik deseni vardır. En iyi fiyatı almak istiyorsanız, hepsinin tümüyle entegrasyon sağlamanız, fiyatlarını sürekli izlemeniz, bireysel özelliklerini yönetmeniz ve bir tanesi indiğinde yedek mantığı kurmanız gerekir.

Ya da MERX'e tek bir API çağrısı yapabilirsiniz.

Bu makale, MERX'in tüm büyük enerji sağlayıcılarını tek bir API'de nasıl topladığını açıklamaktadır - fiyat izleme, en iyi fiyata yönlendirme, otomatik yedekleme mimarisinin ardındaki mimari ve yedi entegrasyonu biriyle değiştirmekten kaynaklanan operasyonel basitleştirmeyi anlatan bir rehber.

---

## Sağlayıcı Ortamı

TRON enerji pazarı bağımsız olarak çalışan birden fazla sağlayıcıyı içerir. 2026 başı itibarıyla, büyük sağlayıcılar şunları içerir:

- **TronSave** - en erken enerji kiralama hizmetlerinden birisi
- **Feee** - API erişimi ile rekabetçi fiyatlandırma
- **itrx** - toplu enerji siparişlerine odaklanmış
- **CatFee** - orta pazar konumlandırması
- **Netts** - agresif fiyatlandırma ile yeni katılımcı
- **SoHu** - Çin pazarına odaklı sağlayıcı
- **PowerSun** - doğrudan stake ve delegasyon

Her sağlayıcının kendi özellikleri vardır:

- API formatı ve kimlik doğrulama yöntemi
- Fiyatlandırma yapısı (bazıları enerji birimi başına ücret alır, diğerleri TRX başına)
- Minimum ve maksimum sipariş büyüklükleri
- Mevcut süreler (1h, 1d, 3d, 7d, vb.)
- Desteklenen ödeme yöntemleri
- Durum sayfaları ve çalışma süresi özellikleri

### Entegrasyon Vergisi

Tek bir sağlayıcı ile entegrasyon basittir. En iyi fiyatlandırmayı elde etmek için tümünün tümüyle entegrasyonu önemli bir mühendislik çabası gerektirmektedir:

```
Sağlayıcı başına:
  - API istemci uygulaması:       2-3 gün
  - Fiyat normalizasyonu:         1 gün
  - Hata işleme:                  1 gün
  - Test etme:                    1-2 gün
  - Devam eden bakım:             2-4 saat/ay

7 sağlayıcı x 5-7 gün = 35-49 gün başlangıç entegrasyonu
7 sağlayıcı x 3 saat/ay = 21 saat/ay devam eden bakım
```

Bu, MERX'in ortadan kaldırdığı entegrasyon vergisidir. Yedi sağlayıcı entegrasyonunu korumak yerine, bir MERX entegrasyonunu korursunuz. MERX geri kalanını halleder.

---

## MERX Mimarisi

MERX, uygulamanız ile sağlayıcı ekosistemi arasında yer almaktadır. Mimari üç temel bileşenden oluşur:

### 1. Fiyat İzleyicisi

Fiyat izleyicisi, her entegre sağlayıcıyı sürekli olarak geçerli fiyatlandırma için sorgulayan özel bir hizmettir. Her 30 saniyede bir, her sağlayıcının API'sini sorgular, yanıtı standart bir biçime normalleştirir ve sonucu bir Redis pub/sub kanalına yayınlar.

```
Her 30 saniye:
  Her sağlayıcı için:
    1. Sağlayıcı API'sini geçerli fiyatlar için sorgula
    2. Standart biçime normalleştir (SUN başına enerji birimi)
    3. Yanıtı doğrula (aykırı değerleri, eski verileri reddet)
    4. Redis'e yayınla: kanal "prices:{provider}"
    5. Fiyat geçmişinde depola (PostgreSQL)
```

30 saniye aralığı bilinçli olarak seçilmiştir. Daha hızlı polling sağlayıcı API'larını strese sokacak ve minimal değer katacaktır (fiyatlar nadiren saniye bazında değişir). Daha yavaş polling eski fiyatları sunma riski taşır.

### 2. Redis Fiyat Önbelleği

Redis, gerçek zamanlı fiyat önbelleği olarak görev yapar. Fiyat izleyicisinden her fiyat güncellemesi Redis'te 60 saniyelik TTL (yaşam süresi) ile saklanır - polling aralığının iki katı. Bir sağlayıcının fiyat verileri 60 saniyeden eski ise, otomatik olarak sona erer ve yönlendirme kararlarından hariç tutulur.

```
Redis anahtar yapısı:
  prices:tronsave     -> { energy: 88, bandwidth: 2, updated: 1711756800 }
  prices:feee         -> { energy: 92, bandwidth: 3, updated: 1711756800 }
  prices:itrx         -> { energy: 85, bandwidth: 2, updated: 1711756800 }
  prices:catfee       -> { energy: 95, bandwidth: 3, updated: 1711756800 }
  ...

  prices:best         -> { provider: "itrx", energy: 85, updated: 1711756800 }
```

`prices:best` anahtarı her fiyat güncellemesinde yeniden hesaplanır ve API'ye tüm sağlayıcıları taramadan geçerli en iyi fiyata anında erişim sağlar.

### 3. Sipariş Yürütücüsü

MERX API'si aracılığıyla bir sipariş verdiğinizde, sipariş yürütücüsü bunu alır ve optimal yönlendirmeyi belirler:

```
Sipariş alındı: TBuyerAddress için 65.000 enerji

1. Redis'ten prices:best oku -> itrx at 85 SUN/unit
2. itrx tarafından 65.000 enerji kullanılabilirliğini kontrol et -> mevcut
3. Siparişi itrx'e gönder
4. Zincir üstü delegasyon onayını izle
5. Enerjinin TBuyerAddress'e ulaştığını doğrula
6. Alıcıyı bilgilendir (webhook + WebSocket)
```

En ucuz sağlayıcı siparişi dolduramaz ise (yetersiz stok, API hatası, zaman aşımı), yürütücü otomatik olarak sonraki en ucuz sağlayıcıya düşer.

---

## Fiyat Normalizasyonu

Farklı sağlayıcılar fiyatları farklı biçimlerde fiyatlandırır. Bazıları SUN başına enerji birimi olarak fiyatlandırır. Bazıları belirli bir enerji miktarı için toplam TRX olarak fiyatlandırır. Bazıları bandwidth'i fiyata dahil eder; diğerleri ayrı olarak ücret alır.

MERX her şeyi tek bir biçime normalleştirir:

```typescript
interface NormalizedPrice {
  provider: string;
  energyPricePerUnit: number;    // SUN başına enerji birimi
  bandwidthPricePerUnit: number; // SUN başına bandwidth birimi
  minOrder: number;              // Minimum enerji birimleri
  maxOrder: number;              // Maksimum enerji birimleri
  availableEnergy: number;       // Şu anda mevcut
  durations: string[];           // Desteklenen süreler
  lastUpdated: number;           // Unix zaman damgası
}
```

Bu normalizasyon kritiktir. Olmaksızın, sağlayıcılar arasında fiyat karşılaştırması, tüketicinin her sağlayıcının fiyatlandırma modelini anlaması gerekir. Bununla birlikte, fiyat karşılaştırması basit bir sayısal sıralamadır.

---

## Ayrıntıda En İyi Fiyata Yönlendirme

Yönlendirme algoritması sadece "en ucuzunu seç" değildir. Birkaç faktör yönlendirme kararını etkiler:

### Faktör 1: Fiyat

Birincil faktör. Diğer her şey eşit ise, en ucuz sağlayıcı kazanır.

### Faktör 2: Kullanılabilirlik

80 SUN fiyatlandıran ancak sadece 10.000 enerji mevcut olan bir sağlayıcı 65.000 enerji siparişini dolduramaz. Yönlendirici mevcut envanteri kontrol etmelidir.

### Faktör 3: Güvenilirlik

MERX her sağlayıcının tarihsel doldurma oranını, yanıt süresini ve başarısızlık oranını izler. %95 doldurma oranına sahip bir sağlayıcı, %99 doldurma oranına sahip biriyle karşılaştırıldığında cezalandırılır, %95 sağlayıcı biraz daha ucuz bile olsa.

```
Etkili fiyat = alıntı_fiyat / doldurma_oranı

Sağlayıcı A: 85 SUN, %99 doldurma oranı -> 85.86 etkili
Sağlayıcı B: 82 SUN, %94 doldurma oranı -> 87.23 etkili
Kazanan: Daha yüksek alıntılı fiyata rağmen Sağlayıcı A
```

### Faktör 4: Süre Desteği

Tüm sağlayıcılar tüm süreleri desteklemez. 1 saatlik delegasyon gerekiyorsa, sadece günlük minimumlar sunan sağlayıcılar hariç tutulur.

### Sipariş Bölme

Herhangi bir sağlayıcının kapasitesini aşan büyük siparişler için, yönlendirici siparişi birden fazla sağlayıcıya böler:

```
Sipariş: 500.000 enerji

Sağlayıcı A: 85 SUN'da 200.000 mevcut -> 200.000 doldur
Sağlayıcı B: 87 SUN'da 180.000 mevcut -> 180.000 doldur
Sağlayıcı C: 92 SUN'da 300.000 mevcut -> 120.000 doldur

Toplam dolduruldu: 500.000 enerji
Karışık oran: 87.28 SUN/unit
```

Alıcı tek bir sipariş ile karışık bir oran görür. Çok sağlayıcılı yürütmenin karmaşıklığı tamamen gizlenmiştir.

---

## Otomatik Yedekleme

Yedekleme, toplamlaştırmanın gerçekten değer kazandığı yerdir. Doğrudan bir sağlayıcı ile entegrasyon sağladığınız ve indikleri zaman, uygulamanız çalışmayı durdurur. MERX ile sağlayıcı arızaları şeffaf olarak işlenir.

### Yedekleme Zinciri

```
Birincil sağlayıcı başarısız oluyor
  |
  v
Sağlayıcıyı sağlıksız olarak işaretle (5 dakika boyunca yönlendirmeden hariç tut)
  |
  v
Sonraki en ucuz sağlayıcı ile yeniden dene
  |
  v
İkinci sağlayıcı başarısız olur ise, üçüncüyü dene
  |
  v
Tüm sağlayıcılar başarısız olur ise, alıcıya yeniden deneme rehberi ile hata döndür
```

### Sağlık İzleme

Fiyat izleyicisi her sağlayıcı için bir sağlık puanı tutar:

```
Sağlık puanı bileşenleri:
  - Son başarılı fiyat getirme: 60s içinde olmalı
  - API yanıt süresi: > 2 saniye ise cezalandır
  - Son sipariş doldurma oranı: < %95 ise cezalandır
  - Son hata oranı: > %5 ise cezalandır
```

Sağlıksız sağlayıcılar, kurtulana kadar yönlendirmeden hariç tutulur. Kurtarma, fiyat izleyicisi onlardan başarıyla fiyatları getirdiğinde otomatik olarak algılanır.

### Alıcılar için Sıfır Kapalı Kalma Süresi

Alıcı perspektifinden, sağlayıcı arızaları görünmezdir. API çağrıları, en az bir sağlayıcı çalıştığı sürece başarılı olur. Pratikte, yedi veya daha fazla sağlayıcıya sahip olmak, toplam pazar kesintisinin esasen imkansız olduğu anlamına gelir - tüm sağlayıcıların aynı anda inme olasılığı önemsizdir.

---

## Bir API Birçok Şeyi Değiştirir

MERX ile doğrudan sağlayıcı entegrasyonu ile karşı karşıya entegrasyon şu şekildedir:

### MERX Olmadan

```typescript
// Pseudo-kod: doğrudan çok sağlayıcılı entegrasyon

// 7 sağlayıcı istemcisini başlat
const tronsave = new TronSaveClient(apiKey1);
const feee = new FeeeClient(apiKey2);
const itrx = new ItrxClient(apiKey3);
// ... 4 daha fazla

// Tüm sağlayıcılardan fiyatları getir
const prices = await Promise.allSettled([
  tronsave.getPrice(65000),
  feee.getPrice(65000),
  itrx.getPrice(65000),
  // ... 4 daha fazla
]);

// Farklı yanıt biçimlerini normalleştir
const normalized = prices
  .filter(p => p.status === 'fulfilled')
  .map(p => normalizePrice(p.value)); // sağlayıcı başına karmaşık mantık

// Fiyata göre sırala, kullanılabilirliği kontrol et, hataları işle...
const best = normalized.sort((a, b) => a.price - b.price)[0];

// En iyi sağlayıcı ile sipariş ver
try {
  const order = await getClient(best.provider).createOrder({
    energy: 65000,
    target: buyerAddress,
    // Sağlayıcıya özel parametreler...
  });
} catch (e) {
  // Sonraki sağlayıcıya yedekle...
  // Daha fazla sağlayıcıya özel hata işleme...
}
```

### MERX ile

```typescript
import { MerxClient } from 'merx-sdk';

const client = new MerxClient({ apiKey: 'your-merx-key' });

// Tüm sağlayıcılar arasında en iyi fiyatı al
const prices = await client.getPrices({ energy: 65000 });
console.log(`Best: ${prices.bestPrice.provider} at ${prices.bestPrice.perUnit} SUN`);

// Sipariş ver - otomatik olarak en iyi sağlayıcıya yönlendir
const order = await client.createOrder({
  energy: 65000,
  targetAddress: buyerAddress,
  duration: '1h'
});

// Bitti. Yedekleme, yeniden denemeler, doğrulama otomatik olarak işlenir.
```

Yedi entegrasyon bire dönüşür. Yüzlerce satır yönlendirme ve yedekleme kodu dört satıra dönüşür. Sağlayıcı API değişikliklerinin devam eden bakımı sıfıra düşer.

---

## Gerçek Zamanlı Fiyat Beslemesi

Canlı fiyatları görüntülemek veya gerçek zamanlı yönlendirme kararları almak isteyenler, MERX WebSocket fiyat beslemesi sağlamaktadır:

```typescript
const client = new MerxClient({ apiKey: 'your-key' });

client.onPriceUpdate((update) => {
  console.log(`${update.provider}: ${update.energyPrice} SUN/unit`);
  console.log(`Best price: ${update.bestPrice} SUN/unit`);
});
```

WebSocket beslemesi, fiyat izleyicisinden her fiyat güncellemesini yayınlar - kabaca sağlayıcı başına her 30 saniye. Bu, uygulamaların yoklamadan canlı fiyatlandırmayı göstermesini etkinleştirir.

---

## Sağlayıcı Saydamlığı

MERX, hangi sağlayıcının siparişinizi doldurduğunu gizlemez. Her sipariş yanıtı sağlayıcı adını, ödenen fiyatı ve zincir üstü delegasyon işlem kodunu içerir:

```json
{
  "orderId": "ord_abc123",
  "status": "completed",
  "provider": "itrx",
  "energy": 65000,
  "pricePerUnit": 85,
  "totalCostSun": 5525000,
  "delegationTxHash": "abc123def456...",
  "verifiedAt": "2026-03-30T12:00:00Z"
}
```

Her zaman enerjinizin nereden geldiğini, ne ödediğinizi bilirsiniz ve delegasyonu zincirde bağımsız olarak doğrulayabilirsiniz.

---

## Başlama

MERX toplamlaştırması REST API'si, JavaScript SDK'sı, Python SDK'sı ve yapay zeka ajanları için MCP sunucusu aracılığıyla kullanılabilir:

- **API belgelendirmesi**: [https://merx.exchange/docs](https://merx.exchange/docs)
- **JavaScript SDK**: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- **Python SDK**: [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)
- **MCP Sunucusu**: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)

[https://merx.exchange](https://merx.exchange) adresinde bir hesap oluşturun, bir API anahtarı alın ve enerji siparişlerini tek bir API çağrısı ile mevcut en iyi fiyata yönlendirmeye başlayın.

---

*Bu makale, MERX teknik serisinin bir parçasıdır. MERX, tüm büyük TRON enerji sağlayıcılarını en iyi fiyata yönlendirme ve otomatik yedekleme ile tek bir API'de topladığı ilk blockchain kaynak borsasıdır.*


## Şimdi AI ile Deneyin

Claude Desktop veya herhangi bir MCP uyumlu istemciye MERX ekleyin -- kurulum olmaksızın, salt okunur araçlar için API anahtarına gerek yok:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Yapay zeka ajanınızdan şunu isteyin: "Şu anda en ucuz TRON enerjisi nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgelendirmesi: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)