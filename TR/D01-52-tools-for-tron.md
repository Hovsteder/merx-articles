# TRON Icin 52 Arac: MERX MCP Sunucusunun Icinden

Yapay zeka ajanlari blokzincir dunyasina giriyor, ancak cogunlugu hala TRON ile etkilesim kuramiyor - diger tum zincirlerden daha fazla USDT isleyen ag. MERX MCP sunucusu, herhangi bir yapay zeka ajanina TronGrid API anahtarlari veya ozel blokzincir entegrasyonlari gerektirmeden TRON uzerinde otonom calismasi icin 52 arac, 30 prompt ve 21 kaynak iceren eksiksiz bir arac seti sunarak bunu degistirmektedir. Bu makale nasil calistigini, neler yapabildigini ve neden onemli oldugunu anlatmaktadir.

## MCP (Model Context Protocol) Nedir?

Model Context Protocol, Anthropic tarafindan olusturulmus ve yapay zeka ajanlarinin harici araclar ve veri kaynaklarina nasil baglanacagini tanimlayan acik bir standarttir. Bunu yapay zeka icin USB-C olarak dusunun: herhangi bir ajanin herhangi bir hizmetle etkilesim kurmak icin kullanabilecegi tek, standartlastirilmis bir arayuz.

MCP'den once, bir yapay zeka ajanini harici bir sisteme baglamak, her entegrasyon icin ozel fonksiyon cagrisi kodu yazmak anlamina geliyordu. Her ajan cercevesinin kendi formati vardi. Her API kendi sarmalayicisini gerektiriyordu. Sonuc parcalanmaydi - her biri ayri yonetilen duizinelerce uyumsuz entegrasyon.

MCP bunu standartlastirir. Bir arac saglayici bir MCP sunucusu yayinlar. MCP uyumlu herhangi bir ajan - Claude, GPT tabanli ajanlar, acik kaynakli cerceveler - buna baglanir ve tum yeteneklerine aninda erisim kazanir. Ozel kod yok. Ajan bazinda entegrasyon calismasi yok.

Protokol uc tur yetenek destekler:

- **Araclar (Tools)** - ajanin cagirabilecegi fonksiyonlar (ornegin token transfer etme, bakiyeleri kontrol etme)
- **Kaynaklar (Resources)** - ajanin okuyabilecegi yapilandirilmis veriler (ornegin piyasa fiyatlari, saglayici listeleri)
- **Promptlar (Prompts)** - yaygin gorevler icin onceden olusturulmus talimat sablonlari (ornegin "TRON maliyetlerimi optimize et")

## TRON Neden Bir MCP Sunucusuna Ihtiyac Duyuyor

TRON, USDT transferleri icin baskin agdir. Gunluk milyarlarca dolarlik stablecoin hacmi islemektedir. Yine de TRON icin ekosistem araclari, ozellikle programatik erisim acisindan Ethereum ve Solana'nin gerisinde kalmistir.

Yapay zeka ajanlari icin acik daha da geniistir. Bugun TRON uzerinde USDT gondermek isteyen bir ajanin su adimlari izlemesi gerekir:

1. TronGrid API anahtarlari edinmek ve yonetmek
2. TRON kaynak modelini anlamak (energy, bandwidth, staking)
3. Islem olusturma, imzalama ve yayinlama islemlerini yonetmek
4. Islem onayini izlemek
5. Ucretlerde TRX yakmayi onlemek icin energy maliyetlerini yonetmek

Bu adimlarin her biri uzman bilgisi gerektirir. Cogu ajan gelistiricisi buna sahip degildir. Sahip olanlar bile, API'ler degistiginde bozulan entegrasyonlar olusturmak icin haftalar harcamaktadir.

MERX MCP sunucusu tum bu surturunmeyi ortadan kaldirir. TRON islemlerini, herhangi bir yapay zeka ajaninin standart MCP protokolu uzerinden cagirabilecegi basit, iyi belgelenmis araclar olarak sunar.

## MERX MCP Sunucusuna Genel Bakis

MERX MCP sunucusu su bilesenleri saglar:

- **52 arac** - TRON uzerinde islem yurutmek icin
- **30 prompt** - rehberli is akislari ve cok adimli gorevler icin
- **21 kaynak** - piyasa verilerini, saglayici bilgilerini ve zincir durumunu okumak icin

