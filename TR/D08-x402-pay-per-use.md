# x402 Ödeme Kullanımı: Hesap Olmadan TRON Energy Satın Alın

## Kayıt Problemi

Her blockchain hizmeti aynı deseni izler: hesap oluştur, e-postanı doğrula, API anahtarı oluştur, iç bakiyeyi finanse et, sonra hizmeti kullanmaya başla. İnsan kullanıcı için bu küçük bir rahatsızlıktır. Otonom bir yapay zeka ajanı için ise bu tam bir engeldir.

Ajanların e-posta adresleri yoktur. Üçüncü taraflar tarafından yönetilen iç bakiyeleri istemezler. Bir platformaya fonlarının saklama görevini vermek istemezler. Ajanların istedikleri basittir: bir hizmet için öde, hizmeti al, devam et. İlişki yok. Durum yok. Güven yok.

x402 protokolü bunu mümkün kılar. 1997'de tanımlanan ancak hiçbir zaman yaygın olarak uygulanmayan HTTP 402 "Ödeme Gerekli" durum kodundan ilham alan x402, ödemenin hesaplar ve API anahtarları yerine blokzincir üzerinde doğrulandığı kullanım başına ödeme ticaretini sağlar.

MERX, enerji satın alımları için x402'yi uygular. TRON cüzdanına sahip herhangi bir varlık (insan, ajan veya akıllı sözleşme), hesap oluşturmadan, fon yatırmadan ve MERX platformu ile herhangi bir önceki ilişki olmadan tek bir işlemde enerji satın alabilir.

## Nasıl Çalışır?

x402 akışının beş adımı vardır. Her adım blokzincir üzerinde doğrulanabilir ve alıcı hiçbir noktada MERX'e fonlarının saklama görevini vermek zorunda değildir.

### Adım 1: Fatura İste

Alıcı, istenen enerji parametreleriyle `create_paid_order` çağrısı yapar:

```
Tool: create_paid_order
Input: {
  "energy_amount": 65000,
  "duration_hours": 1,
  "target_address": "TBuyerAddress..."
}

Response:
{
  "invoice": {
    "order_id": "xpay_abc123",
    "amount_trx": 1.43,
    "amount_sun": 1430000,
    "pay_to": "TMerxTreasuryAddress...",
    "memo": "merx_xpay_abc123",
    "expires_at": "2026-03-30T12:05:00Z",
    "energy_amount": 65000,
    "duration_hours": 1,
    "target_address": "TBuyerAddress..."
  }
}
```

Fatura bir teklif olup, bir taahhüt değildir. Hiçbir fon hareket etmemiştir. Alıcı fiyatı inceleyebilir, alternatifleriyle karşılaştırabilir ve devam edip etmemesine karar verebilir. Fatura 5 dakika sonra sona erer - bu süre içinde ödenmezse, alıntı fiyat artık garantili değildir.

### Adım 2: Ödemeyi Yerel Olarak İmzala

Alıcı, cüzdanından MERX hazine adresine bir TRX transfer işlemi oluşturur. Kritik detay, faturadan alınan tam dizeyi içermesi gereken memo alanıdır.

```javascript
const tx = await tronWeb.transactionBuilder.sendTrx(
  invoice.pay_to,        // TMerxTreasuryAddress
  invoice.amount_sun,    // 1430000 (1.43 TRX in SUN)
  buyerAddress
);

// İşlem verilerine memo ekle
const txWithMemo = await tronWeb.transactionBuilder.addUpdateData(
  tx,
  invoice.memo,          // "merx_xpay_abc123"
  'utf8'
);

// Yerel olarak imzala - özel anahtar asla alıcının makinesinden ayrılmaz
const signedTx = await tronWeb.trx.sign(txWithMemo);
```

Ödeme alıcının özel anahtarıyla alıcının makinesinde imzalanır. Özel anahtar hiçbir zaman MERX ile paylaşılmaz, ağ üzerinden asla iletilmez ve alıcının kontrolü dışında hiçbir yerde depolanmaz.

### Adım 3: Ödemeyi Yayınla

İmzalı işlem TRON ağına yayınlanır:

```javascript
const result = await tronWeb.trx.sendRawTransaction(signedTx);
const txHash = result.txid;
```

Bu noktada, ödeme blokzincir üzerindedir. TRON blokzincirini sorgulayan herkese görünürdür. TRX, alıcının adresinden MERX hazine adresine taşınmış ve memo alanı fatura tanımlayıcısını içerir.

### Adım 4: MERX Blokzincir Üzerinde Doğrula

MERX, hazine adresini gelen işlemler için izler. İşlem geldiğinde, MERX:

