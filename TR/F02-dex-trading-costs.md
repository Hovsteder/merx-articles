# DEX İşlem Maliyetlerini Enerji Agregasyonu ile Azaltma

TRON üzerinde merkeziyetsiz borsa işlemleri pahalıdır -- işlemlerin kendileri değil, akıllı kontrat etkileşimlerinin tükettiği enerji yüzünden. SunSwap'ta tek bir token takası, token çifti, likidite havuzu yönlendirmesi ve kayma koşullarına bağlı olarak 120.000 ila 223.000 enerji tüketebilir. Enerji delegasyonu olmadan, bu maliyetler doğrudan cüzdanınızdan yakılan TRX'e dönüşür.

Bu makale, TRON üzerinde DEX işlemlerinin enerji ekonomisini açıklar, kesin simülasyonun aşırı harcamayı nasıl engellediğini gösterir ve MERX enerji agregasyonunun işlem maliyetlerini %80 veya daha fazla nasıl azaltabileceğini gösterir.

## DEX Enerji Maliyetlerini Anlama

SunSwap'ta (TRON'un birincil DEX'i) bir takası gerçekleştirdiğinizde, bir akıllı kontrat fonksiyonunu çağırıyorsunuz. Enerji maliyeti, bu çağrının hesaplama karmaşıklığına bağlıdır.

### Tek Atlama İşlemleri

Aynı likidite havuzunu paylaşan iki token arasında doğrudan takası yaklaşık 120.000-150.000 enerji tüketir. Örneğin, TRX/USDT havuzu aracılığıyla TRX'i USDT'ye takas etmek, tek atlama işlemidir.

### Çok Atlama İşlemleri

Token çiftiniz arasında doğrudan likidite havuzu olmadığında, DEX yönlendiricisi işlemi birden fazla havuz arasında böler. Token A'dan Token B'ye bir takası şu şekilde yönlendirebilir: Token A -> TRX -> Token B. Her atlama yaklaşık 50.000-70.000 enerji ekler.

### Karmaşık Rotalar

Bazı takaslar üç veya daha fazla atlama gerektirir ve enerji tüketimini 200.000'in üzerine çıkarır. Fiyat etkisi hesaplamaları, kayma koruması ve diğer kontrat mantığını ekleyin ve tek bir takası 223.000 enerjiye ulaştırabilir.

### Enerji Olmadan Maliyet

Mevcut ağ oranlarında, TRX olarak yakılan 200.000 enerji yaklaşık 41 TRX (~$4.90) maliyetlidir. Günde 10-20 takası gerçekleştiren aktif tüccarlar için, bu ücretlerin kendileri sayılmadan günlük $49-$98 ücrettir.

| Takası Türü | Enerji | TRX Yakma Maliyeti | USD Maliyeti |
|---|---|---|---|
| Basit takası (tek atlama) | ~130.000 | ~27 TRX | ~$3.24 |
| İki atlama takası | ~180.000 | ~37 TRX | ~$4.44 |
| Karmaşık rota (3+ atlama) | ~223.000 | ~46 TRX | ~$5.52 |

## Sabit Kodlanmış Tahminlerin Sorunu

Çoğu DEX arayüzü ve ticaret botu sabit kodlanmış enerji tahminleri kullanır. Bir takasın 200.000 enerji (veya başka bir sabit sayı) maliyetli olduğunu varsayar ve buna göre satın alır veya ayırır.

Bu iki sorunu yaratır:

**Aşırı satın alma.** Takasınız yalnızca 130.000 enerji gerektiriyorsa ancak siz 200.000 satın alırsanız, 70.000 enerji birimi israf edersiniz. 28 SUN per birim fiyatında, bu işlem başına 1.960.000 SUN (1.96 TRX) harcanan enerji demektir.

**Yetersiz satın alma.** Takasınız 223.000 enerji gerektiriyorsa ancak yalnızca 200.000 satın alırsanız, 23.000 enerjinin eksikliği tam ağ oranında TRX yakılarak karşılanır. Geri kalanı için yüksek ücretler ödersiniz.

Her iki senaryo da size para malı olur. Çözüm kesin simülasyondur.

## MERX ile Kesin Simülasyon

MERX, yürütmeden önce özel takasınızı simüle etmek için TRON ağının `triggerConstantContract` uç noktasını kullanır. Bu kuru çalıştırma, takasın tükettiği enerji miktarını tam olarak söyler -- tahmin değil, ortalama değil, özel işleminiz için kesin sayı.

```typescript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY });

// Kesin enerji gereksinimlerini almak için takasını simüle edin
const estimate = await merx.estimateEnergy({
  contract_address: SUNSWAP_ROUTER,
  function_selector: 'swapExactTokensForTokens(uint256,uint256,address[],address,uint256)',
  parameter: [
    amountIn,
    amountOutMin,
    path,         // örn: [tokenA, WTRX, tokenB]
    walletAddress,
    deadline
  ],
  owner_address: walletAddress
});

console.log(`Gerekli kesin enerji: ${estimate.energy_required}`);
// Çıktı: Gerekli kesin enerji: 187432

// Tam olarak ihtiyacınız olanı satın alın -- hiçbir boşa harcanmış enerji yok
const order = await merx.createOrder({
  energy_amount: estimate.energy_required,
  duration: '5m',
  target_address: walletAddress
});
```

