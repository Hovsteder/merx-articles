# Delegasyon İzleyicileri: TRON Enerjinizin Asla Süresi Dolmasın

## Süre Bitme Sorunu

TRON enerji delegasyonları sabit bir süreye sahiptir. 1 saat için enerji satın aldığınızda, tam olarak 1 saat elde edersiniz. O saat sona erdiğinde, delegasyon iptal edilir ve adresiniz sıfır delege enerjiye döner. Delegasyon başladıktan 61 dakika sonra uygulamanız bir USDT transferi gönderse, o işlem TRX'ı tam fiyatla yakar.

24/7 çalışan uygulamalar için bu bir yönetim yükü oluşturur. Her delegasyonun ne zaman sona ereceğini izlemeniz, mevcut olanın süresi bitmeden bir yenisini satın almanız ve sona erme ile yeni delegasyonun varması arasındaki boşluğu yönetmeniz gerekir. Yenilemeyi sadece birkaç saniye kaçırırsanız, bu pencere sırasında yürütülen işlemler tam TRX yakmasına neden olur.

MERX delegasyon izleyicileri bu sorunu ortadan kaldırır. Delegasyonlarınızı izlerler, sona erme zamanlarını takip ederler ve enerjiniz bitmeden otomatik olarak yenilerler. İzleyici MERX sunucusunda 24/7 çalışır - uygulamanızın çevrimiçi olmasına, MCP istemcinizin bağlı olmasına veya AI ajanınızın etkin olmasına bağlı değildir.

## İzleyici Türleri

MERX izleme sistemi, her biri farklı bir operasyonel kaygı için tasarlanmış üç tür izleyiciyi destekler.

### delegation_expiry

Aktif enerji delegasyonlarını izler ve süresi bitmeden önce harekete geçer.

```
Tool: create_monitor
Input: {
  "type": "delegation_expiry",
  "config": {
    "address": "TYourAddress...",
    "renew_before_minutes": 10,
    "auto_renew": true,
    "max_price_per_unit": 0.00006,
    "renewal_amount": 500000,
    "renewal_duration_hours": 24
  },
  "notifications": {
    "channels": ["webhook"],
    "webhook_url": "https://your-app.com/hooks/energy-monitor"
  }
}
```

Bu izleyici, belirtilen adrese yapılan tüm aktif delegasyonları takip eder. Herhangi bir delegasyon süresinin bitmesine 10 dakika kaldığında, izleyici harekete geçer:

1. `auto_renew` true ise ve mevcut en iyi fiyat `max_price_per_unit` değerinde veya altındaysa, `renewal_duration_hours` süresi ile `renewal_amount` enerji için otomatik olarak yeni bir sipariş verir.
2. Fiyat `max_price_per_unit` değerini aşarsa, izleyici satın almak yerine bir bildirim gönderir ve fiyatların otomatik yenileme için çok yüksek olduğunu sizi uyarır.
3. `auto_renew` false ise, izleyici sadece bildirim gönderir ve siz manuel işlem yapma zamanı kazanırsınız.

`renew_before_minutes` parametresi kritiktir. Yeni delegasyonun yerleştirilmesi, onaylanması ve eski olanın süresi bitmeden etkinleştirilmesi için yeterince yüksek ayarlayın. 10 dakikalık bir pencere, sipariş yerleştirme (saniyeler), sağlayıcı işleme (1-2 dakika) ve delegasyon onayı (3-6 saniye) için, ağ tıkanıklığı için bir marj ile geniş zaman sağlar.

### balance_threshold

Zincir üstü enerji bakiyesini izler ve belirtilen seviyenin altına düştüğünde tetiklenir.

```
Tool: create_monitor
Input: {
  "type": "balance_threshold",
  "config": {
    "address": "TYourAddress...",
    "energy_threshold": 200000,
    "bandwidth_threshold": 2000,
    "action": "ensure_resources",
    "energy_target": 500000,
    "bandwidth_target": 5000
  },
  "notifications": {
    "channels": ["telegram"],
    "telegram_chat_id": "-1001234567890"
  }
}
```

`delegation_expiry`'den farklı olarak, `balance_threshold` tüketim temelli değildir. Adresinizin kullanılabilir enerjisi neden olursa olsun 200.000'in altına düştüğünde tetiklenir. Bu, sona erme izleyicisinin karşılayamayacağı senaryoları kapsar:

