# TRON Kaynak Ekonomisi: Geliştirici Rehberi

TRON'daki her işlem kaynakları tüketir. Bu kaynakların nasıl çalıştığını anlamıyorsanız, uygulamanızın gerçekleştirdiği her işlem için fazla ödeme yaparsınız. Bu teorik bir endişe değildir -- kaynakları optimize edilmiş bir TRON uygulaması ile saf bir uygulama arasındaki fark işletme maliyetlerinde %90'ı aşabilir.

Bu makale, TRON kaynak ekonomisi için tam bir geliştirici referansıdır. İşlem türü başına enerji maliyetlerini, bandwidth mekaniklerini, staking oranlarını, yakma oranlarını, kiralama pazarını ve kullanım durumunuz için doğru kaynak stratejisini seçmek için bir karar çerçevesini kapsar.

## İki Kaynak: Enerji ve Bandwidth

TRON'un işlemlerin tükettiği iki hesaplama kaynağı vardır:

**Enerji**, akıllı sözleşme yürütülmesi tarafından tüketilir. TVM'deki (TRON Sanal Makinesi) her opcode'un bir enerji maliyeti vardır. Sözleşme etkileşimi ne kadar karmaşıksa, o kadar fazla enerji tüketir. Enerji, pahalı olan kaynaktır -- TRON'daki işlem maliyetlerinin çoğunluğunu yönlendirir.

**Bandwidth**, bir işlemin ham veri boyutu tarafından tüketilir. İşlem verilerinin her baytı bandwidth tüketir. Basit TRX transferleri, token transferleri ve sözleşme çağrıları hepsi seri hale getirilmiş boyutlarıyla orantılı bandwidth tüketir. Bandwidth, daha ucuz olan kaynaktır ve TRON her aktif adrese günde 600 bandwidth puanı ücretsiz olarak verir.

Kritik ayrım: basit bir TRX transferi yalnızca bandwidth tüketir (hiçbir akıllı sözleşme söz konusu değildir). Bir USDT transferi, bir DEX swapı veya diğer herhangi bir akıllı sözleşme etkileşimi hem enerji hem de bandwidth tüketir.

## İşlem Türüne Göre Enerji Maliyetleri

Enerji tüketimi işlem türüne göre önemli ölçüde değişir. İşte mainnet işlemlerinden alınan ampirik ölçümler:

### TRC-20 Token Transferleri

```
USDT transferi (mevcut alıcı):          ~31,895 - 64,285 enerji
USDT transferi (yeni alıcı):            ~65,527 enerji
USDT transferFrom (onay ile):           ~45,000 - 68,000 enerji
USDC transferi:                         ~32,000 - 65,000 enerji
Özel TRC-20 transferi:                  ~30,000 - 100,000+ enerji
```

USDT transferlerindeki varyans açıklamayı hak ediyor. Alıcı daha önce hiç USDT tutmadığında, sözleşme iç mapping'inde yeni bir depolama yuvası tahsis etmelidir. Bu SSTORE işlemi, mevcut bir bakiyeyi güncellemeye göre (mevcut slotta SLOAD + SSTORE) önemli ölçüde daha fazla enerji maliyeti yaşar. İlk kez ve tekrarlanan alıcı arasındaki fark 30.000 enerjiyi aşabilir.

### DEX İşlemleri

```
SunSwap V2 tek çift swapı:              ~200,000 - 300,000 enerji
SunSwap V2 multi-hop swapı:             ~400,000 - 600,000 enerji
Likidite ekleme:                        ~250,000 - 350,000 enerji
Likidite çıkarma:                       ~200,000 - 300,000 enerji
```

DEX işlemleri pahalıdır çünkü tek bir işlem içinde birden fazla sözleşme çağrısı içerirler: router sözleşmesi, pair sözleşmesi, token onayları, fiyat hesaplamaları ve token transferleri.

### NFT İşlemleri

```
TRC-721 mint:                           ~100,000 - 200,000 enerji
TRC-721 transferi:                      ~50,000 - 80,000 enerji
TRC-1155 mint (tek):                    ~80,000 - 150,000 enerji
TRC-1155 toplu mint:                    ~150,000 - 500,000+ enerji
```

### Sözleşme Dağıtımı

```
Basit sözleşme dağıtımı:                ~500,000 - 1,000,000 enerji
Karmaşık sözleşme (DEX pair):           ~2,000,000 - 5,000,000 enerji
Proxy sözleşme dağıtımı:                ~300,000 - 600,000 enerji
```

### Önemli İçgörü: Hardcode Yapmayın

Bu sayılar sabitler değil, aralıklardır. Tüketilen gerçek enerji, yürütme sırasında sözleşmenin iç durumuna bağlıdır. Enerji satın almadan önce tam maliyeti simüle etmek için `triggerConstantContract` kullanın. MERX bunu `estimate_contract_call` aracı ve `/api/v1/estimate` uç noktası aracılığıyla sunar.