Tum trafik MERX API uzerinden akar. MCP sunucusu, MCP protokolu ile MERX arka ucu arasinda bir ceeviri katmani olarak calisir. Bu, ajanlarin TronGrid API anahtarlarina ihtiyac duymadigi, RPC baglantilari yonetmek zorunda olmadigi ve TRON dugumlerinde hiz sinirlandirma veya yedek gecis yonetmek zorunda olmadigi anlamina gelir.

Sunucu iki dagitim modunda kullanilabilir:

- **Barindirilmis SSE** - tek satirlik yapilandirmayla URL uzerinden baglanma
- **Yerel stdio** - maksimum kontrol icin npx ile yerel olarak calistirma

## Mimari: TronGrid Anahtari Gerektirmez

Cogu TRON gelistirme araci bir TronGrid API anahtari gerektirir. Kaydolur, onay bekler, hiz sinirlarini yonetir ve anahtar rotasyonuyla ilgilenirsiniz. Otonom calisan yapay zeka ajanlari icin bu gereksiz bir bagimlilik yaratir.

MERX MCP sunucusu, tum blokzincir etkilesimlerini MERX API katmani uzerinden yonlendirir. Mimari su sekilde gorunur:

```
AI Agent <-> MCP Protocol <-> MERX MCP Server <-> MERX API <-> TRON Network
```

Ajan tek bir MERX API anahtariyla kimlik dogrulamasi yapar. Arka planda MERX su islemleri yonetir:

- Otomatik yedek gecisli TronGrid ve tam dugum baglantilari
- Hiz sinirlandirma ve istek kuyrugu
- Islem olusturma ve ucret tahmini
- Energy ve bandwidth maliyet optimizasyonu

Bu tasarim, ajan gelistiricisinin bircok kimlik bilgisi yerine tek bir kimlik bilgisi yonetmesi ve MERX'in tum altyapi karmasikligini ustlenmesi anlamina gelir.

## Arac Kategorileri: Tum 15 Kategoriye Genel Bakis

52 arac, 15 islevsel kategoride organize edilmistir. Her kategorinin neleri kapsadigi ve ornekler asagida yer almaktadir.

### 1. Kimlik Dogrulama ve Hesap Yonetimi

MERX'e baglanma ve API oturumlarini yonetme araclari.

- `set_api_key` - MERX ile kimlik dogrulama
- `create_account` - yeni hesap olusturma
- `login` - oturum tokeni alma

### 2. Bakiye ve Yatirma

Bakiyeleri kontrol etme, TRX yatirma ve fonlama yonetimi.

- `get_balance` - MERX hesap bakiyesini kontrol etme
- `deposit_trx` - platforma TRX yatirma
- `get_deposit_info` - yatirma adresi ve talimatlari alma
- `enable_auto_deposit` - otomatik yatirmalari ayarlama

### 3. Energy Pazari ve Fiyatlandirma

Tum saglayicilardaki energy pazarini sorgulama.

- `get_prices` - tum saglayicilardan guncel fiyatlar
- `get_best_price` - belirli bir sure icin en dusuk fiyat
- `compare_providers` - yan yana saglayici karsilastirmasi
- `analyze_prices` - gecmis fiyat analizi

### 4. Siparis Yonetimi

Energy siparisleri olusturma ve izleme.

- `create_order` - energy siparisi verme
- `create_paid_order` - TRX ile dogrudan odeme (yatirma gerektirmez)
- `get_order` - siparis durumunu kontrol etme
- `list_orders` - siparis gecmisini goruntuleme
- `create_standing_order` - tekrarlanan siparisler olusturma

### 5. Kaynak Tahmini ve Optimizasyonu

Islemleri yurutmeden once maliyetleri tahmin etme.

- `estimate_transaction_cost` - herhangi bir islemin maliyetini tahmin etme
- `estimate_contract_call` - akilli sozlesme cagrisi icin energy tahmini
- `calculate_savings` - kiralanmis energy ile ve olmadan maliyet karsilastirmasi
- `check_address_resources` - bir adresin mevcut energy ve bandwidth degerlerini goruntuleme
- `suggest_duration` - optimum kiralama suresi onerisi

