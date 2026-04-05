# Tam Enerji Simülasyonu: MERX Bir Swapın Maliyetini Nasıl Tam Olarak Biliyor?

## Tahmin Etmenin Sorunu

Çoğu TRON aracı ve cüzdanı sabit kodlanmış enerji tahminlerini kullanır. Bir USDT transferi? 65.000 enerji bütçelendir. SunSwap'ta bir swap? Belki 200.000. Bir onay? Yaklaşık 50.000. Bu numaralar kodun bir yerinde sabit olarak depolanır ve her işlem gerçek koşullar ne olursa olsun bunları kullanır.

Bu yaklaşımın iki başarısızlık modu vardır ve ikisi de sizi paraya mal olur.

Tahmin çok düşükse, işlem yürütme sırasında enerjiden tükenir. Tüm işlem başarısız olur, ancak yine de denemede tüketilen bant genişliğini ödersiniz. Satın aldığınız enerji, hiçbir sonuç üretmeyen bir işlem için boşa harcanır. Daha fazla enerji satın almanız ve tekrar denemeniz gerekir.

Tahmin çok yüksekse, hiç kullanmayacağınız enerji satın alırsınız. Devredilen enerjinin minimum kiralama süresi vardır - tipik olarak bir saat. Delegasyon sona erdiğinde kalan her şey basitçe kaybolur. Her işlemde %30 fazla tahmin ederseniz ve binlerce işlem içinde, atık ciddi paraya dönüşür.

MERX tahmin etmeyi tamamen ortadan kaldırır. Her enerji tahmini, tam işlemin canlı blockchain durumuna karşı simüle edilmesiyle üretilir. Sonuç bir yaklaşım veya aralık değildir - işlemin tüketecek olduğu tam enerji birimi sayısıdır.

## triggerConstantContract Nasıl Çalışır?

TRON ağı `triggerConstantContract` adlı salt okunur bir simülasyon uç noktası sağlar. Bu uç nokta, akıllı bir sözleşme çağrısını mevcut blockchain durumunu yansıtan ancak onu değiştirmeyen sanal bir ortamda yürütür. Hiçbir işlem yayınlanmaz. Hiçbir kaynak tüketilmez. Hiçbir durum değişikliği işlenir.

Simülasyon gerçek bir işlemde yürütülecek olan tam aynı bytecode'u çalıştırır. Aynı depolama yuvalarını, aynı hesap bakiyelerini, aynı sözleşme mantığını kullanır. Tek fark, sonuçların blockchain'e yazılmak yerine atılmasıdır.

Anahtar çıktı `energy_used`'dir - simüle edilen yürütme tarafından tüketilen tam enerji birimi sayısı.

### API Çağrısı

```javascript
const result = await tronWeb.transactionBuilder.triggerConstantContract(
  contractAddress,       // Çağrılan sözleşme
  functionSelector,      // örn: "transfer(address,uint256)"
  { callValue: 0 },     // Gönderilen herhangi bir TRX değeri dahil seçenekler
  [                      // İşlev parametreleri
    { type: 'address', value: recipientAddress },
    { type: 'uint256', value: amount }
  ],
  senderAddress          // İşlemi gönderecek olan adres
);

const energyUsed = result.energy_used;
```

Bu, Ethereum'un `eth_estimateGas` gibi bir gaz tahmin buluşsal yöntemi değildir, bu da güvenlik marjı olan bir üst sınır döndürür. `triggerConstantContract` yürütme izlemesi tarafından tüketilen tam enerjiyi döndürür. MERX 223.354 enerji döndüren bir swap simüle ettiğinde, zincir üstü yürütme 223.354 enerji tüketecektir.

## Gönderici Adresinin Neden Önemli Olduğu

İnce ama kritik bir ayrıntı: gönderici adresi simülasyon sonucunu etkiler.

Bir USDT transferi düşünün. Gönderici daha önce USDT sözleşmesiyle hiç etkileşimde bulunmadıysa, sözleşme göndericinin bakiyesi için yeni bir depolama yuvası ayırmalıdır. Bu ayırma enerji maliyeti. Göndericinin zaten bir bakiye girişi varsa, sözleşme yalnızca mevcut bir yuvayı güncellemelidir - binlerce enerji birimi daha ucuz.