1. Memo alanını okur
2. Onu olağanüstü faturaların karşısında eşleştirir
3. Miktarın fatura ile eşleştiğini doğrular (tam miktar gerekli)
4. Faturanın sona ermediğini doğrular
5. Faturanın daha önce ödenmediğini doğrular (çift talep etmeyi önler)

```
Doğrulama:
  TX hash: 7f3a2b...
  From: TBuyerAddress...
  To: TMerxTreasuryAddress...
  Amount: 1,430,000 SUN (1.43 TRX) - UYUŞUYOR
  Memo: "merx_xpay_abc123" - UYUŞUYOR
  Invoice status: ÖDENMEMIŞ - TAMAM
  Invoice expiry: 2026-03-30T12:05:00Z - SÜRESI BİTMEMİŞ

  Result: DOĞRULAND
```

### Adım 5: Enerji Delegasyonu

Ödeme doğrulandıktan sonra, MERX enerji siparişini sağlayıcı ağı üzerinden yerleştirir:

```
Sipariş yerleştirildi:
  Energy: 65,000
  Duration: 1 hour
  Target: TBuyerAddress...
  Provider: sohu (sipariş zamanında en iyi fiyat)

4.1 saniye sonra delegasyon onaylandı
Energy available at TBuyerAddress: 65,000
```

Alıcı artık adreslerine 1 saatliğine 65.000 enerji delegasyonuna sahiptir. Bunu USDT transferleri, DEX takasları veya başka herhangi bir akıllı sözleşme etkileşimi için kullanabilir.

## Tam Ajan Akışı

Yapay zeka ajanının MERX MCP sunucusu aracılığıyla tüm x402 akışını nasıl yürüttüğü aşağıda verilmiştir:

```
Agent: "MERX hesabım yok, 100 USDT göndermem gerekiyor TRecipient adresine."

Step 1: Agent create_paid_order çağrısı yapar
  -> Alır: 1.43 TRX fatura, memo "merx_xpay_abc123"

Step 2: Agent transfer_trx çağrısı yapar
  -> TMerxTreasury adresine "merx_xpay_abc123" memo ile 1.43 TRX gönderir
  -> TX hash: 7f3a2b...

Step 3: Agent doğrulamayı bekler (otomatik)
  -> MERX ödemeyi algılar, blokzincir üzerinde doğrular
  -> Enerji ajan adresine delegasyona yerleştirilir

Step 4: Agent transfer_trc20 çağrısı yapar
  -> TRecipient'e 100 USDT gönderir
  -> TRX yanmak yerine delegasyona alınmış enerji kullanır
  -> Cost: 1.43 TRX yerine ~27 TRX

MERX ile toplam etkileşim: 2 araç çağrısı
Hesap oluşturuldu: Hayır
API anahtarı kullanıldı: Hayır
Fon yatırıldı: Hayır
Gerekli güven: Minimal (ödeme blokzincir üzerinde doğrulandı)
```

## Güvenlik: Memo Neden Önemlidir

Memo alanı x402 güvenliğinin anahtarıdır. Onsuz, MERX hazinesine yapılan herhangi bir TRX transferi herhangi bir fatura için ödeme olarak talep edilebilir. Bu iki saldırı vektörü yaratır:

### Çapraz Ödeme Saldırısı

Memo doğrulaması olmadan, saldırgan şu işlemleri yapabilir:
1. 65.000 enerji faturası talep et (1.43 TRX)
2. Birisinin ilgisiz bir nedenle MERX hazinesine 1.43 TRX göndermesini bekle
3. O ilgisiz ödemeyi kendi fatura ödemesi olarak talep et
4. Bedava enerji al

Memo bunu önler. Her faturanın benzersiz bir memo dizesi vardır. Ödeme, yalnızca memo alanı tam dizeyi içeriyorsa bir fatura ile eşleştirilir. Hazine adresine yapılan rastgele TRX transferleri (herhangi bir aktif adrese gerçekleşir) geçerli bir memo eksik oldukları için yoksayılır.

### Replay Saldırısı

Geçerlilik süresi ve tek kullanımlı zorlama olmadan, saldırgan şu işlemleri yapabilir:
1. Faturayı meşru şekilde öde
2. Aynı ödeme işlem hash'ını ikinci bir fatura için referans göster
3. Bir ödeme için iki kez enerji al

MERX bunu iki mekanizmayla önler:
- Her fatura ilk başarılı doğrulamadan sonra ÖDENMEMIŞ olarak işaretlenir. Aynı memo talep eden ikinci bir ödeme reddedilir.
- Her ödeme işlemi yalnızca bir fatura ile eşleştirilir. İşlem hash'ı kaydedilir ve yeniden kullanılamaz.