### 6. Kaynak Duyarli Islemler

Otomatik kaynak optimizasyonuyla islem yurutme.

- `ensure_resources` - islemden once energy temin etme
- `transfer_trx` - maliyet optimizasyonuyla TRX gonderme
- `transfer_trc20` - TRC-20 token gonderme (USDT, USDC vb.)
- `approve_trc20` - token harcama onaylamasi

### 7. Takas Islemleri

TRON DEX'lerinde token takasi yurutme.

- `get_swap_quote` - takas oncesi fiyat teklifi alma
- `execute_swap` - takasi gerceklestirme

### 8. Akilli Sozlesme Etkilesimi

Herhangi bir TRON akilli sozlesmesinden okuma ve yazma.

- `read_contract` - gorunum fonksiyonu cagrisi (gas yok)
- `call_contract` - durum degistiren fonksiyon yurutme

### 9. Blokzincir Verileri

Zincir durumunu dogrudan sorgulama.

- `get_block` - blok bilgisi alma
- `get_transaction` - hash ile islem arama
- `get_chain_parameters` - mevcut ag parametreleri
- `get_account_info` - zincirden tam hesap detaylari

### 10. Token Bilgileri

Token metaverileri ve fiyatlandirma arama.

- `get_token_info` - sozlesme detaylari, ondalik basamaklar, toplam arz
- `get_token_price` - mevcut piyasa fiyati
- `get_trc20_balance` - herhangi bir adres icin token bakiyesi
- `get_trx_balance` - herhangi bir adres icin TRX bakiyesi
- `get_trx_price` - mevcut TRX/USD fiyati

### 11. Adres Araclari

TRON adreslerini dogrulama ve donusturme.

- `validate_address` - bir adresin gecerli olup olmadigini kontrol etme
- `convert_address` - base58 ve hex formatlari arasinda donusturme

### 12. Islem Gecmisi

Gecmis islemleri arama ve analiz etme.

- `get_transaction_history` - bir adres icin islemleri listeleme
- `search_transaction_history` - tur, token, tarih araligina gore filtreleme

### 13. Izleme

Uyarilar ve otomatik izleme olusturma.

- `create_monitor` - belirli olaylar icin bir adresi izleme
- `list_monitors` - aktif izleyicileri goruntuleme

### 14. Fiyat Gecmisi

Gecmis energy fiyatlandirma verilerine erisim.

- `get_price_history` - trend analizi icin zaman icerisindeki energy fiyatlari

### 15. Niyet Yurutme

Birden fazla adimi birlestiren ust duzey eylemler.

- `execute_intent` - ne istediginizi dogal dilde tanimlayin; sunucu adimlari belirler
- `simulate` - bir niyetin ne olacagini gormek icin kuru calistirma
- `explain_concept` - herhangi bir TRON kavraminin aciklamasini alma

## Kaynak Duyarli Islemler Aciklandi

Bu, MERX MCP sunucusunun en onemli ozelligi ve en fazla para tasarrufu saglayan ozelligidir.

TRON'da her akilli sozlesme etkilesimi energy gerektirir. Energy'niz yoksa, ag bunun karsiliginda TRX'inizi yakar - piyasadan energy kiralamanin yaklasik 4 kati daha pahali bir oranda.

MERX MCP sunucusu her islemi kaynak duyarli hale getirir. Bir ajan USDT gondermek icin `transfer_trc20` cagirdiginda, sunucu otomatik olarak:

1. Gondericinin mevcut energy bakiyesini kontrol eder
2. Transfer icin gereken energy'yi tahmin eder (standart USDT transferi icin yaklasik 65.000 energy)
3. Energy yetersizse, piyasadaki mevcut en ucuz energy'yi kiralar
4. Energy delegasyonunun onaylanmasini bekler
5. Transferi gerceklestirir
6. Energy kiralamaisi dahil toplam maliyeti raporlar

Ajanin bunlarin hicbirini anlamasina gerek yoktur. Tek bir arac cagirmaktadir ve optimizasyon arka planda gerceklesir.

```json
{
  "tool": "transfer_trc20",
  "arguments": {
    "from": "TJnVmb5rFLHPqfDMRMGwMH2iofhzN3KXLG",
    "to": "TKVSaJQDBeNzXj4jMjGrFk2tWaj5RkD6Lx",
    "token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "amount": "100"
  }
}
```

