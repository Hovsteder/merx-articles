# 30 Prompt ve 21 Kaynak: Neden MERX Tek Tam Protokol MCP Sunucusu

## MCP'nin Bir Değil Üç Primitifi Var

Model Context Protocol, bir sunucunun bir AI istemcisine açığa çıkarabileceği üç tür yeteneği tanımlar:

1. **Tools** - yürütülebilir eylemler (işlem gönder, enerji satın al, takas yap)
2. **Prompts** - modeli karmaşık iş akışlarında rehber yapan önceden oluşturulmuş şablonlar
3. **Resources** - modelin okuyabileceği yapılandırılmış veriler (fiyat beslemeleri, ağ parametreleri, belgeler)

Çoğu MCP sunucusu sadece tools uygular. Bir avuç çağrılabilir fonksiyonu açığa çıkarır, işi bittiği söyler ve kendilerini "MCP-uyumlu" olarak pazarlarlar. Bu, teknik olarak motoru olan ama direksiyonu olmayan bir arabaya sahip olmanın teknik olarak bir araç sayılması gibi doğrudur.

Tools tek başına modele eylemler verirler ama ne zaman ve nasıl kullanılacağı konusunda hiçbir rehberlik vermezler. Promptlar olmadan, model her seferinde karmaşık çok adımlı iş akışlarını sıfırdan anlamalıdır. Resources olmadan, modelin referans alacağı yapılandırılmış verisi yoktur - sadece erişilebilir olması gereken bilgileri okumak için tool çağrısı yapmak zorundadır.

MERX üç primitifi de uygular: 55 tool, 30 prompt ve 21 resource. Bu makale her primitifin ne yaptığını, üçünün neden önemli olduğunu ve MERX'in bunu tools-only uygulamalardan niteliksel olarak farklı bir MCP sunucusu oluşturmak için nasıl kullandığını açıklar.

## Primitive 1: Tools (Eylemler)

Tools en sezgisel primitiftir. Bir tool, modelin bir eylemi gerçekleştirmek için çağırabileceği bir fonksiyondur. Bir adı, açıklaması, yazılı parametreleri ve yapılandırılmış bir yanıtı vardır.

MERX, TRON blokzincirinin tüm işlem alanını kapsayan 55 tool açığa çıkarır:

### Cüzdan İşlemleri
- `set_private_key` - Cüzdanı özel anahtarla yapılandır (adresi otomatik türet)
- `get_trx_balance` - Herhangi bir adresin TRX bakiyesini kontrol et
- `get_trc20_balance` - TRC20 token bakiyesini kontrol et
- `transfer_trx` - TRX'i bir adrese gönder
- `transfer_trc20` - TRC20 tokenlarını bir adrese gönder
- `check_address_resources` - Enerji ve bandwidth tahsisini görüntüle

### Enerji Pazarı
- `get_prices` - Tüm sağlayıcılardan mevcut fiyatlar
- `get_best_price` - En ucuz enerji teklifini bul
- `create_order` - Pazardan enerji satın al
- `create_paid_order` - x402 aracılığıyla enerji satın al (hesap gerekmez)
- `ensure_resources` - Gerekli kaynakları otomatik satın al
- `list_orders` - Sipariş geçmişini görüntüle
- `get_order` - Belirli bir siparişin detaylarını al

### DEX Ticareti
- `get_swap_quote` - Tam simülasyonlu SunSwap V2 teklifi al
- `execute_swap` - Otomatik kaynak yönetimi ile takas yap
- `approve_trc20` - Token harcama onayı ayarla

### Otomasyon
- `create_standing_order` - Otomatik enerji satın alımları ayarla
- `list_standing_orders` - Aktif standing order'ları görüntüle
- `create_monitor` - Delegasyon veya bakiye monitörü kur
- `list_monitors` - Aktif monitörleri görüntüle

### İleri Seviye
- `execute_intent` - Çok adımlı işlem planlarını yürüt
- `estimate_contract_call` - Enerji tahmini için herhangi bir sözleşme çağrısını simüle et

Her tool'un parametreleri için JSON Schema tanımı, doğal dil açıklaması ve yapılandırılmış dönüş türleri vardır. Model her tool'un ne yaptığını, hangi parametrelere ihtiyaç duyduğunu ve ne döndüreceğini tam olarak bilir.

Ama tools tek başına yeterli değildir.

## Primitive 2: Prompts (Şablonlar)