Benzer şekilde, alıcı adresi önemlidir. USDT'yi hiç USDT tutmayan bir adrese aktarmak, alıcı için bir depolama yuvası oluşturmayı tetikler. USDT bakiyesi olan bir adrese aktarmak bunu tetiklemez.

Bu farklılıklar basit bir transfer için enerji maliyetini 10.000-20.000 birim kaydırabilir. Birden çok iç çağrı ve depolama değişiklikleri içeren karmaşık DeFi işlemleri için varyans daha da büyüktür.

Bu yüzden MERX her işlem için gerçek gönderici ve gerçek alıcı adreslerini simüle eder. Yer tutucu adreslerle genel bir simülasyon, gerçek yürütmeden farklı bir enerji değeri döndürecektir.

## Sabit Kodlanmış Tahminler vs Tam Simülasyon

TRON mainnet'ten gerçek veriler kullanarak somut bir karşılaştırma aşağıda verilmiştir.

### USDT Transfer Senaryoları

| Senaryo | Sabit Kodlanmış Tahmin | Tam Simülasyon |
|---|---|---|
| Gönderici USDT'ye sahip, alıcı USDT'ye sahip | 65.000 | 29.631 |
| Gönderici USDT'ye sahip, alıcı hiç USDT tutmadı | 65.000 | 64.895 |
| Gönderici hiç onay vermedi, alıcı yeni | 65.000 | 64.895 |
| Sözleşme adresine transfer | 65.000 | 47.222 |

Sabit kodlanmış yaklaşım tüm durumlar için 65.000 kullanır. Tam simülasyon, gerçek maliyetlerde 2x aralık ortaya çıkarır. İlk senaryoda, sabit kodlanmış bir tahmin, gerçekten gerekli olandan iki kattan fazla enerji satın almanıza neden olur.

### SunSwap V2 Swap

| Senaryo | Sabit Kodlanmış Tahmin | Tam Simülasyon |
|---|---|---|
| TRX -> USDT, doğrudan havuz | 200.000 | 223.354 |
| USDT -> TRX, doğrudan havuz | 200.000 | 218.847 |
| Token A -> Token B, çok atlamalı | 250.000 | 312.668 |
| Küçük miktar, aynı havuz | 200.000 | 223.354 |

Doğrudan TRX -> USDT swap'ı için 200.000'lik sabit kodlanmış tahmin, 23.354 birim az tahmin ederdi. Bu işlem başarısız olurdu. Alternatif - güvenli olmak için sabit kodlanmış tahmini 250.000'e yükseltmek - her tek swap'ta 26.646 enerji birimi boşa harcar.

## MERX Tam Simülasyonu Nasıl Kullanır?

MERX MCP sunucusu simülasyonu soyutlama düzeylerinde farklı çalışan iki araç aracılığıyla gösterir.

### Düşük Seviye: estimate_contract_call

Bu araç, herhangi bir keyfi sözleşme çağrısını simüle etmenizi sağlar:

```
Araç: estimate_contract_call
Giriş: {
  "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "function_selector": "transfer(address,uint256)",
  "parameters": [
    { "type": "address", "value": "TRecipient..." },
    { "type": "uint256", "value": "100000000" }
  ],
  "from_address": "TSender...",
  "call_value": 0
}

Yanıt:
{
  "energy_used": 64895,
  "result": "0x0000...0001",
  "success": true
}
```

Yanıt hem enerji maliyetini hem de simüle edilen çağrının dönüş değerini içerir. Bir transfer için, dönüş değeri transferin başarılı olup olmayacağını gösterir. Bir swap için, çıktı miktarını döndürür.

### Yüksek Seviye: get_swap_quote

DEX işlemleri için MERX, simülasyonu fiyat alıntılaması ile birleştiren özel bir araç sağlar:

```
Araç: get_swap_quote
Giriş: {
  "from_token": "TRX",
  "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "amount": "100000",
  "from_address": "TSender..."
}

Yanıt:
{
  "output_amount": "0.032847",
  "output_token": "USDT",
  "energy_required": 223354,
  "price_impact": "0.001%",
  "route": ["WTRX", "USDT"],
  "minimum_received": "0.032519"
}
```