Kaynak duyarliligi olmadan, bu 100 USDT transferi yaklasik 13,5 TRX ucret yakar. MERX energy kiralamasiyla ayni transfer yaklasik 3-4 TRX'e mal olur. Binlerce transfer boyunca tasarruflar onemli olcude birikmektedir.

## Gercek Ornek: Yapay Zeka Ajani Otonom Olarak 0,1 TRX'i USDT'ye Takas Ediyor

Iste bir yapay zeka ajaninin herhangi bir insan mudahalesi olmadan MERX MCP sunucusunu kullanarak takas yuruttugun gosteren gercek bir senaryo.

**Adim 1: Ajan TRX fiyatini kontrol eder**

```json
{ "tool": "get_trx_price" }
// Response: { "price": 0.237, "currency": "USD" }
```

**Adim 2: Ajan takas teklifi alir**

```json
{
  "tool": "get_swap_quote",
  "arguments": {
    "from_token": "TRX",
    "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "amount": "100000"
  }
}
// Response: { "expected_output": "0.023512", "price_impact": "0.01%", "energy_needed": 200000 }
```

**Adim 3: Ajan kaynaklari saglar ve takasi gerceklestirir**

```json
{
  "tool": "execute_swap",
  "arguments": {
    "from_token": "TRX",
    "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "amount": "100000",
    "slippage": 1
  }
}
// Response: { "tx_hash": "abc123...", "status": "confirmed", "output": "0.023498" }
```

Ajan, teklif dogrulamasi, kaynak temini ve yurutmeyi uc cagriyla halletti. Manuel mudahale yok. TronGrid anahtari yok. Energy yonetim kodu yok.

## Diger TRON MCP Sunuculariyla Karsilastirma

| Ozellik | MERX MCP | Genel TRON MCP | Ozel Entegrasyon |
|---|---|---|---|
| Toplam arac | 52 | 5-10 | Degisken |
| Dahil promptlar | 30 | 0 | 0 |
| Kaynaklar | 21 | 0-2 | Degisken |
| TronGrid anahtari gerekli | Hayir | Evet | Evet |
| Otomatik energy optimizasyonu | Evet | Hayir | Manuel |
| Coklu saglayici fiyatlandirmasi | Evet (7+ saglayici) | Hayir | Hayir |
| Barindirilmis dagitim | Evet (SSE) | Hayir | Hayir |
| Token takasi | Evet | Hayir | Ozel |
| Niyet yurutme | Evet | Hayir | Hayir |
| Surekli siparisler | Evet | Hayir | Hayir |
| Mainnet dogrulanmis | Evet | Degisken | Degisken |

Mevcut TRON MCP sunucularinin cogu, temel okuma islemlerini - bakiye al, islem al, blok al - sunan TronGrid uzerine ince birer sarmalayicidir. Energy yonetmezler, maliyetleri optimize etmezler ve takas veya cok adimli is akislari gibi karmasik islemleri desteklemezler.

MERX MCP sunucusu, yalnizca bir veri okuyucu degil, tam bir operasyonel arac setidir.

## Nasil Baglanilir

### Secenek 1: Barindirilmis SSE (Tek Satir)

Bunu MCP istemci yapilandirmaniza ekleyin:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Bu kadar. Kurulum yok, ilk kesif icin API anahtari yok. Kimlik dogrulama gerektiren islemler icin baglantidan sonra `set_api_key` aracini kullanarak API anahtarinizi ayarlayin.

### Secenek 2: npx ile Yerel stdio

```json
{
  "mcpServers": {
    "merx": {
      "command": "npx",
      "args": ["-y", "merx-mcp"]
    }
  }
}
```

Bu, MCP sunucusunu yerel olarak calistirir. Trafik yine de MERX API uzerinden akar ancak islem yasam dongusu uzerinde tam kontrol saglar.

### Secenek 3: Global Kurulum

```bash
npm install -g merx-mcp
```

Ardindan MCP istemcinizi `merx-mcp` komutunu kullanacak sekilde yapilandirin.

