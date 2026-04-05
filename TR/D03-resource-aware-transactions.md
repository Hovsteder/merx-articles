# Kaynağa Duyarlı İşlemler: MERX Her TX'i Otomatik Olarak Nasıl Optimize Eder

## TRON Kaynaklarını Göz Ardı Etmenin Gizli Maliyeti

TRON ağındaki her işlem iki tür kaynağı tüketir: enerji ve bant genişliği. Enerji akıllı kontrat yürütmesini güçlendirir - her opcode, her depolama yazması, her token transferi. Bant genişliği işlemin kendisinin ham baytlarını kapsar. Bu kaynakların size devredilmesi veya stake edilmesi yoksa, ağ maliyeti karşılamak için TRX'inizi yakıyor.

Basit bir USDT transferi için bu yakma 27 TRX'e ulaşabilir - şu anki fiyatlarla kabaca 7 dolar. Bir DEX swap'ı için 50 TRX'i aşabilir. Çoğu cüzdan ve SDK bunu olmuş gibi görmezden gelir. İşlemi yayınlarlar, ağ TRX'inizi yakar ve daha ucuz bir seçenek olduğundan hiç haberdar olmadan maksimum olası ücreti ödersiniz.

MERX temelden farklı bir yaklaşım benimser. MERX MCP sunucusundan geçen her işlem, maliyetleri tahmin eden, mevcut kaynakları kontrol eden, yalnızca eksik kısmı satın alan, devredilmeyi bekleyen ve ancak ardından imzalayan ve yayınlayan bir kaynağa duyarlı bir boru hattından geçer. Sonuç her işlemde tutarlı %80-90 tasarruf eder.

Bu makale tam olarak bu boru hattının nasıl çalıştığını açıklar.

## Kaynağa Duyarlı İşlem Boru Hattı

Boru hattının altı aşaması vardır. Her aşama bir sonrakine başlamadan önce tamamlanmalıdır. Bir aşamayı atlamak veya yeniden sıralamak, başarısız işlemlere veya israf edilmiş kaynak satın alımlarına neden olabilecek yarış koşulları oluşturur.

### Aşama 1: Enerji ve Bant Genişliğini Tahmin Edin

Hiçbir şey yapmadan önce, MERX bu belirli işlemin tam olarak kaç kaynak tüketeceğini bilmesi gerekir. Bu bir arama tablosu veya sabit kodlanmış bir sabit değildir. MERX, tam parametrelerle tam işlemi blockchain'in mevcut durumuna karşı simüle etmek için `triggerConstantContract` kullanır.

A adresinden B adresine 100 USDT transferi için:

```
Tool: estimate_transaction_cost
Input: {
  "from": "TAddressA...",
  "to": "TAddressB...",
  "contract": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "function": "transfer(address,uint256)",
  "parameters": ["TAddressB...", 100000000],
  "value": 0
}

Response:
{
  "energy_required": 64895,
  "bandwidth_required": 345,
  "cost_without_resources": 27.12,
  "cost_with_resources": 3.42
}
```

Simülasyon canlı blockchain durumuna karşı çalışır. Alıcı daha önce hiç USDT tutmadıysa, enerji maliyeti daha yüksek olur (çünkü kontrat yeni bir depolama yuvası oluşturması gerekir). Gönderenin izni güncellenmesi gerekiyorsa, bu enerji ekler. Her değişken dikkate alınır.

Bir SunSwap işlemi için simülasyon tam yönlendirmeyi, kayma hesaplamasını ve likidite havuzu durumunu yakalar:

```
Simulation result for SunSwap V2:
- Energy required: 223,354
- Bandwidth required: 420
- Router path: TRX -> USDT via pool 0x...
```

Bu hassasiyet kritiktir. Aşırı tahmin etmek kullanılmayan enerji için para harcaması getirir. Yetersiz tahmin etmek işlemin başarısız olmasına neden olur ve hem enerji satın alma maliyetini hem de başarısız işlemin bant genişliğini kaybedersiniz.