`energy_required` alanı, gerçek parametrelerle swap'ın tam simülasyonundan gelir. `output_amount` de simülasyondan gelir, bu nedenle alıntı mevcut blokta swap'ın üretecek olduğu gerçek çıktıyı yansıtır.

## Gerçek Veriler: Simülasyon vs Zincir Üstü Yürütme

MERX MCP sunucusu aracılığıyla TRON mainnet'te yürütülen gerçek bir swap aşağıdadır.

**İşlem: SunSwap V2 aracılığıyla 0,1 TRX'i USDT'ye takas edin**

Öncesi simülasyonu yürütme:

```
triggerConstantContract(
  contract: TKzxdSv2FZKQrEkFPRQi4qeCokAFkrntVM,  // SunSwap V2 Router
  function: swapExactETHForTokens(uint256,address[],address,uint256),
  parameters: [
    0,                                              // amountOutMin
    ["WTRX_address", "USDT_address"],              // path
    "TSender...",                                    // to
    1711814400                                       // deadline
  ],
  call_value: 100000,                               // 0.1 TRX in SUN
  from: "TSender..."
)

Sonuç:
  energy_used: 223,354
  return_value: [32847]  // 0.032847 USDT
```

Zincir üstü yürütme (enerji delegasyonundan sonra):

```
İşlem: abc123...
  Durum: SUCCESS
  Tüketilen enerji: 223,354
  Tüketilen bant genişliği: 420
  Çıktı: 0.032847 USDT alındı
```

Simülasyon 223.354 enerji döndürdü. Zincir üstü yürütme 223.354 enerji tükettti. Tam eşleşme. Yaklaşım değil. Aralık değil. Aynı sayı.

## Kenar Durumları ve MERX Bunları Nasıl Yönetir?

### Simülasyon ve Yürütme Arasında Durum Değişiklikleri

Blockchain durumu simülasyon ile yayınlama arasında değişebilir. Başka bir işlem aynı sözleşme depolamasını değiştirerek enerji maliyetini değiştirebilir. MERX bunu üç şekilde azaltır:

1. **Boşluğu en aza indir.** Pipeline simüle eder, enerji satın alır, delegasyonu yoklar ve en sıkı dizide yayınlar. Tipik boşluk: 5-10 saniye.

2. **Değişken işlemler için küçük bir tampon ekle.** Likidite havuzu durumunun sık sık değiştiği DEX swap'ları için MERX enerji satın almasına %5'lik bir tampon ekler. Bu tampon, minor durum varyasyonlarını maliyeti önemli ölçüde artırmadan kapsar.

3. **Güvenli şekilde başarısız ol.** İşlem tampona rağmen enerjiden tükense bile, durumu değiştirmeden önce başarısız olur. Enerji tüketilir, ancak hiçbir token yanlış aktarılmaz. Ajan yeni bir simülasyonla tekrar deneyebilir.

### Tersine Çevrilen Simülasyonlar

Bazen `triggerConstantContract` bir revert döndürür. Bu, işlemin zincir üstünde başarısız olacağı anlamına gelir. Yaygın nedenler:

- Yetersiz token bakiyesi
- Swap çıktısı minimumun altında (kayma aşıldı)
- Token harcama için onay ayarlanmadı
- Sözleşme duraklatılmış veya kısıtlanmış

MERX bu revert'leri herhangi bir enerji satın alınmadan önce ortaya çıkarır:

```
Yanıt:
{
  "success": false,
  "revert_reason": "INSUFFICIENT_OUTPUT_AMOUNT",
  "energy_used": 0,
  "message": "Swap başarısız olur: çıktı minimumun altında. Kayma toleransını veya miktarı ayarlayın."
}
```

Bu, en pahalı başarısızlık türünü önler: asla başarılı olmayacak olan bir işlem için enerji satın almak.

### Onay İşlemleri

