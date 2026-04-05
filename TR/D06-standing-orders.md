# Kalıcı Siparişler: TRON Enerji Satın Almalarını 24/7 Otomatikleştirin

## Manuel Yaklaşım Ölçeklenmiyor

TRON enerjisini manuel olarak satın alıyorsanız, bunu yanlış yapıyorsunuz. Süreç zor olduğu için değil - bir siparişi vermek saniyeler alır - ama enerji piyasası dinamiktir. Fiyatlar gün içinde dalgalanır. Delegasyonlar sabit programlarda sona erer. Uygulamanızın enerji ihtiyaçları işlem hacmine göre değişir. Tüm bunları manuel yönetmek ya da sabit dikkat ya da harcanan parayla sonuçlanır.

Kalıcı siparişler bunu çözer ve kuralları tanımlamanıza izin verir ve bunlar otomatik olarak, haftada yedi gün, günde 24 saat yürütülür. Bir koşul karşılandığında - fiyat eşik değerinizin altına düştüğünde, bir program çalıştığında veya enerji bakiyeniz çok düştüğünde - MERX önceden tanımladığınız parametrelerle siparişi sizin adınıza yerleştirir.

Bu makale tam kalıcı sipariş sistemini kapsar: tetikleyici türleri, eylem türleri, bütçe kontrolleri ve en çok para tasarrufu sağlayan pratik düzenler.

## Kalıcı Siparişler Nasıl Çalışır?

Kalıcı sipariş, MERX'in PostgreSQL veritabanında depolanan kalıcı bir kuraldır. Üç bileşenden oluşur:

1. **Tetikleyici** - siparişi etkinleştiren koşul
2. **Eylem** - tetikleyici ateşlendiğinde ne olur
3. **Kısıtlamalar** - bütçe limitleri, yürütme sıklığı ve son kullanma tarihi

MERX arka ucu tetikleyicileri sürekli değerlendirir. Fiyat tabanlı tetikleyiciler, herhangi bir sağlayıcıdan yeni bir fiyat teklifi geldiğinde kontrol edilir (tipik olarak her 30 saniyede bir). Program tabanlı tetikleyiciler sunucu tarafından değerlendirilen cron ifadeleri kullanır. Bakiye tabanlı tetikleyiciler zincir üstü duruma düzenli olarak polling yapılarak kontrol edilir.

Bir tetikleyici ateşlendiğinde ve tüm kısıtlamalar karşılandığında, eylem herhangi bir manuel müdahale olmaksızın hemen yürütülür.

## Kalıcı Sipariş Oluşturma

MCP sunucusu kalıcı siparişleri `create_standing_order` aracı aracılığıyla sunar:

```
Tool: create_standing_order
Input: {
  "trigger": {
    "type": "price_below",
    "threshold": 0.00005,
    "resource_type": "energy"
  },
  "action": {
    "type": "buy_resource",
    "params": {
      "energy_amount": 500000,
      "duration_hours": 24,
      "target_address": "TYourAddress..."
    }
  },
  "constraints": {
    "max_daily_spend_trx": 100,
    "max_executions_per_day": 3,
    "expires_at": "2026-06-30T00:00:00Z"
  }
}
```

Bu sipariş, pazar fiyatı enerji birimi başına 0.00005 TRX'in altına düştüğünde günde 3 kez, günlük 100 TRX'ten fazla harcamadan, 24 saat için 500.000 enerji satın alacaktır.

## Tetikleyici Türleri

### price_below

En iyi mevcut enerji fiyatı belirtilen eşik değerinizin altına düştüğünde ateşlenir.

```json
{
  "type": "price_below",
  "threshold": 0.00005,
  "resource_type": "energy"
}
```

Bu maliyet optimizasyonu için en yaygın tetikleyicidir. TRON'daki enerji fiyatları arz ve talebe göre dalgalanır. Yoğun olmayan saatlerde (tipik olarak 00:00-06:00 UTC), fiyatlar genellikle tepe seviyelerinin 15-25% altında düşer. `price_below` tetikleyici, fiyatları izlemeden bu dönemlerde otomatik olarak enerji satın almanıza izin verir.

**Pratik ipucu:** Eşik değerinizi son 7 günün fiyatlarının 25. yüzdelik diliminde ayarlayın. Bu, fiyatlar son aralığının alt çeyreğinde olduğunda satın aldığınız anlamına gelir - para tasarrufu yapmak için yeterince ucuz, adresinizi tedarik etmek için yeterince sık.