Bir prompt, modelin belirli bir istek türünü ele almak için kullanabileceği önceden oluşturulmuş bir şablondur. Bunu bir reçete gibi düşün: "Kullanıcı X için sorduğunda, işte adım adım yaklaşım, hangi tool'ları çağrılacak, hangi sırada ve sonuçlar nasıl yorumlanacak."

MERX, 10 kategori arasında organize edilmiş 30 prompt sağlar.

### Başlarken (3 prompt)

**setup_wallet**
```
Şablon: Kullanıcıyı cüzdan yapılandırmasında yönlendir
Adımlar:
  1. Özel anahtar güvenliğini açıkla
  2. set_private_key'i çağır
  3. Doğrulamak için get_trx_balance'ı çağır
  4. Temel için check_address_resources'ı çağır
  5. Bakiyeye göre sonraki adımları öner
```

**first_energy_purchase**
```
Şablon: İlk kez enerji satın alanları rehberlik et
Adımlar:
  1. Enerji nedir ve neden önemli olduğunu açıkla
  2. Pazar genel görünümü göstermek için get_prices'ı çağır
  3. Miktar ve süreyi seçmede yardımcı ol
  4. create_order veya create_paid_order'ı çağır
  5. check_address_resources ile delegasyonu doğrula
```

**platform_overview**
```
Şablon: MERX yeteneklerini açıkla
İçerik: Her bir tool, prompt ve resource'un ne zaman kullanılacağını
  içeren tüm tool'ların yapılandırılmış genel görünümü
```

### Enerji Yönetimi (4 prompt)

**buy_cheapest_energy**
```
Şablon: En iyi fiyatta enerji bul ve satın al
Adımlar:
  1. Kullanıcının parametreleriyle get_best_price'ı çağır
  2. Sağlayıcılar arasında fiyat karşılaştırması göster
  3. Kullanıcı ile satın alma işlemini onayla
  4. create_order'ı çağır
  5. Delegasyon onayı için poll yap
```

**compare_providers**
```
Şablon: Detaylı sağlayıcı karşılaştırması
Adımlar:
  1. Tüm sağlayıcılar için get_prices'ı çağır
  2. Kullanıcının özel ihtiyaçları için maliyeti hesapla
  3. Karşılaştırma tablosunu sun
  4. Güvenilirlik ve hız metriklerini dahil et
  5. Akıl yürütme ile en iyi seçeneği öner
```

**optimize_costs**
```
Şablon: Harcamaları analiz et ve optimizasyon öner
Adımlar:
  1. Sipariş geçmişini inceleyebilirsiniz
  2. Fiyat modellerini analiz et
  3. Standing order'lar için fırsatları belirle
  4. Olası tasarrufu hesapla
  5. Optimizasyon planını sun
```

**ensure_resources_for_tx**
```
Şablon: Belirli bir işlem için kaynakları hazırla
Adımlar:
  1. Planlanan işlem için enerjiyi tahmin et
  2. Mevcut kaynakları kontrol et
  3. Açığı hesapla
  4. Gerekirse satın al
  5. Hazır olduğunu onayla
```

### İşlem Yürütme (4 prompt)

**send_usdt**
```
Şablon: Kaynak optimizasyonu ile tam USDT transferi
Adımlar:
  1. Alıcı adresini doğrula
  2. Enerjiyi tahmin et
  3. Kaynakları sağla
  4. Transferi yürüt
  5. Zincir üzerinde doğrula
```

**send_trx**
```
Şablon: TRX transferi (daha basit - enerji gerekmez)
Adımlar:
  1. Alıcı adresini doğrula
  2. Bandwidth'i kontrol et
  3. Transferi yürüt
  4. Zincir üzerinde doğrula
```

**swap_tokens**
```
Şablon: Tam pipeline ile DEX takas
Adımlar:
  1. Teklif al
  2. Fiyat etkisini ve kaymayı gözden geçir
  3. Onay gerekli mi kontrol et
  4. Onay + takas için kaynakları sağla
  5. Takas'ı yürüt
  6. Sonucu doğrula
```

**execute_complex_intent**
```
Şablon: Çok adımlı işlem planı
Adımlar:
  1. Kullanıcının adımları tanımlamasına yardımcı ol
  2. Tüm adımları simüle et
  3. Toplam maliyetleri gözden geçir
  4. Kaynak stratejisini seç
  5. Intent'i yürüt
  6. Sonuçları bildir
```

### Otomasyon (4 prompt)

