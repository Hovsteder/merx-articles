# MERX ile Tanisin: Ilk TRON Kaynak Borsasi

MERX, TRON ag kaynaklari - energy ve bandwidth - icin insa edilmis ilk agregator-borsadir. Isletmeleri TRON blokzincirindeki her USDT transferinde fazla odemeye zorlayan parcali saglayici pazarini cozmek icin tasarlanmistir. Birden fazla energy saglayicisini gercek zamanli fiyat karsilastirmasi ve otomatik en iyi fiyat yonlendirmesiyle tek bir platformda birlestirerek, MERX, USDT transfer maliyetlerini varsayilan TRX yakma mekanizmasina kiyasla yuzde 94'e kadar azaltir.

## MERX'in Cozdugu Sorun

TRON uzerindeki her USDT transferi, energy adinda bir ag kaynagi gerektirir. Bu kaynak olmadan, protokol cuzdaninizdan TRX yakar - standart transfer basina yaklasik 27 TRX. Bu sorunu cozmek icin, yakma maliyetinin bir kesmesi karsiliginda delege edilmis energy sunan energy kiralama saglayicilari pazari mevcuttur.

Ancak bu pazar parcalidir. Her birinin kendi API'si, fiyatlandirma modeli, kimlik dogrulama sistemi ve kullanilabilirlik kaliplari olan en az yedi buyuk saglayici vardir. Fiyatlar herhangi bir dakikada energy birimi basina 22 SUN'dan 80 SUN'a kadar degisir - en ucuz ve en pahali secenek arasinda 3,6 katlik bir fark.

Odeme sistemi, borsa veya programatik olarak USDT gonderen herhangi bir uygulama gelistiren bir gelistirici icin bu parcalanma gercek muhendislik sorunlari yaratir:

**Coklu entegrasyonlar.** Her saglayicinin kendi SDK'si, API formati ve hata yonetimi vardir. En iyi fiyati almak icin hepsiyle entegre olmak birden fazla kod tabani yonetmek anlamina gelir.

**Fiyat seffafligi yoktur.** Merkezi bir emir defteri veya fiyat akisi yoktur. En iyi fiyati bilmek icin her saglayiciyi ayri ayri sorgulamaniz ve karsilastirmaniz gerekir.

**Yedek gecis yoktur.** Tek saglayiciniz coker veya kapasitesi biterse, islemleriniz basarisiz olur veya pahali TRX yakma mekanizmasina geri doner.

**Manuel fiyat izleme.** Fiyatlar gun boyunca degisir. Sabah 9'da en ucuz olan saglayici, ogle sonrasi 3'te en pahali olabilir. Surekli izleme olmadan en iyi fiyati aldiginizi garanti edemezsiniz.

MERX, tek bir entegrasyon noktasiyla tum bu sorunlari ortadan kaldirir.

## MERX Nasil Calisir

Mimari, surekli calisan uc temel operasyon uzerine kuruludur.

### Fiyat Sorgulama

MERX, her buyuk TRON energy saglayicisina baglanir ve guncel fiyatlarini her 30 saniyede bir sorgular. Bu, tum piyasada gercek zamanli bir fiyat endeksi olusturur. MERX'te fiyatlari kontrol ettiginizde, her saglayicinin mevcut oranini ve hangisinin o anda en ucuz oldugunu gorursunuz.

Sorgulama basit bir fiyat kontrolu degildir. MERX ayrica saglayici kullanilabilirligini, kapasitesini ve desteklenen sure araligini da dogrular. Bir saglayicinin iyi bir fiyati olabilir ancak siparisiniizi karsilayacak kapasitesi olmayabilir. MERX, secenekleri sunmadan once bunlari filtreler.

### En Iyi Fiyat Yonlendirme

MERX araciligiyla bir energy siparisi verdiginizde, platform bunu varsayilan bir saglayiciya iletmez. Tum bagli saglayicilari siparisiniizin spesifik parametrelerine - miktar, sure, hedef adres - gore degerlendirir ve siparisi karsilayabilecek en ucuz saglayiciya yonlendirir.

