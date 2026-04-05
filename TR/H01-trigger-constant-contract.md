# triggerConstantContract Nasıl Tam Enerji Simülasyonunu Sağlar

TRON enerjisi satın alan her geliştirici aynı sorunu yaşamıştır: işlemim gerçekten ne kadar enerji tüketiyor? Standart yaklaşım, sabit tahminleri kullanmaktır -- USDT transferi için 65.000, DEX swap'i için 200.000 -- ve tahminlerin yeterince yakın olduğunu umut etmektir. Genellikle değildir.

Çözüm TRON protokolünün kendisinde mevcuttur: `triggerConstantContract`, bir işlemi yayınlamadan akıllı kontrat yürütümünü simüle eden kuru çalışma API'ı. Bu makale, bu mekanizmanın nasıl çalıştığına, MERX'in bunu hassas enerji tahmini için nasıl kullandığına ve neden sabit değerlerden temelde daha iyi sonuçlar ürettiğine ilişkin teknik bir inceleme sunmaktadır.

## Sabit Tahminlerin Sorunu

Bir USDT transferini düşünün. Yaygın olarak belirtilen rakam 65.000 enerjidir. Ancak gerçek tüketilen enerji birden fazla faktöre bağlıdır:

- **İlk kez alıcı**: Alıcı adresi hiç USDT tutmadıysa, kontrat yeni bir bakiye eşlemesi oluşturmalıdır. Bu depolama ayırması, mevcut bir bakiyeyi güncellemeye kıyasla önemli ölçüde daha fazla enerji maliyetine neden olur.
- **Kontrat durumu**: USDT kontratının dahili durumu (toplam tutucu sayısı, depolama düzeni) gaz tüketimini etkiler.
- **Onay durumu**: Transfer onaylı bir allowance'ı içeriyorsa (transferFrom vs doğrudan transfer), yürütme yolu ve enerji maliyeti farklılık gösterir.
- **Token miktarı**: Miktarı doğrudan çoğu ERC-20/TRC-20 uygulamasında enerjiyi etkilemese de, özel mantığı olan bazı tokenler (vergiler, yeniden tabanlama, hook'lar) miktara bağlı olarak değişken enerji tüketir.

"65.000 enerji" USDT transferi gerçekten şunları tüketebilir:

- 31.895 enerji (mevcut sahibine doğrudan transfer, optimal yol)
- 64.285 enerji (mevcut sahibine standart transfer)
- 65.527 enerji (yeni sahibine transfer, yeni depolama yuvası)
- 94.000+ enerji (transfer hook'larıyla karmaşık token)

65.000'i sabit tahmin olarak kullanmak, bazı durumlarda aşırı satın almayı (para kaybetme) ve diğerlerinde yetersiz satın almayı (kısmi TRX yakma) anlamına gelir.

## triggerConstantContract Ne Yapar

`triggerConstantContract`, TRON tam nodu API yöntemidir ve akıllı kontrat çağrısını salt okunur simülasyon ortamında yürütür. Nodu, çağrıyı gerçek bir işlem için yapacağı gibi işler -- tüm depolama okumalarını, durum kontrollerini ve hesaplama adımlarını içererek -- ancak aşağıdakileri yapmaz:

- İşlemi ağa yayınlamak
- Herhangi bir blockchain durumunu değiştirmek
- Herhangi bir gerçek enerji veya TRX tüketmek
- Herhangi bir bakiye veya yetkilendirme gerektirmek

Yanıt, simülasyon sırasında tüketilen tam enerjiyi (gas), dönüş değerini ve meydana gelecek herhangi bir durum değişikliğini içerir.

### API Uç Noktası

Yöntem TRON tam düğümleri ve TronGrid aracılığıyla mevcuttur:

```
POST https://api.trongrid.io/wallet/triggerconstantcontract
```

### İstek Yapısı

```json
{
  "owner_address": "TYourAddress...",
  "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "function_selector": "transfer(address,uint256)",
  "parameter": "0000000000000000000000...",
  "visible": true
}
```

Ana alanlar:

- **owner_address**: İşlemi gönderecek adres (depolama okuma maliyetlerini ve yetkilendirme kontrollerini etkiler)
- **contract_address**: Çağrılacak akıllı kontrat
- **function_selector**: Solidity formatında işlev imzası
- **parameter**: ABI-kodlanmış işlev parametreleri

### Yanıt Yapısı

```json
{
  "result": {
    "result": true,
    "code": "SUCCESS",
    "message": ""
  },
  "energy_used": 64285,
  "constant_result": ["0000000000000000000000000000000000000001"],
  "transaction": {
    "ret": [{ "contractRet": "SUCCESS" }]
  }
}
```

`energy_used` alanı, mevcut kontrat durumunda bu belirli parametrelerle bu belirli çağrı için tam enerji tüketimini içerir.

## ABI Kodlaması

`parameter` alanı ABI-kodlanmış işlev argümanları gerektirir. Doğru simülasyon istekleri oluşturmak için ABI kodlamasını anlamak gereklidir.

### Temel Türler

ABI kodlaması tüm değerleri 32 bayta (64 hex karakteri) doldurur:

```
address: 32 bayta doğru doldurmak
  TJGPeXwDpe6MBY2gwGPVbXbNJhkALrfLjX
  -> 0000000000000000000000005e09d2c48fee51bfb71e4f4a5d3e2f2c3a8b7d01

uint256: 32 bayta doğru doldurmak
  1000000 (6 ondalak biçiminde 1 USDT)
  -> 00000000000000000000000000000000000000000000000000000000000f4240
```

### USDT Transferi Kodlaması

`transfer(address,uint256)` için `TRecipient...` alıcı ve `1000000` miktarıyla:

```
parameter = <recipient_padded_32_bytes><amount_padded_32_bytes>
```

### ABI Kodlaması için TronWeb Kullanması

TronWeb ABI kodlamasını basitleştirir:

```typescript
import TronWeb from 'tronweb';

const tronWeb = new TronWeb({
  fullHost: 'https://api.trongrid.io',
  headers: { 'TRON-PRO-API-KEY': process.env.TRONGRID_KEY }
});

// Yöntem 1: triggerConstantContract'ı doğrudan kullanma
const result = await tronWeb.transactionBuilder.triggerConstantContract(
  'TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t', // USDT kontratı
  'transfer(address,uint256)',
  {},
  [
    { type: 'address', value: recipientAddress },
    { type: 'uint256', value: 1000000 }
  ],
  senderAddress
);

console.log(`Kullanılan enerji: ${result.energy_used}`);
```

### Karmaşık İşlev İmzaları

Daha karmaşık çağrılar (DEX swap'leri, NFT basımlar) için ABI kodlaması birden fazla parametre ve potansiyel olarak dinamik türler içerir:

```typescript
// SunSwap swap simülasyonu
const swapResult = await tronWeb.transactionBuilder.triggerConstantContract(
  SUNSWAP_ROUTER_ADDRESS,
  'swapExactTokensForTokens(uint256,uint256,address[],address,uint256)',
  {},
  [
    { type: 'uint256', value: amountIn },
    { type: 'uint256', value: amountOutMin },
    { type: 'address[]', value: [tokenA, WTRX, tokenB] },
    { type: 'address', value: recipientAddress },
    { type: 'uint256', value: deadline }
  ],
  senderAddress
);

console.log(`Swap enerjisi: ${swapResult.energy_used}`);
// Varsayılan 200.000 yerine 187.432 döndürebilir
```

## MERX triggerConstantContract'ı Nasıl Kullanır

MERX, triggerConstantContract işlevselliğini `estimateEnergy` yönteminde sarmalamakta ve birden fazla değer katmanı eklemektedir:

### Basitleştirilmiş Arayüz

```typescript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY });

const estimate = await merx.estimateEnergy({
  contract_address: 'TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t',
  function_selector: 'transfer(address,uint256)',
  parameter: [recipientAddress, 1000000],
  owner_address: senderAddress
});

console.log(`Tam enerji: ${estimate.energy_required}`);
```

MERX ABI kodlamasını dahili olarak işler, bu nedenle hex-kodlanmış baytlar yerine okunabilir parametreleri geçersiniz.

### Fiyatlandırma ile Entegrasyon

Tahmin doğrudan fiyatlandırma motoruyla entegre edilir:

```typescript
// Enerji tahmin et
const estimate = await merx.estimateEnergy({
  contract_address: USDT_CONTRACT,
  function_selector: 'transfer(address,uint256)',
  parameter: [recipient, amount],
  owner_address: sender
});

// Tam miktar için fiyat al
const prices = await merx.getPrices({
  energy_amount: estimate.energy_required,
  duration: '5m'
});

// En iyi fiyatla tam olarak ihtiyacın olan miktarı satın al
const order = await merx.createOrder({
  energy_amount: estimate.energy_required,
  duration: '5m',
  target_address: sender
});

// Toplam maliyet en aza indirilir: tam miktar en iyi fiyatla
const costTrx =
  (prices.best.price_sun * estimate.energy_required) / 1e6;
console.log(`Toplam maliyet: ${costTrx.toFixed(4)} TRX`);
```

### Hata Algılama

Simüle edilen işlem geri dönerse (yetersiz bakiye, yetkisiz çağrı, kontrat hatası), simülasyon bunu enerji için para harcamadan önce yakalar:

```typescript
try {
  const estimate = await merx.estimateEnergy({
    contract_address: USDT_CONTRACT,
    function_selector: 'transfer(address,uint256)',
    parameter: [recipient, amount],
    owner_address: sender
  });
} catch (error) {
  if (error.code === 'SIMULATION_REVERT') {
    console.error(
      'İşlem başarısız olacak: ' + error.message
    );
    // Başarısız olacak bir işlem için enerji satın almayın
  }
}
```

Bu, başarısız olamayan bir işlem için enerji satın almak gibi yaygın ve pahalı hatayı önler.

## Sabit Tahminlerle Karşılaştırma

### Doğruluk

| İşlem Türü | Sabit Tahmin | triggerConstantContract | Fark |
|---|---|---|---|
| USDT transferi (mevcut tutucu) | 65.000 | 64.285 | -1,1% |
| USDT transferi (yeni tutucu) | 65.000 | 65.527 | +0,8% |
| USDT transferFrom | 65.000 | 51.481 | -20,8% |
| SunSwap basit swap | 200.000 | 143.287 | -28,4% |
| SunSwap çok atlamalı swap | 200.000 | 212.456 | +6,2% |
| NFT basım (basit) | 150.000 | 112.340 | -25,1% |
| NFT basım (karmaşık) | 150.000 | 267.891 | +78,6% |

Farklar rastgele gürültü değildir -- verilen işlem türleri ve durumlar için tutarlıdırlar. Sabit tahminler işleme bağlı olarak %1-80 yanlıştır.

### Maliyet Etkisi

28 SUN'da günde 1.000 USDT transferi işleyen bir sistem için sabit (65.000) ve tam (ortalama 63.500) tahmin arasındaki maliyet farkı:

- Sabit: 65.000 x 1.000 x 28 = 1.820.000.000 SUN = 1.820 TRX
- Tam: 63.500 x 1.000 x 28 = 1.778.000.000 SUN = 1.778 TRX
- Günlük tasarruf: 42 TRX ($5,04)
- Aylık tasarruf: 1.260 TRX ($151)
- Yıllık tasarruf: 15.330 TRX ($1.840)

Sabit tahminin daha da yanlış olduğu DEX işlemleri (200.000 vs gerçek ~155.000 ortalama) için tasarruflar orantılı olarak daha büyüktür.

## Kenar Durumlar ve Dikkat Edilecekler

### Durum Bağımlı Sonuçlar

Simülasyon sonuçları mevcut kontrat durumu için geçerlidir. Simülasyon ve yürütme arasında kontrat durumu değişirse (başka bir işlem ilgili bir depolama yuvasını değiştirse), gerçek enerji tüketimi biraz farklılık gösterebilir.

Uygulamada, bu token transferleri gibi yaygın işlemler için nadiren önemlidir. Havuz bakiyeleri veya küresel duruma bağlı olan karmaşık DeFi etkileşimleri için simülasyon sonucuna küçük bir tampon ekleyin (%2-5):

```typescript
const estimate = await merx.estimateEnergy({
  contract_address: DEX_ROUTER,
  function_selector: 'swap(...)',
  parameter: swapParams,
  owner_address: sender
});

// Duruma bağlı işlemler için %5 tampon ekle
const energyToOrder =
  Math.ceil(estimate.energy_required * 1.05);
```

### İlk Çağrı vs Sonraki Çağrılar

Bazı akıllı kontratlar, yeni bir adresin ilk etkileşiminde çalışan başlatma mantığına sahiptir. İlk çağrı sonraki çağrılardan daha fazla enerji maliyetine neden olabilir. Simülasyon bunu doğru şekilde yakalar çünkü mevcut durumu yansıtır -- adres hiç kontratla etkileşim kurmadıysa, simülasyon başlatma maliyetini içerir.

### Gaz vs Enerji

TRON'un EVM uygulamasında, gaz ve enerji kavramsal olarak eşdeğerdir ancak farklı birimleri kullanırlar. `triggerConstantContract` yanıtı, doğrudan sağlayıcılardan satın almanız gereken enerji birimlerinde değeri döndürür.

### Oran Sınırları

TronGrid, `triggerConstantContract` dahil API çağrılarına oran sınırları uygular. Yüksek frekanslı işlemler için ücretli TronGrid planı kullanın veya kendi tam düğümünüzü çalıştırın. MERX'in tahmin uç noktası, sorguları birden fazla tam düğüm bağlantısı arasında dağıtarak oran sınırlamasını dahili olarak işler.

## Entegrasyon Desenleri

### İşlem Öncesi Tahmin

En yaygın desen: her işlemden önce tahmin yapın.

```typescript
async function sendWithExactEnergy(
  contract: string,
  method: string,
  params: any[],
  sender: string
): Promise<string> {
  // 1. Simüle et
  const estimate = await merx.estimateEnergy({
    contract_address: contract,
    function_selector: method,
    parameter: params,
    owner_address: sender
  });

  // 2. Tam enerji satın al
  const order = await merx.createOrder({
    energy_amount: estimate.energy_required,
    duration: '5m',
    target_address: sender
  });

  await waitForFill(order.id);

  // 3. Sıfır atık ile işlemi yürüt
  return await broadcastTransaction(
    contract, method, params, sender
  );
}
```

### Toplu Tahmin

Toplu işlemler için tüm işlemleri simüle edin ve enerjileri toplamda satın alın:

```typescript
async function batchWithExactEnergy(
  operations: Operation[]
): Promise<void> {
  let totalEnergy = 0;

  for (const op of operations) {
    const estimate = await merx.estimateEnergy({
      contract_address: op.contract,
      function_selector: op.method,
      parameter: op.params,
      owner_address: op.sender
    });
    totalEnergy += estimate.energy_required;
  }

  // Tüm işlemler için tek satın alma
  await merx.createOrder({
    energy_amount: Math.ceil(totalEnergy * 1.02),
    duration: '30m',
    target_address: operations[0].sender
  });
}
```

### Önbelleğe Alınmış Tahmin

Aynı kontrat ve benzer parametrelerle tekrarlayan işlemler için tahmini önbelleğe alın ve periyodik olarak yenileyin:

```typescript
class EstimationCache {
  private cache = new Map<string, {
    energy: number;
    timestamp: number;
  }>();
  private ttlMs = 300000; // 5 dakika

  async getEstimate(
    contract: string,
    method: string,
    params: any[],
    sender: string
  ): Promise<number> {
    const key = `${contract}:${method}:${sender}`;
    const cached = this.cache.get(key);

    if (cached && Date.now() - cached.timestamp < this.ttlMs) {
      return cached.energy;
    }

    const estimate = await merx.estimateEnergy({
      contract_address: contract,
      function_selector: method,
      parameter: params,
      owner_address: sender
    });

    this.cache.set(key, {
      energy: estimate.energy_required,
      timestamp: Date.now()
    });

    return estimate.energy_required;
  }
}
```

Önbelleğe alma, enerji maliyetinin stabil olduğu işlemler (mevcut tutucuların sahip olduğu tokenler) için uygulanmalıdır ancak maliyetin önemli ölçüde değiştiği işlemler (havuz durumu sık sık değişen DeFi swap'ları) için kaçınılmalıdır.

## Sonuç

`triggerConstantContract` enerji satın almayı tahmin oyunundan tam hesaplamaya dönüştürür. İşleminizin ne kadar enerji gerektirdiğini tahmin etme ve tahminin yeterince yakın olduğunu umma yerine, mevcut kontrat durumuna karşı tam işlemi simüle edin ve tam sayıyı alın.

MERX bu özelliği doğrudan enerji satın alma iş akışına entegre etmiştir. Simüle edin, tam miktarı alın, yedi sağlayıcıdan en iyi mevcut fiyattan satın alın ve işlemi sıfır atık ve sıfır TRX yakma ile yürütün.

Teknik mekanizm açıktır -- enerji tüketimini raporlayan ancak yayınlamayan akıllı kontrat çağrınızın kuru işletimi. Pratik etki önemlidir -- hem aşırı satın almak hem de yetersiz satın alma cezaları ortadan kaldırarak, herhangi bir anlamlı ölçekte enerji için para harcamadan önce başarısız olacak işlemleri yakalarken.

TRON'da inşa eden geliştiriciler için tam simülasyon bir optimizasyon değildir -- herhangi bir anlamlı ölçekteki maliyet etkin işlemler için bir gerekliliktir.

Tahmin API'ını [https://merx.exchange/docs](https://merx.exchange/docs) adresinde keşfedin veya platformu [https://merx.exchange](https://merx.exchange) adresinde deneyin. Tahmin yetenekleriyle yapay zeka aracısı entegrasyonu için MCP sunucusuna bakın [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp).


## Şimdi Yapay Zeka ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin -- yüklemesiz, salt okunur araçlar için API anahtarı gerekmez:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Yapay zeka aracınıza sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)