- Birden fazla işlem enerjiyi beklenenden daha hızlı tüketme
- Bir delegasyonun sağlayıcı tarafından erken iptal edilmesi
- Staking parametrelerindeki değişiklik nedeniyle stake edilen enerjinin azalması

Tetiklendiğinde, `ensure_resources` eylemi mevcut bakiyeyi kontrol eder, hedefi ulaşmak için gereken açığı hesaplar ve sadece gereken miktarı satın alır.

### price_alert

Enerji pazar fiyatlarını izler ve koşullar karşılandığında bildirir.

```
Tool: create_monitor
Input: {
  "type": "price_alert",
  "config": {
    "condition": "below",
    "threshold": 0.000040,
    "resource_type": "energy",
    "duration_hours": 24,
    "cooldown_minutes": 360
  },
  "notifications": {
    "channels": ["webhook", "telegram"],
    "webhook_url": "https://your-app.com/hooks/price-alert",
    "telegram_chat_id": "-1001234567890"
  }
}
```

Fiyat uyarıları varsayılan olarak bilgilendiricidir - satın almadan bildirir. Bu, enerjinin tarihsel olarak düşük fiyatlarda satın almak isteyen ancak satın alma kararları için bir insanın sürece dâhil olmasını tercih eden ekipler için yararlıdır.

`cooldown_minutes` parametresi uyarı yorgunluğunu önler. Fiyat saatlerce eşik altında kalırsa, her 30 saniyede bir yerine her 6 saatte bir uyarı alırsınız.

## Otomatik Yenileme Ayrıntılı

`delegation_expiry` izleyicileri için otomatik yenileme akışı şu şekilde çalışır:

### Adım 1: Aktif Delegasyonları İzle

MERX, platform aracılığıyla yerleştirilen tüm enerji siparişlerinin bir kaydını tutar. Her sipariş, delegasyon başlama zamanı, süresi ve sona erme zamanını içerir. İzleyici bu kayıtları mevcut zaman ile karşılaştırır.

```
Active delegations for TYourAddress:
  Order #1234: 500,000 energy, expires 2026-03-30T14:00:00Z (47 min remaining)
  Order #1235: 200,000 energy, expires 2026-03-30T15:30:00Z (137 min remaining)
```

### Adım 2: Yenileme Penceresini Değerlendir

Herhangi bir delegasyon yenileme penceresine girdiğinde (örn. sona ermesine 10 dakika kalmış):

```
Order #1234: 500,000 energy, expires in 9 minutes 42 seconds
-> Within renewal window (10 minutes)
-> Initiating renewal check
```

### Adım 3: Fiyat Kontrolü

İzleyici mevcut pazar fiyatlarını sorgular:

```
Best price for 500,000 energy / 24 hours:
  Provider: sohu
  Price: 26.30 TRX
  Price per unit: 0.0000526

Max allowed: 0.00006 per unit
  0.0000526 <= 0.00006: PASS
```

### Adım 4: Yenileme Siparişi Yerleştir

```
Order placed:
  Amount: 500,000 energy
  Duration: 24 hours
  Provider: sohu
  Cost: 26.30 TRX
  Target: TYourAddress...
  Status: CONFIRMED
```

### Adım 5: Delegasyonu Doğrula

İzleyici, yeni delegasyonun alındığını onaylamak için adresi yoklar:

```
Delegation confirmed:
  Previous energy: 487,231 (remaining from expiring delegation)
  New energy: 987,231 (previous + new delegation)
```

### Adım 6: Bildir

```
Webhook sent:
{
  "event": "delegation_renewed",
  "address": "TYourAddress...",
  "old_order": "1234",
  "new_order": "1236",
  "energy_amount": 500000,
  "cost_trx": 26.30,
  "provider": "sohu",
  "new_expiry": "2026-03-31T13:50:18Z"
}
```

Tüm akış 30 saniyeden kısa sürede tamamlanır. Adres hiçbir zaman enerji kapsamında bir boşluk yaşamaz.

## Fiyat Koruması

Otomatik yenileme izleyicilerindeki `max_price_per_unit` parametresi kritik bir güvenlik mekanizmasıdır. Yüksek talep dönemlerinde enerji fiyatları artabilir. Fiyat koruması olmaksızın, bir fiyat artışı sırasında otomatik yenileme normal oranın 2-3 katına mal olabilir.

