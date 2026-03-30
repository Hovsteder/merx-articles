# TRON Energy ve Bandwidth: Her Kaynak Ne Ise Yarar ve Ne Zaman Ihtiyaciniz Olur

TRON, islem ucretlerini odemek icin iki ayri kaynak kullanir: energy ve bandwidth. Aralarindaki farki anlamak, TRON uzerinde gelistirme yapan veya islem maliyetlerini yoneten herkes icin onemlidir. Energy, akilli sozlesme calistirmasi icin odeme yapar - blockchain uzerinde kod calistirmanin hesaplama maliyeti. Bandwidth, veri iletimi icin odeme yapar - isleminizin ham baytlari. Bu makale, her kaynaginin ne yaptigini, birini veya her ikisini ne zaman kullanmaniz gerektigini, ne kadara mal olduklarini ve bunlari verimli bir sekilde nasil elde edebileceginizi aciklar.

## Iki Kaynak, Iki Amac

Cogu blockchain tek bir ucret mekanizmasina sahiptir. Ethereum her sey icin gas alir. Bitcoin islem boyutuna gore ucret alir. TRON farklidir: hesaplama maliyetini veri maliyetinden ayirir.

**Energy**, akilli sozlesme kodunun calistirilmasinin hesaplama maliyetini kapsar. USDT gonderdiginizde, TRON Virtual Machine (TVM) USDT sozlesmesinin `transfer` fonksiyonunu calistirir. Bu calistirma energy tuketir. Sozlesme mantigi ne kadar karmasiksa, o kadar fazla energy gerekir.

**Bandwidth**, isleminizin aga yayinlanmasinin veri maliyetini kapsar. Her islemin bayt cinsinden bir boyutu vardir - ilgili adresler, fonksiyon parametreleri, imza. Bu baytlarin aga iletilmesi bandwidth tuketir.

Bu ayrim onemlidir cunku farkli islem turleri cok farkli kaynak profillerine sahiptir. Basit bir TRX transferi bandwidth kullanir ancak sifir energy tuketir (akilli sozlesme yoktur). Karmasik bir DeFi etkilesimi ise her ikisinden de buyuk miktarda kullanir.

## Energy: Akilli Sozlesmeler Icin Odeme

Energy, TRON Virtual Machine akilli sozlesme kodu calistirdiginda tuketilir. Buna su islemler dahildir:

- **USDT transferleri** (TRC-20 token transferleri) - yaklasik 65.000 energy
- **USDC, TUSD ve diger TRC-20 token transferleri** - 50.000-65.000 energy
- **Token onaylari** (bir sozlesmenin tokenlarinizi harcamasina izin verme) - 30.000-50.000 energy
- **DEX takasi** (SunSwap, JustSwap) - 200.000-500.000 energy
- **NFT basimi** - 100.000-300.000 energy
- **Staking islemleri** - 50.000-150.000 energy
- **Karmasik DeFi etkilesimleri** (borclama, getiri ciftciligi) - 200.000-1.000.000+ energy

Tuketilen energy miktari, akilli sozlesmenin dahili olarak ne yaptigina baglidir. Basit bir token transferi birkaç duzine islem calistirir. Bir DEX takasi, birden fazla likidite havuzundan yonlendirme yapar, fiyat hesaplamalari gerceklestirir, kaymayi kontrol eder ve bakiyeleri gunceller - toplanan yuzlerce islem.

### Energy Olmadiginda Ne Olur

Adresinizde sifir energy varken bir akilli sozlesme islemi calistirirseniz, TRON bunu reddetmez. Bunun yerine, energy maliyetini karsilamak icin bakiyenizdeki TRX'i yakar. Buna "energy yakma" denir ve pahalidir.

Yakma orani bir ag parametresi tarafindan belirlenir. Mevcut ayarlarda, eksik olan her energy birimi yakildiginda yaklasik 0,00021 TRX'e mal olur. 65.000 energy gerektiren standart bir USDT transferi icin:

- Energy ile: hesaplama bileseni icin 0 TRX maliyet
- Energy olmadan: yaklasik 13,5 TRX yakilir

Bu 4-5 katlik maliyet carpani, TRON uzerinde ara sira yapilan islemlerin otesinde herhangi bir sey yapan kisiler icin energy yonetiminin neden bu kadar onemli oldugunu gostermektedir.

### Energy Kendiligindenin Yenilenmez

Bandwidth'in aksine, energy otomatik olarak yenilenmez. Uc yontemden biriyle energy elde edersiniz:

1. **TRX stake etme** - orantili energy almak icin TRX'i en az 14 gun boyunca kilitleyin. Stake edilen TRX basina dusecek energy miktari, toplam ag stake'ine baglidir.
2. **Energy kiralama** - belirli bir sure icin (1 saat ila 30 gun) adresinize energy delege etmesi icin bir saglayiciya odeme yapin.
3. **Agregator kullanma** - MERX gibi birden fazla kiralama saglayicisini karsilastiran ve en ucuz saglayiciya yonlendiren hizmetler.

Her yontemin farkli ekonomisi vardir. Staking, sermaye kilitleme gerektirir ve surekli energy saglar. Kiralama, sermaye kilitleme gerektirmez ancak kiralama basina ucret odemek gerekir. Cogu kullanici icin, kullanmayi planlamadiklari buyuk TRX varliklari olmadikca kiralama daha maliyet etkindir.

## Bandwidth: Islem Verisi Icin Odeme

Bandwidth, TRON uzerindeki her islemin veri iletim maliyetini kapsar. Bandwidth puanlari olarak olculur; burada 1 bandwidth puani, islem verisinin 1 baytina esittir.

Tipik bir TRON islemi 250-350 bayttir. Tam boyut, islem turune ve parametrelere baglidir:

- **TRX transferi** - yaklasik 270 bayt (270 bandwidth puani)
- **TRC-20 token transferi** - yaklasik 350 bayt (350 bandwidth puani)
- **Akilli sozlesme dagitimi** - sozlesme boyutuna bagli olarak 1.000-10.000+ bayt
- **Coklu imza islemi** - imzaci sayisina bagli olarak 400-800 bayt

### Ucretsiz Bandwidth: Gunde 600 Puan

Aktive edilmis her TRON adresi gunde 600 ucretsiz bandwidth puani alir. Bu, 24 saatlik kayan bir pencere uzerinden yenilenir. Baglamiyla:

- 600 bandwidth puani gunde yaklasik 2 basit TRX transferini kapsar
- Veya yaklasik 1-2 TRC-20 token transferini kapsar (yalnizca bandwidth bileseni)

Gunde birkaç islem yapan bireysel kullanicilar icin, ucretsiz bandwidth veri maliyetini tamamen karsilayabilir. Gunde yuzlerce islem yapan isletmeler veya uygulamalar icin ucretsiz bandwidth ihmal edilebilir duzeydedir ve ek bandwidth elde edilmelidir.

### Bandwidth Olmadiginda Ne Olur

Energy'ye benzer sekilde, adresinizde bandwidth (ucretsiz veya stake edilmis) yoksa, TRON maliyeti karsilamak icin TRX yakar. Bandwidth yakma orani, bandwidth puani basina yaklasik 0,001 TRX'tir. 350 baytlik bir TRC-20 transferi icin:

- Bandwidth ile: veri bileseni icin 0 TRX maliyet
- Bandwidth olmadan: yaklasik 0,35 TRX yakilir

Bandwidth yakma maliyeti, mutlak degerler acisindan energy yakma maliyetinden cok daha kucuktur. Bir USDT transferi icin energy yakma (13,5 TRX), bandwidth yakmayi (0,35 TRX) golgede birakir. Bu nedenle cogu maliyet optimizasyon calismasi energy uzerine odaklanir.

### Bandwidth Yenilenme

Staking yoluyla elde edilen bandwidth, 24 saatlik bir pencere uzerinden yenilenir. 10.000 bandwidth puani alacak kadar TRX stake ederseniz, gunde 10.000 puana kadar kullanabilirsiniz ve bakiyeniz surekli olarak yenilenir.

Ucretsiz bandwidth de ayni takvimde yenilenir. Herhangi bir sey yapmaniz gerekmez - otomatik olarak dolar.

## Energy Ne Zaman Gerekir

Akilli sozlesme calistirma iceren herhangi bir islem icin energy gerekir. Pratikte bu, basit TRX transferlerinin otesindeki cogu TRON islemini kapsar.

### TRC-20 Token Transferleri

En yaygin energy tuketen islem. USDT, USDC, WTRX veya herhangi bir TRC-20 token gondermek, token sozlesmesinin transfer fonksiyonunu calistirmak icin energy gerektirir.

Tahmini energy tuketimi:
- USDT (TRC-20): ~65.000 energy
- USDC (TRC-20): ~50.000-65.000 energy
- Diger TRC-20 tokenlar: degisken, tipik olarak 30.000-65.000 energy

