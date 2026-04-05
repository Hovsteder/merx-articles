# TRON Energy Agregatörü Nedir ve Neden Önemlidir

Eğer hiç merkezi olmayan bir borsada token ticareti yaptıysanız, muhtemelen bunu fark etmeden bir agregatör kullanmışsınızdır. Örneğin 1inch, kendisi likidite tutmaz. Düzinelerce DEX'i tarar -- Uniswap, SushiSwap, Curve, Balancer -- sizin takas için en iyi fiyatı bulur ve siparişinizi buna göre yönlendirir. Herhangi bir tek DEX'ten daha iyi bir fiyat alırsınız ve her birini manuel olarak kontrol etmeniz gerekmez.

MERX de TRON energy için aynı şeyi yapar.

TRON energy piyasası, kendi API'leri, fiyatlandırma modelleri ve kullanılabilirlik pencereleri olan bağımsız sağlayıcıların parçalanmış bir ekosistemine dönüşmüştür. Bir energy agregatörü bu sağlayıcıların üzerinde yer alır, tümünü gerçek zamanlı olarak sorgular ve satın almanızı o anki en iyi teklifi sunan kaynağa yönlendirir. Bu makale TRON energy bağlamında agregatörün ne anlama geldiğini, neden önemli olduğunu ve MERX'in bunu nasıl uyguladığını açıklar.

## Sorun: Yedi Sağlayıcı, Yedi Fiyat

TRON ağı, her akıllı kontrat etkileşimi için energy ücretlendirir. Bir USDT transferi yaklaşık 65.000 energy maliyeti. Bir DEX takas 300.000'i aşabilir. Energy olmadan TRX olarak ödersiniz -- ve mevcut oranlarında, bu her bir USDT transferi için 13-27 TRX yakmanız anlamına gelir; bunun yerine energy kiralama yoluyla 1-2 TRX harcayabilirsiniz.

Energy kiralama piyasası bu talebe yanıt vermiştir. 2026 başı itibariyle, en az yedi büyük sağlayıcı energy delegasyon hizmetleri sunmaktadır:

- **TronSave** -- eşler arası energy pazarı
- **Feee** -- rekabetçi fiyatlandırma ve API erişimi
- **itrx** -- toplu energy odağı
- **CatFee** -- orta pazar konumlandırması
- **Netts** -- yeni oyuncudan agresif fiyatlandırma
- **SoHu** -- Çin pazarına odaklanma
- **PowerSun** -- doğrudan staking ve delegasyon

Her sağlayıcı bağımsız olarak kendi fiyatlarını belirler. Herhangi bir anda, en ucuz sağlayıcı Feee olabilir (energy birimi başına 28 SUN), TronSave 35 SUN ve Netts 31 SUN'da listelenir. On dakika sonra sıralama tamamen değişebilir. Fiyatlar arz mevcudiyeti, talep dalgalanmaları, staking havuzu kullanımı ve tahmin edilmesi imkansız olan rekabet dinamiklerine göre değişir.

## DEX Agregatörü Analojisi

DEX agregatörü ile paralellik doğrudan ve öğreticidir.

1inch ortaya çıkmadan önce, ETH'den USDC'ye en iyi fiyattan takas yapmak isteyen bir tüccar, Uniswap, SushiSwap, Curve ve diğer tüm ilgili havuzları manuel olarak kontrol etmek zorundaydı. Kaymayı, gaz maliyetlerini ve zamanlamayı hesaba katması gerekirdi. Pratikte çoğu tüccar bir DEX'i seçti ve sunduğu fiyatı kabul etti. Her seferinde para masaya bıraktılar.

1inch bunu bir yönlendirme katmanı tanıtarak çözdü. Tüm mevcut likidite kaynaklarını sorgular, optimal bölünmeyi hesaplar (bazen siparişinizin parçalarını farklı havuzlar aracılığıyla yönlendirir) ve ticareti yürütür. Kullanıcı bir arabirim ile etkileşime girer ve herhangi bir tek DEX'e eşit veya daha iyi bir fiyat alır.