### schedule

Pazar koşullarından bağımsız olarak bir cron programında ateşlenir.

```json
{
  "type": "schedule",
  "cron": "0 */6 * * *"
}
```

Bu tetikleyici her 6 saatte ateşlenir. Öngörülebilir enerji ihtiyaçları olan uygulamalar için kullanışlıdır - uygulamanız çoğu işlemleri iş saatleri sırasında işliyorsa, enerji satın almalarını acele öncesine gelmesi için programlayın.

Yaygın cron düzenleri:

```
"0 */6 * * *"    - Her 6 saatte
"0 8 * * 1-5"   - Her hafta içi günü saat 8:00 UTC'de
"*/30 * * * *"   - Her 30 dakikada
"0 0 * * *"      - Günlük saat 00:00 UTC'de
```

### balance_below

Adresinizin mevcut enerjisi eşik değerin altına düştüğünde ateşlenir.

```json
{
  "type": "balance_below",
  "energy_threshold": 100000,
  "check_address": "TYourAddress..."
}
```

Bu tetikleyici reaktiftir - bakiye düşük olduğunda otomatik olarak yenileyerek adresinizin asla enerjisiz kalmamasını sağlar. `buy_resource` eylemiyle birleştirildiğinde, kendi kendini idame ettiren bir enerji kaynağı yaratır.

**Nasıl çalışır:** MERX, belirtilen adresin zincir üstü kaynak bakiyesini düzenli aralıklarla (varsayılan olarak her 60 saniyede bir) polling yapar. Mevcut enerji eşik değerin altına düştüğünde, tetikleyici ateşlenir.

**Önemli uyarı:** Bakiyenin düşmesi ile yeni enerjinin gelmesi arasında gerçek bir gecikme vardır (polling aralığı + sipariş yerleştirme + delegasyon onayı). Eşik değeri, uygulamanızın bu pencere sırasında enerjisiz kalmaması için yeterince yüksek ayarlayın. Uygulamanız dakika başına 50.000 enerji tüketiyorsa, 100.000 eşik değeri size yaklaşık 2 dakikalık bir pist süresi verir - MERX'in yenilemesi için yeterlidir.

## Eylem Türleri

### buy_resource

Piyasadan enerji veya bandwidth satın alın.

```json
{
  "type": "buy_resource",
  "params": {
    "energy_amount": 500000,
    "duration_hours": 24,
    "target_address": "TYourAddress..."
  }
}
```

Bu enerji tedariki için standart eylemdir. MERX, yürütme zamanında en ucuz mevcut sağlayıcıyı otomatik olarak seçer. `target_address` temsilci edilen enerjiyi alır.

### ensure_resources

Satın almadan önce mevcut kaynakları kontrol eden daha yüksek seviye bir eylem.

```json
{
  "type": "ensure_resources",
  "params": {
    "target_address": "TYourAddress...",
    "energy_target": 500000,
    "bandwidth_target": 5000
  }
}
```

Her zaman belirtilen miktarı satın alan `buy_resource` olmakla aksine, `ensure_resources` önce adresin zaten sahip olduğunu kontrol eder ve yalnızca açığı satın alır. Adresin zaten 300.000 enerjisi varsa ve hedef 500.000 ise, 200.000 satın alır.

Birden fazla tetikleyici hızlı ardışıklıkla ateşlendiğinde fazla satın almayı engellediği için `balance_below` tetikleyicileri için daha güvenli eylemdir.

### notify_only

Zincir üstü hiçbir işlem yapmadan bir bildirim gönderin.

```json
{
  "type": "notify_only",
  "params": {
    "channels": ["webhook", "telegram"],
    "message": "Energy price dropped below threshold"
  }
}
```

Otomasyonun olmadığı farkındalık istediğinizde bunu kullanın. Kalıcı sipariş durumu izler ve sizi uyarır, ama satın alma kararını siz yaparsınız. Bu, tam otomatik satın almalarla henüz rahat olmayan ekipler için iyi bir başlangıç noktasıdır.

## Bütçe Limitleri ve Kısıtlamalar

Kısıtlamalar olmayan kalıcı siparişler tehlikelidir. Harcama limiti olmayan `price_below` tetikleyici, sürekli bir fiyat düşüşü sırasında tüm bakiyenizi tüketebilir. MERX birkaç kısıtlama mekanizması sağlar:

### max_daily_spend_trx

Kalıcı siparişin 24 saatlik bir dönemde harcayabileceği maksimum toplam TRX.

