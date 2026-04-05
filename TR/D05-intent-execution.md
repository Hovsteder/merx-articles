# İşlem Yürütme: TRON'da AI Ajanları için Çok Adımlı Planlar

## Tek Seferde Bir Problemi ile İlgili Sorun

TRON ile etkileşime giren AI ajanları, görevler birden fazla zincir üstü işlem içerdiğinde yapısal bir sorunla karşılaşır. Basit bir senaryoyu düşünün: bir ajanın 100 USDT'yi Alice'e göndermesi ve ardından 50 TRX'i USDT için takas etmesi gerekiyor. Her işlem kendi energy satın almasını, kendi delegasyon beklentisini ve kendi yayın döngüsünü gerektirir.

Tek seferde bir yaklaşımla, ajan en az 8 araç çağrısı yapar:

1. USDT transferi için energy tahmini
2. USDT transferi için energy satın alma
3. Delegasyon bekleme
4. USDT transferi yürütme
5. Takas için energy tahmini
6. Takas için energy satın alma
7. Delegasyon bekleme
8. Takası yürütme

Her energy satın alması, kendi işlem yükü ile ayrı bir pazar etkileşimidir. Her delegasyon beklentisi 3-6 saniye gecikme ekler. Toplam duvar saati süresi, basit bir iki adımlı görev için 30 saniyeyi aşabilir.

MERX bunu niyet yürütme ile çözer - bir AI ajanından çok adımlı bir plan alan, her adımı simüle eden, tüm adımlar arasında kaynak satın almalarını optimize eden ve tüm planı sırayla yürüten bir sistem.

## Niyet Nedir?

MERX sisteminde, bir niyet ajanın başarmak istediğini ifade eden bildirimsel bir açıklama olup, sıralı bir eylemler listesi olarak gösterilir. Ajan istenen sonucu belirtir ve MERX yürütme mekaniğini işler.

Bir niyet, bir araç çağrısı dizisinden üç önemli şekilde farklıdır:

1. **Kaynak optimizasyonu** - MERX, tüm adımlar arasında energy satın almalarını toplu olarak yapabilir, adım adım yerine tek bir siparişte ihtiyaç duyulan toplam energy'yi satın alabilir.

2. **Ön doğrulama** - Her adım herhangi bir adım yürütülmeden önce simüle edilir. 5 adımlı bir planın 3. adımı başarısız olursa, ajan bunu 1. adım yayınlanmadan önce bilir.

3. **Atomik planlama** - Ajan tüm planı bir kez gönderir ve bu da MERX'e çalışın tam kapsamı hakkında görünürlük sağlar. Bu, adımlar bireysel olarak gönderildiğinde imkansız olan optimizasyonları sağlar.

## execute_intent Aracı

MCP sunucusu niyet yürütmesini `execute_intent` aracı aracılığıyla ortaya çıkarır:

```
Tool: execute_intent
Input: {
  "steps": [
    {
      "action": "transfer_trc20",
      "params": {
        "to": "TAliceAddress...",
        "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "100000000"
      }
    },
    {
      "action": "swap",
      "params": {
        "from_token": "TRX",
        "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "50000000",
        "slippage": 0.5
      }
    }
  ],
  "resource_strategy": "batch_cheapest"
}
```

Yanıt, her adım için simülasyon sonuçlarını, toplam kaynak maliyetini ve tamamlanmadan sonra her adımın yürütme durumunu içerir.

## Desteklenen İşlemler

Niyet sistemi aşağıdaki işlem türlerini destekler:

### transfer_trx

Bir adrese TRX gönderin. Bu, bandwidth'i tüketir ancak energy tüketmez yerel bir transferdir.

```json
{
  "action": "transfer_trx",
  "params": {
    "to": "TRecipient...",
    "amount_sun": 1000000
  }
}
```

### transfer_trc20

Bir adrese TRC20 tokenini (USDT, USDC, vb.) gönderin. Akıllı kontrat çağrısı için energy tüketir.

```json
{
  "action": "transfer_trc20",
  "params": {
    "to": "TRecipient...",
    "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "amount": "100000000"
  }
}
```