**setup_standing_order**
```
Şablon: Otomatik enerji satın almalarını yapılandır
Adımlar:
  1. Kullanıcının ihtiyaçlarını anla (hacim, zaman, bütçe)
  2. Tetikleyici türü öner
  3. Uygun kısıtlamaları ayarla
  4. Standing order'ı oluştur
  5. İzleme ve yönetimi açıkla
```

**setup_delegation_monitor**
```
Şablon: Enerji delegasyonlarının otomatik yenilenmesini yapılandır
Adımlar:
  1. Mevcut delegasyonları inceleyebilirsiniz
  2. Yenileme penceresi ayarla
  3. Fiyat korumasını yapılandır
  4. Bildirim kanallarını ayarla
  5. Monitor'ı oluştur
```

**setup_balance_monitor**
```
Şablon: Bakiye tabanlı uyarılar ve eylemler yapılandır
Adımlar:
  1. Mevcut kaynak kullanım modellerini analiz et
  2. Eşik seviyelerini öner
  3. Eylem yapılandır (satın al, sağla veya bildir)
  4. Bütçe kısıtlamaları ayarla
  5. Monitor'ı oluştur
```

**review_automation**
```
Şablon: Mevcut otomasyonu denetim ve optimize et
Adımlar:
  1. Tüm standing order'ları ve monitörleri listele
  2. Yürütme geçmişini inceleyebilirsiniz
  3. Düşük performanslı kuralları belirle
  4. İyileştirmeler öner
  5. Onaylanırsa değişiklikleri uygula
```

### Analiz (4 prompt)

**analyze_prices**
```
Şablon: Pazar fiyat analizi
Adımlar:
  1. Tüm sağlayıcılardan mevcut fiyatları çek
  2. Fiyat geçmişini çek
  3. Eğilimleri ve modelleri belirle
  4. Optimal satın alma zamanlarını hesapla
  5. Grafikler ile analiz sunebilirsiniz
```

**calculate_savings**
```
Şablon: Delegeli enerji kullanımından yapılan tasarrufu hesapla
Adımlar:
  1. Kullanıcının işlem türleri için enerjiyi tahmin et
  2. TRX yanması ile maliyeti hesapla
  3. Enerji satın alma ile maliyeti hesapla
  4. Tasarruf yüzdesini göster
  5. Verilen hacimde yıllık tasarrufu projeksiyonu
```

**estimate_transaction_cost**
```
Şablon: Herhangi bir işlem türü için maliyet tahmini
Adımlar:
  1. İşlem türünü belirle
  2. Tam parametrelerle simüle et
  3. Enerji ve bandwidth gereksinimlerini göster
  4. Delegasyonu olan ve olmayan maliyeti göster
  5. Optimal yaklaşımı öner
```

**portfolio_overview**
```
Şablon: Tam hesap genel görünümü
Adımlar:
  1. Tüm bakiyeleri kontrol et (TRX, USDT, diğer tokenler)
  2. Kaynak tahsisini kontrol et
  3. Aktif delegasyonları gözden geçir
  4. Aktif otomasyon kurallarını özetleyebilirsiniz
  5. Mali genel görünümü sun
```

### Hesap Yönetimi (3 prompt)

**account_setup**, **deposit_guide**, **withdrawal_guide** - Hesap yaşam döngüsü işlemleri için adım adım şablonlar.

### x402 (2 prompt)

**x402_purchase** - Sıfır kayıt satın alma akışında rehberlik.
**x402_explain** - x402'nin nasıl çalıştığını ve ne zaman kullanılacağını açıkla.

### Ağ Bilgisi (2 prompt)

**explain_energy**, **explain_bandwidth** - Resources'deki verileri kullanarak TRON'ın kaynak modelini açıklayan eğitim şablonları.

### Sorun Giderme (2 prompt)

**transaction_failed**, **delegation_not_arrived** - Yaygın sorunları tanımlamaya ve çözmeye yardımcı olan tanı şablonları.

### Entegrasyon (2 prompt)

**sdk_quickstart**, **api_integration** - MERX'i uygulamalarına entegre eden geliştiriciler için şablonlar.

### Prompt'lar Neden Önemli

Prompt'lar olmadan, "bana enerji satın almada yardımcı ol" isteğini alan bir model şunları yapması gerekir:
1. Hangi tool'ların ilgili olduğunu bulabilirsebilirsiniz
2. İşlem sırasını belirle
3. Önce hangi bilgiyi toplayacağına karar ver
4. Bilmeyebileceği kenar durumlarını ele al