### Miktar Doğrulaması

Ödeme miktarı fatura miktarıyla tam olarak eşleşmelidir. 1.42 TRX yerine 1.43 TRX göndermek doğrulamanın başarısız olmasıyla sonuçlanır. Bu, saldırganın minimum bir miktar (örn. 0.000001 TRX) gönderdiği ve faturayı fiyatının bir kısmında talep etmeye çalıştığı saldırıları önler.

## Gerçek Mainnet İşlemi

TRON mainnet'ten doğrulanmış bir x402 işlemi aşağıda verilmiştir:

```
Invoice:
  Order ID: xpay_m7k2p9
  Energy: 65,000
  Duration: 1 hour
  Price: 1.43 TRX
  Memo: merx_xpay_m7k2p9

Payment Transaction:
  Hash: 8d4f1a7b3c2e...
  Block: 58,234,891
  From: TWallet...
  To: TMerxTreasury...
  Amount: 1,430,000 SUN
  Memo: merx_xpay_m7k2p9
  Status: CONFIRMED

Verification:
  Timestamp: 2026-03-28T14:22:17Z
  Match: Amount OK, Memo OK, Not expired, Not previously paid
  Result: VERIFIED

Energy Delegation:
  Delegated: 65,000 energy
  Provider: catfee
  Delegation TX: 3a8b2c...
  Confirmed at: 2026-03-28T14:22:23Z
  Expires at: 2026-03-28T15:22:23Z
```

Fatura talebinden enerji delegasyonuna: 23 saniye. Hesap yok. API anahtarı yok. Önceden ilişki yok.

## x402 ve Hesap Tabanlı Erişim Ne Zaman Kullanılır

x402 ideal şu durumlar için:

- **Yapay zeka ajanları** otonom olarak çalışan ve yönetilen hesaplara bağlı olmayan ajanlar
- **Bir kez satın almalar** hesap oluşturmanın overhead'ı işlem değerini aştığı durumlar
- **Gizlilik bilinçli kullanıcılar** hesap oluşturmak veya tanımlayıcı bilgi sağlamak istemeyen kullanıcılar
- **Test ve değerlendirme** geliştiricilerin hesaba taahhüt etmeden önce hizmeti denemek istedikleri durumlar
- **Çapraz platform ajanları** birçok hizmetle etkileşime giren ve her biri için ayrı hesapları korulamayan ajanlar

Hesap tabanlı erişim şu durumlar için daha iyidir:

- **Yüksek hacimli işlemler** x402'nin işlem başına overhead'inin (fatura oluşturma + ödeme yayınlama + doğrulama) önemli olduğu durumlar
- **Duran emirler ve izleyiciler** hesapla bağlantılı kalıcı sunucu taraflı durum gerektiren durumlar
- **Bakiye tabanlı faturalandırma** önceden ödenmiş kredinin işlem başına ödemelerden daha verimli olduğu durumlar
- **Takım işlemleri** birden fazla kullanıcının rol tabanlı erişimle hesabı paylaştığı durumlar

Birçok kullanıcı değerlendirme için x402 ile başlar ve kullanımları arttıkça hesap tabanlı erişime geçer. İki model rakip değil, tamamlayıcıdır.

## x402 ve Ajan Ticaretinin Geleceği

x402 protokolü, hizmetlerin nasıl tüketildiğinin daha geniş bir değişimini temsil eder. Geleneksel SaaS faturalandırması - aylık abonelikler, katmanlı fiyatlandırma, kullanım sınırları - hesap oluşturan, fiyatlandırma katmanlarını değerlendiren ve satın alma kararı alan insan müşterisi varsayar. Müşteri gerçek zamanlı olarak, belirli görevler için otonom olarak satın alma kararları almması gereken bir yapay zeka ajanı olduğunda bu model bozulur.

x402, ajan ekonomisi ile hizalanır:

- **Kayıt yok** - Ajanların geleneksel anlamda kimlikleri yoktur. x402, yalnızca bir cüzdan adresi gerektirir.
- **Kullanım başına fiyatlandırma** - Ajanlar, ne zaman kullanırsanız kullanın, elde ettiğiniz şeyin fiyatını, görev elle orantılı miktarlarda ödemesi gerekir.
- **Blokzincir üzerinde doğrulama** - Güven, hesap kimlik bilgileri veya ticari ilişkiler yoluyla değil, kriptografik kanıt aracılığıyla kurulur.
- **Bileşkenlik** - Ajan, x402'yi kullanarak MERX'ten enerji için ödeme yapabilir, ardından bu enerjiyi herhangi bir TRON akıllı sözleşmesiyle etkileşime girmek için kullanabilir. Ödeme ve hizmet ayrıştırılmıştır.

