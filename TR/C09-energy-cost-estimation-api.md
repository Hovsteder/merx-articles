# MERX Enerji Maliyeti Tahmin API'si: Satın Almadan Önce Maliyeti Bilin

Her TRON işlemi kaynak tüketir. Bir USDT transferi yaklaşık 65.000 enerji birimi gerektirir. Akıllı bir sözleşme onayı yaklaşık 15.000 yakar. Karmaşık bir DeFi etkileşimi 200.000 veya daha fazlasına ihtiyaç duyabilir. Gerçek sayılar sözleşmeye, işleme ve mevcut ağ parametrelerine bağlıdır.

Kullanıcılar adına TRON işlemlerini yöneten bir ürün geliştiriyorsanız - bir cüzdan, bir ödeme işlemcisi, bir işlem botu - işlemi gerçekleştirmeden önce maliyeti bilmeniz gerekir. Ne kadar enerji gereklidir? Kiralamaya karşı yakmak ne kadar maliyetli olur? Kullanıcı ne kadar tasarruf eder?

MERX, bu soruları kesin olarak cevaplayan iki endpoint sağlar: genel maliyet tahmini için `POST /api/v1/estimate` ve siparişe özgü maliyet önizlemesi için `GET /api/v1/orders/preview`. Birlikte, tek bir SUN değişmeden önce kullanıcılara tam maliyetler ve tasarrufları göstermenize olanak tanırlar.

## Tahmin Endpoint'i

`POST /api/v1/estimate`, belirli bir işlem türü için enerji ve bandwidth gereksinimlerini hesaplar ve MERX aracılığıyla enerji kiralama ile protokol seviyesinde TRX yakma arasındaki maliyet karşılaştırmasını döndürür.

### Temel İstek

```bash
curl -X POST https://merx.exchange/api/v1/estimate \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "trc20_transfer",
    "target_address": "TTargetAddressHere"
  }'
```

### Yanıt Formatı

```json
{
  "operation": "trc20_transfer",
  "target_address": "TTargetAddressHere",
  "energy_required": 64895,
  "bandwidth_required": 345,
  "costs": {
    "burn": {
      "trx_cost": 27370000,
      "trx_cost_readable": "27.37 TRX",
      "usd_equivalent": 2.19
    },
    "rental": {
      "best_provider": "sohu",
      "price_per_unit_sun": 22,
      "total_cost_sun": 1427690,
      "total_cost_trx": "1.43 TRX",
      "usd_equivalent": 0.11,
      "duration_hours": 1
    },
    "savings": {
      "trx_saved": "25.94 TRX",
      "percent": 94.8,
      "usd_saved": 2.08
    }
  },
  "address_resources": {
    "current_energy": 0,
    "current_bandwidth": 1200,
    "energy_deficit": 64895,
    "bandwidth_deficit": 0
  },
  "timestamp": "2026-03-30T10:00:00Z"
}
```

Yanıt, işlem maliyeti kararı için bilmeniz gereken her şeyi söyler:

- **energy_required** - işlemin ihtiyaç duyduğu enerji miktarı. Standart bir TRC-20 transferi için bu yaklaşık 65.000 birimdir, ancak tam sayı sözleşmeye ve hedef adres durumuna bağlıdır.
- **bandwidth_required** - işlemin kullandığı bandwidth miktarı. Çoğu basit transfer 300-400 bandwidth puanı gerektirir. Yeni hesaplar (bir adresi ilk kez etkinleştirmek) daha fazlasına ihtiyaç duyar.
- **costs.burn** - kullanıcı hiçbir şey yapmazsa ve protokolün TRX yakmasına izin verirse ne kadar maliyetlidir. Bu "varsayılan" maliyettir.
- **costs.rental** - MERX aracılığıyla mevcut olan en ucuz kiralama seçeneği. Sağlayıcı, birim başı fiyat, toplam maliyet ve kiralama süresini içerir.
- **costs.savings** - yakma ve kiralama arasındaki fark, tasarruf edilen mutlak TRX, tasarruf edilen yüzde ve USD eşdeğeri olarak ifade edilir.
- **address_resources** - hedef adresteki mevcut enerji ve bandwidth. Adres zaten staking'ten veya önceki bir delegasyondan enerji varsa, açık verilen değer buna göre azaltılır.