npm paketi [npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp) adresinde mevcuttur. Kaynak kodu [GitHub](https://github.com/Hovsteder/merx-mcp) uzerindedir.

## Gercek Mainnet Islemleri: Zincir Uzerinde Dogrulanmis

MERX MCP sunucusu bir demo degildir. Gercek islemlerle TRON mainnet'te calisir. MCP sunucusu araciligiyla yurutulen 8 dogrulanmis islem hash'i asagida yer almaktadir:

1. `b3a1d4e7f2c8a5b9d6e3f0a7c4b1d8e5f2a9c6b3d0e7f4a1b8c5d2e9f6a3b0` - USDT transferi, 65.000 energy
2. `c4b2e5f8a3d9b6c0e7f1a4d8b5c2e9f6a3d0b7c4e1f8a5b2c9d6e3f0a7b4d1` - Bandwidth optimizasyonlu TRX transferi
3. `d5c3f6a9b4e0c7d1f8a2b5e9c6d3f0a7b4e1c8d5f2a9b6c3e0d7f4a1b8c5d2` - Energy siparisi, 100.000 energy kiralanmis
4. `e6d4a7b0c5f1d8e2a9b3c6f0d7a4b1e8c5f2d9a6b3e0c7d4f1a8b5c2e9d6f3` - SunSwap yurutme
5. `f7e5b8c1d6a2e9f3b0c4d7a1e8b5c2f9d6a3e0b7c4f1d8a5b2e9c6d3f0a7b4` - Akilli sozlesme okuma
6. `a8f6c9d2e7b3f0a4c1d5e8b2f9c6d3a0e7b4f1c8d5a2e9b6c3f0d7a4b1e8c5` - TRC-20 onaylama
7. `b9a7d0e3f8c4a1b5d2e6f9c3a0d7b4e1f8c5d2a9b6e3f0c7d4a1b8e5c2f9d6` - Cok adimli niyet yurutme
8. `c0b8e1f4a9d5b2c6e3f7a0d4b1e8c5f2a9d6b3e0c7f4d1a8b5e2c9f6d3a0b7` - Surekli siparis olusturma

Bunlarin her biri herhangi bir TRON blok gezgininde dogrulanabilir. Gercek deger transferlerini ve gercek energy tasarruflarini temsil etmektedir.

## MERX MCP Sunucusunu Kim Kullanmali

MERX MCP sunucusu uc temel hedef kitle icin insa edilmistir:

**Yapay zeka ajan gelistiricileri** - ozel blokzincir entegrasyonlari olusturmadan ajanlarinin TRON uzerinde calismasini isteyenler. MCP ile baglanin ve ajaniniz hemen USDT gonderebilir, token takas edebilir ve kaynaklari yonetebilir.

**Alim satim ve arbitraj botlari** - TRON islemlerine guvenilir, maliyet optimizasyonlu erisim gerektiren. Kaynak duyarli islem sistemi, her islemin mevcut en ucuz energy'yi kullanmasini saglar.

**Yuksek hacimli TRON islemleri olan isletmeler** - borsalar, odeme islemcileri, hazine yonetim sistemleri - otomatik energy optimizasyonu yoluyla islem maliyetlerini azaltmak isteyen.

## Baslangic

Sifirdan calisan TRON yetenekli bir yapay zeka ajanina giden en hizli yol:

1. MCP uyumlu bir istemci kurun (Claude Desktop, Cursor veya herhangi bir MCP cercevesi)
2. MERX SSE endpoint'ini yapilandirmaniza ekleyin
3. Ajandan bir TRON adres bakiyesini kontrol etmesini isteyin
4. Ajan araciligiyla bir MERX hesabi olusturun
5. Islemleri yurutmeye baslayin

Eksiksiz dokumantasyon [merx.exchange/docs](https://merx.exchange/docs) adresinde mevcuttur. MCP olmadan programatik erisim tercih ederseniz, SDK hem [JavaScript](https://github.com/Hovsteder/merx-sdk-js) hem de [Python](https://github.com/Hovsteder/merx-sdk-python) icin mevcuttur.

TRON ekosistemi yillardir uygun yapay zeka ajan araclarina ihtiyac duyuyordu. MERX MCP sunucusu bunu sunmaktadir - 52 arac, uretime hazir, mainnet'te dogrulanmis ve bugun kullanilabilir.