MERX, x402'yi uygulayan ilk enerji değişimidir. Protokol olgunlaştıkça, diğer blockchain hizmetlerinin ajan tüketimine optimize edilmiş benzer kullanım başına ödeme modellerini benimsemeleri beklenir.

## Geliştiriciler için Uygulama Detayları

### x402 İstemcisi Oluşturma

MCP sunucusu olmadan MERX x402'yi kullanan bir uygulama oluşturuyorsanız, minimal uygulama aşağıdadır:

```javascript
const TronWeb = require('tronweb');

async function buyEnergyX402(tronWeb, energyAmount, durationHours, targetAddress) {
  // Step 1: Fatura al
  const invoiceRes = await fetch('https://merx.exchange/api/v1/x402/invoice', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      energy_amount: energyAmount,
      duration_hours: durationHours,
      target_address: targetAddress
    })
  });
  const invoice = await invoiceRes.json();

  // Step 2: Ödeme işlemi oluştur
  const tx = await tronWeb.transactionBuilder.sendTrx(
    invoice.pay_to,
    invoice.amount_sun,
    tronWeb.defaultAddress.base58
  );

  const txWithMemo = await tronWeb.transactionBuilder.addUpdateData(
    tx, invoice.memo, 'utf8'
  );

  // Step 3: İmzala ve yayınla
  const signedTx = await tronWeb.trx.sign(txWithMemo);
  const result = await tronWeb.trx.sendRawTransaction(signedTx);

  // Step 4: Sipariş tamamlanması için yokla
  let order;
  for (let i = 0; i < 20; i++) {
    await new Promise(r => setTimeout(r, 3000));
    const orderRes = await fetch(
      `https://merx.exchange/api/v1/x402/order/${invoice.order_id}`
    );
    order = await orderRes.json();
    if (order.status === 'completed') break;
  }

  return order;
}
```

### Hata Yönetimi

Yaygın hata modları ve çözümleri:

| Hata | Neden | Çözüm |
|---|---|---|
| `INVOICE_EXPIRED` | Ödeme 5 dakika içinde gönderilmedi | Yeni fatura talep et |
| `AMOUNT_MISMATCH` | Ödeme miktarı faturadan farklı | Tam miktarı gönder; fiyat değiştiyse yeni fatura talep et |
| `MEMO_NOT_FOUND` | Ödeme memo alanından yoksun | Doğru memo ile tekrar gönder; başarısız deneme fonları otomatik olarak geri alınmaz |
| `ALREADY_PAID` | Fatura zaten yerine getirildi | Sipariş durumunu kontrol et; bu yoklama yapıyorsanız bir hata değildir |
| `PROVIDER_UNAVAILABLE` | Sağlayıcı siparişi dolduramadı | Birkaç dakika sonra yeniden dene; fatura ödemesi manuel çekme için MERX bakiyesine itibar edilir |

### İade Politikası

MERX ödeme doğrulandıktan sonra siparişi yerine getiremezse (örneğin, tüm sağlayıcılar geçici olarak kullanılamaz), ödeme miktarı ödeyenin TRON adresiyle ilişkili talep edilebilir bir bakiye olarak kredilenir. Ödeyici, hesap oluşturmadan ayrı bir uç nokta aracılığıyla bu bakiyeyi talep edebilir - talep, orijinal ödemeyi gönderen aynı özel anahtar ile bir mesajı imzalayarak doğrulanır.

## Sonuç

HTTP 402 durum kodu, yerel internet ödemeleri vizyonuyla neredeyse 30 yıl önce tanımlanmıştır. Bu vizyon zamanından ilerideydi. Blokzincirler nihayet bunu gerçek kılmak için altyapıyı sağlıyor.

MERX x402, enerji satın almayı tek bir atomik akışa dönüştürür: talep, öde, al. Hesap yok. API anahtarı yok. Blokzincirin sağladığı şeyin ötesinde güven varsayımı yok.

Yapay zeka ajanları için bu doğal satın alma modelidir. Geliştiriciler için en basit entegrasyondur. Ekosistem için bu, ajan ekonomisinde blockchain hizmetlerinin nasıl tüketileceğinin bir kanıtıdır.

---

**Bağlantılar:**
- MERX Platform: [https://merx.exchange](https://merx.exchange)
- MCP Server (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP Server (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)


## Şimdi Yapay Zeka ile Deneyin

MERX'i Claude Desktop veya herhangi bir MCP uyumlu istemciye ekleyin - sıfır yükleme, salt okunur araçlar için API anahtarı gerekli değildir:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Yapay zeka ajanınıza sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)