### Desteklenen İşlemler

`operation` alanı, en yaygın TRON işlemlerini kapsayan birkaç önceden tanımlanmış türü kabul eder:

#### trc20_transfer

Standart TRC-20 token transferi. Bu en yaygın işlemdir - USDT, USDC veya başka herhangi bir TRC-20 tokenini bir adresten diğerine göndermek.

```json
{
  "operation": "trc20_transfer",
  "target_address": "TSenderAddressHere"
}
```

Gerekli enerji: Standart bir transferde USDT için yaklaşık 64.895 birim (adres zaten USDT bakiyesiyle etkinleştirilmiş). Hiç tokenini tutmamış bir adrese ilk kez transfer edilmesi daha fazla maliyetlidir - 100.000 enerjiye kadar - çünkü sözleşme yeni bir depolama yuvası oluşturmalıdır.

#### trc20_approve

TRC-20 onay işlemi, akıllı bir sözleşmenin sizin adınıza token harcamasına izin vermek için kullanılır. DEX sözleşmeleriyle, kredi protokolleriyle ve çoğu DeFi uygulamasıyla etkileşime geçmeden önce gereklidir.

```json
{
  "operation": "trc20_approve",
  "target_address": "TApproverAddressHere"
}
```

Gerekli enerji: Yaklaşık 15.000-18.000 birim, bir transferden önemli ölçüde daha az.

#### trx_transfer

Basit bir TRX transferi. Bunlar öncelikle enerji değil, bandwidth tüketirler, ancak tahmin endpoint'i tamlık açısından bunları da işler.

```json
{
  "operation": "trx_transfer",
  "target_address": "TSenderAddressHere"
}
```

Gerekli enerji: 0 (TRX transferleri enerji tüketmez). Gerekli bandwidth: Yaklaşık 270 bayt.

#### custom

Önceden tanımlanmış türlere uymayan akıllı sözleşme çağrıları için. Sözleşme adresini ve işlev seçicisini sağlayın ve MERX, çağrıyı simüle ederek enerji tüketimini tahmin eder.

```json
{
  "operation": "custom",
  "target_address": "TCallerAddressHere",
  "contract_address": "TContractAddressHere",
  "function_selector": "stake(uint256)",
  "parameters": [
    {
      "type": "uint256",
      "value": "1000000"
    }
  ]
}
```