### DEX Takasi

SunSwap veya diger TRON DEX'lerinde token takasi, islemin birden fazla sozlesme cagrisi icermesi nedeniyle onemli olcude daha fazla energy gerektirir: yonlendirme, fiyat hesaplama, likidite havuzu etkilesimi ve token transferleri.

Tahmini energy tuketimi:
- Basit takas (tek havuz): ~200.000 energy
- Cok atlamali takas (2-3 havuz uzerinden yonlendirilen): ~300.000-500.000 energy

### DeFi Islemleri

TRON uzerindeki borclama, borc alma, getiri ciftciligi ve diger DeFi etkilesimleri energy tuketir. Tek bir islemde birden fazla sozlesmeyle etkilesime giren karmasik islemler 500.000+ energy tuketebilir.

### NFT Islemleri

TRON pazaryerlerinde NFT basimi, transferi ve listelenmesi energy tuketir. Basim, sozlesmenin yeni depolama olusturmasi gerektiginden tipik olarak transferden daha pahalidir.

## Bandwidth Ne Zaman Gerekir

TRON uzerindeki her islem icin istisnasiz bandwidth gerekir. En basit islem bile - TRX'i bir adresten digerine gondermek - bandwidth tuketir.

### Basit TRX Transferleri

TRX transferi, bandwidth gerektiren ancak energy gerektirmeyen tek yaygin islemdir. Akilli sozlesme icermez, bu nedenle energy tuketimi sifirdir.

TRX'i toplu olarak gonderen kullanicilar icin (ornegin, birden fazla adrese TRX dagitimi), bandwidth birincil maliyet kaygisi haline gelir. Ucretsiz bandwidth (gunde 600 puan) yalnizca 2 transferi kapsar. 1.000 TRX transferinin toplu dagitimi yaklasik 270.000 bandwidth puanina ihtiyac duyar.

### Yuksek Hacimli Islem Adresleri

Gunde binlerce islem yayinlayan borsalar ve odeme islemcileri onemli bandwidth'e ihtiyac duyar. Islem basina bandwidth maliyeti kucuk olsa da toplanir:

- Gunde 1.000 TRC-20 transferi: ~350.000 bandwidth puani gerekli
- Gunde 10.000 TRC-20 transferi: ~3.500.000 bandwidth puani gerekli

Bu hacimlerde, bandwidth icin TRX stake etmek veya bir agregator araciligiyla bandwidth kiralamak ekonomik olarak onemli hale gelir.

## Her Ikisine de Ne Zaman Ihtiyaciniz Olur

Gercek dunya TRON islemlerinin cogu, ayni anda hem energy hem de bandwidth gerektirir. Yaygin islemler icin kaynaklarin nasil birlestigini asagida gorebilirsiniz.

### USDT Transferi (En Yaygin)

| Kaynak | Miktar | Kaynak Olmadan Maliyet |
|---|---|---|
| Energy | ~65.000 | ~13,5 TRX |
| Bandwidth | ~350 puan | ~0,35 TRX |
| **Toplam** | | **~13,85 TRX** |

Kiralanmis energy ve stake edilmis/ucretsiz bandwidth ile:

| Kaynak | Miktar | Kaynak Ile Maliyet |
|---|---|---|
| Energy | ~65.000 | ~2-3 TRX (kiralama maliyeti) |
| Bandwidth | ~350 puan | 0 TRX (ucretsiz veya stake edilmis) |
| **Toplam** | | **~2-3 TRX** |

### DEX Takasi

| Kaynak | Miktar | Kaynak Olmadan Maliyet |
|---|---|---|
| Energy | ~300.000 | ~63 TRX |
| Bandwidth | ~500 puan | ~0,5 TRX |
| **Toplam** | | **~63,5 TRX** |

Kiralanmis energy ile:

| Kaynak | Miktar | Kaynak Ile Maliyet |
|---|---|---|
| Energy | ~300.000 | ~8-12 TRX (kiralama maliyeti) |
| Bandwidth | ~500 puan | 0 TRX (ucretsiz veya stake edilmis) |
| **Toplam** | | **~8-12 TRX** |

Kalip aciktir: hemen hemen her senaryoda energy maliyete hakim olur. Bandwidth kucuk ve tutarli bir ek yuktur.

## Maliyet Karsilastirmasi: Energy ve Bandwidth