```json
{
  "max_daily_spend_trx": 200
}
```

Bu limit ulaşıldığında, tetikleyici ateşlenmeye devam eder ama eylem 24 saatlik pencere ileriye gidene kadar bastırılır.

### max_executions_per_day

Eylemin 24 saatlik bir dönemde yürütülebileceği maksimum sayı.

```json
{
  "max_executions_per_day": 5
}
```

Bu, değişken dönemler sırasında hızlı ateşlenmeyi engeller. Fiyat eşiğin üstü ve altında 20 kez zıplasa bile, eylem günde en fazla 5 kez yürütülür.

### min_interval_minutes

Art arda yürütmeler arasında minimum süre.

```json
{
  "min_interval_minutes": 60
}
```

Bu bir sakinleştirme dönemini uygular. Eylem yürütüldüğünde, koşullardan bağımsız olarak tetikleyici 60 dakika boyunca bastırılır.

### expires_at

Kalıcı sipariş bu tarihten sonra otomatik olarak devre dışı bırakılır.

```json
{
  "expires_at": "2026-06-30T00:00:00Z"
}
```

Kalıcı siparişlerde her zaman bir son kullanma tarihi ayarlayın. Unuttuğunuz sonra süresiz olarak çalışan yetim kalıcı sipariş bir yükümlülüktür.

### Total Budget

Kalıcı siparişin ömrü boyunca toplam harcamaya sert bir üst sınır.

```json
{
  "max_total_spend_trx": 5000
}
```

Yaşam boyu harcama bu limite ulaştığında, kalıcı sipariş kalıcı olarak devre dışı bırakılır.

## Bildirim Kanalları

Kalıcı siparişler tetikleyici etkinleştirmesi, başarılı yürütme, yürütme hatası ve bütçe limitine ulaşılmış durumlarda bildirimleri gönderebilir.

### Webhook

```json
{
  "notification_channels": [
    {
      "type": "webhook",
      "url": "https://your-app.com/hooks/merx",
      "headers": { "Authorization": "Bearer your-secret" }
    }
  ]
}
```

Webhook, tetikleyici ayrıntıları, alınan işlem ve yürütme sonucu içeren bir JSON yükü alır.

### Telegram

```json
{
  "notification_channels": [
    {
      "type": "telegram",
      "chat_id": "-1001234567890"
    }
  ]
}
```

MERX, belirtilen Telegram sohbetine biçimlendirilmiş bir mesaj gönderir. Telegram grupları aracılığıyla işlemleri izleyen ekipler için kullanışlıdır.

## Kalıcı Siparişleri Yönetme

### Etkin Siparişleri Listele

```
Tool: list_standing_orders

Response:
{
  "standing_orders": [
    {
      "id": "so_abc123",
      "trigger": { "type": "price_below", "threshold": 0.00005 },
      "action": { "type": "buy_resource", "energy_amount": 500000 },
      "status": "active",
      "executions_today": 1,
      "total_spent_trx": 247.50,
      "created_at": "2026-03-15T10:00:00Z",
      "last_executed": "2026-03-30T02:15:00Z"
    }
  ]
}
```

### Duraklat ve Sürdür

Kalıcı siparişler silinmeden duraklatılabilir. Duraklatılmış bir sipariş yapılandırmasını ve yürütme geçmişini korur ama ateşlenmez.

### Sil

Kalıcı siparişi kalıcı olarak kaldırır. Yürütme geçmişi denetim amaçları için tutulur.

## Pratik Düzenler

### Desen 1: Maliyet Optimize Edilmiş 24/7 Kapsama

Sürekli enerji kapsaması gereken uygulamalar için en düşük maliyette:

```
Kalıcı Sipariş 1:
  Tetikleyici: fiyat 0.000045 TRX/enerji'nin altında
  Eylem: 24 saat için 1.000.000 enerji satın al
  Kısıtlama: günde maksimum 2 yürütme

Kalıcı Sipariş 2:
  Tetikleyici: bakiye 200.000 enerjinin altında
  Eylem: kaynakları 500.000 enerjiye sağla (1 saat)
  Kısıtlama: günde maksimum 100 TRX
```

Sipariş 1 fiyatlar düştüğünde büyük miktarlarda ucuz enerji satın alır ve 24 saatlik kapsama sağlar. Sipariş 2 güvenlik ağıdır - fiyatlar Sipariş 1'i tetikleyecek kadar düşük asla düşmezse, Sipariş 2 bakiye kritik düzeye düştüğünde pazar fiyatıyla satın alarak adresinizin asla kurumasını sağlar.