`buy_cheapest_energy` prompt'u ile, model kenar durumları ele alan, bilgiyi doğru biçimde sunan ve işlem sırasını takip eden test edilmiş, optimize edilmiş bir iş akışına sahiptir. Fark, birine tool vermek ile tool'lar artı bir kılavuz vermek arasındakidir.

## Primitive 3: Resources (Veriler)

Resources, modelin bir eylem çağrısı yapmadan okuyabileceği yapılandırılmış verilerdir. Tool'lardan farklı olarak, eylemi yaparlar, resource'lar sağlarlar - bilgiyi pasif olarak kullanılabilir hale getirirler.

MERX, 21 resource sağlar: 14 statik ve 7 dinamik şablon.

### Statik Resources (14)

Statik resource'lar oturumlar arasında değişmeyen sabit referans veriler sağlarlar:

```
merx://docs/getting-started
merx://docs/energy-explained
merx://docs/bandwidth-explained
merx://docs/resource-model
merx://docs/providers-overview
merx://docs/x402-protocol
merx://docs/standing-orders-guide
merx://docs/monitors-guide
merx://docs/intent-execution-guide
merx://docs/api-reference
merx://docs/sdk-js
merx://docs/sdk-python
merx://docs/faq
merx://docs/troubleshooting
```

Model `merx://docs/energy-explained`'i okuduğunda, TRON'ın enerji modelini açıklayan yapılandırılmış bir belge alır - enerji nedir, nasıl tüketilir, delegasyon nasıl çalışır ve TRX maliyetleriyle nasıl ilişkilidir. Bu bilgi herhangi bir API çağrısı veya tool çağrısı yapılmadan hemen kullanılabilir.

### Dinamik Şablon Resources (7)

Şablon resource'lar parametreleri kabul eder ve istege özgü veriler döndürür:

```
merx://prices/current
merx://prices/history/{period}
merx://providers/{provider_name}
merx://account/{address}/resources
merx://account/{address}/balances
merx://network/parameters
merx://network/chain-info
```

Örneğin, `merx://prices/current` tüm sağlayıcılardan mevcut enerji fiyatlarını yapılandırılmış biçimde döndürür:

```json
{
  "timestamp": "2026-03-30T12:00:00Z",
  "prices": [
    {
      "provider": "sohu",
      "energy_1h": 0.0000526,
      "energy_24h": 0.0000498,
      "available": true,
      "min_amount": 65000
    },
    {
      "provider": "catfee",
      "energy_1h": 0.0000540,
      "energy_24h": 0.0000510,
      "available": true,
      "min_amount": 65000
    }
  ]
}
```

Ve `merx://account/{address}/resources` herhangi bir adres için mevcut kaynak tahsisini döndürür:

```json
{
  "address": "TYourAddress...",
  "energy": {
    "available": 487231,
    "total": 500000,
    "used": 12769,
    "delegated_from": [
      {
        "amount": 500000,
        "provider": "sohu",
        "expires_at": "2026-03-31T02:00:00Z"
      }
    ]
  },
  "bandwidth": {
    "available": 1342,
    "total": 1500,
    "free_remaining": 1342
  }
}
```

### Resources Neden Önemli

Resources, modele tool çağrılarının ek yükü olmadan bağlam sağlar. Kullanıcı "enerji bakiyem nedir?" diye sorduğunda, model `merx://account/{address}/resources` resource'unu doğrudan kontrol edebilir. Model standing order'ları oluşturmadan önce bunlar hakkında referans alma ihtiyacı duyduğunda, `merx://docs/standing-orders-guide` okur.

Resources olmadan, her bilgi parçası bir tool çağrısı gerektirir. Model mevcut fiyatları görmek için `get_prices`'ı çağırması gerekir, hatta bu bilgi pasif olarak kullanılabilir olsa bile. Ayırım, her bilgi parçasını aktif olarak isteyen bir model (tools-only) ile başvuru için kullanabileceği zengin bir bilgi ortamına sahip bir model (tools + resources) arasındadır.

## Tam Protokol Avantajı

Üç primitive birlikte çalışırken, model temelde farklı bir seviyede çalışır:

### Senaryo: Kullanıcı "Bana enerji otomasyonu ayarlamada yardımcı ol" diyor

**Tools-only MCP sunucusu:**
1. Model hangi tool'ların otomasyon için var olduğunu tahmin eder
2. Tool'ları deneme yanılma sırasıyla çağırır
3. Önemli yapılandırma seçeneklerini kaçırabileceğiniz
4. En iyi uygulamalar konusunda rehberlik yok
5. Referans materyali yok