Pazar fiyatı maksimumu aştığında:

```
Best price for 500,000 energy / 24 hours:
  Provider: catfee
  Price: 42.50 TRX
  Price per unit: 0.0000850

Max allowed: 0.00006 per unit
  0.0000850 > 0.00006: FAIL - Price exceeds maximum

Action: Notification sent instead of purchase
  "Energy delegation expiring in 8 minutes. Auto-renewal skipped:
   market price 0.0000850 exceeds maximum 0.00006. Manual action required."
```

Bildirim size aşağıdaki seçenekleri verir:
- Daha yüksek fiyatı kabul edin ve manuel sipariş verin
- Fiyatların normalleşmesini bekleyin ve kapsama da kısa bir boşluk kabul edin
- İzleyici üzerindeki max_price_per_unit değerini ayarlayın

### Doğru Maksimum Fiyat Ayarlama

Etkili bir maksimum fiyat ayarlamak için:

1. Son 30 gün için `get_price_history` kaynağını kontrol edin
2. 95. persentil fiyatını tanımlayın (tekliklerin %95'inin o seviyede veya altında olduğu fiyat)
3. Maksimumunuzu bu seviyede veya biraz üzerinde ayarlayın

Bu yaklaşım normal dalgalanmaları yakalarken gerçek fiyat artışlarını reddeder.

## Bir Ajan Olmadan 24/7 Çalıştırma

Bu, MERX izleyicilerinin ajan tarafı mantığına kıyasla temel ayırt edici özelliktir. Bir AI ajan bir konuşma oturumu sırasında çalışır. Oturum sona erdiğinde, ajan durur. Delegasyon izlemeyi ajanınızın kodunda uyguladıysanız, yalnızca ajan etkin olsa çalışırdı.

MERX izleyicileri MERX sunucu altyapısında çalışır:

- **PostgreSQL kalıcılığı** - İzleyici konfigürasyonları veritabanında depolanır ve sunucu yeniden başlatmalarından sonra da kalır
- **Sunucu tarafı değerlendirme** - Tetikleyiciler, herhangi bir istemci tarafından değil, MERX arka uç işlemi tarafından değerlendirilir
- **MCP bağlantılarından bağımsız** - İzleyicilerin çalışması için herhangi bir istemcinin bağlı olması gerekmez
- **Kilitlenme kurtarma** - MERX hizmeti yeniden başlatılırsa, izleyiciler son bilinen durumlarından otomatik olarak devam eder

Bir ajan bir monitörü bir kere oluşturur. Bu izleyici süresiz olarak (veya son kullanma tarihine kadar) ajan tekrar bağlanıp bağlanmasa da çalışır.

```
Day 1: Agent creates delegation_expiry monitor
Day 2: Agent is offline. Monitor renews delegation at 14:00.
Day 3: Agent is offline. Monitor renews delegation at 13:55.
Day 7: Agent reconnects. Checks monitor history:
  - 6 successful auto-renewals
  - 0 gaps in energy coverage
  - Total spent: 157.80 TRX
  - Average price: 0.0000526 per energy unit
```

## Sağlam Kapsama için İzleyicileri Birleştirme

Tek bir izleyici türü tüm arıza modlarını kapsamaz. Üretim kullanımı için önerilen konfigürasyon iki veya üç izleyici türünü birleştirir:

### Önerilen Kurulum

```
Monitor 1: delegation_expiry
  Purpose: Proactive renewal before expiry
  Config:
    renew_before_minutes: 10
    auto_renew: true
    max_price: 0.00006
    renewal_amount: 500,000
    renewal_duration: 24 hours

Monitor 2: balance_threshold
  Purpose: Catch unexpected energy depletion
  Config:
    energy_threshold: 100,000
    action: ensure_resources
    energy_target: 500,000

Monitor 3: price_alert
  Purpose: Opportunity buying at low prices
  Config:
    condition: below
    threshold: 0.000035
    cooldown: 360 minutes
    action: notify_only
```

İzleyici 1 normal durumu işler - kabul edilebilir fiyatlarda planlanan yenilemeleri. İzleyici 2 anormal durumları işler - ani enerji tüketim artışlarını, delegasyonun erken iptal edilmesini veya İzleyici 1'in fiyat koruması tarafından engellenmesini. İzleyici 3 sizi manuel toplu satın alma için olağanüstü fırsatlar konusunda uyarır.

Birlikte, bu üç izleyici sağlar:
- Normal koşullar altında sıfır boşluk enerji kapsamı
- Fiyatlar geçici olarak artarken otomatik geri dönüş
- Maliyet optimizasyonu fırsatları için uyarılar

## İzleyicileri Yönetme

### Aktif İzleyicileri Listele

```
Tool: list_monitors

Response:
{
  "monitors": [
    {
      "id": "mon_abc123",
      "type": "delegation_expiry",
      "status": "active",
      "address": "TYourAddress...",
      "last_triggered": "2026-03-30T02:00:00Z",
      "total_renewals": 14,
      "total_spent_trx": 368.20,
      "next_expiry": "2026-03-31T02:00:00Z"
    },
    {
      "id": "mon_def456",
      "type": "balance_threshold",
      "status": "active",
      "address": "TYourAddress...",
      "last_triggered": "2026-03-28T15:42:00Z",
      "total_triggers": 2,
      "total_spent_trx": 52.60
    }
  ]
}
```

### İzleyici Geçmişi

Her izleyici ayrıntılı bir yürütme günlüğü tutar:

```json
{
  "history": [
    {
      "timestamp": "2026-03-30T02:00:12Z",
      "trigger_reason": "Delegation expiring in 9m48s",
      "action_taken": "auto_renew",
      "order_id": "ord_xyz789",
      "energy_purchased": 500000,
      "cost_trx": 26.30,
      "provider": "sohu",
      "status": "success"
    },
    {
      "timestamp": "2026-03-29T01:55:33Z",
      "trigger_reason": "Delegation expiring in 9m27s",
      "action_taken": "notification_only",
      "reason": "Price 0.0000780 exceeds max 0.0000600",
      "status": "skipped"
    }
  ]
}
```

Bu geçmiş tam denetlenebilirlik sağlar. Her yenilemenin tam olarak ne zaman gerçekleştiğini, ne kadar maliyeti olduğunu, hangi sağlayıcının kullanıldığını ve herhangi bir yenilemenin neden atlandığını görebilirsiniz.

## Hiçbir Zaman Süresi Bitmeyen Ekonomisi

Günde 500 USDT transferi işleyen bir uygulamayı düşünün. Her transfer yaklaşık 65.000 enerji gerektirir.

İzleyiciler olmadan (bazen boşluk ile manuel yönetim):

```
Haftada ortalama boşluk: 3 (her biri ~15 dakika süren)
Boşluklar sırasında işlemler: ~15
Boşluklarda yakalanan TRX: ~15 x 27 = 405 TRX/hafta
Boşluklarda yıllık yakma: ~21.060 TRX (~$5.475)
```

MERX delegasyon izleyicileri ile:

```
Haftada boşluk: 0
Boşluklarda yakalanan TRX: 0
İzleyici maliyeti (otomatik yenileme): ~0 ek (aynı enerji yine de satın alınırdı)
Yıllık tasarruf: ~$5.475
```

İzleyicilerin ekstra maliyeti yoktur. Her iki durumda da aynı enerjiyi satın alırsınız - izleyiciler sadece satın almalar arasında boşluk olmadığını garanti eder. Tasarruflar tamamen yönetilmeyen sona erme pencerelerinde TRX yakmasını ortadan kaldırmaktan gelir.

## Sonuç

Enerji delegasyonlarının süresi biter. Bu, TRON ağında kaçınılamayan bir gerçektir. Kaçınılabilir olan şey, hazır bir yenisi olmaksızın süresi bitirmesini izlemektir.

MERX delegasyon izleyicileri manuel, hatalara açık bir süreci otomatikleştirilmiş, güvenilir bir sisteme dönüştürür. Sunucuda çalışırlar, herhangi bir istemci bağlantısından bağımsızlar. Delegasyonları süresi bitmeden yenilerler. Fiyat limitinize saygı duyarlar. İstisnalar hakkında sizi bildirir.

Bir kere ayarlayın. Delegasyon süresinin bitmesi hakkında bir daha asla düşünmeyin.

---

**Bağlantılar:**
- MERX Platformu: [https://merx.exchange](https://merx.exchange)
- MCP Sunucusu (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP Sunucusu (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)


## Şimdi AI ile Deneyin

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

AI ajanınıza sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve bağlı tüm sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgelendirmesi: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)