### Desen 2: İş Saatleri Optimizasyonu

Öngörülebilir yoğun saatlere sahip uygulamalar için:

```
Kalıcı Sipariş 1:
  Tetikleyici: program "0 7 * * 1-5" (UTC sabah 7, hafta içi)
  Eylem: 12 saat için 2.000.000 enerji satın al
  Kısıtlama: günde maksimum 200 TRX

Kalıcı Sipariş 2:
  Tetikleyici: program "0 19 * * 1-5" (UTC akşam 7, hafta içi)
  Eylem: 14 saat için 500.000 enerji satın al
  Kısıtlama: günde maksimum 50 TRX
```

İş saatleri öncesinde ağır enerji, gece için hafif enerji. Hafta sonu siparişleri daha düşük miktarlar ile ayrı olabilir.

### Desen 3: Yalnızca Fiyat Uyarısı

İnsan karar vermeyi otomatik izleme ile isteyen ekipler için:

```
Kalıcı Sipariş:
  Tetikleyici: fiyat 0.000040 TRX/enerji'nin altında
  Eylem: yalnızca bildir
  Kanallar: webhook + telegram
  Mesaj: "Enerji tarihi düşük - manuel toplu satın almayı düşünün"
```

Otomatik satın alma yok. Ekip fiyatlar istisnai olarak düşük olduğunda uyarı alır ve büyük manuel sipariş vermeyi tercih edip edemeyeceğine karar verebilir.

### Desen 4: Değişken Yük İçin Otomatik Ölçeklendirme

İşlem hacmi öngörülemez şekilde değişen uygulamalar için:

```
Kalıcı Sipariş 1:
  Tetikleyici: bakiye 500.000 enerjinin altında
  Eylem: kaynakları 1.000.000 enerjiye sağla (1 saat)
  Kısıtlama: minimum aralık 15 dakika, günde maksimum 500 TRX

Kalıcı Sipariş 2:
  Tetikleyici: bakiye 100.000 enerjinin altında
  Eylem: kaynakları 500.000 enerjiye sağla (1 saat)
  Kısıtlama: minimum aralık 5 dakika, günde maksimum 1000 TRX
```

Sipariş 1 orta seviye tamamlama ile normal yükü ele alır. Sipariş 2 daha kısa aralık ve daha yüksek bütçe ile trafik çıkışları için yüksek aciliyet backstop'udur.

## Kalıcı Siparişler ve Ajan Entegrasyonu

AI ajanları için kalıcı siparişler kalıcı davranış mekanizmasıdır. Ajan oturumu geçicidir - MCP bağlantısı kapandığında, ajan artık fiyatları izleyemez veya enerji satın alamaz. Kalıcı siparişler MERX sunucusunda çalışır, herhangi bir istemci bağlantısından bağımsız.

Ajan, bir oturum sırasında kalıcı siparişleri ayarlayabilir ve gelecekteki tüm oturumlardan faydalanabilir:

```
Oturum 1 (kurulum):
  Ajan fiyat tabanlı satın alma için kalıcı siparişler oluşturur
  Ajan otomatik yenileme ile bakiye monitörleri oluşturur

Oturum 2 (günler sonra):
  Ajan kalıcı sipariş yürütme geçmişini kontrol eder
  Enerji otomatik olarak 14 kez satın alındı
  Tüm işlemler temsilci edilen enerji ile yürütüldü
  Sıfır manuel müdahale gerekli
```

Bu, gerçek anlamda otonom blokzincir işlemlerine giden yoldur. Ajan kuralları bir kez tanımlar ve kurallar sürekli yürütülür.

## Sonuç

Manuel enerji yönetimi çözülmüş bir sorundur. Kalıcı siparişler insan izlemesini herhangi bir manuel süreçten daha hızlı, daha tutarlı ve daha düşük maliyetle yürütülen otomatik kurallarla değiştirir.

Tetikleyicileri tanımlayın. Kısıtlamalarınızı ayarlayın. Geri kalanını MERX'e bırakın.

Uygulamanız 24/7 çalışır. Enerji tedariki de öyle olmalı.

---

**Linkler:**
- MERX Platform: [https://merx.exchange](https://merx.exchange)
- MCP Sunucusu (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP Sunucusu (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)


## Şimdi AI ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin - kurulum yok, salt okunur araçlar için API anahtarı yok:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI ajanınıza sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)