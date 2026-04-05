# MCP Aracılığıyla SunSwap: Bir DEX'te İşlem Yapan AI Aracıları

## Eksik Köprü

Merkezi olmayan borsalar yıllardır web arayüzleri, mobil cüzdanlar ve programlı SDK'lar aracılığıyla erişilebilir durumdadır. Ancak doğal dil aracılığıyla erişilebilir değillerdir. Bir AI aracısı takas düğmesine tıklayamaz. Bir web kullanıcı arayüzünde gezinemeyen. Ve bir aracı teknik olarak ham işlem yapısı aracılığıyla akıllı bir sözleşmeyi çağırabilse de, bunun yapılması aracının ABI kodlaması, router sözleşme adreslerini, likidite havuzu mekaniklerini ve TRON kaynak modelini anlamasını gerektirir.

MERX bu boşluğu kapatır. MCP sunucusu aracılığıyla, bir AI aracısı takas teklifi isteyebilir, beklenen çıktıyı ve maliyetleri anlayabilir ve işlemi gerçekleştirebilir - hepsi protokol düzeyindeki karmaşıklığı soyutlarken blok zincir üzerinde neler olduğu konusunda tam şeffaflığı koruyacak yapılandırılmış araç çağrıları aracılığıyla.

Bu makale, tam SunSwap entegrasyonunu kapsamaktadır: teklif alma, takasları yürütme, onayları işleme, enerji maliyetlerini simüle etme ve mainnet işlemlerinden gerçek sayıları anlama.

## Takas Teklifi Alma

Herhangi bir işlemdeki ilk adım, ne alacağınızı anlamaktır. `get_swap_quote` aracı, belirli bir giriş için beklenen çıktıyı hesaplamak amacıyla SunSwap V2'nin router sözleşmesini sorgular:

```
Tool: get_swap_quote
Input: {
  "from_token": "TRX",
  "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "amount": "100000",
  "from_address": "TYourAddress..."
}

Response:
{
  "input_token": "TRX",
  "input_amount": "0.1",
  "input_amount_sun": 100000,
  "output_token": "USDT",
  "output_amount": "0.032847",
  "output_amount_raw": "32847",
  "minimum_received": "0.032519",
  "price_impact": "0.001%",
  "route": ["WTRX", "USDT"],
  "router": "TKzxdSv2FZKQrEkFPRQi4qeCokAFkrntVM",
  "energy_required": 223354,
  "energy_cost_estimate_trx": 11.76,
  "liquidity": {
    "pool_address": "TPool...",
    "trx_reserve": "45,234,891.23",
    "usdt_reserve": "14,832,445.67"
  }
}
```

Bu yanıtta birkaç önemli ayrıntı bulunmaktadır:

### Fiyat Etkisi

`price_impact` alanı, işleminizin havuz fiyatını ne kadar hareket ettirdiğini gösterir. Yukarıdaki 0.1 TRX işleminde etki 0.001% olup, ihmal edilebilirdir. Daha büyük işlemler için bu sayı artar:

```
0.1 TRX takası:      0.001% etki
1,000 TRX takası:    0.012% etki
100,000 TRX takası:  1.2% etki
1,000,000 TRX takası: 11.8% etki
```

Fiyat etkisi bir ücret değildir - sabit ürün AMM mekaniklerinin yapısal bir sonucudur. İşleminiz havuzun likiditesine kıyasla ne kadar büyük olursa, yürütme fiyatınız o kadar kötü olur.

### Minimum Alınan Tutar

`minimum_received` alanı slipaj korumasını hesaba katar. Varsayılan olarak, MERX %1'lik bir slipaj toleransı hesaplar, bu da takas, alınan miktar, teklif edilen tutarın %1'den fazla azalırsa geri alınır anlamına gelir. Bu, ön çalıştırmaya ve teklif ile yürütme arasındaki hızlı fiyat hareketlerine karşı koruma sağlar.

### Rota

`route` alanı, token yolunu gösterir. TRX'den USDT'ye, rota WTRX (Sarılı TRX) üzerinden geçer çünkü SunSwap V2 çiftleri TRC20 tokenler arasındadır ve yerli TRX'in önce sarılması gerekir. Doğrudan çifti olmayan token'ten token'e takaslar için rota ara tokenler içerebilir:

```
Token A -> WTRX -> Token B  (iki atlama rotası)
```

Çok atlama rotaları, ek sözleşme etkileşimleri nedeniyle daha fazla enerji tüketir.

### Gerekli Enerji

Bu, `triggerConstantContract` simülasyonundan tam enerji tahminidir. Sabit kodlanmış bir sabit değil, bir aralık değil - bu spesifik takas, bu spesifik parametrelerle, mevcut blok zincir durumunda tüketecek olan kesin enerji birimleri.

