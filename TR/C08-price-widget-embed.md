# MERX Fiyat Widget'ı: Canlı TRON Enerji Fiyatlarını Herhangi Bir Web Sitesine Gömün

TRON enerji fiyatları sürekli değişir. Sağlayıcılar talep, ağ koşulları ve mevcut kapasite temelinde oranları ayarlar. Web siteniz TRON kullanıcılarına hizmet veriyorsa - ister bir cüzdan, ister bir dApp, ister bir blockchain explorer, ister bir kaynak rehberi olsun - canlı enerji fiyatlarını göstermek, ziyaretçileriniz için anında pratik bir değer katar.

MERX, tüm ana sağlayıcılardan gerçek zamanlı enerji fiyatlarını maliyet sırasına göre gösteren gömülebilir bir fiyat widget'ı sağlar. İki satırlık HTML gerektirir, her 60 saniyede otomatik olarak yenilenir ve çoğu blockchain ile ilgili web sitesine herhangi bir değişiklik olmadan uygun profesyonel koyu tema kullanır.

Bu makale widget'ı nasıl gömeceğini, arkasında nasıl çalıştığını ve kullanım örneğinize nasıl özelleştireceğini kapsar.

## İki Satırlık HTML

En basit entegrasyon. Bu iki satırı HTML'nizin herhangi bir yerine ekleyin:

```html
<div id="merx-prices"></div>
<script src="https://merx.exchange/widget/prices.js"></script>
```

Hepsi bu. Script otomatik olarak başlatılır, MERX genel API'sinden mevcut fiyatları alır ve `div` içinde stillenmiş bir tablo işler. API anahtarı gerekli değil. Derleme adımı yok. Çerçeve bağımlılığı yok.

Widget herhangi bir HTML sayfasında çalışır - statik siteler, WordPress, Webflow, Squarespace (özel kod blokları aracılığıyla) veya tarayıcıya işlenen herhangi bir çerçeve. Asynchronously yüklenir ve sayfa işlemesini engellemez.

## Widget'ın Gösterdikleri

Widget, tüm etkin TRON enerji sağlayıcılarını mevcut fiyatlandırmayla gösteren kompakt bir tablo işler. Her satır şunları içerir:

- **Sağlayıcı adı** - enerji sağlayıcısı (Sohu, CatFee, NETTs, TronSave, Feee, iTRX, PowerSun).
- **Birim başına fiyat** - SUN cinsinden mevcut enerji fiyatı. Bu, standart delegasyon için birim enerji maliyetidir.
- **Min/Max sipariş** - sağlayıcının halihazırda kabul ettiği minimum ve maksimum enerji miktarı.
- **Süre** - mevcut kira süreleri (1 saat, 3 saat, 1 gün, vb.).
- **Durum** - sağlayıcının halihazırda çevrimiçi olup olmadığı ve siparişleri kabul edip etmediği.

Sağlayıcılar fiyata göre sıralanır, en ucuz ilk sırada. Sıralama her yenilemede güncellenir, yani bir sağlayıcı fiyatını düşürürse otomatik olarak yukarı taşınır.

### Varsayılan Görünüm

Widget varsayılan olarak koyu temayı kullanır:

- Arka plan: `#0a0a0a` (neredeyse siyah)
- Metin: `#e0e0e0` (açık gri)
- Tablo sınırları: `#1a1a1a` (incelikli koyu sınırlar)
- Aksent rengi: `#00d4aa` (MERX yeşili, en ucuz fiyat vurgulama için kullanılır)
- Yazı tipi: IBM Plex Mono (zaten mevcut değilse Google Fonts'tan yüklenir)

Görsel stil kasıtlı olarak minimaldir. Yuvarlak köşe yok, gradyan yok, gölge yok. Çoğu kripto ve blockchain arayüzünün estetiğine uyar.

## Arkasında Nasıl Çalışır

Widget'ın işlevini anlamak özelleştirme ve sorun gidermeye yardımcı olur.

### Veri Kaynağı

Widget, MERX genel fiyatları endpoint'inden veri alır:

```
GET https://merx.exchange/api/v1/prices
```

Bu endpoint herkese açıktır - kimlik doğrulama gerekli değil, API anahtarı gerekmez. Bağlantılı tüm sağlayıcılardan mevcut fiyatları döndürür:

```json
{
  "prices": [
    {
      "provider": "sohu",
      "energy_price_sun": 22,
      "min_energy": 32000,
      "max_energy": 10000000,
      "durations": [1, 3, 24],
      "available": true,
      "updated_at": "2026-03-30T10:30:00Z"
    },
    {
      "provider": "catfee",
      "energy_price_sun": 25,
      "min_energy": 10000,
      "max_energy": 5000000,
      "durations": [1, 24],
      "available": true,
      "updated_at": "2026-03-30T10:30:15Z"
    }
  ],
  "timestamp": "2026-03-30T10:30:20Z"
}
```

### Yenileme Döngüsü

Widget, fiyatları endpoint'ini her 60 saniyede bir yoklar. Her yenileme sessizdir - yükleme çarkı yok, boş içeriğin yanıp sönmesi yok. Tablo yerinde güncellenir. Bir getirme başarısız olursa (ağ sorunu, sunucu zaman aşımı), widget son başarılı verileri tutar ve sonraki döngüde tekrar dener.

Widget altbilgisinde küçük bir zaman damgası, verilerin ne zaman son güncellendiğini gösterir, böylece kullanıcılar fiyatların güncel olup olmadığını bir bakışta anlayabilir.

### Script Yüklemesi

`prices.js` script'i MERX CDN'sinden agresif önbelleğe alma (1 saat) ve gzip sıkıştırması ile sunulur. Tipik yükleme süresi geniş bant bağlantılarında 50ms altındadır. Script, minifiye ve gzip sıkıştırılmış halde yaklaşık 8 KB'dir.

Yükleme sırasında script:

1. `#merx-prices` elementini (veya yapılandırılmışsa özel bir hedefi) bulur.
2. Kapsamlı CSS stillerini enjekte eder (sayfa stillerinizle çatışmalarını önlemek için ön ek eklenmiştir).
3. İlk API çağrısını yapar.
4. Tabloyu işler.
5. 60 saniyelik yenileme aralığını ayarlar.

Tüm stiller sayfa stillerinizle CSS çatışmalarını önlemek için `.merx-widget` altında kapsamlandırılır.

## Özelleştirme Seçenekleri

Widget, kapsayıcı `div` üzerinde veri öznitelikleri veya JavaScript yapılandırma nesnesi aracılığıyla yapılandırmayı kabul eder.

### Veri Özniteliği Yapılandırması

```html
<div
  id="merx-prices"
  data-refresh="30"
  data-providers="sohu,catfee,netts"
  data-duration="1"
  data-theme="light"
  data-max-rows="5"
></div>
<script src="https://merx.exchange/widget/prices.js"></script>
```

Mevcut veri öznitelikleri:

| Öznitelik | Varsayılan | Açıklama |
|----------|---------|-------------|
| `data-refresh` | `60` | Yenileme aralığı (saniye cinsinden, minimum 15) |
| `data-providers` | tümü | Gösterilecek sağlayıcıların virgülle ayrılmış listesi |
| `data-duration` | tümü | Belirli süreye filtrele (1, 3, 24 saat) |
| `data-theme` | `dark` | `dark` veya `light` |
| `data-max-rows` | tümü | Gösterilecek maksimum sağlayıcı sayısı |
| `data-show-header` | `true` | "MERX Enerji Fiyatları" başlığını göster veya gizle |
| `data-show-footer` | `true` | Zaman damgası altbilgisini göster veya gizle |
| `data-compact` | `false` | Kompakt mod - daha az sütun, daha küçük metin |

### JavaScript Yapılandırması

Daha fazla kontrol için widget'ı programatik olarak başlatın:

```html
<div id="energy-prices"></div>
<script src="https://merx.exchange/widget/prices.js"></script>
<script>
  MerxWidget.init({
    container: '#energy-prices',
    refresh: 30,
    providers: ['sohu', 'catfee', 'netts', 'tronsave'],
    duration: 1,
    theme: 'dark',
    maxRows: 5,
    showHeader: true,
    showFooter: true,
    compact: false,
    onUpdate: function (prices) {
      console.log('Fiyatlar güncellendi:', prices.length, 'sağlayıcı');
    },
    onError: function (error) {
      console.error('Widget hatası:', error.message);
    },
  });
</script>
```

`onUpdate` ve `onError` geri çağrıları widget olaylarına kendi kodunuzda tepki vermenize izin verir. `onUpdate` geri çağrısı, her başarılı yenilemede ayrıştırılan fiyat dizisini alır.

### Açık Tema

Açık arka plana sahip web siteler için:

```html
<div id="merx-prices" data-theme="light"></div>
<script src="https://merx.exchange/widget/prices.js"></script>
```

Açık tema renkleri:

- Arka plan: `#ffffff`
- Metin: `#1a1a1a`
- Tablo sınırları: `#e0e0e0`
- Aksent: `#00a88a` (açık arka planlar için daha koyu yeşil)

### Özel Stil Oluşturma

Widget'ın CSS sınıfları stabildir ve belgelenir. Kendi stil sayfanızda bunları geçersiz kılın:

```css
/* Widget'ı tam genişliğe yapt */
.merx-widget {
  width: 100%;
  max-width: none;
}

/* Özel yazı tipi */
.merx-widget table {
  font-family: 'JetBrains Mono', monospace;
  font-size: 13px;
}

/* Özel aksent rengi */
.merx-widget .merx-best-price {
  color: #ff6b00;
}

/* Belirli sütunları gizle */
.merx-widget .merx-col-duration {
  display: none;
}
```

Widget'ın varsayılan `max-width` değeri 640px'dir. Bunu `100%` olarak ayarlamak, kapsayıcısını doldurmaya izin verir.

## Tam HTML Sayfası Örneği

Widget gömülü tam, bağımsız bir HTML sayfası. Kopyalayın, tarayıcıda açın ve canlı TRON enerji fiyat takip edicisine sahip olursunuz:

```html
<!DOCTYPE html>
<html lang="tr">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>TRON Enerji Fiyatları - Canlı</title>
  <style>
    body {
      background: #0a0a0a;
      color: #e0e0e0;
      font-family: 'IBM Plex Mono', monospace;
      display: flex;
      justify-content: center;
      padding: 40px 20px;
      margin: 0;
    }
    .container {
      max-width: 720px;
      width: 100%;
    }
    h1 {
      font-family: 'Cormorant Garamond', serif;
      font-size: 28px;
      font-weight: 400;
      margin-bottom: 8px;
    }
    p {
      color: #888;
      font-size: 14px;
      margin-bottom: 32px;
    }
  </style>
  <link
    href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:wght@400;600&family=IBM+Plex+Mono:wght@400;500&display=swap"
    rel="stylesheet"
  />
</head>
<body>
  <div class="container">
    <h1>TRON Enerji Fiyatları</h1>
    <p>Tüm ana sağlayıcılardan canlı fiyatlar. Her 60 saniyede güncellenir.</p>

    <div id="merx-prices" data-refresh="60" data-compact="false"></div>
    <script src="https://merx.exchange/widget/prices.js"></script>
  </div>
</body>
</html>
```

## Genel API ile Kendi Widget'ınızı Oluşturma

Önceden oluşturulmuş widget'ı ihtiyaçlarınıza uymazsa, genel fiyatları endpoint'ini doğrudan kullanarak kendi widget'ınızı oluşturabilirsiniz. Bu, UI'ın tamamen kontrolünü size verir.

### Vanilla JavaScript Örneği

```javascript
async function fetchMerxPrices() {
  const response = await fetch('https://merx.exchange/api/v1/prices');
  const data = await response.json();
  return data.prices
    .filter((p) => p.available)
    .sort((a, b) => a.energy_price_sun - b.energy_price_sun);
}

function renderPriceTable(prices) {
  const table = document.getElementById('custom-price-table');

  const rows = prices
    .map(
      (p) =>
        `<tr>
      <td>${p.provider}</td>
      <td>${p.energy_price_sun} SUN</td>
      <td>${(p.min_energy / 1000).toFixed(0)}K</td>
      <td>${p.available ? 'Çevrimiçi' : 'Çevrimdışı'}</td>
    </tr>`
    )
    .join('');

  table.innerHTML = `
    <thead>
      <tr>
        <th>Sağlayıcı</th>
        <th>Fiyat</th>
        <th>Min Sipariş</th>
        <th>Durum</th>
      </tr>
    </thead>
    <tbody>${rows}</tbody>
  `;
}

// İlk yükleme ve otomatik yenileme
async function refreshPrices() {
  try {
    const prices = await fetchMerxPrices();
    renderPriceTable(prices);
  } catch (err) {
    console.error('Fiyatlar alınamadı:', err);
  }
}

refreshPrices();
setInterval(refreshPrices, 60000);
```

### React Bileşeni Örneği

```jsx
import { useState, useEffect } from 'react';

function MerxPrices({ refreshInterval = 60000 }) {
  const [prices, setPrices] = useState([]);
  const [lastUpdated, setLastUpdated] = useState(null);

  useEffect(() => {
    async function fetchPrices() {
      try {
        const res = await fetch('https://merx.exchange/api/v1/prices');
        const data = await res.json();
        const sorted = data.prices
          .filter((p) => p.available)
          .sort((a, b) => a.energy_price_sun - b.energy_price_sun);
        setPrices(sorted);
        setLastUpdated(new Date());
      } catch (err) {
        console.error('Fiyat getirme başarısız:', err);
      }
    }

    fetchPrices();
    const interval = setInterval(fetchPrices, refreshInterval);
    return () => clearInterval(interval);
  }, [refreshInterval]);

  return (
    <div className="merx-prices">
      <table>
        <thead>
          <tr>
            <th>Sağlayıcı</th>
            <th>Fiyat (SUN)</th>
            <th>Min</th>
            <th>Max</th>
          </tr>
        </thead>
        <tbody>
          {prices.map((p) => (
            <tr key={p.provider}>
              <td>{p.provider}</td>
              <td>{p.energy_price_sun}</td>
              <td>{(p.min_energy / 1000).toFixed(0)}K</td>
              <td>{(p.max_energy / 1000000).toFixed(1)}M</td>
            </tr>
          ))}
        </tbody>
      </table>
      {lastUpdated && (
        <small>Güncellendi: {lastUpdated.toLocaleTimeString()}</small>
      )}
    </div>
  );
}

export default MerxPrices;
```

## Genel Endpoint için Hız Sınırları

`GET /api/v1/prices` endpoint'i herkese açıktır ve kimlik doğrulama gerektirmez, ancak IP adresi başına dakikada 300 istek ile hız sınırlandırılmıştır. Her 60 saniyede yenilenen bir widget için bu limitin iyi içinde yer alırsınız.

Daha agresif şekilde polling yapan özel bir çözüm oluşturursanız - örneğin her 5 saniyede fiyatları toplayan bir backend - sonuçları önbelleğe almayı ve kendi sunucunuzdan frontend'inize sunmayı düşünün. Bu, MERX API kullanımınızı düşük tutar ve kullanıcılarınız için yanıt sürelerini iyileştirir.

```javascript
// Sunucu tarafı önbelleğe alma örneği (Node.js/Express)
let cachedPrices = null;
let cacheTimestamp = 0;

app.get('/api/energy-prices', async (req, res) => {
  const now = Date.now();

  if (!cachedPrices || now - cacheTimestamp > 15000) {
    const response = await fetch('https://merx.exchange/api/v1/prices');
    cachedPrices = await response.json();
    cacheTimestamp = now;
  }

  res.json(cachedPrices);
});
```

Bu, MERX yanıtlarını sunucunuzda 15 saniye boyunca önbelleğe alır, frontend'inizi istediği sıklıkta MERX API'sinde yükleme artırmadan polling yapmasına izin verir.

## SEO Hususları

Widget, JavaScript aracılığıyla istemci tarafında işlenir, bu da JavaScript yürütmeyen arama motorlarının fiyat verilerini indekslemeyeceği anlamına gelir. SEO'nuz fiyat sayfanız için önemliyse, ilk fiyat verilerini sunucu tarafında işlemyi ve sonra canlı güncellemeler için widget'ı hydrate etmeyi düşünün.

Alternatif olarak, sayfayı TRON enerji fiyatlandırması hakkında statik içerikle yapılandırın ve widget'ı ek bir canlı öğe olarak kullanın. Çevreleyen metin SEO değeri sağlarken widget pratik canlı faydası sağlar.

## Sıkça Sorulan Sorular

**Widget sayfamı yavaşlatır mı?**
Hayır. Script gziplenmiş halde yaklaşık 8 KB'dir ve asynchronously yüklenir. İşlemeyi engellemez. İlk API çağrısı bir ağ isteği ekler, ancak fiyatlar tipik olarak 100ms içinde yanıt verir.

**Sadece belirli sağlayıcıları gösterebilir miyim?**
Evet. Ekranı belirli sağlayıcılara filtrelemek için `data-providers="sohu,catfee"` kullanın.

**MERX API'si kapanırsa ne olur?**
Widget son başarıyla getirilen verileri görüntüler. İlk yükleme sırasında hiçbir zaman başarıyla yüklenmediyse (API kapıyken ilk sayfa yüklemesi), fiyatların geçici olarak kullanılamadığını gösteren bir mesaj gösterir.

**Widget mobil responsive'dir mi?**
Evet. Kompakt modda (`data-compact="true"`), 320px kadar dar ekranlarda iyi çalışır. Varsayılan mod yaklaşık 500px minimum genişlik gerektirir.

**Widget'ı ticari olarak kullanabilir miyim?**
Evet. Widget ve altta yatan genel fiyatları API'si ücretsiz olarak kullanılabilir. Atıf takdir edilir ancak gerekli değil.

## Sonuç

Web sitenize canlı TRON enerji fiyatları eklemek iki satırlık HTML alır ve sıfır backend çalışması. MERX fiyat widget'ı veri getirmeyi, sıralamayı, stillendirmeyi ve otomatik yenilemeyi şıklı bir şekilde işler. Özel uygulamalar için genel fiyatları API'si kimlik doğrulama olmadan mevcuttur.

İster bir TRON cüzdanı, ister bir blockchain blogu, ister bir enerji yönetim aracı yönetiyor olun, gerçek zamanlı sağlayıcı fiyatlarını göstermek, kullanıcılarınıza başka yerlerde kolayca bulamayacakları harekete geçilebilir bilgiler verir.

- MERX platformu: [merx.exchange](https://merx.exchange)
- API belgeleri: [merx.exchange/docs](https://merx.exchange/docs)
- Tam kaynak örnekleri: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)


## Şimdi AI ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin -- yükleme yok, salt okunur araçlar için API anahtarı gerekmez:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI aracınıza şunu sorun: "Şu anda en ucuz TRON enerji nedir?" ve bağlantılı tüm sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)