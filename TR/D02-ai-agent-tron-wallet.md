# AI Agentenize TRON Cüzdanı Verin

## AI Agentelerin Neden On-Chain Erişime İhtiyacı Vardır

AI agenteleri hakkındaki konuşma sohbet botlarının ve kod asistanlarının ötesine geçmiştir. Sonraki sınır, blockchain ağlarında özerk şekilde hareket edebilen agenteldir - bakiyeler kontrol etme, işlem gönderme, kaynaklar satın alma ve her adımda insan müdahalesi olmaksızın portföyleri yönetme.

TRON, stablecoin transferlerinin temelini oluşturur. Günde 50 milyardan fazla USDT TRON üzerinde hareket eder ve ağın kaynak modeli - gaz ücretleri yerine energy ve bandwidth - bunu yüksek frekansı, düşük maliyetli işlemler için benzersiz şekilde uygun hale getirir. Ancak bir AI agentini TRON'a bağlamak tarihsel olarak özel entegrasyonlar oluşturmayı, RPC uç noktalarını yönetmeyi, kaynak tahminini ele almayı ve TronWeb'in karmaşıklıklarıyla uğraşmayı gerektirmiştir.

MERX bunu, herhangi bir uyumlu AI agent'e - Claude, Cursor veya herhangi bir Model Context Protocol istemcisine - tam özellikli bir TRON cüzdanı ve kaynak değişimini tek bir entegrasyon halinde veren bir MCP sunucusu ile çözer.

Bu makale, sıfırdan TRON mainnet'te anahtarlar tutan, bakiyeleri kontrol eden ve energy satın alan özerk bir agente kadar kurulumu ayrıntılı olarak açıklamaktadır.

## MCP Nedir ve Neden Önemlidir

Model Context Protocol (MCP), Anthropic tarafından oluşturulan açık bir standart olup, AI modellerinin dış araçlar, veri kaynakları ve hizmetlerle birleşik bir arayüz aracılığıyla etkileşim kurmasını sağlar. Bunu AI için bir USB portu olarak düşünün - herhangi bir MCP uyumlu istemci, özel entegrasyon kodu olmaksızın herhangi bir MCP sunucusuna bağlanabilir.

Bir MCP sunucusu üç temel işlemi ortaya çıkarır:

- **Tools** - agentenin gerçekleştirebileceği eylemler (TRX gönderme, energy satın alma, swap yürütme)
- **Prompts** - agenteyi karmaşık iş akışlarında rehberlik eden önceden oluşturulmuş şablonlar
- **Resources** - agentenin okuyabileceği yapılandırılmış veriler (fiyat beslemeleri, ağ parametreleri, hesap durumu)

MERX, 21 tool, 30 prompt ve 21 resource ile her üç temel işlemi uygular. Başka hiçbir blockchain MCP sunucusu bu düzeyde kapsama alanı sunmaz.

## Kurulum

MERX MCP sunucusu standart bir npm paketi olarak yayınlanmıştır. Bunu genel olarak yükleyin veya doğrudan çalıştırmak için npx kullanın:

```bash
npm install -g merx-mcp
```

Veya MCP istemci konfigürasyonunuza ekleyin. Claude Desktop için `claude_desktop_config.json` dosyasını düzenleyin:

```json
{
  "mcpServers": {
    "merx": {
      "command": "npx",
      "args": ["-y", "merx-mcp"],
      "env": {
        "MERX_NETWORK": "mainnet"
      }
    }
  }
}
```

Cursor için, konfigürasyon proje dizininizdeki `.cursor/mcp.json` dosyasına gider:

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

Bu, tüm kurulumdur. API anahtarı yok. Kayıt yok. OAuth akışları yok.

## Cüzdanı Kurma

Agentenizin ilk ihtiyacı bir TRON adresidir. MERX, hex kodlu özel bir anahtar kabul eden ve karşılık gelen TRON adresini otomatik olarak türeten `set_private_key` aracını sağlar.