## Takası Yürütme

Aracı teklifi inceledikten ve devam etmeye karar verdikten sonra, `execute_swap` aracı tüm yürütme boru hattını işler:

```
Tool: execute_swap
Input: {
  "from_token": "TRX",
  "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "amount": "100000",
  "slippage": 1.0,
  "deadline_minutes": 5
}

Response:
{
  "status": "completed",
  "tx_hash": "9c7b3a2f...",
  "input": {
    "token": "TRX",
    "amount": "0.1"
  },
  "output": {
    "token": "USDT",
    "amount": "0.032847"
  },
  "energy_used": 223354,
  "energy_purchased": 225000,
  "energy_cost_trx": 11.84,
  "block": 58234892,
  "gas_saved_trx": 41.16
}
```

### execute_swap İçinde Neler Olur

`execute_swap` aracı, tam kaynak farkında işlem boru hattını yönetir:

1. **Takası simüle etme** - Tam takas parametreleriyle `triggerConstantContract` çalıştırarak tam enerji gereksinimini alma (bu durumda 223,354)

2. **Mevcut kaynakları kontrol etme** - Gönderenin adresini sorgulamak için mevcut enerji ve bandwidth'i sorgulamak

3. **Enerji eksikliğini satın alma** - En iyi sağlayıcı fiyatını bulmak, siparişi yerleştirmek ve delegasyon onayını beklemek

4. **Takas işlemini oluşturma** - Doğru işlev seçiciyle, parametrelerle ve çağrı değeriyle SunSwap V2 router çağrısını yapılandırmak

5. **Yerel olarak imzalama** - Özel anahtar, işlemi aracının makinesinde imzalar

6. **Yayınlama** - İmzalanan işlem TRON ağına gönderilir

7. **Doğrulama** - İşlem onayı için yoklamak ve sonucu ayrıştırmak

Aracı bir araç çağırır. Arka planda yedi adım yürütülür. Aracı tam sonuç içeren tek bir yanıt alır.

## Token Onayları

TRC20 token'lerini (yerel TRX'i değiştirmeye karşı) değiştirirken, SunSwap router'ı sizin adınıza token harcamak için izne ihtiyaç duyar. Bu standart TRC20 `approve` mekanizmasıdır.

MERX bunu otomatik olarak işler. TRC20'den TRC20'ye veya TRC20'den TRX'e takas yürütülmeden önce, execute_swap aracı router sözleşmesinin giriş token'ı için yeterli izne sahip olup olmadığını kontrol eder:

```
Onay kontrolü:
  Token: USDT (TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t)
  Harcayan: SunSwap V2 Router (TKzxdSv2FZKQrEkFPRQi4qeCokAFkrntVM)
  Geçerli izin: 0
  Gerekli: 100,000,000 (100 USDT)

  -> Onay gerekli
```

Onay gerekiyorsa, MERX:

1. Onay işlemini simüle ederek enerji maliyetini alır
2. Onay enerjisini takas enerjisine ekler, birleşik satın almaya
3. Önce onay işlemini yürütür
4. Onay bekler
5. Ardından takası yürütür

```
Birleşik enerji gereksinimi:
  Onay: 46,312 enerji
  Takas: 223,354 enerji
  Toplam: 269,666 enerji

  Satın alınan: 270,000 enerji, 14.20 TRX karşılığında
```

Aracının onaylar hakkında bilgi sahibi olması gerekmez. Gerekiyorsa, olur. Token zaten onaylanmışsa, adım atlanır.

### Onay Miktarı

Varsayılan olarak, MERX maksimum uint256 değerini onaylar (sınırsız onay). Bu, aynı token'in gelecekteki takaslarının başka bir onay işlemi gerektirmeyeceği anlamına gelir, enerji ve zaman tasarrufu sağlar. Aracı tam onayları tercih ederse (yalnızca gerekli belirli tutarı onaylamak), bunu belirtebilir:

```
Tool: approve_trc20
Input: {
  "token_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "spender": "TKzxdSv2FZKQrEkFPRQi4qeCokAFkrntVM",
  "amount": "100000000"
}
```

Tam onaylar daha güvenlidir (router sözleşmesi tehlikeye atılırsa maruziyeti sınırla) ancak her takas için enerji maliyeti.

## Gerçek Takas: 0.1 TRX'den USDT'ye

MERX MCP sunucusu aracılığıyla yürütülen gerçek bir mainnet takasının tam ayrıntıları aşağıdadır:

### Takas Öncesi Durum