### Aşama 2: Mevcut Kaynakları Kontrol Edin

Ajanın adresi önceki delegasyonlardan, stake etmeden veya günlük ücretsiz bant genişliği limitinden bazı kaynakları zaten sahibi olabilir. MERX nelerin zaten kullanılabilir olduğunu kontrol eder:

```
Tool: check_address_resources
Input: { "address": "TAddressA..." }

Response:
{
  "energy": {
    "available": 12000,
    "total": 12000,
    "used": 0
  },
  "bandwidth": {
    "available": 1400,
    "total": 1500,
    "used": 100,
    "free_available": 1400,
    "free_total": 1500
  }
}
```

Bu örnekte, adres 12.000 enerji ve günlük ücretsiz dağıtımdan 1.400 bant genişliğine sahiptir.

### Aşama 3: Açığı Hesaplayın

MERX mevcut kaynakları gerekli kaynaklardan çıkararak tam olarak neyin satın alınması gerektiğini belirler:

```
Energy needed:     64,895
Energy available:  12,000
Energy deficit:    52,895
-> Rounded up to:  65,000 (minimum order unit)

Bandwidth needed:    345
Bandwidth available: 1,400
Bandwidth deficit:     0 (sufficient)
```

Burada iki önemli kural geçerlidir:

**Enerji minimum: 65.000 birim.** TRON enerji delegasyonu pazarı yaklaşık 65.000 enerjinin minimum blokları içinde çalışır. Açık 65.000'den azsa, MERX 65.000'e yuvarlar. Açık 0 ise (adres zaten yeterli energiye sahipse), satın alma yapılmaz.

**Bant genişliği eşiği: 1.500 birim.** Bant genişliği açığı 1.500'den azsa, MERX bant genişliği satın almayı tamamen atlar. Her TRON adresi günde 1.500 ücretsiz bant genişliği alır ve bu sürekli yenilenir. Çoğu tek işlem için ücretsiz tahsis yeterlidir. Bant genişliği satın alma yalnızca günlük tahsisi tüketen yüksek frekans işlemleri için mantıklıdır.

### Aşama 4: Açığı Satın Alın

Tam açık hesaplandığında, MERX en iyi fiyat için tüm mevcut enerji sağlayıcılarını sorgular:

```
Tool: get_best_price
Input: {
  "energy_amount": 65000,
  "duration_hours": 1
}

Response:
{
  "best_price": 3.42,
  "provider": "sohu",
  "all_prices": [
    { "provider": "sohu", "price": 3.42 },
    { "provider": "catfee", "price": 3.51 },
    { "provider": "netts", "price": 3.65 },
    { "provider": "tronsave", "price": 3.78 }
  ]
}
```

MERX siparişi en ucuz sağlayıcıyla yerleştirir:

```
Tool: create_order
Input: {
  "energy_amount": 65000,
  "duration_hours": 1,
  "target_address": "TAddressA..."
}
```

Sipariş yerleştirilir ve sağlayıcı delegasyon sürecini başlatır.

### Aşama 5: Delegasyon Gelinceye Kadar Yokla

Bu, çoğu uygulamanın yanlış yaptığı aşamadır. TRON'da enerji delegasyonu anlık değildir. Sağlayıcı delegasyon işlemini yayınladıktan sonra, ağ tarafından doğrulanması gerekir. Bu tipik olarak 3-6 saniye sürer ancak ağ tıkanıklığı sırasında daha uzun sürebilir.

MERX hedef adresin kaynak bakiyesini düzenli aralıklarla yoklar:

```
Polling cycle:
  t+0s:  check_address_resources -> energy: 12,000 (not yet)
  t+3s:  check_address_resources -> energy: 12,000 (not yet)
  t+6s:  check_address_resources -> energy: 77,000 (delegation arrived)
  -> Proceed to Stage 6
```

Yoklama, 3 saniyeden başlayan ve zamanı dolmadan 60 saniye maksimum bekleme ile üstel geri çekilme kullanır. Delegasyon zaman penceresi içinde gelmezse, boru hattı, yine de TRX'i yakacak bir işlemi yayınlamak yerine bir hata bildirir.