Simülasyon, özel takasınız için, özel parametrelerinizle, likidite havuzlarının mevcut durumunda, gerçek enerji tüketimini döndürür. Tahmin yok, sabit arabellek yok.

### Kesin Simülasyondan Tasarruf

100 işlem üzerinde kesin simülasyon ile sabit kodlanmış tahminleri karşılaştırın:

| Yaklaşım | İşlem başına enerji | Toplam enerji (100 işlem) | 28 SUN'da Maliyet |
|---|---|---|---|
| Sabit 200K | 200.000 | 20.000.000 | 560 TRX |
| Kesin simülasyon (ort 155K) | 155.000 | 15.500.000 | 434 TRX |
| **Tasarruf** | | **4.500.000** | **126 TRX (~$15)** |

Aktif ticaretin bir ayı boyunca (500+ işlem), kesin simülasyon yüzlerce TRX tasarrufu sağlar.

## Birden Fazla Takası için Batch Enerji

Ard arda birden fazla takası gerçekleştiriyorsanız -- örneğin bir portföyü yeniden dengeliyorsanız -- her takası için ayrı ayrı enerji satın almak verimsizdir. Bunun yerine, tüm takasları simüle edin ve enerjiyi tek bir batch'te satın alın:

```typescript
async function batchSwaps(
  swaps: SwapParams[]
): Promise<void> {
  // Gerekli toplam enerjiyi almak için tüm takasları simüle edin
  let totalEnergy = 0;
  const estimates = [];

  for (const swap of swaps) {
    const estimate = await merx.estimateEnergy({
      contract_address: SUNSWAP_ROUTER,
      function_selector: swap.functionSelector,
      parameter: swap.params,
      owner_address: swap.wallet
    });
    estimates.push(estimate);
    totalEnergy += estimate.energy_required;
  }

  console.log(`${swaps.length} takası için toplam enerji: ${totalEnergy}`);

  // Tüm enerjiyi bir kerede satın alın
  const order = await merx.createOrder({
    energy_amount: totalEnergy,
    duration: '30m', // Tüm takasları yürütmek için yeterli zaman
    target_address: swaps[0].wallet
  });

  await waitForOrderFill(order.id);

  // Önceden satın alınmış enerji ile tüm takasları yürütün
  for (const swap of swaps) {
    await executeSwap(swap);
  }
}
```

Batch satın alma iki avantaj sağlar:

1. **Daha iyi oranlar.** Daha büyük enerji miktarları, sağlayıcılardan genellikle daha iyi birim başına fiyatlandırma için uygunluk kazanır.
2. **Tek işlem maliyeti.** N enerji satın alması yerine bir tane, API çağrılarını ve işleme süresini azaltır.

## Ticaret Botu Entegrasyonu

Otomatik ticaret botları için, enerji yönetimi sorunsuz olmalıdır. Bot ticaret mantığına odaklanmalı, enerji tedariki ile ilgilenmemelidir. Bir SunSwap ticaret botuna MERX entegre etmek için bir kalıp:

```typescript
class EnergyAwareTrader {
  private merx: MerxClient;

  constructor() {
    this.merx = new MerxClient({
      apiKey: process.env.MERX_API_KEY
    });
  }

  async executeSwap(params: SwapParams): Promise<SwapResult> {
    // Adım 1: Kesin enerjiyi almak için simüle edin
    const estimate = await this.merx.estimateEnergy({
      contract_address: params.router,
      function_selector: params.method,
      parameter: params.args,
      owner_address: params.wallet
    });

    // Adım 2: Cüzdanın yeterli enerji olup olmadığını kontrol edin
    const resources = await this.merx.checkResources(
      params.wallet
    );

    if (resources.energy.available < estimate.energy_required) {
      // Adım 3: Eksikliği satın alın
      const deficit =
        estimate.energy_required - resources.energy.available;

      const order = await this.merx.createOrder({
        energy_amount: deficit,
        duration: '5m',
        target_address: params.wallet
      });

      await this.waitForFill(order.id);
    }

    // Adım 4: Sıfır TRX yakma ile takasını yürütün
    return await this.broadcastSwap(params);
  }
}
```

Bu kalıp, mevcut enerjiyi satın almadan önce kontrol eder ve yalnızca eksikliği satın alır. Cüzdanın önceki bir satın almadan veya stake etmeden zaten enerjiye sahipse, bot zaten sahip olduğu şeyi satın almaya para harcamaz.

## Aktif Tüccarlar için Daimi Siparişler

Aktif tüccarlar, elverişli fiyatlarla enerjiyi önceden satın alan daimi siparişlerden yararlanır:

```typescript
// Ticaret için bir enerji rezervi oluşturun
const standing = await merx.createStandingOrder({
  energy_amount: 500000, // ~2-3 karmaşık takası değerinde
  max_price_sun: 25,     // Sadece 25 SUN'dan aşağısında satın alın
  duration: '1h',
  repeat: true,
  target_address: tradingWallet
});
```

Bu, ticaret cüzdanınızın ticaret fırsatı ortaya çıktığında her zaman enerji bulundurmasını sağlar. Önceden satın alınmış enerji olmadan, enerji delegasyonunun tamamlanması beklenirken zamana bağlı bir ticaret fırsatını kaçırabilirsiniz.

## Gerçek Dünyada Maliyet Karşılaştırması

Gerçekçi bir aktif ticaret senaryosunu modelleyelim:

**Profil: Aktif DEX tüccarı, günde 15 takası, basit ve karmaşık rotaların karışımı.**

Takası başına ortalama enerji: 165.000 (kesin simülasyon tabanlı)

### Enerji Optimizasyonu Olmadan

15 takası x 165.000 enerji x ~0.206 TRX başına 1.000 enerji yakma = 509 TRX/gün = $61/gün = **$1.830/ay**

### MERX Enerjisi Ortalama 28 SUN'da

15 takası x 165.000 enerji x 28 SUN = 69.300.000 SUN = 69.3 TRX/gün = $8.32/gün = **$250/ay**

### Aylık Tasarruf: $1.580

Bir yıl boyunca, bu $18.960 tasarruf -- muhtemelen birçok perakende tüccar için ticaret kârını aşan.

### Ortalama 23 SUN'da Daimi Siparişler Kullanan

Fiyat düşüşlerini yakalamak için daimi siparişleri kullanmak, ortalama maliyeti daha da azaltır:

15 takası x 165.000 enerji x 23 SUN = 56.925.000 SUN = 56.9 TRX/gün = $6.83/gün = **$205/ay**

**Optimizasyon yapılmayan duruma karşı yıllık tasarruf: $19.500.**

## Ticaret Maliyetlerini İzleme

Stratejinizi optimize etmek için zaman içinde enerji verimliliğini takip edin:

```typescript
interface TradeMetrics {
  swapType: string;
  estimatedEnergy: number;
  actualEnergy: number;
  energyCostSun: number;
  provider: string;
  savedVsBurn: number;
}

async function logTradeEnergy(
  trade: TradeResult,
  energyOrder: Order
): Promise<void> {
  const burnCost =
    (trade.energyUsed / 1000) * 0.206; // TRX
  const energyCost =
    (energyOrder.price_sun * trade.energyUsed) / 1e6; // TRX
  const saved = burnCost - energyCost;

  console.log(
    `Takası: ${trade.pair} | ` +
    `Enerji: ${trade.energyUsed} | ` +
    `Maliyet: ${energyCost.toFixed(2)} TRX | ` +
    `Tasarrufu: ${saved.toFixed(2)} TRX (yakma karşılaştırıldığında)`
  );
}
```

## Çok-DEX Hususları

TRON'un SunSwap'ın ötesinde birden fazla DEX platformu vardır. Her birinin farklı enerji profilleri vardır:

- **SunSwap V2**: Standart AMM, takası başına 120K-200K+ enerji
- **SunSwap V3**: Yoğunlaştırılmış likidite, muhtemelen farklı enerji desenleri
- **Diğer DEX'ler**: Farklı enerji ayak izlerine sahip çeşitli akıllı kontrat uygulamaları

MERX'in kesin simülasyonu TRON'daki herhangi bir akıllı kontrat ile çalışır. Siz kontrat adresini ve fonksiyon çağrısını sağlarsınız, o da hangi DEX'i kullanıyorsanız kullanın kesin enerji gereksinimini döndürür.

## Sonuç

TRON'da enerji optimizasyonu olmadan DEX ticareti gereksiz yere pahalıdır. MERX aracılığıyla kesin enerji simülasyonu ve çok sağlayıcı agregasyonunun kombinasyonu, ham TRX yakılmasına karşı ticaret maliyetlerini %80-90 oranında azaltabilir.

Aktif tüccarlar için, aylık tasarruflar kolaylıkla dört basamağa ulaşır. Ölçekte faaliyet gösteren ticaret botları için, tasarruflar karlılık ile zararla faaliyet göstermek arasındaki farktır.

Entegrasyon basittir: ticaretten önce simüle edin, tam olarak ihtiyacınız olanı satın alın ve agregatöre yedi sağlayıcı arasında en iyi fiyatı bulmasını bırakın. Ticaret mantığınız temiz kalır, maliyetleriniz düşük kalır.

API'yi [https://merx.exchange/docs](https://merx.exchange/docs) adresinde keşfedin veya [https://merx.exchange](https://merx.exchange) adresinde optimizasyon yapmaya başlayın.


## Şimdi AI ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin -- kurulum yok, salt okunur araçlar için API anahtarı gerekmiyor:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI aracınıza şunu sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve tüm bağlı sağlayıcılardan gerçek zamanlı fiyatlar alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)