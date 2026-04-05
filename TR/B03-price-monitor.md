# MERX Fiyat İzleyicisi: Her Sağlayıcıyı 30 Saniyede Bir Nasıl İzliyoruz

Fiyat izleyicisi MERX'in kalbidir. Her 30 saniyede bir, entegre edilen her enerji sağlayıcısına bağlanır, mevcut fiyatlandırmalarını getirir, verileri normalleştirir ve sistem genelindeki diğer servislere yayınlar. Bunu olmadan, en iyi fiyat yönlendirmesi tahmin olurdu. Bununla, her sipariş 30 saniyeden eski olmayan verilere dayalı olarak en ucuz mevcut sağlayıcıya yönlendirilir.

Bu makale, fiyat izleyicisinin mimarisine derinlemesine bir bakış: sağlayıcıları nasıl yokladığı, adapter deseni sistemi nasıl genişletilebilir tuttuğu, Redis pub/sub fiyat verilerini gerçek zamanlı olarak nasıl dağıttığı ve fiyat geçmişi analitik ve karar almayı nasıl güçlendirdiği.

---

## Neden 30 Saniye

Yoklama aralığı, bilinçli bir tasarım seçimidir. TRON'daki enerji fiyatları her saniye değişmez - spot forex veya kripto emir defterleri gibi değildir. Sağlayıcı fiyatlandırması tipik olarak saatte birkaç kez değişir, bazen daha az. 30 saniyelik bir aralık, çeşitli sorunları önlerken her anlamlı fiyat değişikliğini yakalar:

- **Sağlayıcı API hız sınırlamaları**: çoğu sağlayıcı saniyede 1-2 istek sağlar. 30 saniyelik aralıklarla, yeniden denemelerle bile iyi bir şekilde sınırlar içinde kalırız.
- **Ağ yükü**: 7+ sağlayıcıyı yoklamak HTTP trafiği oluşturur. 30 saniyede, bu ihmal edilebilir. 1 saniyede, önemli olurdu.
- **Veri tazeliği vs gürültü**: enerji piyasalarında 30 saniyeden kısa fiyat değişiklikleri neredeyse her zaman sinyal değil, gürültüdür. 30 saniye gürültüyü filtreler, gerçek hareketleri yakalarken.
- **Sistem kaynağı kullanımı**: fiyat izleyicisi diğer hizmetlerle birlikte çalışır. Agresif yoklama, değer katmadan CPU ve bellek için rekabet edecektir.

Önbelleğe alınmış fiyatlardaki TTL 60 saniyeye ayarlanmıştır - yoklama aralığının iki katı. Bir yoklama başarısız olursa, önceki fiyat sona ermeden önce bir çevrim daha geçerli kalır. Bu, tek bir başarısız yoklamanın bir sağlayıcıyı emir defterinden çıkarmasını önler.

---

## Adapter Deseni

Her enerji sağlayıcısının farklı bir API'si vardır. Farklı uç noktalar, farklı kimlik doğrulama yöntemleri, farklı yanıt formatları, farklı hata kodları. Fiyat izleyicisi, bu farklılıkları çekirdek yoklama mantığından izole etmek için adapter desenini kullanır.

### Sağlayıcı Arayüzü

Her sağlayıcı adaptörü ortak bir arayüzü uygular:

```typescript
interface IEnergyProvider {
  name: string;

  // Mevcut fiyatlandırmayı getir
  getPrices(): Promise<ProviderPriceResponse>;

  // Sağlayıcının çalışır durumda olup olmadığını kontrol et
  healthCheck(): Promise<boolean>;

  // Mevcut envanteri al
  getAvailability(): Promise<AvailabilityResponse>;
}

interface ProviderPriceResponse {
  energyPricePerUnit: number;   // SUN cinsinden enerji birimi başına
  bandwidthPricePerUnit: number;
  minOrder: number;
  maxOrder: number;
  durations: Duration[];
  timestamp: number;
}
```

### Bir Sağlayıcı Adaptörü

Her sağlayıcı kendi adaptör dosyasını alır. İşte bir sağlayıcı adaptörünün neye benzediğinin basitleştirilmiş bir örneği:

```typescript
// providers/tronsave/index.ts
import { IEnergyProvider, ProviderPriceResponse } from '../base';

export class TronSaveProvider implements IEnergyProvider {
  name = 'tronsave';
  private apiUrl: string;
  private apiKey: string;

  constructor(config: ProviderConfig) {
    this.apiUrl = config.apiUrl;
    this.apiKey = config.apiKey;
  }

  async getPrices(): Promise<ProviderPriceResponse> {
    const response = await fetch(`${this.apiUrl}/v1/prices`, {
      headers: { 'Authorization': `Bearer ${this.apiKey}` }
    });

    const data = await response.json();

    // Sağlayıcıya özgü formatı standart formata normalleştir
    return {
      energyPricePerUnit: this.normalizePrice(data.energy_price),
      bandwidthPricePerUnit: this.normalizePrice(data.bandwidth_price),
      minOrder: data.min_energy || 10000,
      maxOrder: data.max_energy || 10000000,
      durations: this.normalizeDurations(data.available_durations),
      timestamp: Date.now()
    };
  }

  async healthCheck(): Promise<boolean> {
    try {
      const response = await fetch(`${this.apiUrl}/health`);
      return response.ok;
    } catch {
      return false;
    }
  }

  // ... sağlayıcıya özgü normalleştirme yöntemleri
}
```

### Yeni Bir Sağlayıcı Ekleme

Bu mimarinin temel avantajlarından biri, yeni bir sağlayıcı eklemek için tam olarak bir yeni dosya gerekir:

1. `IEnergyProvider` uygulayan `providers/newsaglaycı/index.ts` oluştur.
2. Sağlayıcıyı yapılandırmada kaydet.
3. Fiyat izleyicisi otomatik olarak onu yoklamaya başlar.
4. Sipariş yürütücüsü otomatik olarak onu yönlendirme kararlarına dahil eder.

Fiyat izleyicisinde değişiklik yok, sipariş yürütücüsünde değişiklik yok, API'de değişiklik yok. Adapter deseni, sağlayıcıya özgü karmaşıklığın encapsülasyonunu sağlar.

---

## Yoklama Döngüsü

Fiyat izleyicisinin çekirdek döngüsü basit ama güvenilirlik için dikkatli tasarlanmıştır:

```typescript
class PriceMonitor {
  private providers: IEnergyProvider[];
  private redis: RedisClient;
  private pollInterval = 30_000; // 30 saniye

  async start() {
    // Başlangıçta ilk yoklama
    await this.pollAll();

    // Tekrarlayan yoklamaları planla
    setInterval(() => this.pollAll(), this.pollInterval);
  }

  async pollAll() {
    const results = await Promise.allSettled(
      this.providers.map(provider => this.pollProvider(provider))
    );

    // Başarılı sonuçlardan en iyi fiyatı hesapla
    const validPrices = results
      .filter(r => r.status === 'fulfilled')
      .map(r => r.value);

    if (validPrices.length > 0) {
      await this.updateBestPrice(validPrices);
    }
  }

  async pollProvider(provider: IEnergyProvider) {
    const startTime = Date.now();

    try {
      const prices = await provider.getPrices();
      const responseTime = Date.now() - startTime;

      // Redis'e 60s TTL ile depola
      await this.redis.setex(
        `prices:${provider.name}`,
        60,
        JSON.stringify(prices)
      );

      // Fiyat güncelleme olayını yayınla
      await this.redis.publish(
        'price-updates',
        JSON.stringify({
          provider: provider.name,
          prices,
          responseTime
        })
      );

      // Sağlık ölçümlerini güncelle
      await this.updateHealthMetrics(provider.name, {
        success: true,
        responseTime,
        timestamp: Date.now()
      });

      return { provider: provider.name, prices };

    } catch (error) {
      await this.updateHealthMetrics(provider.name, {
        success: false,
        error: error.message,
        timestamp: Date.now()
      });

      throw error; // Promise.allSettled'e işletmesini ver
    }
  }
}
```

### Anahtar Tasarım Kararları

**Promise.allSettled, Promise.all değil**: Tek bir sağlayıcı hatası diğer sağlayıcılardan güncellemeleri engellememesi gerekir. `allSettled` her sağlayıcının bağımsız olarak yoklandığından emin olur.

**60 saniyelik TTL**: Bir sağlayıcı art arda iki çevrimde (60 saniye) yanıt vermezse, önbelleğe alınmış fiyatı otomatik olarak sona erer. Sipariş yürütücüsü, önbelleğe alınmış fiyatı olmayan bir sağlayıcıya yönlendirmeyecektir.

**Fiyatlarla birlikte sağlık ölçümleri**: Her yoklama yanıt süresini ve başarı/başarısızlığını kaydeder. Bu veriler, yönlendirme algoritmasının güvenilirlik puanlamasına beslenmiştir.

---

## Redis Pub/Sub Dağıtımı

Fiyat izleyicisi fiyatları doğrudan API tüketicilerine sunmaz. Bunun yerine Redis'e yayınlar ve diğer hizmetler ihtiyaç duyduğu güncellemelere abone olurlar.

### Kanal Yapısı