```
Tool: set_private_key
Input: { "private_key": "your_64_char_hex_private_key" }

Response:
{
  "address": "TYourDerivedTronAddress...",
  "network": "mainnet",
  "status": "ready"
}
```

Özel anahtar hiçbir zaman yerel makineden ayrılmaz. Oturum süresi boyunca MCP sunucusunun çalışma zamanı belleğinde depolanır ve yayınlamadan önce işlemleri yerel olarak imzalamak için özel olarak kullanılır. MERX sunucuları asla özel anahtarınızı görmez - tüm imzalama istemci tarafında gerçekleşir.

### Yeni Bir Cüzdan Oluşturma

Agenteniz için yeni bir cüzdana ihtiyacınız varsa, herhangi bir TRON uyumlu aracı kullanarak bir tane oluşturabilirsiniz. Agent'in kendisi standart kriptografik kütüphaneleri kullanarak bir tane oluşturabilir:

```javascript
const TronWeb = require('tronweb');
const account = TronWeb.utils.accounts.generateAccount();
console.log('Address:', account.address.base58);
console.log('Private Key:', account.privateKey);
```

Adresi aktivasyon ve temel işlemler için küçük bir miktar TRX ile finanse edin, ardından özel anahtarı `set_private_key`'e geçirin.

## Temel Cüzdan İşlemleri

Cüzdan yapılandırıldıktan sonra, agent tam bir on-chain işlem paketine erişim sağlar.

### Bakiye Kontrol Etme

```
Tool: get_trx_balance
Input: { "address": "TYourAddress..." }

Response:
{
  "balance": 1523.456789,
  "balance_sun": 1523456789
}
```

TRC-20 tokenları için:

```
Tool: get_trc20_balance
Input: {
  "address": "TYourAddress...",
  "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t"
}

Response:
{
  "balance": "2500.00",
  "token": "USDT",
  "decimals": 6
}
```

### TRX Gönderme

```
Tool: transfer_trx
Input: {
  "to": "TRecipientAddress...",
  "amount_trx": 100
}
```

Agent işlemi yerel olarak imzalar ve TRON ağına yayınlar. Araç, on-chain doğrulaması için işlem hash'ini döndürür.

### TRC-20 Tokenleri Gönderme

```
Tool: transfer_trc20
Input: {
  "to": "TRecipientAddress...",
  "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "amount": "500"
}
```

TRC-20 transferleri energy tüketir. MERX burada parlak olur - agent, transferi yürütmeden önce energy'yi otomatik olarak tahmin edebilir, satın alabilir ve devredebilir; maliyeti yaklaşık 27 TRX (yakılmış) seviyesinden 4 TRX'in altına (devredilmiş energy) indirir.

## Energy Satın Alma - Temel Değer Önerisi

TRON'un kaynak modeli, her akıllı sözleşme etkileşiminin energy maliyeti olduğu anlamına gelir. Basit bir USDT transferi yaklaşık 65.000 energy gerektirir. SunSwap işlemi 200.000'den fazla energy tüketebilir. Devredilmiş energy olmadan, bu maliyetler mevcut ağ oranlarında TRX yakılarak ödenir.

MERX, birden fazla sağlayıcıdan energy toplayıp agent'e bunu satın almak için tek bir araç verir:

```
Tool: create_order
Input: {
  "energy_amount": 65000,
  "duration_hours": 1,
  "target_address": "TYourAddress..."
}
```

Agent, satın almadan önce mevcut pazar fiyatlarını da kontrol edebilir:

```
Tool: get_best_price
Input: {
  "energy_amount": 65000,
  "duration_hours": 1
}

Response:
{
  "best_price": 3.42,
  "provider": "sohu",
  "price_per_unit": 0.0000526
}
```

### ensure_resources Aracı

Mekaniklerin yerini niyet almak isteyen agenteleri için, `ensure_resources` her şeyi ele alan üst düzey araçtır:

```
Tool: ensure_resources
Input: {
  "address": "TYourAddress...",
  "energy_needed": 65000,
  "bandwidth_needed": 350
}
```

Bu araç, adresinde halihazırda nelerin olduğunu kontrol eder, eksik olanları hesaplar, yalnızca gerekenini satın alır ve geri dönmeden önce devredilmenin gelmesini bekler. Agent, energy pazarlarını, sağlayıcı API'lerini veya devredilme mekaniklerini anlamak zorunda değildir.

## x402 ile Kayıt Gerektirmeyen Yol

MERX, hiçbir hesap oluşturmayı gerektirmeyen bir yol sunar. x402 protokolü, ödemenin on-chain doğrulandığı kullanım başına ödeme enerjisi satışlarını sağlar.

Akış şu şekilde çalışır:

1. Agent, bir fatura almak için `create_paid_order` çağırır
2. Fatura, tam bir TRX tutarı ve bir memo dizesi belirtir
3. Agent, yerel cüzdanını kullanarak ödeme işlemini imzalar ve yayınlar
4. MERX, memo'yu korelasyon anahtarı olarak kullanarak ödemeyi on-chain doğrular
5. Energy hedef adrese devredilir

```
Tool: create_paid_order
Input: {
  "energy_amount": 65000,
  "duration_hours": 1,
  "target_address": "TYourAddress..."
}

Response:
{
  "invoice": {
    "amount_trx": 1.43,
    "pay_to": "TMerxTreasuryAddress...",
    "memo": "merx_ord_abc123",
    "expires_at": "2026-03-30T12:05:00Z"
  }
}
```

Agent daha sonra belirtilen memo ile TRX gönderir ve sipariş yerine getirilir. API anahtarı yok. E-posta yok. Kayıt formu yok. Özerk bir agent ile bir hizmet arasında saf on-chain ticaret.

## Gerçek Özerk Agent Akışı

İşte MERX'e bağlandığında özerk bir agent oturumunun nasıl görüneceğine dair tam bir örnek:

**Adım 1: Agent cüzdanını yapılandırır**

Agent, özel anahtarını güvenli bir ortam değişkeninden yükler ve `set_private_key` çağırır. MERX, TRON adresini türetir ve cüzdanın hazır olduğunu doğrular.

**Adım 2: Agent finansal pozisyonunu kontrol eder**

Agent, `get_trx_balance` ve USDT için `get_trc20_balance` çağırır. Artık 500 TRX ve 2.000 USDT'ye sahip olduğunu bilir.

**Adım 3: Agent bir görev alır - "Bu adrese 100 USDT gönder"**

Agent, mevcut energy ve bandwidth'i görmek için `check_address_resources` çağırır. 0 energy devredildiğini bulur.

**Adım 4: Agent maliyeti tahmin eder**

Agent, bir USDT transferi için `estimate_transaction_cost` çağırır. Yanıt, 65.000 energy gerekli olduğunu gösterir. Energy olmadan, bu yaklaşık 27 TRX yakmak olur. Energy satın alımıyla, maliyet yaklaşık 3,5 TRX'tir.

**Adım 5: Agent energy satın alır**

Agent, energy gereksinimi ile `ensure_resources` çağırır. MERX, en ucuz sağlayıcıyı bulur, siparişi yerleştirir ve devredilmenin gelmesini bekler.

**Adım 6: Agent transferi yürütür**

Devredilmiş energy ile agent, 100 USDT göndermek için `transfer_trc20` çağırır. İşlem, TRX yakma yerine devredilmiş energy'yi tüketir.

**Adım 7: Agent sonucu doğrular**

Agent, başarıyı doğrulamak için işlem hash'i ile `get_transaction` çağırır.

Toplam maliyet: insan müdahalesi olmaksızın 27 TRX yerine yaklaşık 3,5 TRX. Agent ücretlerde %87 tasarruf sağladı.