## Bandwidth Maliyetleri

Bandwidth, enerjiden daha basittir. Her işlem, serileştirilmiş bayt boyutuyla orantılı bandwidth tüketir. Formül:

```
tüketilen_bandwidth = işlem_boyutu_bayt
```

Tipik bandwidth maliyetleri:

```
Basit TRX transferi:                    ~270 bandwidth
TRC-20 transferi:                       ~345 bandwidth
DEX swapı:                              ~400 - 600 bandwidth
Sözleşme dağıtımı:                      ~1,000 - 10,000 bandwidth
```

### Ücretsiz Tahsisat

Her TRON adresi günde 600 ücretsiz bandwidth puanı alır ve UTC gece yarısında yenilenir. Bu günde 1-2 basit TRX transferi için yeterlidir ancak sözleşme etkileşimleri için eksiktir.

Bandwidth'iniz tükenirse, ağ maliyeti karşılamak için hesabınızdan TRX yakar. Yakma oranı şu anda bandwidth noktası başına 1.000 SUN (0.001 TRX) olur. 345 bandwidth'li bir USDT transferi için, bandwidth için 0.345 TRX yakılır -- enerji maliyetlerine kıyasla nispeten küçüktür, ancak yüksek hacimde birikir.

### Bandwidth Staking

Enerji için yapabildiğiniz gibi TRX'i bandwidth için stake edebilirsiniz. Bandwidth-per-TRX oranı bandwidth için toplam ağ stake'ine bağlıdır:

```
sizin_bandwidth = (stake_ettiğiniz_trx / toplam_ağ_bandwidth_stake) * toplam_bandwidth_limiti
```

Bandwidth staking daha az tartışılır çünkü:
1. Ücretsiz 600 puanlık günlük tahsisat hafif kullanımı kapsar
2. Bandwidth maliyetleri enerji maliyetlerine kıyasla küçüktür
3. Çoğu geliştirici önce enerji optimizasyonuna odaklanır

Günde yüzlerce işlem işleyen yüksek hacimli uygulamalar için, bandwidth staking anlamlı tutarlar tasarruf edebilir. Ancak enerji optimizasyonu her zaman ilk gelmesi gerekir.

## Staking Oranları

TRX'i stake etmek, kaynakları elde etmenin birinci taraf yöntemidir. TRX'i Stake 2.0'da kilitler ve enerji veya bandwidth alırsınız.

### Enerji Staking Oranı

Staking'ten aldığınız enerji iki faktöre bağlıdır: kaç TRX stake ettiğiniz ve tüm ağın enerji için kaç TRX stake ettiği.

```
sizin_enerji = (stake_ettiğiniz_trx / toplam_ağ_enerji_stake) * toplam_enerji_limiti
```

2026 başı itibarıyla yaklaşık oran:

```
Toplam ağ enerji limiti:        ~90,000,000,000 enerji/gün
Toplam ağ enerji stake:         ~50,000,000,000 TRX

Stake edilen TRX başına enerji:  ~1.8 enerji/gün stake edilen TRX başına
```

Tek bir USDT transferini (65.000 enerji) karşılamak için, yaklaşık olarak stake etmeniz gerekirdi:

```
65,000 / 1.8 = ~36,111 TRX stake edildi
```

Mevcut TRX fiyatlarında, bu önemli bir sermaye taahhüdüdür -- ve günde bir USDT transferi için yeterli enerji elde edersiniz. Bu nedenle kiralama pazarı vardır: enerji kiralamak, stake etmekten çok daha fazla sermaye verimlidir.

### Staking Paradoksu

Stake'in sıfır marjinal maliyeti vardır (14 günlük bekleme süresinden sonra stake'inizi geri alırsınız), ancak fırsat maliyeti gerçektir. Stake ettiğiniz TRX ticaret, ödünç verme veya diğer getiri üreten faaliyetler için kullanılamaz. Çoğu geliştirici ve işletme için, pazardan enerji kiralamak, kilitli sermaye gerektiren kendi stake'i yapmaktan daha uygun maliyetlidir.

Kırılma noktası günlük enerji tüketiminize bağlıdır:

```
Başabaş analizi (yaklaşık):

  Günlük enerji ihtiyacı:       65,000 (bir USDT transferi)
  Kiralama maliyeti:            ~1.5 TRX/gün
  Gerekli stake:                ~36,111 TRX
  TRX yıllık getirisi:          ~%4 (staking ödülleri)
  Fırsat maliyeti:              ~36,111 * 0.04 / 365 = ~3.96 TRX/gün

  Sonuç: Kiralama daha ucuzdur (1.5 < 3.96 TRX/gün)

  Günlük enerji ihtiyacı:       6,500,000 (100 USDT transferi)
  Kiralama maliyeti:            ~150 TRX/gün
  Gerekli stake:                ~3,611,111 TRX
  Fırsat maliyeti:              ~3,611,111 * 0.04 / 365 = ~395 TRX/gün

  Sonuç: Kiralama hala daha ucuzdur (150 < 395 TRX/gün)
```

Çoğu kullanım durumu için, saf ekonomi açısından kiralama kazanır. Staking, pazar koşullarından bağımsız olarak garantili kaynak kullanılabilirliğine ihtiyacınız olduğunda veya stake'i yönetim için yapan ve enerjiyi bir yan ürün olarak gören bir doğrulayıcı olduğunuzda mantıklıdır.

## Yakma Oranları: Kaynak Olmadan Ne Olur

Yeterli enerji veya bandwidth olmadan bir işlem yürütürseniz, TRON işlemi reddetmez. Bunun yerine, açığı kapatmak için hesabınızdan TRX yakar.

### Enerji Yakma Oranı

```
1 enerji birimi = 420 SUN yakılır (0.00042 TRX)
```

Herhangi bir enerji olmadan 65.000 enerjili bir USDT transferi için:

```
65,000 * 420 = 27,300,000 SUN = 27.3 TRX yakılır
```

Bunu MERX üzerinden birim başına 28 SUN'da 65.000 enerji kiralamakla karşılaştırın:

```
65,000 * 28 = 1,820,000 SUN = 1.82 TRX kiralama maliyeti
```

Tasarruf: 27.3 - 1.82 = 25.48 TRX tasarruf edilir veya %93 maliyet azalması. Bu nedenle enerji kiralama pazarı vardır ve neden önemli olduğudur.

### Bandwidth Yakma Oranı

```
1 bandwidth noktası = 1,000 SUN yakılır (0.001 TRX)
```

345 bandwidth'li bir USDT transferi için:

```
345 * 1,000 = 345,000 SUN = 0.345 TRX yakılır
```

### Kısmi Kapsam

Biraz enerji var ama yeterli değilse, TRON önce mevcut enerji kullanır ve kalanı için TRX yakar:

```
Gerekli enerji:         65,000
Mevcut enerji:          40,000
Enerji açığı:           25,000
Yakılan TRX:            25,000 * 420 = 10,500,000 SUN = 10.5 TRX
```

Bu nedenle doğru enerji tahmini önemlidir. 65.000'e ihtiyaçken 40.000 enerji satın almak kiralama maliyetini boşa harcar ve yine de önemli bir TRX yakması ile sonuçlanır.

## Kiralama Pazarına Genel Bakış

Enerji kiralama pazarı, tahmin edilebilir desenlere sahip yapılandırılmış bir ekosisteme evrimleşmiştir.

### Kiralamaların Nasıl Çalıştığı

Enerji sağlayıcıları büyük miktarlarda TRX stake eder ve enerji biriktirir. Daha sonra bu enerjiyi alıcılara bir ücret karşılığında devrederlerse. Devreden, zincirde bir işlemdir: sağlayıcının adresi, belirli miktarda enerjiyi belirli bir süre için alıcının adresine devrediler.

```
Sağlayıcı 1,000,000 TRX stake eder
  -> ~1,800,000 enerji/gün alır
  -> Alıcılara pazar oranında enerji devreder
  -> Kiralama ücretleri toplar
  -> TRX stake'de kalır (ana para korunur)
```

### Süre Seçenekleri

Çoğu sağlayıcı birden fazla süre seviyesi sunar:

```
1 saat        En ucuz birim fiyatı, tek işlemler için ideal
1 gün         Orta fiyatlandırma, günlük toplu işlemler için iyi
3 gün         Hacim indirimi başlar
7 gün         Taahhütlü kullanım için anlamlı indirim
14 gün        En düşük birim oranlar
30 gün        En iyi oranlar, talep hakkında güven gerekir
```

Optimal süre, kullanım şeklinize bağlıdır. Günde 10 USDT transferi işliyorsanız, 650.000 enerji'nin 24 saatlik kiralanması, 65.000 enerji'nin on ayrı 1 saatlik kiralama'sından daha uygun maliyetlidir.

### Fiyat Keşfi

Enerji fiyatları arz ve talebe göre dalgalanır. 2026 başı itibarıyla tipik fiyat aralıkları:

```
1 saatlik kiralama:     25 - 40 SUN birim başına
1 günlük kiralama:      20 - 35 SUN birim başına
7 günlük kiralama:      15 - 30 SUN birim başına
30 günlük kiralama:     10 - 25 SUN birim başına
```