En ucuz saglayici teslim edemezse (ag sorunlari, kapasite tukenme, zaman asimi), MERX otomatik olarak bir sonraki en ucuz secenege gecer. Bu yedek gecis seffaftir. Energy'nizi her iki durumda da alirsiniz; platform yonlendirme karmasikligini yonetir.

### Siparis Yurutme

Saglayici secildikten sonra MERX, energy delegasyonunu zincir uzerinde gerceklestirir. Energy hedef adresinizde gorunur ve kullanima hazir hale gelir. Tum surec - API cagrisindan energy delegasyonuna kadar - genellikle saniyeler icerisinde tamamlanir.

## Platform Bilesenleri

MERX yalnizca bir web arayuzu degildir. Farkli kullanim senaryolari icin tasarlanmis birden fazla entegrasyon yoluna sahip tam bir platformdur.

### Web Borsasi

[merx.exchange](https://merx.exchange) adresindeki birincil arayuz, energy pazarinin gercek zamanli gorunumunu saglar. Tum saglayicilardaki guncel fiyatlari gorebilir, manuel olarak siparis verebilir, bakiyenizi kontrol edebilir ve siparis gecmisinizi inceleyebilirsiniz. Arayuz profesyoneller icin tasarlanmistir: koyu tema, gorsel karmasiklik yok, veri yogun duzenler.

### REST API

API, energy ticaretinin tam yasam dongusu kapsayan 46 endpoint sunar:

- **Piyasa verileri** - guncel fiyatlar, fiyat gecmisi, saglayici karsilastirmasi
- **Siparisler** - energy ve bandwidth siparisleri olusturma, izleme, listeleme
- **Surekli siparisler** - takvime gore tekrarlanan energy satin alimlari
- **Hesap yonetimi** - bakiyeler, yatirmalar, cekmeler
- **Izleme** - kaynak izleyicileri ve esik degeri uyarilari olusturma
- **Adres araclari** - adresleri dogrulama, zincir uzerindeki kaynaklari kontrol etme

Tum endpoint'ler `/api/v1/` altinda surumlenmistir ve standart hata yaniti dondurmektedir. POST endpoint'leri, tekrarlanan siparisleri onlemek icin idempotency key destekler.

Eksiksiz dokumantasyon [merx.exchange/docs](https://merx.exchange/docs) adresinde mevcuttur ve endpoint referanslari, kimlik dogrulama rehberleri, hata kodu tablolari ve entegrasyon orneklerini kapsayan 36 sayfadan olusmaktadir.

### WebSocket

Sorgulama olmadan gercek zamanli fiyat guncellemelerine ihtiyac duyan uygulamalar icin MERX bir WebSocket baglantisi saglar. Fiyat kanallarina abone olun ve fiyatlar degistiginde - her 30 saniyede bir - guncellemeler alin.

### JavaScript SDK

Resmi JavaScript/TypeScript SDK, REST API'yi tipli bir istemciye sarar ve yerlesik hata yonetimi, yeniden deneme mantigi ve kolaylik yontemleri sunar.

```javascript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY
});

// Mevcut en iyi energy fiyatini alin
const prices = await merx.getPrices();
console.log('Best price:', prices.energy.best.price, 'SUN/unit');
console.log('Provider:', prices.energy.best.provider);

// Tum saglayicilari karsilastirin
const comparison = await merx.compareProviders();
for (const provider of comparison) {
  console.log(`${provider.name}: ${provider.price} SUN`);
}

// En iyi fiyattan energy satin alin
const order = await merx.createOrder({
  resourceType: 'ENERGY',
  amount: 65000,
  duration: '1h',
  targetAddress: 'TRecipientAddressHere'
});

console.log('Cost:', order.totalCost, 'SUN');
```

[npm](https://www.npmjs.com/package/merx-sdk) uzerinde `merx-sdk` olarak mevcuttur. Kaynak kodu [GitHub](https://github.com/Hovsteder/merx-sdk-js) uzerindedir.

### Python SDK

Python SDK, Python uygulamalari icin esdefer (senkron) ve esdefer olmayan (asenkron) destekle ayni islevselliigi saglar.

```python
from merx_sdk import MerxClient

client = MerxClient(api_key="YOUR_API_KEY")

# Mevcut en iyi energy fiyatini alin
prices = client.get_prices()
print(f"Best price: {prices['energy']['best']['price']} SUN/unit")

# Potansiyel tasarruflari hesaplayin
savings = client.calculate_savings(
    energy_amount=65000,
    num_transfers=1000
)
print(f"Monthly savings: {savings['savings_trx']} TRX")

# Siparis verin
order = client.create_order(
    resource_type="ENERGY",
    amount=65000,
    duration="1h",
    target_address="TRecipientAddressHere"
)
print(f"Order {order['id']}: {order['status']}")
```

[PyPI](https://pypi.org/project/merx-sdk/) uzerinde `merx-sdk` olarak mevcuttur. Kaynak kodu [GitHub](https://github.com/Hovsteder/merx-sdk-python) uzerindedir.

### Yapay Zeka Ajanlari Icin MCP Server

MERX, yapay zeka ajanlarinin ve LLM tabanli uygulamalarin TRON energy pazariyla dogrudan etkilesime girmesini saglayan bir Model Context Protocol (MCP) sunucusu icerir. Bir yapay zeka ajani, dogal arac cagrilari araciligiyla fiyatlari kontrol edebilir, siparis verebilir, adresleri izleyebilir ve energy satin alimlarini yonetebilir.

Bu, yapay zeka ajanlarinin giderek daha fazla zincir uzerindeki islemleri yonettigi goz onunde bulunduruldugunda ozellikle ilgilidir. Bir hazine yoneten, odemeleri isleyen veya bir DeFi stratejisini yoneten bir yapay zeka ajani, ozel entegrasyon kodu olmadan energy maliyetlerini optimize etmek icin MERX MCP sunucusunu kullanabilir.

[npm](https://www.npmjs.com/package/merx-mcp) uzerinde `merx-mcp` olarak mevcuttur. Kaynak kodu [GitHub](https://github.com/Hovsteder/merx-mcp) uzerindedir.

## Onemli Rakamlar

- **Bagli saglayici sayisi:** 7 buyuk TRON energy saglayicisi
- **Fiyat sorgulama:** tum saglayicilarda her 30 saniyede bir
- **Komisyon:** energy siparislerinde %0
- **API endpoint'leri:** 46 surumlu endpoint
- **Dokumantasyon:** tam API referansi dahil 36 sayfa
- **SDK'lar:** JavaScript (npm) ve Python (PyPI)
- **Maliyet azaltma:** TRX yakmaya kiyasla %94'e kadar
- **Mainnet dogrulanmis:** TRON mainnet'te 8 islem onaylandi

## MERX'i Tek Saglayiclardan Ayiran Nedir

Tek bir energy saglayicisi bir saticidir. MERX bir pazaryeridir. Bu ayrim bircok somut sekilde onemlidir.

**Fiyat.** Tek bir saglayici kendi fiyatini belirler. MERX size her saglayicinin fiyatini gosterir ve en ucuz olana yonlendirir. Herhangi bir dakikada, en ucuz saglayici farklidir. Bir ay boyunca, her zaman en iyi fiyata ulasmanin tasarruflari onemli olcude birikmektedir.

**Kullanilabilirlik.** Tek bir saglayicinin kapasitesi biterse veya cevrimdisi olursa, energy satin aliminiz basarisiz olur. MERX otomatik olarak bir sonraki mevcut saglayiciya yonlendirir. Uygulamanizin yedek gecis mantigi yonetmesi gerekmez.

**Seffaflik.** Tek bir saglayiciyla, rekabetci bir fiyat alip almadiginiza dair gorunurluguzuz yoktur. MERX size tam piyasa farkini gosterir, boylece siparisiniizin nereye ve neden yonlendirildigini goerebilirsiniz.

**Entegrasyon basitligi.** 7 saglayiciyla entegre olmak, 7 API entegrasyonu, 7 kimlik dogrulama akisi ve 7 set hata yonetimi yonetmek demektir. MERX ile entegre olmak tek API, tek SDK, tek kimlik bilgi seti demektir.

**Tarafsizlik.** MERX, platformdaki saglayicilerla rekabet eden kendi energy staking operasyonunu isletmez. Kendi arzini kayirmak yerine en iyi fiyata yonlendirmeyle uyumlu saf bir agregatordur.

## Dokumantasyon

MERX dokumantasyonu, kurumsal API referanslari standardinda insaa edilmistir. Otuz alti sayfa su konulari kapsar:

- Web ve API kullanicilari icin baslangic rehberleri
- Kimlik dogrulama ve API anahtari yonetimi
- Istek/yanit semalariyla eksiksiz endpoint referansi
- Cozum adimlariyla hata kodu tablolari
- SDK kurulum ve kullanim rehberleri
- WebSocket abonelik dokumantasyonu
- Idempotency ve yeniden deneme en iyi uygulamalari
- Hiz sinirlandirma ve kota bilgileri

Dokumantasyon [merx.exchange/docs](https://merx.exchange/docs) adresinde mevcuttur ve API ile senkronize tutulur - belgelenen her endpoint, canli API ile eslesmektedir.

## Gercek Zincir Uzerindeki Sonuclar

MERX, dogrulanmis sonuclarla TRON mainnet'te energy siparisleri yurutmustur. API siparisinden energy delegasyonuna ve energy icin sifir TRX yakilarak gerceklestirilen USDT transferine kadar tam akisi gosteren sekiz islem zincir uzerinde onaylanmistir.

Bu mainnet islemlerinde, yakma mekanizmasiyla 27,30 TRX'e mal olacak USDT transferleri, MERX yonlendirmeli energy kiralamasiyla 1,43 TRX'e mal olmustur. Delegasyon ve transfer islemleri herhangi bir TRON blok gezgininde herkese acik sekilde dogrulanabilir.

Bunlar testnet simuelasyonlari degildir. Gercek TRX ve gercek USDT ile gercek mainnet islemleridir ve yonlendirme, saglayici entegrasyonu ve zincir uzerindeki yurutmenin tamaminina uretimde calistigini kanitlamaktadir.

## MERX Kimin Icin

**Odeme islemcileri** - saticilar, serbest calisanlar veya tedarikcilere USDT gonderen. Her transferin energy maliyeti dogrudan kar marjini etkiler.

**Kripto borsalari** - TRC-20 cekim islemleri yapan. Energy maliyetleri ya absorbe edilir (kari azaltir) ya da kullanicilara yansitilir (rekabet gucunu azaltir).

**Alim satim botlari** - arbitraj veya piyasa yapicilik stratejilerinin parcasi olarak sik zincir uzerindeki transferler yapan. Transfer basina tasarruf edilen her TRX kesri hacimle olceklenir.

**DeFi protokolleri** - TRON akilli sozlesmeleriyle etkilesime giren. Energy maliyetleri her zincir uzerindeki islemin ekonomisini etkiler.

**Hazine yonetimi** ekipleri - USDT ve diger TRC-20 tokenlerin zincir uzerindeki hareketleri icin operasyonel maliyetleri optimize etmesi gereken.

**Yapay zeka ajanlari** - zincir uzerindeki islemleri otonom olarak yoneten ve en iyi fiyattan energy elde etmek icin programatik bir yola ihtiyac duyan.

## Baslangic

Hesap olusturmak ve canli energy pazarini gormek icin [merx.exchange](https://merx.exchange) adresini ziyaret edin. Platform calisir durumda ve TRON mainnet'te siparis islemektedir.

API entegrasyonu icin [dokumantasyon](https://merx.exchange/docs) ile baslayin. Diliniz icin SDK'yi kurun:

```bash
# JavaScript / TypeScript
npm install merx-sdk

# Python
pip install merx-sdk
```

Yapay zeka ajan entegrasyonu icin:

```bash
npm install merx-mcp
```

Ilk adim her zaman aynidir: guncel fiyatlari kontrol edin. Piyasanin nasil gorunugunu gorun. Energy icin su anda odediginizi, toplanan piyasanin sunduguyla karsilastirin. Rakamlar kendileri icin konusmaktadir.

---

*Etiketler: merx borsasi, tron energy borsasi, tron kaynak pazaryeri, tron energy agregator, usdt transfer optimizasyonu*