## Güvenlik Hususları

### Özel Anahtar Yönetimi

Özel anahtar yalnızca MCP sunucusunun çalışma zamanı belleğinde bulunur. MERX sunucularına hiçbir zaman aktarılmaz, MCP sunucusu tarafından hiçbir zaman diske yazılmaz ve API çağrılarına hiçbir zaman dahil edilmez. Tüm işlem imzalaması yerel olarak gerçekleşir.

Üretim dağıtımları için, özel anahtarı altyapınızın gizli yönetim sisteminde depolayın (AWS Secrets Manager, HashiCorp Vault veya güvenli bir çalışma zamanında ortam değişkenleri) ve MCP sunucusuna başlangıçta geçirin.

### İşlem Limitleri

Özerk agenteleri için koruma görevleri uygulamayı düşünün:

- Agenteniz mantığında maksimum işlem tutarını belirleyin
- Tekrar eden satınalmalar için bütçe limitli kalıcı siparişleri kullanın
- Agenteniz cüzdan bakiyesini izleyin ve belirli bir eşiğin altına düşerse uyarı verin
- Ana hazinenizdense sınırlı fonlar içeren özel bir cüzdan kullanın

### Ağ Seçimi

MERX hem mainnet hem de Shasta testnet'i destekler. Yeni agent iş akışlarını her zaman Shasta'da önce test edin:

```json
{
  "env": {
    "MERX_NETWORK": "shasta"
  }
}
```

Tam akışı doğruladıktan sonra mainnet'e geçin.

## Temel Cüzdan İşlemlerinin Ötesinde

Agenteniz bir cüzdana sahip olduktan sonra, MERX MCP sunucusu basit transferlerin çok ötesine uzanan yetenekler ortaya çıkarır:

- **DEX trading** SunSwap aracılığıyla tam energy simülasyonuyla
- **Kalıcı siparişler** fiyatlar belirli bir eşiğin altına düştüğünde otomatik olarak energy satın alıyor
- **Devredilme monitörleri** energy'nin süresi dolmadan önce otomatik olarak yenileniyordu
- **Çok adımlı niyetler** optimize edilmiş kaynak satın alımı ile birden fazla işlemi toplu olarak işleme tabi tutma
- **Fiyat analizi** en iyi anlaşmaları bulmak için tüm energy sağlayıcıları arasında

Bu yeteneklerin her biri agentenin çağırabileceği bir araç olarak ortaya çıkarılır; agenteyi karmaşık iş akışlarında rehberlik eden prompt'lar ve gerçek zamanlı pazar verileri sağlayan kaynaklar.

## Bugün Başlayın

Sıfırdan çalışan bir agent cüzdanına kadar en hızlı yol:

1. MCP sunucusunu yükleyin: `npm install -g merx-mcp`
2. MCP istemcinizi yapılandırın (Claude Desktop, Cursor veya herhangi bir MCP uyumlu araç)
3. Bir TRON özel anahtarı oluşturun veya içeri aktarın
4. Cüzdanı etkinleştirmek için `set_private_key` çağırın
5. Bağlantıyı doğrulamak için `get_trx_balance` çağırın

Tüm kurulum beş dakikadan az sürer. Kayıt yok, API anahtarları yok, onay süreci yok.

MERX, AI agenteleri ile TRON blockchain'i arasındaki köprüdür. MCP sunucusu açık kaynaklıdır, protokol standartlaştırılmıştır ve energy pazarı canlıdır.

Agenteniz bir cüzdana hazırdır. Ona bir tane verin.

---

**Bağlantılar:**
- MERX Platform: [https://merx.exchange](https://merx.exchange)
- MCP Server (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP Server (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)


## Şimdi AI ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin - kurulum yok, salt okunur araçlar için API anahtarı gerekmez:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI agentenize sorun: "Şu anda en ucuz TRON energy nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgelendirmesi: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)