Fiyatlar yoğun olmayan saatlerde (kabaca 00:00-08:00 UTC) en düşüktür ve yüksek işlem dönemlerinde en yüksektir. MERX fiyat monitörü, 30 saniyede bir tüm sağlayıcılar arasında bu dalgalanmaları yakalar.

## Karar Çerçevesi: Kaynak Stratejisini Seçme

### Düşük Hacim (günde 1-10 işlem)

**Önerilen strateji: MERX üzerinden işlem başına kiralama**

Düşük hacimde, en basit yaklaşım, gerektiğinde her işlem için enerji satın almaktır. Staking veya uzun süreli kiralamaları sürdürmenin ek yükü haklı değildir.

```typescript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY });

// Her USDT transferinden önce, tam ihtiyacınız olan enerjiyi satın alın
const estimate = await merx.estimateContractCall({
  contract_address: 'TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t',
  function_selector: 'transfer(address,uint256)',
  parameter: encodedParams,
  owner_address: senderAddress
});

const order = await merx.createOrder({
  energy_amount: estimate.energy_required,
  target_address: senderAddress,
  duration: '1h'
});
```

### Orta Hacim (günde 10-100 işlem)

**Önerilen strateji: Buffer'lı günlük enerji kiralamaları**

Beklenen günlük hacminiz artı %20 buffer için boyutlandırılmış 24 saatlik enerji kiralaması satın alın. Bu saatlik kiralama'dan daha düşük birim fiyatı verir ve işlem başına satın alma ek yükünü ortadan kaldırır.

```
Günlük USDT transferleri:   50
Transfer başına enerji:     65,000
Günlük enerji ihtiyacı:     3,250,000
%20 buffer ile:             3,900,000
Süre:                       24 saat
```

### Yüksek Hacim (günde 100+ işlem)

**Önerilen strateji: Haftalık veya aylık kiralamaları sabit siparişlerle**

Yüksek hacimde, daha uzun süreler en iyi birim fiyatlandırması sunar. MERX sabit siparişlerini kullanarak fiyatlar eşiğinizin altına düştüğünde otomatik olarak enerji satın alın:

```typescript
// Sabit sipariş: fiyat 25 SUN'nun altına düştüğünde otomatik satın al
await merx.createStandingOrder({
  energy_amount: 30000000,
  duration: '7d',
  max_price_sun: 25,
  target_address: operationalAddress
});
```

### Kurumsal / Altyapı

**Önerilen strateji: Hibrit staking + kiralama**

Binlerce işlem işleyen altyapı operatörleri için, kendi stake'i (temel kapasite için) ve pazar kiralamaları (patlama kapasitesi için) kombinasyonu en iyi ekonomi ve güvenilirlik sağlar:

```
Temel (stake edildi):   Günlük enerji ihtiyacının ortalama %60'ını kapla
Patlama (kirala):       Kalan %40 + MERX üzerinden çıkıntıları kapla
```

Bu, pazar bozulmaları sırasında bile kaynak kullanılabilirliğini sağlarken makul sermaye verimliliğini tutar.

## İzleme ve Optimizasyon

Hangi stratejiyi seçerseniz seçin, gerçek enerji tüketiminizi tahminlerinize karşı izleyin. MERX buna yönelik araçlar sağlar:

- **Fiyat geçmişi API**: Enerji fiyatlarının zaman içinde nasıl değiştiğini izle
- **Sipariş geçmişi**: Ne ödediğinizi gözden geçirin ve optimize edin
- **WebSocket fiyat akışları**: Fiyat hareketlerine gerçek zamanda tepki verin
- **Devreden monitörler**: Devredilen enerji süreniz dolmak üzereyken bildirim alın

TRON kaynak modeli, bunu anlayanları ödüllendirir. Enerji maliyetleri işlem ekonomisine hakim olur ve protokol düzeyinde TRX yakma ile enerjiyi pazar oranında kiralama arasındaki fark tutarlı bir şekilde %90 veya daha fazladır. İşlem başına kiralama yapın, günlük bloklar satın alın veya kendi stake'inizi yapın, anahtar kiralanan enerji ile karşılanabilecek hiçbir işlemin TRX yakmamasını sağlamaktır.

Tam dokümantasyon: [https://merx.exchange/docs](https://merx.exchange/docs)
Platform: [https://merx.exchange](https://merx.exchange)
MCP sunucusu: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)


## Şimdi AI ile Deneyin

MERX'i Claude Desktop veya herhangi bir MCP uyumlu istemciye ekleyin -- sıfır kurulum, salt okunur araçlar için API anahtarı gerekli değildir:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI aracınızdan şunu sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve bağlı tüm sağlayıcılardan canlı fiyatlar alın.

Tam MCP dokümantasyonu: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)