```
Adres: TWallet...
TRX Bakiyesi: 523.456789 TRX
USDT Bakiyesi: 0.00 USDT
Mevcut Enerji: 0
Mevcut Bandwidth: 1,420 (ücretsiz)
```

### Teklif

```
get_swap_quote:
  Giriş: 0.1 TRX (100,000 SUN)
  Çıkış: 0.032847 USDT
  Fiyat etkisi: 0.001%
  Gerekli enerji: 223,354
  Rota: WTRX -> USDT
```

### Yürütme

```
execute_swap:
  Satın alınan enerji: 225,000, 11.84 TRX (sağlayıcı: sohu)
  Delegasyon onaylandı: 4.1 saniye
  Onay: Gerekli değil (TRX -> USDT, yerli TRX onay gerektirmez)
  Takas TX yayınlandı: 9c7b3a2f...
  Takas TX onaylandı: Blok 58,234,892
```

### Takas Sonrası Durum

```
Adres: TWallet...
TRX Bakiyesi: 511.516789 TRX  (523.456789 - 0.1 takas - 11.84 enerji)
USDT Bakiyesi: 0.032847 USDT
Mevcut Enerji: 1,646  (225,000 satın alınan - 223,354 kullanılan)
Mevcut Bandwidth: 1,000 (ücretsiz, TX tarafından kısmen tüketildi)
```

### Maliyet Ayrıntısı

```
Enerji satın alımı:      11.84 TRX
Takas girdisi:            0.10 TRX
Toplam harcanan:         11.94 TRX

Enerji satın alımı olmadan (TRX yakılması):
  Takas enerji yakılması:   ~53.00 TRX
  Takas girdisi:             0.10 TRX
  Toplam:                   ~53.10 TRX

Tasarruf:                41.16 TRX (%77.5)
```

## Fiyat Etkisi Tahmini

Daha büyük işlemler yapan aracılar için, fiyat etkisini anlamak kritiktir. `get_swap_quote` aracı bu tahmini sağlar, ancak nasıl hesaplandığını anlamak aracıların daha iyi kararlar almalarına yardımcı olur.

SunSwap V2, sabit ürün formülünü kullanır: `x * y = k`, burada x ve y havuzdaki token rezervleridir. Token X'in dx'ini Token Y'nin dy'si için değiştirdiğinizde:

```
dy = y * dx / (x + dx)
```

Aldığınız etkili fiyat (dy/dx) her zaman spot fiyattan (y/x) daha kötüdür çünkü işleminiz X arzını artırır ve Y arzını azaltır.

45 milyon TRX ve 14.8 milyon USDT havuzu için:

```
Spot fiyatı: 14,800,000 / 45,000,000 = 0.328889 USDT/TRX

100 TRX takası:
  Çıkış: 14,800,000 * 100 / (45,000,000 + 100) = 0.032888 USDT/TRX
  Etki: 0.0003%

100,000 TRX takası:
  Çıkış: 14,800,000 * 100,000 / (45,000,000 + 100,000) = 32.797 USDT
  Etkili fiyat: 0.32797 USDT/TRX
  Etki: 0.28%

1,000,000 TRX takası:
  Çıkış: 14,800,000 * 1,000,000 / (45,000,000 + 1,000,000) = 321,739 USDT
  Etkili fiyat: 0.32174 USDT/TRX
  Etki: 2.17%
```

MERX bu etki hesaplamasını her teklife dahil ederek aracıya işlem boyutunun mevcut likidite için uygun olup olmadığını değerlendirmesine izin verir.

## Çok Atlama Takasları

Tüm token çiftleri doğrudan likidite havuzlarına sahip değildir. WTRX'ye karşı işlem gören ancak birbirleri arasında işlem görmeyen token'ler için, SunSwap V2 ara bir token aracılığıyla yönlendirir:

```
Token A -> WTRX -> Token B
```

Bu, tek bir işlemde iki takas işlemi gerektirir (router sözleşmesi tarafından işlenebilir). Enerji tüketimi daha yüksektir:

```
Doğrudan takas (ör. TRX -> USDT): ~223,000 enerji
İki atlama takası (ör. USDC -> USDT, WTRX aracılığıyla): ~340,000 enerji
Üç atlama takası: ~460,000 enerji
```

`get_swap_quote` aracı otomatik olarak optimal rotayı bulur ve tam yol için toplam enerji gereksinimini raporlar.

## Aracılar İçin Takas Stratejileri

### Dolar-Maliyet Ortalaması

USDT biriktirmeyle görevlendirilen bir aracı, sabit emirleri takas yürütmesiyle birlikte kullanabilir:

```
Sabit emir:
  Tetikleyici: program "0 */4 * * *" (her 4 saatte bir)
  İşlem: 100 TRX -> USDT pazar fiyatında takas yürüt
  Kısıtlama: günde maksimum 600 TRX
```

Günde altı takas, her biri 100 TRX için, fiyat oynaklığını yumuşatır.

### Eşik Tabanlı İşlem

```
Sabit emir:
  Tetikleyici: TRX/USDT fiyatı 0.35'in üzerinde
  İşlem: 1000 TRX -> USDT takası yürüt
  Kısıtlama: günde maksimum 1 yürütme
```

Aracı fiyat uygun olduğunda TRX satarak USDT biriktirir operasyonel giderler için.

### Yeniden Dengeleme

Portföy yöneten bir aracı, yeniden dengeleme için niyetleri kullanabilir:

```
execute_intent:
  Adım 1: 500 TRX -> USDT takası yap (TRX tahsisi > 60% ise)
  Adım 2: 200 USDT -> TRX takası yap (TRX tahsisi < 40% ise)
  Kaynak stratejisi: batch_cheapest
```

## Karşılaştırma: Ham TronWeb vs MERX MCP

### Ham TronWeb (MERX olmadan)

```javascript
// 1. Enerjiyi tahmin et (manuel)
const estimation = await tronWeb.transactionBuilder.triggerConstantContract(
  routerAddress, 'swapExactETHForTokens(uint256,address[],address,uint256)',
  { callValue: 100000 },
  [{ type: 'uint256', value: 0 },
   { type: 'address[]', value: [wtrxAddr, usdtAddr] },
   { type: 'address', value: myAddr },
   { type: 'uint256', value: deadline }],
  myAddr
);
// 2. Bir yere enerji satın al (ayrı entegrasyon)
// 3. Delegasyonu bekle (manuel yoklama)
// 4. Takas işlemini oluştur
// 5. İmzala
// 6. Yayınla
// 7. Doğrula
// ~50 satır kod, çoklu API entegrasyonları
```

### MERX MCP (tek araç çağrısı)

```
Tool: execute_swap
Input: {
  "from_token": "TRX",
  "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "amount": "100000",
  "slippage": 1.0
}
// Bir çağrı. Her şey dahili olarak işlenir.
```

Aracının router adreslerini, WTRX sarılmasını, ABI kodlamasını, enerji pazarlarını, delegasyon mekaniklerini veya işlem yapısını bilmesi gerekmez. Niyet ifade eder ("TRX'i USDT'ye dönüştür") ve bir sonuç alır.

## Riskler ve Sınırlamalar

### Slipaj

Slipaj koruması olsa bile, hızlı fiyat hareketleri takasların geri alınmasına neden olabilir. Takas geri alınırsa, geri alınan işlem için kullanılan enerji kaybolur (token'ler olmasa da). Aracı daha yüksek bir slipaj toleransı veya daha küçük bir miktar ile yeniden deneyebilir.

### MEV ve Ön Çalıştırma

TRON'un blok üretim modeli Ethereum'dan farklıdır ve MEV ortamı daha az gelişmiştir. Ancak, SunSwap'ta büyük takaslar hala mempool'u izleyen sofistike aktörlerin ön çalıştırması yapılabilir. Büyük işlemler için, birden fazla küçük takaslar içine bölmeyi farklı bloklar genelinde düşünün.

### Likidite

SunSwap V2 likiditesi token çiftleri arasında önemli ölçüde değişir. Büyük çiftler (TRX/USDT, TRX/USDC) derin likiditeye sahiptir. Küçük tokenler ince havuzlara sahip olabilir ve burada bile orta ölçekli işlemler önemli fiyat etkisine neden olur. Yürütmeden önce her zaman teklifi kontrol edin.

## Sonuç

Bir AI aracısı aracılığıyla DEX ticareti artık teorik bir kapasite değildir. MERX bunu iki araç çağrısıyla operasyonel hale getirir: biri teklif etmek, biri yürütmek. Enerji simülasyonu tam. Kaynak satın alımı otomatik. Token onayları şeffaf olarak işlenir.

TRON'da faaliyet gösteren AI aracıları için, MCP aracılığıyla SunSwap, zincir üzerinde ticaret için en hızlı yoldur. Web UI yok. Hiçbir manuel işlem yapısı yok. Kaynak yönetimi ek yükü yok.

Teklif. Yürüt. Bitti.

---

**Bağlantılar:**
- MERX Platformu: [https://merx.exchange](https://merx.exchange)
- MCP Sunucusu (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP Sunucusu (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)


## Şimdi AI ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin -- sıfır yükleme, salt okunur araçlar için API anahtarı gerekli değil:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI aracınıza sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatları alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)