| Faktor | Energy | Bandwidth |
|---|---|---|
| Neyin icin odenir | Akilli sozlesme calistirma | Islem verisi iletimi |
| Ucretsiz tahsis | Yok | Gunde 600 puan |
| Yenilenir | Hayir (stake edilmedikce) | Evet (24 saatlik kayan pencere) |
| Yakildigindaki maliyet | Birim basina ~0,00021 TRX | Birim basina ~0,001 TRX |
| Tipik islem basina maliyet | 13,5 TRX (USDT transferi) | 0,35 TRX (USDT transferi) |
| En buyuk etki | Token transferleri, takaslar, DeFi | Toplu TRX transferleri |
| Kiralama pazari | Aktif (birden fazla saglayici) | Daha kucuk ama buyuyen |
| Staking kilitleme | Minimum 14 gun | Minimum 3 gun |

Energy, cogu TRON kullanicisi icin birincil maliyet etkenidir. Bandwidth, esas olarak yuksek hacimli TRX transferleri veya her TRX kesrinin onemli oldugu olcekte calisma durumlarinda onem kazanir.

## Her Kaynak Nasil Elde Edilir

### Energy Elde Etme

**Secenek 1: TRX Stake Etme (Stake 2.0)**

Agin toplam energy havuzundan orantili bir pay almak icin TRX'i dondurun. Stake edilen TRX basina alacaginiz energy miktari, toplam ag stake'ine baglidir.

```
Alinan energy = (stake edilen TRX'iniz / toplam ag stake TRX) * toplam energy havuzu
```

Mevcut ag kosullarinda, stake edilen yaklasik 1 TRX gunde 10-15 energy uretir. Tek bir USDT transferini karsilamak icin (65.000 energy), yaklasik 4.300-6.500 TRX stake etmeniz gerekir. TRX basina 0,24 dolarla bu, kilitli sermaye olarak 1.032-1.560 dolar demektir.

Cogu kullanici icin bu, kiralama ile karsilastirildiginda ekonomik degildir.

**Secenek 2: Bir saglayicidan kiralama**

Birden fazla saglayici energy kiralamalari satar. Fiyatlar, saglayiciya, sureye ve piyasa kosullarina bagli olarak energy birimi basina gunde 22 ila 80 SUN arasinda degisir.

Tek bir USDT transferi icin (65.000 energy, 1 saatlik kiralama):
- En ucuz saglayici: ~1,5 TRX (~0,36 $)
- Ortalama saglayici: ~2,0 TRX (~0,48 $)
- En pahali saglayici: ~3,5 TRX (~0,84 $)

**Secenek 3: Agregator kullanma**

MERX, birden fazla saglayicidan energy'yi toplar ve her siparisi mevcut en ucuz secenege yonlendirir. Bu, birden fazla saglayiciyla hesap tutma veya fiyatlari manuel olarak karsilastirma ihtiyacini ortadan kaldirir.

```typescript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({ apiKey: 'your-api-key' });

// USDT transferi icin energy kiralama
const order = await merx.createOrder({
  energy: 65000,
  duration: '1h',
  target: 'YOUR_TRON_ADDRESS'
});
```

### Bandwidth Elde Etme

**Secenek 1: Ucretsiz bandwidth**

Hicbir sey yapmayin. Aktive edilmis her TRON adresi gunde 600 ucretsiz bandwidth puani alir. Dusuk hacimli kullanicilar icin bu yeterli olabilir.

**Secenek 2: Bandwidth icin TRX stake etme**