Bu yoklama aşaması, naif uygulamaları rahatsız eden yarış koşulunu önleyen şeydir. Onsuz, dizi şöyle olur: enerji satın al, hemen işlemi yayınla, işlem delegasyon doğrulanmadan önce yürütülür, TRX'e yine de yakılır ve hem enerji hem de yakma için ödeme yaptınız.

### Aşama 6: Yerel Olarak İmzala ve Yayınla

Yalnızca devredilen enerjinin zincir üzerinde mevcut olduğu doğrulandıktan sonra MERX gerçek işlemi imzalar ve yayınlar:

```
1. Build transaction object with TronWeb
2. Sign with local private key (never leaves the machine)
3. Broadcast to TRON network
4. Return transaction hash
```

İşlem artık devredilen enerjiyi kullanarak yürütülür, 27 TRX yakma yerine yaklaşık 65.000 enerji birimi tüketir.

## ensure_resources Aracı: Bir Çağrıda Boru Hattı

Her aşamayı ayrı ayrı yönetmek istemeden boru hattını kullanmak isteyen ajanlar için, MERX `ensure_resources` sağlar:

```
Tool: ensure_resources
Input: {
  "address": "TAddressA...",
  "energy_needed": 65000,
  "bandwidth_needed": 345
}
```

Bu tek araç çağrısı Aşamalar 2 ile 5'i dahili olarak çalıştırır. Mevcut kaynakları kontrol eder, açığı hesaplar, en iyi fiyatı bulur, siparişi yerleştirir ve delegasyon gelinceye kadar yoklar. Ajan, yalnızca adres tamamen donatılmış ve işlem için hazır olduğunda bir yanıt alır.

## Gerçek Örnek: Bir SunSwap İşlemi

İşte gerçek bir SunSwap V2 işlemi için tam boru hattı - 0.1 TRX'i USDT için takas etme.

**Aşama 1 - Simülasyon:**

```
triggerConstantContract(
  contract: SunSwapV2Router,
  function: swapExactETHForTokens,
  parameters: [0, [WTRX, USDT], address, deadline],
  call_value: 100000  // 0.1 TRX in SUN
)

Result: energy_estimate = 223,354
```

**Aşama 2 - Kaynakları kontrol et:**

```
Address resources:
  Energy: 0
  Bandwidth: 1,420 (free)
```

**Aşama 3 - Açığı hesapla:**

```
Energy deficit: 223,354
-> Rounded to nearest order unit: 225,000
Bandwidth deficit: 0 (free allocation covers 345 needed)
```

**Aşama 4 - Satın al:**

```
Best price for 225,000 energy / 1 hour:
  Provider: catfee
  Price: 11.82 TRX
```

**Aşama 5 - Yokla:**

```
Delegation confirmed after 4.2 seconds
Address now has 225,000 energy available
```

**Aşama 6 - Yürütün:**

```
Swap transaction broadcast
TX hash: abc123...
Energy consumed: 223,354
Energy remaining: 1,646 (will expire with delegation)
Net cost: 11.82 TRX instead of ~53 TRX burned
Savings: 78%
```

Tüm boru hattı otonomik olarak yürütüldü. Ajan TRX'i USDT için takas etmesini istedi ve MERX sahnenin arkasında her kaynak hesaplaması, satın alımı ve zamanlama sorununun işini yaptı.

## Operasyonlar Sırası Neden Önemlidir

Boru hattının sıkı sırası üç başarısızlık kategorisini önler:

### Yarış Koşulu: Satın Al Sonra Hemen Yayınla

Enerji satın alırsan ve işlemi aynı blokta yayınlarsan, delegasyon henüz işlemmiş olmayabilir. İşlem devredilen enerji olmadan yürütülür, TRX'i yakar ve iki kez ödeme yaptınız - bir kez enerji için (kullanılmamaz) ve bir kez TRX yakması için.

MERX bunu devredilmeyi zincir üzerinde doğrulama bitmeden yoklayarak önler.