SunSwap'ta TRC20 token swap'ları, swap'tan önce bir onay işlemi gerektirir. Bu, kendi enerji maliyeti olan ayrı bir sözleşme çağrısıdır. MERX onay gerektiğinde algılar ve her iki işlemi simüle eder:

```
Simülasyon sonuçları:
  1. approve(router, MAX_UINT256): 46,312 enerji
  2. swapExactTokensForTokens(...): 223,354 enerji
  Toplam: 269,666 enerji
```

Ajan daha sonra her iki işlem için enerjiyi tek bir siparişle satın alabilir ve iki ayrı enerji pazar etkileşiminin genel masrafından kaçınabilir.

## Simülasyonu İş Akışınıza Entegre Etme

TRON'da oluşturuyorsanız ve tam simülasyon kullanmıyorsanız, masanın üzerinde para bırakıyor veya kendinizi işlem başarısızlıklarına açıyorsunuz. İşte nasıl entegre edeceksiniz:

### MCP Ajan Geliştiricileri için

MERX MCP sunucusunu kullanın. `ensure_resources` ve `execute_swap` araçları simülasyonu otomatik olarak çalıştırır. `estimate_contract_call`'u ayrıca çağırmanız gerekmez çünkü sonuçları incelemek isterseniz.

### API İntegratörleri için

Her işlemden önce MERX tahmin uç noktasını çağırın:

```bash
curl -X POST https://merx.exchange/api/v1/estimate \
  -H "Content-Type: application/json" \
  -d '{
    "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "function_selector": "transfer(address,uint256)",
    "parameters": ["TRecipient...", 100000000],
    "from_address": "TSender...",
    "call_value": 0
  }'
```

### Doğrudan TronWeb Kullanıcıları için

Her akıllı sözleşme etkileşiminden önce `triggerConstantContract`'ı kendiniz çağırın:

```javascript
const simulation = await tronWeb.transactionBuilder.triggerConstantContract(
  contractAddress,
  functionSelector,
  options,
  parameters,
  senderAddress
);

if (!simulation.result.result) {
  console.error('İşlem başarısız olur:', simulation.result.message);
  return;
}

const energyNeeded = simulation.energy_used;
// Şimdi yayınlamadan önce tam bu kadar enerji satın alın
```

## Hassasiyetin Ekonomisi

Günde 1.000 USDT transferi işleme koyan bir uygulamayı düşünün.

Sabit kodlanmış tahminlerle (transfer başına 65.000 enerji):

```
Günlük satın alınan enerji: 65.000.000
Ortalama gerçek kullanım: 47.000.000  (senaryoya göre değişir)
Günlük atık: 18.000.000 enerji
Günlük atık maliyeti: ~94 TRX (~$24)
Yıllık atık: ~$8.760
```

Tam simülasyonla:

```
Günlük satın alınan enerji: 47.200.000  (gerçek + %0,4 yuvarlama)
Günlük atık: 200.000  (minimum sipariş birimlerine yuvarlama)
Günlük atık maliyeti: ~1 TRX (~$0,26)
Yıllık atık: ~$95
```

Fark, tek bir işlem türü için yıllık $8.665'tir. Daha yüksek hacimlerde birden çok işlem türü çalıştıran uygulamalar için tasarruflar doğrusal olarak ölçeklendirilir.

## Sonuç

Tam enerji simülasyonu bir özellik değildir. Ciddi herhangi bir TRON uygulaması için bir gereksinimdir. Sabit kodlanmış tahmin ile tam simülasyon arasındaki fark, tahmin etmek ile bilmek arasındaki farktır. MERX bilmeyi seçer.

Her işlem simüle edilir. Her enerji birimi hesaplanır. Her swap tam olarak alıntılanır. Simülasyon 223.354 der ve zincir 223.354'ü doğrular.

O yaklaşık değildir. O mühendisliktir.

---

**Bağlantılar:**
- MERX Platform: [https://merx.exchange](https://merx.exchange)
- MCP Sunucusu (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP Sunucusu (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)


## Şimdi AI ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin -- kurulum yok, salt okunur araçlar için API anahtarı yok:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI ajanınıza şunu sorun: "Şu anda TRON enerjisinin en ucuzu nedir?" ve bağlı tüm sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)