TRON energy piyasası bugün, agregatörler ortaya çıkmadan önceki DEX likidite piyasasındaki aynı konumdadır. Bireysel sağlayıcılara erişim mümkündür, ancak karşılaştırma çaba gerektirir. Çoğu kullanıcı bir sağlayıcı seçer ve kalır, başka bir yerde daha iyi bir fiyatın mevcut olabileceğinden habersiz, o sağlayıcının talep ettiği fiyatı öder.

### Analojinin Geçerli Olduğu Yerler

| DEX Agregatörü | Energy Agregatörü |
|---|---|
| Birden fazla DEX'i sorgular | Birden fazla energy sağlayıcısını sorgular |
| En iyi fiyatlı likidite havuzuna yönlendirir | En ucuz energy kaynağına yönlendirir |
| Bir arabirim birçok yerine geçer | Bir API yedi entegrasyonun yerine geçer |
| Kullanıcı token'lerinin saklaması yok | Kullanıcı özel anahtarlarının saklaması yok |
| Ticarete işaretlemeler eklemez | Energy fiyatına işaretlemeler eklemez |

### Farklı Olduğu Yerler

DEX agregatörleri, siparişleri havuzlara bölebilir. Uniswap ilk 50 ETH için en iyi fiyata sahipse ancak Curve sonraki 50 için daha iyiyse, agregatör ticareti böler. Energy agregatörü şu anda siparişleri sağlayıcılar arasında bölmez -- tüm energy satın almanız tek bir sağlayıcıya gider. Bu, TRON'daki delegasyon mekanizmasinin bir sonucudur: energy, staking adresinden alıcı adrese tek bir zincir üzerindeki işlem olarak delege edilir.

DEX agregatörleri de kaymalar ile uğraşır -- fiyat alıntı ile yürütme arasında değişebilir. Energy fiyatları daha kararlıdır (saniyeye değil dakikalara göre değişir), bu yüzden kayma önemli bir sorun değildir. Alıntıda gördüğünüz fiyat, ödediğiniz fiyattır.

## Neden Hiçbir Sağlayıcı Her Zaman En Ucuz Değildir

Bir sağlayıcı sürekli en ucuz olsaydı, agregatöre ihtiyacınız olmayacaktı. Bu sağlayıcı ile entegrasyonu sağlardınız ve bitirirdiniz. Ancak energy piyasası bu şekilde çalışmaz.

### Arz Tarafı Dinamikleri

Her sağlayıcının sonlu energy arzı vardır. Bu arz TRX'in energy için stake edilmesinden gelir ve stake edilen TRX miktarı, sağlayıcının ne kadar energy delege edebileceğini belirler. Bir sağlayıcının arzı düştüğünde -- çünkü birçok alıcı aynı anda satın alıyor veya staking havuzları yeniden dengeleniyor -- bu sağlayıcı fiyatları artırır veya geçici olarak kullanılamaz hale gelir.

Sağlayıcı A'nın UTC 08:00'de bol arzı olabilir ve pazardaki en düşük fiyatları sunabilir. 10:00'de, büyük bir alıcı bu arzın çoğunu tüketebilir, Sağlayıcı A'nın fiyatlarını artırabilir. Bu arada, Sağlayıcı B, 08:00'de orta aralıkta olup, arzı etkilenmediği için artık en düşük fiyata sahiptir.

### Fiyatlandırma Modelleri Değişir

Farklı sağlayıcılar farklı fiyatlandırma stratejileri kullanır:

- **Sabit fiyatlandırma**: Bazı sağlayıcılar, seyrek olarak değişen düz bir oran belirler. Bu sağlayıcılar yüksek talep dönemlerinde en uygun fiyatlı olup ancak düşük talep dönemlerinde daha pahalı olabilir.
- **Dinamik fiyatlandırma**: Bazı sağlayıcılar fiyatları mevcut arz kullanım oranına göre ayarlar. Bu sağlayıcılar arz bol olduğunda çok ucuz olabilir ancak kullanım yüksek olduğunda pahalı olabilir.
- **Süreler bağlı fiyatlandırma**: Fiyatlar kiralama süresine göre değişir. Bir sağlayıcı 1 saatlik kiralamalar için en ucuz olabilir ancak 24 saatlik kiralamalar için daha pahalı olabilir ya da tam tersi.