### swap

SunSwap V2'de token takası yürütün. Belirli takas parametreleri için tam energy simülasyonunu içerir.

```json
{
  "action": "swap",
  "params": {
    "from_token": "TRX",
    "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "amount": "100000000",
    "slippage": 0.5
  }
}
```

### approve

Bir TRC20 tokeninin harcama onayını ayarlayın. Token takası yapmadan önce gereklidir (TRX takası için gerekli değildir).

```json
{
  "action": "approve",
  "params": {
    "token_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "spender": "TRouterAddress...",
    "amount": "unlimited"
  }
}
```

### call_contract

Rastgele akıllı kontrat çağrısını yürütün. Bu, belirli işlem türleri tarafından kapsanmayan işlemler için kaçış yoludur.

```json
{
  "action": "call_contract",
  "params": {
    "contract_address": "TContractAddress...",
    "function_selector": "stake(uint256)",
    "parameters": [{ "type": "uint256", "value": "1000000" }],
    "call_value": 0
  }
}
```

### buy_resource

Planda bir adım olarak energy veya bandwidth satın alın. Ajan kaynak zamanlaması üzerinde açık kontrol istediğinde kullanışlıdır.

```json
{
  "action": "buy_resource",
  "params": {
    "resource_type": "energy",
    "amount": 130000,
    "duration_hours": 1
  }
}
```

## Kaynak Stratejileri

`resource_strategy` parametresi MERX'in niyetin adımları arasında energy satın almalarını nasıl işlediğini kontrol eder.

### batch_cheapest

Bu varsayılan ve önerilen stratejidir. MERX tüm adımları simüle eder, gerekli toplam energy'yi toplar, mevcut kaynakları çıkarır ve tüm niyet için tek bir energy satın alması yapar.

```
Step 1 (transfer_trc20): 64,895 energy
Step 2 (swap):           223,354 energy
Toplam gerekli:          288,249 energy
Şu anda mevcut:          0 energy
Satın alma:              290,000 energy (sipariş birimine yuvarlanmış)
```

Bir satın alma. Bir delegasyon beklentisi. Sonra tüm adımlar havuzlanmış energy'yi kullanarak sırayla yürütülür.

Faydaları:
- Tek pazar etkileşimi (daha düşük yük)
- Tek delegasyon beklentisi (daha düşük gecikme)
- Daha büyük siparişlerde potansiyel hacim indirimi
- Daha basit hata işleme

### per_step

Her adım bağımsız olarak kendi energy'sini satın alır. Adımlar koşullu olduğunda veya riski en aza indirmek istediğinizde (1. adım başarısız olursa, 2. adım için energy satın almamışsınızdır) bunu kullanın.

```
Step 1: 65,000 energy satın al -> bekle -> transferi yürüt
Step 2: 225,000 energy satın al -> bekle -> takası yürüt
```

Bu strateji daha yavaştır (iki delegasyon beklentisi) ancak yürütme planda durdurulursa daha az energy boşa harcar.

## Durum Saklı Simülasyon

Niyetin simülasyon motoru adımlar arasında durumu saklı tutar. Bu, sonraki adımların önceki adımların sonuçlarına bağlı olduğu planlar için kritiktir.

Bu niyeti düşünün: "50 TRX'i USDT için takas edin, sonra alınan USDT'yi Alice'e gönderin."

Simülasyon motoru:

1. Adım 1'i (takas) simüle eder. Sonuç: ajan 16.42 USDT alır.
2. Yeni USDT bakiyesini yansıtmak için simüle edilen durumu günceller.
3. Adım 2'yi (16.42 USDT'yi Alice'e transfer etme) güncellenmiş duruma karşı simüle eder.
4. Adım 2'nin Adım 1'den gelen bakiye ile başarılı olacağını doğrular.