Energy staking'e benzer sekilde, TRX'i ozellikle bandwidth icin dondurabilirsiniz. Bandwidth icin staking kilitleme suresi 3 gundur (energy'nin 14 gununen daha kisa).

Bandwidth basina TRX orani genellikle energy'den daha elverislidir. Stake edilen yaklasik 1 TRX gunde 100-150 bandwidth puani uretir - bir islemin bandwidth ihtiyacinin yaklasik yarisini karsilamaya yeter.

**Secenek 3: MERX araciligiyla kiralama**

MERX, energy'ye ek olarak bandwidth agregasyonunu da yonetir. Her iki kaynaga da ihtiyac duyan kullanicilar icin, MERX uzerinden tek bir API cagrisi her ikisini de guvenli hale getirebilir.

## Pratik Senaryolar

### Senaryo 1: Gunde 50 USDT Odemesi Gonderen Kucuk Isletme

**Gunluk gereken kaynaklar:**
- Energy: 50 x 65.000 = 3.250.000 energy
- Bandwidth: 50 x 350 = 17.500 bandwidth puani

**Oneri:** MERX araciligiyla gunluk veya cok gunluk bazda energy kiralanmasi. Ucretsiz bandwidth (gunde 600) yetersizdir, bu nedenle bandwidth icin kucuk miktarda TRX stake edin veya yakilmasina izin verin (bandwidth icin gunde yalnizca ~17,5 TRX, energy maliyetlerinden cok daha az).

### Senaryo 2: Haftada 2-3 USDT Transferi Gonderen Bireysel Kullanici

**Gereken kaynaklar:**
- Energy: haftada 2-3 x 65.000 = 130.000-195.000 energy
- Bandwidth: haftada 2-3 x 350 = 700-1.050 bandwidth puani

**Oneri:** Islem basina en ucuz saglayicidan (veya MERX'ten) 1 saatlik energy kiralanmasi. Ucretsiz bandwidth cogu gunu kapsar (gunde 600). Gunde 2+ islem yapilan gunlerde, bandwidth yakma maliyeti (~0,35 TRX) ihmal edilebilir duzeydedir.

### Senaryo 3: Gunde 5.000 Islem Isleyen Borsa

**Gunluk gereken kaynaklar:**
- Energy: 5.000 x 65.000 = 325.000.000 energy
- Bandwidth: 5.000 x 350 = 1.750.000 bandwidth puani

**Oneri:** Temel yuk icin MERX araciligiyla uzun vadeli energy kiralamalari (30 gun), ani artislar icin kisa vadeli takviyelerle birlikte. Gunluk veri maliyetini karsilamak icin bandwidth icin TRX stake edin. Bu hacimde, birim basina kucuk tasarruflar bile onemli miktarlara ulasmaktadir.

## MERX Her Ikisini de Yonetiyor: Energy ve Bandwidth Agregasyonu

Cogu kullanici, baskin maliyet oldugu icin yalnizca energy optimizasyonuna odaklanir. Ancak yuksek hacimli islemler icin bandwidth maliyetleri de onemlidir ve MERX her iki kaynagi da toplar.

MERX araciligiyla siparis verdiginizde energy, bandwidth veya her ikisini belirtebilirsiniz. Platform tum aktif saglayicilari sorgular, en ucuz kombinasyonu bulur ve delegasyonu yonetir. Kaynak duyarli islemler icin MERX gereksinimleri tahmin eder, bakiyelerinizi kontrol eder, en ucuz saglayicidan eksik kaynaklari temin eder, islemi gerceklestirir ve tam maliyet dokumunu raporlar.

```python
from merx_sdk import MerxClient

merx = MerxClient(api_key="your-api-key")

# Herhangi bir adres icin kaynaklari kontrol edin
resources = merx.check_address_resources("YOUR_TRON_ADDRESS")
print(f"Energy: {resources['energy']['available']}")
print(f"Bandwidth: {resources['bandwidth']['available']}")
print(f"Free bandwidth: {resources['bandwidth']['free']}")

# Bir USDT transferinin ne kadara mal olacagini tahmin edin
estimate = merx.estimate_transaction_cost(
    from_address="YOUR_TRON_ADDRESS",
    to_address="RECIPIENT_ADDRESS",
    token="TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    amount="100"
)
print(f"Energy needed: {estimate['energy_needed']}")
print(f"Bandwidth needed: {estimate['bandwidth_needed']}")
print(f"Estimated cost: {estimate['total_cost_trx']} TRX")
```

Eksiksiz API dokumantasyonu [merx.exchange/docs](https://merx.exchange/docs) adresinde mevcuttur. SDK'lar: [JavaScript](https://www.npmjs.com/package/merx-sdk) | [Python](https://pypi.org/project/merx-sdk/). Yapay zeka ajan entegrasyonu icin [MERX MCP server](https://github.com/Hovsteder/merx-mcp) sayfasina bakin.

## Ozet

Energy ve bandwidth, her ikisi de temel TRON kaynaklaridir, ancak akilli sozlesme etkilesimleri icin - ki bu TRON uzerinde bugun olan seylerin cogunu olusturur - energy islem maliyetlerine hakim olur. Haftada birkaçtan fazla islem yapan herkes icin, energy'yi kiralama veya agregator araciligiyla elde etmek, agin TRX'inizi yakmasina izin vermekten yuzde 75-85 daha ucuzdur. Cogu kullanici icin, MERX gibi bir agregator araciligiyla islem basina energy kiralamak en basit ve en maliyet etkin yaklasimdir.