Hiçbir tek strateji tüm koşullarda hakim değildir. Optimal sağlayıcı günün saatine, ihtiyacınız olan energy miktarına, kiralama süresine ve genel pazar durumuna göre değişir.

### Ampirik Kanıtlar

MERX, her otuz saniyede tüm yedi sağlayıcıdan fiyatları izler ve sonuçları depolar. Tarihsel fiyat verilerinin analizi, en ucuz sağlayıcının günde birden çok kez değiştiğini gösterir. Tipik bir günde, en düşük 1 saatlik energy fiyatını sunan sağlayıcı, üç ila dört farklı sağlayıcı arasında döner ve en ucuz ile en pahalı arasındaki boşluk genellikle %20'yi aşar.

```
Örnek fiyat anlık görüntüsü (SUN başına energy birimi, 1s süresi):

  TronSave:   35
  Feee:       28  <-- en ucuz
  itrx:       32
  CatFee:     30
  Netts:      31
  SoHu:       34
  PowerSun:   33

Dört saat sonra:

  TronSave:   30
  Feee:       33
  itrx:       29  <-- en ucuz
  CatFee:     31
  Netts:      28  <-- en ucuz bağlantılı
  SoHu:       35
  PowerSun:   32
```

İlk anlık görüntüde Feee en ucuz olduğu için yalnızca onunla entegrasyonu yaptıysanız, dört saat sonra 33 SUN ödeyebilirsiniz; pazar tabanı 28-29 SUN'da iken. O %15'lik fark her işlemle birleştirilir.

## MERX Nasıl Agregatör Olur

MERX, sürekli çalışan üç bileşen aracılığıyla agregatörü uygular.

### Fiyat İzleyici

Özel bir hizmet, her otuz saniyede her sağlayıcının API'sini yoklar. Her yanıt standart bir format'a normalleştirilir -- SUN cinsinden fiyat/energy birimi -- ve bir Redis kanalına yayımlanır. Bu normalleştirme kritiktir çünkü sağlayıcılar fiyatları farklı şekilde ifade ederler: bazıları SUN'da, bazıları TRX'de, bazıları birim başına, bazıları paket için. MERX her şeyi doğrudan karşılaştırma için SUN başına energy birimine dönüştürür.

### Fiyat Önbelleği

Redis, her sağlayıcıdan en son fiyatı 60 saniyelik TTL ile tutar. Bir sağlayıcının verisi 60 saniyeden eski ise, yönlendirme kararlarından otomatik olarak hariç tutulur. Bu, eski fiyatların yönlendiriciyi yanlış yönlendirmesini önler.

### Sipariş Yönlendiricisi

Sipariş verdiğinizde, yönlendirici Redis'ten tüm mevcut fiyatları sorgular, spesifik isteklerinizi yerine getirebilen sağlayıcıları filtreler (miktar, süre) ve en ucuz seçeneği seçer. Tüm işlem milisaniye cinsindedir.

```typescript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY });

// Bir çağrı. Tüm sağlayıcılar arasında en iyi fiyat. Entegrasyon vergisi yok.
const order = await merx.createOrder({
  energy_amount: 65000,
  target_address: 'TRecipientAddress...',
  duration: '1h'
});

console.log(`Sağlayıcı: ${order.provider}`);
console.log(`Fiyat: ${order.price_sun} SUN/birim`);
console.log(`Toplam: ${order.total_trx} TRX`);
```

Sağlayıcıyı siz seçmezsiniz. MERX sizin için seçer ve seçim her zaman siparişiniz anındaki mevcut en ucuz seçenektir.

## Otomatik Yedekleme

Agregatör, fiyat optimizasyonunun ötesinde ikinci bir avantaj sağlar: güvenilirlik. Tek bir sağlayıcı inerse -- ve gerçekten inerler -- entegrasyonunuz bozulur. MERX ile sağlayıcı kesintisi sizin için görünmezdir.