```
Kanal: price-updates
  -> Tüm fiyat güncelleme olayları (API hizmeti, WebSocket yayını tarafından tüketilir)

Kanal: price-alerts
  -> Önemli fiyat değişiklikleri (bildirim hizmeti tarafından tüketilir)

Kanal: provider-health
  -> Sağlık durumu değişiklikleri (admin paneli tarafından tüketilir)
```

### Neden Pub/Sub Yerine Doğrudan Çağrılar

Fiyat izleyicisi ve API hizmeti ayrı işlemlerdir (aslında ayrı Docker konteynerlerdir). Birbirlerine yalnızca Redis aracılığıyla iletişim kurarlar - doğrudan içeri aktarma yok, işlev çağrıları yok, paylaşılan bellek yok. Bu izolasyon şu anlama gelir:

- Fiyat izleyicisi API'yi etkilemeden yeniden başlatılabilir.
- API yatay olarak ölçeklenebilir (birden fazla örnek) ve hepsi aynı fiyat güncellemelerini alır.
- Fiyat izleyicisindeki bir hata API hizmetini çökeremez.
- Her hizmet bağımsız olarak dağıtılabilir.

### WebSocket Yayını

API hizmeti `price-updates` kanalına abone olur ve güncellemeleri bağlı WebSocket istemcilerine yayınlar:

```
Fiyat İzleyicisi -> Redis pub/sub -> API Hizmeti -> WebSocket -> İstemci

Gecikme: Sağlayıcı yanıtından istemci bildirimine ~5ms
```

WebSocket akışına abone olan istemciler fiyat güncellemelerini neredeyse gerçek zamanlı olarak alır ve canlı panolar ve duyarlı ticaret arayüzlerini etkinleştirir.

---

## Fiyat Geçmişi Depolama

Her fiyat veri noktası, tarihsel analiz için PostgreSQL'de depolanır. Şema, tam fiyat anlık görüntüsünü yakalar:

```sql
CREATE TABLE price_history (
  id BIGSERIAL PRIMARY KEY,
  provider VARCHAR(50) NOT NULL,
  energy_price_sun BIGINT NOT NULL,
  bandwidth_price_sun BIGINT NOT NULL,
  min_order INTEGER,
  max_order INTEGER,
  available_energy BIGINT,
  response_time_ms INTEGER,
  recorded_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_price_history_provider_time
  ON price_history(provider, recorded_at DESC);
```

### Veri Hacmi

7 sağlayıcı genelinde 30 saniyelik aralıklarla:

```
7 sağlayıcı x 2 yoklama/dakika x 60 dakika x 24 saat = 20,160 satır/gün
Aylık: ~604,800 satır
Yıllık: ~7,257,600 satır
```

Her satır küçüktür (kabaca 100 bayt), bu nedenle yıllık depolama 1 GB'tan az. PostgreSQL, uygun dizin oluşturmayla bu hacmi önemsizce işler.

### Analitik Sorguları

Fiyat geçmişi birkaç değerli analizi sağlar:

```sql
-- Son 24 saatte sağlayıcı başına ortalama fiyat
SELECT provider,
       AVG(energy_price_sun) as avg_price,
       MIN(energy_price_sun) as min_price,
       MAX(energy_price_sun) as max_price
FROM price_history
WHERE recorded_at > NOW() - INTERVAL '24 hours'
GROUP BY provider
ORDER BY avg_price;

-- Belirli bir sağlayıcı için fiyat eğilimi
SELECT date_trunc('hour', recorded_at) as hour,
       AVG(energy_price_sun) as avg_price
FROM price_history
WHERE provider = 'tronsave'
  AND recorded_at > NOW() - INTERVAL '7 days'
GROUP BY hour
ORDER BY hour;
```

Bu sorgular, MERX fiyat geçmişi API uç noktasını ve admin panelini besler.

---

## Kenar Durumlarını İşleme

### Sağlayıcı Geçersiz Veriler Döndürür

Fiyat izleyicisi, önbelleğe almadan önce her yanıtı doğrular:

```typescript
function validatePrice(price: ProviderPriceResponse): boolean {
  // Fiyat pozitif olmalı
  if (price.energyPricePerUnit <= 0) return false;

  // Fiyat makul sınırlar içinde olmalı (10-500 SUN)
  if (price.energyPricePerUnit < 10_000_000) return false;  // < 10 SUN
  if (price.energyPricePerUnit > 500_000_000) return false;  // > 500 SUN

  // Geçerli zaman damgası olmalı
  if (price.timestamp > Date.now() + 60_000) return false;  // gelecek
  if (price.timestamp < Date.now() - 300_000) return false;  // > 5dk öncesi

  return true;
}
```

Geçersiz veriler günlüğe kaydedilir ve atılır. Önceki geçerli fiyat, doğal olarak sona erinceye kadar önbellekte kalır.