Durum saklı simülasyon olmadan, Adım 2 ajanın mevcut bakiyesine karşı simüle edilir (bu takastan USDT'yi içermeyebilir). Simülasyon yanlışlıkla Adım 2'nin yetersiz bakiye nedeniyle başarısız olacağını bildirirdi.

```
Tool: execute_intent
Input: {
  "steps": [
    {
      "action": "swap",
      "params": {
        "from_token": "TRX",
        "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "50000000",
        "slippage": 0.5
      }
    },
    {
      "action": "transfer_trc20",
      "params": {
        "to": "TAliceAddress...",
        "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "use_previous_output"
      }
    }
  ],
  "resource_strategy": "batch_cheapest"
}
```

`use_previous_output` parametresi niyet sistemine önceki adımdan çıktı miktarını bu adım için giriş miktarı olarak kullanmasını söyler.

## Simülasyon Yanıtı

Yürütme başlamadan önce, niyet sistemi simülasyon özeti döndürür:

```json
{
  "simulation": {
    "steps": [
      {
        "action": "transfer_trc20",
        "energy_required": 64895,
        "bandwidth_required": 345,
        "simulated_success": true,
        "estimated_cost_trx": 3.42
      },
      {
        "action": "swap",
        "energy_required": 223354,
        "bandwidth_required": 420,
        "simulated_success": true,
        "estimated_output": "16.42 USDT",
        "estimated_cost_trx": 11.76
      }
    ],
    "total_energy": 288249,
    "total_bandwidth": 765,
    "total_cost_trx": 15.18,
    "resource_purchase": {
      "energy": 290000,
      "price": 15.24,
      "provider": "sohu"
    }
  },
  "status": "ready_to_execute"
}
```

Ajan, herhangi bir zincir üstü işlem yapılmadan önce maliyetlerle tam planı görür. Maliyetler kabul edilemez ise veya bir adım başarısız olursa, ajan hiçbir şey harcamadan planı değiştirebilir.

## Yürütme ve Hata İşleme

Ajan plan onayladığında (veya otomatik yürütme etkinleştirilirse), niyet adım adım yürütülür:

```json
{
  "execution": {
    "steps": [
      {
        "action": "transfer_trc20",
        "status": "completed",
        "tx_hash": "abc123...",
        "energy_used": 64895,
        "block": 58234567
      },
      {
        "action": "swap",
        "status": "completed",
        "tx_hash": "def456...",
        "energy_used": 223354,
        "output_amount": "16.42",
        "block": 58234568
      }
    ],
    "total_energy_used": 288249,
    "total_energy_purchased": 290000,
    "energy_wasted": 1751,
    "status": "all_steps_completed"
  }
}
```

### Yürütme Sırasında Başarısızlık

Yürütme sırasında bir adım başarısız olursa (simülasyon sırasında değil), niyet sistemi durur ve başarısızlığı bildirir:

```json
{
  "execution": {
    "steps": [
      {
        "action": "transfer_trc20",
        "status": "completed",
        "tx_hash": "abc123..."
      },
      {
        "action": "swap",
        "status": "failed",
        "error": "SLIPPAGE_EXCEEDED",
        "message": "Output 15.89 USDT below minimum 16.34 USDT"
      }
    ],
    "status": "partial_execution",
    "completed_steps": 1,
    "failed_step": 2,
    "remaining_energy": 223354
  }
}
```

Adım 1 zaten zincir üstü olarak kaydedilmiştir ve tersine çevrilemez. Ajan kalan energy bakiyesini alır ve nasıl devam edeceğine karar verebilir - başarısız adımı ayarlanmış parametrelerle yeniden deneyin, farklı bir işlem yürütün veya energy'nin süresi dolmasını bekleyin.

## Gerçek Dünya Örneği: Hazine Yeniden Dengeleme

Burada bir ajanın hazine yönetimi için yürütebileceği gerçekçi bir çok adımlı niyet:

"1.000 TRX'i USDT için takas edin, 300 USDT'yi operasyon cüzdanına gönderin, 200 USDT'yi pazarlama cüzdanına gönderin, geri kalanı tutun."

```
Tool: execute_intent
Input: {
  "steps": [
    {
      "action": "swap",
      "params": {
        "from_token": "TRX",
        "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "1000000000",
        "slippage": 1.0
      }
    },
    {
      "action": "transfer_trc20",
      "params": {
        "to": "TOpsWallet...",
        "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "300000000"
      }
    },
    {
      "action": "transfer_trc20",
      "params": {
        "to": "TMarketingWallet...",
        "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "200000000"
      }
    }
  ],
  "resource_strategy": "batch_cheapest"
}
```

Simülasyon:

```
Step 1 (takas):     223,354 energy
Step 2 (transfer):  29,631 energy  (OpsWallet zaten USDT'ye sahip)
Step 3 (transfer):  64,895 energy  (MarketingWallet USDT için yeni)
Toplam:             317,880 energy

Toplu satın alma: 320,000 energy catfee'den 16.83 TRX'e

Niyet toplamadan yapılan satın almalar: 3 ayrı satın alma = ~18.20 TRX
Niyet toplamadan yapılan satın almalar: 1 satın alma = 16.83 TRX
Toplamadan tasarruf: 1.37 TRX + azalan gecikme (1 bekleme vs 3)
```

## Niyetleri vs Bireysel Araçları Ne Zaman Kullanacağınız

`execute_intent`'i şu durumlarda kullanın:

- Görev iki veya daha fazla zincir üstü işlem içeriyor
- Adımların bağımlılıkları var (adım 2 adım 1'in çıktısını kullanıyor)
- Toplu işlem aracılığıyla toplam kaynak maliyetini en aza indirmek istiyorsunuz
- Taahhüt etmeden önce tüm planın ön doğrulanmasına ihtiyacınız var

Bireysel araçları şu durumlarda kullanın:

- Görev tek bir işlem
- Ajan, adımlar arasında harici girdiye dayalı kararlar alması gerekiyor
- Adımlar önemli zaman boşluklarıyla ayrılıyor
- Ajan yürütmenin her aşaması üzerinde maksimum kontrol istiyorsa

## Niyetler ve Ajan Özerkliği

Niyet sistemi ajan özerkliği için tasarlanmıştır. "Hazineyi yeniden dengeleme" gibi yüksek seviyeli bir talimat alan bir ajan bunu somut adımlara ayırabilir, bir niyet oluşturabilir, simüle edebilir, maliyetleri gözden geçirebilir ve yürütebilir - tüm bunları herhangi bir aşamada insan müdahalesi olmadan.

Simülasyon adımı ajanın güvenlik kontrolü olarak hizmet eder. Herhangi bir fon taahhütünden önce, ajan her adımın başarılı olacağını, toplam maliyetin bütçe içinde olduğunu ve beklenen çıktıların istenen sonuçla eşleştiğini doğrulayabilir. Bu, bir insanın bir işlemi onaylamadan önce gözden geçirmesine eşdeğerdir, ancak ajan tarafından programlı olarak yürütülür.

Tekrarlanan kaynak satın almalar için sabit siparişlerle ve bakiye uyarıları için monitörlerle birlikte, niyet sistemi insan gözetimi olmadan 24/7 çalışan tamamen özerk zincir üstü işlemleri sağlar.

## Sonuç

Tek adımlı yürütme, blockchain otomasyonunun eğitim tekerlekleridir. Gerçek ajan iş akışları çoklu işlemler, adımlar arasındaki bağımlılıklar ve tüm plan arasında kaynak optimizasyonunu içerir.

MERX niyet yürütmesi AI ajanlarına bireysel eylemler yerine planlar halinde düşünme yeteneği verir. Her şeyi simüle edin. Kaynakları tam kapsam arasında optimize edin. Her adımın ön doğrulama yapıldığından emin olarak güvenle yürütün.

Blockchain tek işlem ortamı değildir. Ajanınız da olmamalıdır.

---

**Bağlantılar:**
- MERX Platformu: [https://merx.exchange](https://merx.exchange)
- MCP Sunucusu (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP Sunucusu (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)


## Şu Anda AI ile Deneyin

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

AI ajanınıza şunları sorun: "Şu anda en ucuz TRON energy nedir?" ve bağlı tüm sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgelendirmesi: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)