Fiyat izleyici bir sağlayıcının yanıt vermediğini tespit ettiğinde, o sağlayıcı için fiyat yayımlamayı durdurur. Redis TTL 60 saniye sonra sona erer ve sağlayıcı otomatik olarak yönlendirmeden hariç tutulur. Siparişler kesinti olmaksızın kalan sağlayıcılar aracılığıyla akışına devam eder.

```
Sağlayıcı hata senaryosu:

  1. Feee API, HTTP 500 döndürür
  2. Fiyat izleyici, Feee'yi kullanılamaz olarak işaretler
  3. Feee için Redis TTL sona erer (60s)
  4. Sonraki sipariş, ikinci en ucuz sağlayıcıya yönlendirilir
  5. Alıcıya görünür hata yok
  6. Feee kurtarılır, fiyat izleyici yoklamaya devam eder
  7. Feee yönlendirme havuzuna yeniden girer
```

Doğrudan Feee ile entegrasyonu yaptıysanız, 500 hatası işlem ardınızı çökertecekti. MERX ile hiç bunu bilmezsiniz.

## Sıfır Komisyon

MERX, sağlayıcı fiyatlarına işaretlemeler eklemez. En ucuz sağlayıcı energy'yi birim başına 28 SUN sunuyorsa, birim başına 28 SUN ödersiniz. Agregatör katmanı kullanımı ücretsizdir.

Bu mümkündür çünkü MERX, pazar kazanım aşamasındadır. Hacmi agregatör etme değeri -- sağlayıcılarla daha iyi müzakere kaldıracağı, yönlendirme optimizasyonu için daha zengin veri, ağ etkileri -- bireysel işlemlerden marj çıkarmak değerini aşar. Model, birçok başarılı agregatörün nasıl başladığını yansıtır: büyüme sırasında ücretsiz erişim, kullanıcı tabanı kurulduğunda premium özellikler aracılığıyla para kazanma.

## Agregatör Ne Zaman Kullanmalısınız

Agregatör şu zaman mantıklıdır:

- **Günde birden fazla işleme işlem yaparsınız.** Hatta küçük işlem başına tasarruflar da hızlı birleşir.
- **TRON işlemlerini otomatize edersiniz.** Agregatörün API'si yedi sağlayıcı entegrasyonunu yönetmekten daha basittir.
- **Güvenilirlik önemli ise.** Otomatik yedekleme, tek başarısızlık noktalarını ortadan kaldırır.
- **Sağlayıcı fiyatlandırmasını manuel olarak izlemek istememezsiniz.** Agregatör bunu sizin için sürekli yapar.

Agregatör daha az değer ekler:

- **Tek bir sağlayıcı ile sabit oran altında uzun vadeli sözleşmeniz varsa.** Agregatör faydaları dinamik yönlendirmeden gelir, bu sabit oran sözleşmelerine uygulanmaz.
- **Ayda bir veya iki işlem yaparsanız.** Tasarruf var ama düşük hacimde minimum.

## Başlarken

MERX, REST API, JavaScript ve Python SDK'ları, WebSocket gerçek zamanlı fiyat beslemeleri ve AI ajanları için bir MCP sunucusu sunmaktadır. En hızlı entegrasyon yolu:

```bash
npm install merx-sdk
```

```typescript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({ apiKey: 'YOUR_API_KEY' });

// Şu anda ne ödeyeceğinizi kontrol edin
const prices = await merx.getPrices({
  energy_amount: 65000,
  duration: '1h'
});

// Her sağlayıcının fiyatını görün
for (const p of prices) {
  console.log(`${p.provider}: ${p.price_sun} SUN/birim`);
}
```

Tam dokümantasyon: [https://merx.exchange/docs](https://merx.exchange/docs)
AI ajanları için MCP sunucusu: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
Platform: [https://merx.exchange](https://merx.exchange)


## AI ile Şimdi Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin -- sıfır kurulum, salt okunur araçlar için API anahtarı gerekmez:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI ajanınıza sorun: "Şu anda en ucuz TRON energy nedir?" ve bağlı tüm sağlayıcılardan canlı fiyatları alın.

Tam MCP dokümantasyonu: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)