### Aşırı Tahmin: Sabit Kodlanmış Enerji Değerleri

Birçok araç sabit enerji tahminleri kullanır (örn. "USDT transferleri her zaman 65.000 enerji maliyeti"). Ancak gerçek maliyet söz konusu adreslere, onların token tutma geçmişine, kontratın iç durumuna ve hatta blok numarasına bağlıdır. Yeni bir adrese transfer mevcut tokeni tutan bir adrese transferden daha pahalıdır.

MERX bunu tam parametrelerle tam işlemi canlı blockchain durumuna karşı simüle ederek önler.

### Yetersiz Tahmin: Yetersiz Kaynaklar

Enerji gereksinimlerini yetersiz tahmin ederseniz ve işlem yürütme sırasında enerjinin bitmesi durumunda başarısız olur. İşlem denemesinin bant genişliğini kaybedersiniz ve satın aldığınız enerji başarısız bir işlemde boşa gider.

MERX bunu `triggerConstantContract` kullanarak tam simülasyon yaparak ve sipariş verirken küçük bir tampon ekleyerek önler.

## Uygulamada Fark

Kaynağa duyarlı işlemler olmadan (standart cüzdan davranışı):

```
USDT transfer: 27 TRX burned (~$7.00)
SunSwap trade: 53 TRX burned (~$13.75)
Approve + Swap: 68 TRX burned (~$17.65)
```

MERX kaynağa duyarlı boru hattıyla:

```
USDT transfer: 3.42 TRX (energy purchase)
SunSwap trade: 11.82 TRX (energy purchase)
Approve + Swap: 14.93 TRX (energy purchase)
```

Günde 100 USDT transferi yürüten bir ajan için, bu günde 700 dolar ile günde 342 dolar arasındaki farktır - yıllık 130.000 dolardan fazla tasarruf.

## Geliştiriciler İçin Entegrasyon

TRON ile etkileşime giren bir uygulama oluşturuyorsanız, MERX kaynağa duyarlı boru hattını entegre etmek mimari değişiklik gerektirmez. MCP sunucusu tüm boru hattını dahili olarak işler.

Doğrudan API entegrasyonu için:

```bash
# Step 1: Get estimation
curl -X POST https://merx.exchange/api/v1/estimate \
  -H "Content-Type: application/json" \
  -d '{"from": "T...", "to": "T...", "amount": 100000000}'

# Step 2: Ensure resources
curl -X POST https://merx.exchange/api/v1/ensure-resources \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{"address": "T...", "energy_needed": 65000}'

# Step 3: Broadcast your transaction (signed client-side)
```

`ensure-resources` uç noktası fiyat karşılaştırması, sipariş yerleştirme ve delegasyon yoklamasını işler. Uygulamanız yalnızca adres hazır olduğunda yanıt alır.

## Sonuç

Kaynağa duyarlı işlemler bir optimizasyon değildir. TRON ağı ile etkileşime girmek için doğru yoldur. Önce yeterli kaynakları sağlamadan işlem yayınlamak, sunucuya erişilebilir olup olmadığını kontrol etmeden HTTP isteği göndermek eşdeğerdir - çalışabilir ancak başarısız olduğunda, fiyatı siz ödersiniz.

MERX doğru yaklaşımı varsayılan yaklaşımın yapır. Her işlem boru hattından geçer. Her kaynak açığı tam olarak hesaplanır. Her satın alma en iyi mevcut fiyattan yapılır. Her delegasyon işlem yayınlanmadan önce doğrulanır.

Sonuç TRON blockchain'e karşı her etkileşimde öngörülebilir, minimum maliyetli işlemlerdir.

---

**Bağlantılar:**
- MERX Platformu: [https://merx.exchange](https://merx.exchange)
- MCP Sunucusu (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP Sunucusu (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)


## Şu Anda AI ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin - sıfır kurulum, salt okunur araçlar için API anahtarı gerekmez:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI ajanınıza sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve tüm bağlantılı sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgesi: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)