### Sağlayıcı API Değişiklikleri

Sağlayıcı API'leri zaman zaman değişir - yeni alanlar, kullanımdan kaldırılan uç noktalar, değiştirilen yanıt formatları. Her sağlayıcının kendi adaptörü olduğu için, API değişiklikleri tek bir dosyaya kapsanır. Adaptör güncellenir, test edilir ve sisteminizin başka bir bölümüne dokunmadan dağıtılır.

### Ağ Bölümlemeleri

Fiyat izleyicisi ağ bağlantısını kaybederse, tüm sağlayıcı yoklamaları aynı anda başarısız olur. 60 saniyelik TTL, önbelleğe alınmış fiyatların bir dakika içinde sona ermesini sağlar ve sipariş yürütücüsü tüm sağlayıcılara yönlendirmeyi durdurur. Bağlantı geri yüklendiğinde, sonraki yoklama döngüsü önbelleği otomatik olarak yeniden doldurur.

### Saat Sapması

Fiyat izleyicisi tek bir sunucuda çalışır, bu nedenle hizmetler arasında saat sapması göreli zamanlamada bir sorun değildir. Fiyat geçmişindeki zaman damgaları PostgreSQL'den `NOW()` kullanır, tutarlılığı sağlar. API yanıtlarındaki mutlak zaman damgaları için sunucu NTP çalıştırır.

---

## İzleyiciyi İzlemek

Fiyat izleyicisi kendisi birkaç mekanizma aracılığıyla izlenir:

- **Sağlık uç noktası**: hizmet, her sağlayıcı için son başarılı yoklama zamanını bildiren bir `/health` uç noktası ortaya koymaktadır.
- **Uyarı verme**: 5 dakika boyunca başarılı bir fiyat güncellemesi yayınlanmadıysa, bir uyarı tetiklenir.
- **Ölçümler**: yoklama sayısı, başarı oranı, ortalama yanıt süresi ve önbellek isabet oranı izlenmiştir.
- **Günlükler**: her yoklama sonucu (başarı veya başarısızlık) analiz için yapılandırılmış JSON ile günlüğe kaydedilir.

---

## Performans Özellikleri

```
Tipik yoklama döngüsü (7 sağlayıcı):
  Toplam süre: 1-3 saniye (paralel HTTP istekleri)
  Redis yazmaları: 8 (7 sağlayıcı fiyatı + 1 en iyi fiyat)
  Redis yayınları: 7 (sağlayıcı başına biri)
  PostgreSQL eklemeleri: 7 (sağlayıcı başına biri)
  Bellek kullanımı: < 50 MB
  CPU kullanımı: < %2 ortalama
```

Fiyat izleyicisi kasıtlı olarak hafiftir. Bir şey yapar - sağlayıcılardan fiyat getir ve dağıt - ve bunu verimli bir şekilde yapar. 30 saniyelik aralık, hizmetin zamanının %90'ında boşta olduğu anlamına gelir ve daha işlemci yoğun sipariş yürütücüsü ve API hizmetleri için kaynakları açık bırakır.

---

## Sonuç

Fiyat izleyicisi kavramsal olarak basittir - sağlayıcıları yokla, verileri normalleştir, güncellemeleri yayınla. Ama detaylar önemlidir. Adapter deseni sağlayıcı eklemeyi önemsizleştirir. Redis pub/sub, fiyat toplamasını fiyat tüketiminden ayırır. 60 saniyelik TTL'ler otomatik olarak eski verileri hariç tutar. Sağlık ölçümleri yönlendirme kararlarına beslenmiştir. Ve fiyat geçmişi, kullanıcıların daha iyi kararlar almalarına yardımcı olan analitikleri etkinleştirir.

MERX'te gördüğünüz her fiyat, her en iyi fiyat tavsiyesi, her otomatik yönlendirme kararı fiyat izleyicisinin 30 saniyelik atışına geri döner. Toplama mümkün kılan temeli olmuştur.

Canlı fiyatlandırma ve geçmiş verileri [https://merx.exchange](https://merx.exchange) adresinde keşfedin. API erişimi için, [https://merx.exchange/docs](https://merx.exchange/docs) adresinde belgelere bakın.

---

*Bu makale, MERX teknik serisinin bir parçasıdır. MERX ilk blockchain kaynak borsasıdır. SDK'lar için kaynak kodu: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js), [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python).*


## Şimdi AI ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin - yükleme yok, salt okunur araçlar için API anahtarı gerekmez:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI aracınıza şunu sorun: "TRON enerjisinin en ucuz olanı şu anda nedir?" ve bağlı tüm sağlayıcılardan canlı fiyatları alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)