**Tam protokol MERX MCP sunucusu:**
1. Model referans materyali için `merx://docs/standing-orders-guide` okur
2. Model adım adım iş akışı için `setup_standing_order` prompt'unu etkinleştirir
3. Model fiyat eşikleri önerisinde bulunmak için `merx://prices/history/7d` okur
4. Model optimize edilmiş parametrelerle `create_standing_order` tool'unu çağırır
5. Model `merx://docs/monitors-guide`'i okur ve tamamlayıcı monitörleri önerir
6. Model delegasyon sona ermesi koruması için `create_monitor` tool'unu çağırır
7. Sonuç: Belgeleme destekli kararlarla tam, iyi yapılandırılmış otomasyon

Tam protokol yaklaşımı sadece marjinal olarak daha iyi değildir - model rehberlik (prompts), bilgi (resources) ve yetenek (tools) eşzamanlı olarak kullanabilir çünkü niteliksel olarak farklı sonuçlar üretir.

## Diğer MCP Sunucularının Eksik Oldukları

2026 başı itibariyle blokzincir ile ilgili MCP sunucularının incelemesi tutarlı bir kalıp gösterir:

| Sunucu | Tools | Prompts | Resources | Tam Protokol |
|---|---|---|---|---|
| Genel ETH MCP | 5-8 | 0 | 0 | Hayır |
| Solana MCP | 10-12 | 0 | 0 | Hayır |
| Bitcoin MCP | 3-4 | 0 | 0 | Hayır |
| Multi-chain MCP | 15-20 | 0 | 0 | Hayır |
| MERX | 52 | 30 | 21 | Evet |

Başka hiçbir blokzincir MCP sunucusu prompt'ları veya resource'ları uygulamaz. Hepsinin tools-only, bu da şu anlama gelir:

- Çok adımlı işlemler için iş akışı rehberliği yok
- Model çıkarsama zamanında belge yok
- Bağlam ve referans için pasif veri erişimi yok
- Her etkileşim yapısal bilgi olmadan sıfırdan başlar

MERX, modelin eksiksiz bir işletim ortamına - hareket etme yeteneğine (tools), ne zaman hareket edeceğini bilme bilgisine (prompts) ve kararları bilgilendirmek için verilere (resources) sahip tek MCP sunucusudur.

## Sayılar

MERX rakamlarla:

- **55 tool** 5 kategoride (cüzdan, enerji pazarı, DEX, otomasyon, ileri seviye)
- **30 prompt** 10 kategoride (başlarken, enerji, işlemler, otomasyon, analiz, hesaplar, x402, ağ, sorun giderme, entegrasyon)
- **21 resource** (14 statik belgelendirme, 7 dinamik veri şablonu)
- **72 toplam yetenek** tek bir MCP sunucusu aracılığıyla açığa çıkarılmış

MERX'e bağlı bir AI ajanı için bu şu anlama gelir:
- Herhangi bir TRON işlemini gerçekleştirebilir (tools)
- Herhangi bir senaryo için en iyi yaklaşımı bilir (prompts)
- Her zaman gerçek zamanlı veriler ve belgeler kullanılabilir (resources)

## Sonuç

MCP, bir değil üç primitif olan bir protokoldür. Yalnızca tool'ları uygulamak, uç noktaları olan ancak belgelendirmesi ve veri beslemesi olmayan bir API oluşturmak gibidir. Teknik olarak çalışır, ama her şeyi kendi başına anlaması gereken tüketicisini zorlar.

MERX tam protokolü uygular çünkü tam protokol MCP'nin kullanılmak üzere tasarlandığı şeklidir. Tools eylem için. Prompts rehberlik için. Resources bilgi için. Üçü birlikte çalışarak bir AI ajanının TRON blokzincirinde yıllarca deneyime sahip bir insan geliştirici kadar derinlikli yetenek ile çalışabilmesi için bir ortam oluşturur.

Otuz prompt. Yirmi bir resource. Yirmi bir tool. Başka hiçbir blokzincir MCP sunucusu buna yakın değildir.

---

**Bağlantılar:**
- MERX Platformu: [https://merx.exchange](https://merx.exchange)
- MCP Sunucusu (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP Sunucusu (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)


## Şimdi AI ile Dene

MERX'i Claude Desktop veya herhangi bir MCP-uyumlu istemciye ekle -- sıfır kurulum, salt okunur tool'lar için API anahtarı gerekli değildir:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI ajanına sor: "TRON enerjisinin en ucuz fiyatı nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatlar al.

Tam MCP belgelendirmesi: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)