Özel işlem, TRON ağına karşı gerçek enerji ve bandwidth tüketimini belirlemek için bir simülasyon çalıştırır. Bu, standart olmayan işlemler için en doğru yöntemdir, ancak biraz daha uzun sürer (önceden tanımlanmış türler için 50 ms'ye karşı 200-500 ms).

## Sipariş Önizleme Endpoint'i

`POST /estimate` size kaynak gereksinimlerini ve maliyet karşılaştırmalarını verirken, `GET /api/v1/orders/preview` tam olarak bir MERX siparişinin nasıl görüneceğini gösterir - hangi sağlayıcının seçileceği, bakiyenizden tam olarak ne kadar düşüleceği ve geçerli ücretler dahil olmak üzere.

### İstek

```bash
curl "https://merx.exchange/api/v1/orders/preview?energy_amount=65000&target_address=TTargetAddressHere&duration_hours=1" \
  -H "X-API-Key: your_api_key"
```

### Yanıt

```json
{
  "preview": {
    "energy_amount": 65000,
    "target_address": "TTargetAddressHere",
    "duration_hours": 1,
    "provider": "sohu",
    "price_per_unit_sun": 22,
    "subtotal_sun": 1430000,
    "fee_sun": 14300,
    "total_sun": 1444300,
    "total_trx": "1.44 TRX",
    "your_balance_sun": 50000000,
    "balance_after_sun": 48555700,
    "estimated_fill_time_seconds": 5
  },
  "alternatives": [
    {
      "provider": "catfee",
      "price_per_unit_sun": 25,
      "total_sun": 1641250,
      "total_trx": "1.64 TRX"
    },
    {
      "provider": "netts",
      "price_per_unit_sun": 28,
      "total_sun": 1838200,
      "total_trx": "1.84 TRX"
    }
  ]
}
```

Önizleme şunları içerir:

- **provider** - mevcut fiyatlarda MERX'in siparişi hangi sağlayıcıya yönlendireceği.
- **subtotal_sun** - sağlayıcının oranında enerji maliyeti.
- **fee_sun** - MERX platform ücreti.
- **total_sun** - bakiyenizden düşülecek toplam tutar.
- **balance_after_sun** - siparişten sonraki bakiyeniz, böylece karşılama gücünü doğrulayabilirsiniz.
- **estimated_fill_time_seconds** - delegasyon tipik olarak bu sağlayıcıyla ne kadar sürer.
- **alternatives** - siparişi doldurabilecek diğer sağlayıcılar, fiyata göre sıralanmış. Kullanıcılara seçeneklerini göstermek için yararlıdır.

### Tahmin ve Önizleme Arasındaki Fark

| Özellik | POST /estimate | GET /orders/preview |
|---------|---------------|-------------------|
| Amaç | Genel maliyet analizi | Tam sipariş planlama |
| Yetkilendirme gerekli | Evet | Evet |
| Yakma karşılığında tasarrufu gösterir | Evet | Hayır |
| Bakiyenizi gösterir | Hayır | Evet |
| Platform ücretlerini gösterir | Hayır | Evet |
| Alternatifleri gösterir | Hayır | Evet |
| Adres kaynaklarını gösterir | Evet | Hayır |
| Özel sözleşmeleri destekler | Evet | Hayır |

Bir kullanıcıya "bu işlem TRX yakarsanız X maliyetinde olur, veya enerji kiralarsanız Y maliyetinde olur - yüzde Z tasarruf edersiniz" demek istediğinizde `estimate` kullanın. Kullanıcı enerji satın almaya karar verdiğinde ve onay vermeden önce tam siparişi göstermek istediğinizde `preview` kullanın.

## Sayılarla Gerçek Dünya Örnekleri

### Örnek 1: USDT Ödeme İşlemcisi

Her ödeme öncesinde maliyeti tahmin ettiğiniz bir ödeme işlemcisi işletiyorsunuz:

```javascript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
});

async function estimatePayoutCost(senderAddress) {
  const estimate = await merx.estimate({
    operation: 'trc20_transfer',
    target_address: senderAddress,
  });

  console.log(`Energy needed: ${estimate.energy_required}`);
  console.log(`Burn cost: ${estimate.costs.burn.trx_cost_readable}`);
  console.log(`Rental cost: ${estimate.costs.rental.total_cost_trx}`);
  console.log(`Savings: ${estimate.costs.savings.percent}%`);

  // Decide whether to rent or burn based on savings threshold
  if (estimate.costs.savings.percent > 50) {
    return { method: 'rent', cost: estimate.costs.rental };
  } else {
    return { method: 'burn', cost: estimate.costs.burn };
  }
}
```

Tipik çıktı:

```
Energy needed: 64895
Burn cost: 27.37 TRX
Rental cost: 1.43 TRX
Savings: 94.8%
```

Mevcut fiyatlarla, enerji kiralama hemen hemen her zaman yakmaya kıyasla yüzde 90'dan fazla tasarruf sağlar.

### Örnek 2: Cüzdan Maliyet Gösterimi

Bir TRON cüzdanı oluşturuyorsunuz ve kullanıcıya gönderiyi onaylamadan önce işlem maliyetini göstermek istiyorsunuz:

```python
from merx_sdk import MerxClient

client = MerxClient(api_key="your_api_key")


def get_transfer_cost(sender_address: str, token: str = "usdt") -> dict:
    """Get the cost to send a TRC-20 token, accounting for existing resources."""

    estimate = client.estimate(
        operation="trc20_transfer",
        target_address=sender_address,
    )

    resources = estimate["address_resources"]
    costs = estimate["costs"]

    # If the address already has enough energy, the transfer is free
    if resources["energy_deficit"] == 0:
        return {
            "cost": "0 TRX",
            "note": "Address has sufficient energy",
        }

    return {
        "without_energy": costs["burn"]["trx_cost_readable"],
        "with_energy": costs["rental"]["total_cost_trx"],
        "savings": f'{costs["savings"]["percent"]}%',
        "energy_deficit": resources["energy_deficit"],
    }
```

Cüzdan arayüzü şöyle bir şey gösterebilir:

```
TAlıcı'ya 100 USDT gönder...

İşlem maliyeti:
  Enerji kiralama olmadan:  27.37 TRX (yakıldı)
  MERX enerjisiyle:       1.43 TRX (kiralandı)
  Tasarrufu:               94.8%

[ Enerji Kirala ve Gönder ]    [ Enerji Olmadan Gönder ]
```

### Örnek 3: Toplu Transfer Maliyeti Projeksiyonu

500 adrese USDT göndermek ve toplam maliyeti önceden bilmek istiyorsunuz:

```javascript
async function estimateBatchCost(addresses) {
  let totalBurnCost = 0;
  let totalRentalCost = 0;
  let totalEnergy = 0;

  // Estimate for a representative sample (first 10 addresses)
  // Most TRC-20 transfers cost the same energy
  const sampleSize = Math.min(10, addresses.length);
  const sample = addresses.slice(0, sampleSize);

  for (const address of sample) {
    const estimate = await merx.estimate({
      operation: 'trc20_transfer',
      target_address: address,
    });

    totalBurnCost += estimate.costs.burn.trx_cost;
    totalRentalCost += estimate.costs.rental.total_cost_sun;
    totalEnergy += estimate.energy_required;
  }

  // Extrapolate to full batch
  const avgBurnCost = totalBurnCost / sampleSize;
  const avgRentalCost = totalRentalCost / sampleSize;
  const avgEnergy = totalEnergy / sampleSize;

  const projectedBurnTRX = (avgBurnCost * addresses.length) / 1_000_000;
  const projectedRentalTRX = (avgRentalCost * addresses.length) / 1_000_000;
  const projectedEnergy = avgEnergy * addresses.length;

  console.log(`Batch size: ${addresses.length} transfers`);
  console.log(`Total energy needed: ${projectedEnergy.toLocaleString()}`);
  console.log(`Cost without MERX: ${projectedBurnTRX.toFixed(2)} TRX`);
  console.log(`Cost with MERX: ${projectedRentalTRX.toFixed(2)} TRX`);
  console.log(
    `Total savings: ${(projectedBurnTRX - projectedRentalTRX).toFixed(2)} TRX`
  );
}
```

Mevcut pazar oranlarında 500 transfer için:

```
Batch size: 500 transfers
Total energy needed: 32,447,500
Cost without MERX: 13,685.00 TRX
Cost with MERX: 715.00 TRX
Total savings: 12,970.00 TRX
```

TRX başına yaklaşık 0,08 USD'de, bu tek bir toplu işlemde 1.000 doların üzerinde bir tasarrufdur.

### Örnek 4: Özel Sözleşme Çağrısı Tahmini

Özel bir staking sözleşmesiyle etkileşime girersiniz ve bir `stake()` çağrısının maliyetini tahmin etmek istiyorsunuz:

```bash
curl -X POST https://merx.exchange/api/v1/estimate \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "custom",
    "target_address": "TCallerAddressHere",
    "contract_address": "TStakingContractHere",
    "function_selector": "stake(uint256)",
    "parameters": [
      {
        "type": "uint256",
        "value": "50000000"
      }
    ]
  }'
```

Yanıt, karmaşık bir staking sözleşmesi için simüle edilen enerji tüketimini içerir; bu, basit bir transfer için 120.000 ila 200.000 birim arasında önemli ölçüde daha fazla olabilir.

## Tahminleri Sipariş Akışına Entegre Etme

Tahmin ve önizleme endpoint'leri doğal olarak kullanıcıya yönelik bir sipariş akışına uyum sağlar:

1. **Kullanıcı bir işlem başlatır** (örneğin, USDT gönderin).
2. **POST /estimate'i çağırın** enerji gereksinimleri ve maliyet karşılaştırması almak için.
3. **Yakma maliyetini kiralama maliyetiyle karşılaştırarak gösterin** tasarruf yüzdesini belirtin.
4. **Kullanıcı enerji kiralama seçeneğini seçer.**
5. **GET /orders/preview'i çağırın** ücretleri içeren tam sipariş ayrıntılarını göstermek için.
6. **Kullanıcı onaylar.**
7. **POST /orders'ı çağırın** bir idempotency anahtarıyla siparişi oluşturmak için.
8. **Enerji delegasyonunu onaylayan webhook'u poll edin veya bekleyin.**
9. **Orijinal işlemi çalıştırın** temsilci enerjisiyle.

Adımlar 2-3 bilgilendirici niteliktedir. Para hareket etmez. Kullanıcı hesaplama yapmadan önce şeffaf fiyatlandırma görür. Bu güveni oluşturur ve maliyetlerle şaşırtan kullanıcılardan gelen destek isteklerini azaltır.

## Hata Yönetimi

Her iki endpoint de standart MERX hata yanıtlarını döndürür:

```json
{
  "error": {
    "code": "INVALID_ADDRESS",
    "message": "Target address is not a valid TRON address",
    "details": {
      "address": "invalid_address_here"
    }
  }
}
```

Yaygın hatalar:

| Kod | Sebep | Çözüm |
|------|-------|-----------|
| `INVALID_ADDRESS` | Hedef adres TRON adres doğrulaması başarısız | Adres biçimini doğrulayın (T-öneki, base58) |
| `INVALID_OPERATION` | Tanınmayan işlem türü | Birini kullanın: trc20_transfer, trc20_approve, trx_transfer, custom |
| `SIMULATION_FAILED` | Özel sözleşme çağrısı simülasyonu başarısız oldu | Sözleşme adresini ve işlev seçicisini kontrol edin |
| `NO_PROVIDERS` | Gerekli enerji miktarı için sağlayıcı yok | Daha sonra tekrar deneyin veya enerji miktarını azaltın |
| `INSUFFICIENT_BALANCE` | Önizlenen sipariş için bakiye çok düşük (yalnızca önizleme) | MERX hesabınıza daha fazla TRX yatırın |

## Önbelleğe Alma Hususları

Tahmin sonuçları kısa bir pencere için geçerlidir. Enerji fiyatları sağlayıcılar oranları ayarladıkça değişir ve ağ parametreleri yakma maliyetini değiştirebilir. Çoğu kullanım durumu için:

- **Tahminleri 30-60 saniye boyunca önbelleğe alın** arayüzde maliyetleri gösteriyorsanız. Fiyatlar MERX sağlayıcıları poll etme hızından (her 30 saniye) daha hızlı değişmez.
- **Her zaman sipariş oluşturmadan hemen önce yeni bir önizleme getirin**. Önizleme, o anki tam maliyeti yansıtır.
- **Sözleşme durumu sık sık değişirse özel sözleşme simülasyonlarını önbelleğe almayın**. Simülasyon sonucu, yürütme sırasında zincir üstü duruma bağlıdır.

## Sonuç

Tahmin ve önizleme endpoint'leri TRON enerji satın almaktan varsayımcılığı kaldırır. Sabit miktarda enerji kiralamak ve yeterli olup olmadığını umut etmek yerine, tam olarak ne kadar gerektiğini bilirsiniz. Mevcut olan fiyatı kabul etmek yerine, her seçeneği maliyete göre sıralanmış şekilde görürsünüz.

TRON'da ürün geliştiren geliştiriciler için bu endpoint'ler, enerjiyii öngörülemeyen bir maliyetten bilinen, optimize edilebilir bir satır öğesine dönüştürür. Maliyeti kontrol edin, tasarrufu gösterin, kullanıcının karar vermesine izin verin, ardından güvenle yürütün.

- MERX platformu: [merx.exchange](https://merx.exchange)
- Tam API belgeleri: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- Python SDK: [github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)


## Şimdi Yapay Zeka ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin - yükleme yok, salt okunur araçlar için API anahtarına gerek yok:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Yapay zeka aracınıza şunu sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)