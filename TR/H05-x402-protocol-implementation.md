# x402 Protokolü Uygulaması: Fatura, Ödeme, Doğrulama

HTTP durum kodu 402 -- "Ödeme Gerekli" -- orijinal HTTP/1.1 spesifikasyonunda 1997'de tanımlandı. Spesifikasyon bunu "gelecekteki kullanım için ayrılmış" olarak işaretledi. Yirmi dokuz yıl sonra, gelecek geldi.

x402 protokolü bu ayrılmış durum kodunu gerçek bir ödeme mekanizmasına dönüştürür. Ödemenin hesaplar, API anahtarları veya kredi kartları yerine blok zincir üzerinde doğrulandığı kullanım başına ödeme ticaretini mümkün kılar. Bir blok zincir cüzdanına sahip herhangi bir varlık -- insan, bot veya otonom ajan -- hesap oluşturmadan veya hizmet sağlayıcısıyla önceden herhangi bir ilişki kurmadan tek bir işlemde bir hizmet için ödeme yapabilir.

MERX, x402'yi TRON energy satın alışları için uygular. Bu makale uygulamaya ilişkin tam bir teknik rehberdir: fatura oluşturma, memo ile ödeme, TronGrid doğrulaması (hex ve base58 adres eşleştirme sorununu da içeriyor), x402 sistem kullanıcısı, bakiye kredisi ve sipariş yürütümü. Her adımı, her sınır durumunu ve her güvenlik düşünmesini ele alıyoruz.

## x402 Akış Genel Bakış

Protokolün beş adımı vardır:

```
1. INVOICE   - Alıcı bir teklif talep eder, sunucu ödeme talimatlarını döndürür
2. PAY       - Alıcı fatura kimliği içeren bir memo ile TRX gönderir
3. VERIFY    - Sunucu blok zincir ödemeyi tespit eder ve doğrular
4. CREDIT    - Sunucu x402 sistem hesabını krediye çeker ve defter girişleri oluşturur
5. EXECUTE   - Sunucu energy siparişini yürütür ve alıcıya devreder
```

Her adım güvensiz (trustless) olacak şekilde tasarlanmıştır. Alıcı hiçbir zaman doğrulanmamış bir adrese fon göndermez. Sunucu hiçbir zaman onaylanmış ödeme olmadan sipariş yürütmez. Memo alanı ödemeyi belirli bir faturaya bağlayarak çapraz ödeme saldırılarını önler.

## Adım 1: Fatura Oluşturma

Alıcı istenen energy parametreleri ile `create_paid_order` uç noktasını çağırır:

```
POST /api/v1/orders/paid
Content-Type: application/json

{
  "energy_amount": 65000,
  "duration_hours": 1,
  "target_address": "TBuyerAddress..."
}
```

Sunucu, mevcut tüm sağlayıcılar arasındaki en iyi fiyatlara dayalı maliyeti hesaplar ve bir fatura döndürür:

```json
{
  "invoice": {
    "order_id": "xpay_a7f3c2d1",
    "amount_trx": 1.43,
    "amount_sun": 1430000,
    "pay_to": "TMerxTreasuryAddress...",
    "memo": "merx_xpay_a7f3c2d1",
    "expires_at": "2026-03-30T12:05:00Z",
    "energy_amount": 65000,
    "duration_hours": 1,
    "target_address": "TBuyerAddress..."
  }
}
```

### Fatura Tasarım Kararları

**5 dakikalık son kullanma tarihi.** Fatura oluşturmadan 5 dakika sonra sona erer. Bu pencere, alıcının ödemeyi incelemesi, imzalaması ve yayınlaması için yeterince uzundur. Eski fiyat istismarını önlemek için yeterince kısadır -- energy fiyatları önemli ölçüde değişirse, alıcı geçerli fiyattan yeni bir fatura talep etmelidir.

**Tam tutar gerekli.** Ödeme fatura tutarıyla tam olarak eşleşmelidir. Ne fazla, ne az. Bu, ödemeleri faturalara eşleştirmede belirsizliği engeller. Bir alıcı 1,43 TRX faturası için 2 TRX gönderirse, ödeme reddedilir (fazlalık muhasebe karmaşıklığı yaratır ve herhangi bir artı yönü yoktur).

**Benzersiz memo.** Memo alanı, ödemeyi bu belirli faturaya bağlayan benzersiz bir kimlik içerir. Bu kritik güvenlik mekanizmasıdır -- aşağıda daha fazlası.

### Sunucu Tarafı Fatura Depolaması

```sql
INSERT INTO x402_invoices (
  order_id,
  amount_sun,
  pay_to,
  memo,
  expires_at,
  energy_amount,
  duration_hours,
  target_address,
  status
) VALUES (
  'xpay_a7f3c2d1',
  1430000,
  'TMerxTreasuryAddress...',
  'merx_xpay_a7f3c2d1',
  '2026-03-30T12:05:00Z',
  65000,
  1,
  'TBuyerAddress...',
  'PENDING'
);
```

Fatura PENDING durumu ile depolanır. Doğrulandığında PAID'ye veya TTL ödeme olmadan geçtiğinde EXPIRED'ye geçecektir.

## Adım 2: Memo ile Ödeme

Alıcı bir TRX transfer işlemi oluşturur ve imzalar. Kritik uygulama ayrıntısı memo alanıdır.

```javascript
const tronWeb = new TronWeb({
  fullHost: 'https://api.trongrid.io'
});

// Temel işlemi oluştur
const tx = await tronWeb.transactionBuilder.sendTrx(
  invoice.pay_to,       // TMerxTreasuryAddress
  invoice.amount_sun,   // 1430000
  buyerAddress           // TBuyerAddress
);

// Memo ekle
const txWithMemo = await tronWeb.transactionBuilder.addUpdateData(
  tx,
  invoice.memo,         // "merx_xpay_a7f3c2d1"
  'utf8'
);

// Yerel olarak imzala
const signedTx = await tronWeb.trx.sign(txWithMemo, privateKey);

// Yayınla
const result = await tronWeb.trx.sendRawTransaction(signedTx);
console.log('TX hash:', result.txid);
```

### Neden Memo, Tutar Değil

Önceki bir tasarım, ödemeleri tanımlamak için benzersiz tutarlar kullanmayı düşünmüştü (örneğin, 1.430.000 SUN yerine 1.430.017 SUN). Bu yaklaşım kırılgandır:

- Tutar çarpışmaları mümkündür (iki faturanın aynı fiyatı olabilir)
- Alıcının yuvarlak olmayan bir tutar ödemesini gerektirir
- Birden fazla faturanın özdeş parametreleri olduğunda çalışmaz

Memo alanı, çarpışma riski olmayan net bir kimlik sağlar.

### Özel Anahtar Güvenliği

Alıcının özel anahtarı hiçbir zaman cihazlarını terk etmez. İşlem tamamen alıcının makinesi üzerinde oluşturulur, imzalanır ve yayınlanır. MERX asla alıcının özel anahtarını görmez, talep etmez veya erişimi yoktur. Bu, x402 protokolünün temel bir güvenlik özelliğidir.

## Adım 3: TronGrid Doğrulaması

Ödeme yayınlandıktan sonra, MERX bunu blok zincir üzerinde doğrulamalıdır. Uygulama burada ilginç hale gelir -- ve önemli bir teknik zorluk ortaya çıkar.

### İzleme Döngüsü

MERX mevduat monitörü, gelen işlemleri sürekli olarak hazine adresini izler:

```typescript
async function monitorTreasuryForX402Payments(): Promise<void> {
  const treasuryAddress = process.env.TREASURY_ADDRESS;

  while (true) {
    const transactions = await tronWeb.trx.getTransactionsRelated(
      treasuryAddress,
      'to',
      { limit: 50, only_confirmed: true }
    );

    for (const tx of transactions) {
      await processIncomingTransaction(tx);
    }

    await sleep(3000); // 3 saniye de bir kontrol et (bir blok)
  }
}
```

### Hex ve Base58 Adres Sorunu

İşte x402 uygulamasının diğer herhangi bir bölümünden daha fazla hata ayıklama zamanı tüketen teknik zorluk.

TRON adresleri iki biçimde vardır:

- **Base58**: `TJRabPrwbZy45sbavfcjinPJC18kjpRTv8` (insan tarafından okunabilir, T ile başlar)
- **Hex**: `415a523b449890854c8fc460ab602df9f31fe4293f` (41 ön eki hex, dahili olarak kullanılır)

TronGrid'de işlem ayrıntıları sorguladığınızda, yanıt hex adresler kullanır. Faturanız alıcı adresini ve hazine adresini depoladığında, bunlar base58'dedir. Bunları doğrudan karşılaştırırsanız, hiçbir zaman eşleşmeyeceklerdir.

```typescript
// TronGrid API'dan gelen işlem
const txData = {
  owner_address: '415a523b449890854c8fc460ab602df9f31fe4293f',  // hex
  to_address: '41e552f6487585c2b58bc2c9bb4492bc1f17132cd0',    // hex
  amount: 1430000
};

// Veritabanından fatura
const invoice = {
  pay_to: 'TJRabPrwbZy45sbavfcjinPJC18kjpRTv8',  // base58
  target_address: 'TBuyerAddressBase58...',         // base58
  amount_sun: 1430000
};

// Doğrudan karşılaştırma BAŞARISIZ OLUR
txData.to_address === invoice.pay_to  // false (hex vs base58)
```

### Çözüm: Karşılaştırmadan Önce Dönüştür

Her adres karşılaştırması her iki tarafı aynı biçime dönüştürmelidir:

```typescript
function normalizeAddress(address: string): string {
  if (address.startsWith('41') && address.length === 42) {
    // Hex biçimi -- base58'e dönüştür
    return tronWeb.address.fromHex(address);
  }
  if (address.startsWith('T') && address.length === 34) {
    // Zaten base58
    return address;
  }
  throw new Error(`Geçersiz TRON adres biçimi: ${address}`);
}

function addressesMatch(a: string, b: string): boolean {
  return normalizeAddress(a) === normalizeAddress(b);
}
```

Bu normalizasyon işlevi x402 doğrulama ardışık düzenindeki her adres karşılaştırmasında kullanılır. Tek bir karşılaştırma noktasını kaçırmak bir güvenlik açığı yaratır.

### Tam Doğrulama Mantığı

```typescript
async function verifyX402Payment(tx: TronTransaction): Promise<void> {
  // 1. İşlem verilerinden memo çıkar
  const memo = extractMemo(tx);
  if (!memo || !memo.startsWith('merx_xpay_')) {
    return; // x402 ödeme değil, atla
  }

  // 2. Eşleşen faturayı bul
  const invoice = await findInvoiceByMemo(memo);
  if (!invoice) {
    console.warn(`Memo için fatura bulunamadı: ${memo}`);
    return;
  }

  // 3. Fatura durumunu kontrol et
  if (invoice.status !== 'PENDING') {
    console.warn(`Fatura ${invoice.order_id} zaten ${invoice.status}`);
    return; // Çift talep etmeyi önle
  }

  // 4. Son kullanma tarihini kontrol et
  if (new Date() > new Date(invoice.expires_at)) {
    await markInvoiceExpired(invoice.order_id);
    console.warn(`Fatura ${invoice.order_id} sona erdi`);
    return;
  }

  // 5. Tutarı doğrula (tam eşleşme gerekli)
  if (tx.amount !== invoice.amount_sun) {
    console.warn(
      `Tutar uyuşmazlığı: TX=${tx.amount}, fatura=${invoice.amount_sun}`
    );
    return;
  }

  // 6. Alıcıyı doğrula (hex vs base58 güvenli karşılaştırması)
  if (!addressesMatch(tx.to_address, invoice.pay_to)) {
    console.warn('Alıcı adresi uyuşmazlığı');
    return;
  }

  // Tüm kontroller geçildi -- ödeme geçerli
  await processValidPayment(invoice, tx);
}
```

### Memo Çıkarımı

Memo, işlemin `raw_data.data` alanında hex kodlanmış bir dize olarak depolanır:

```typescript
function extractMemo(tx: TronTransaction): string | null {
  try {
    const hexData = tx.raw_data?.data;
    if (!hexData) return null;

    // Hex'i UTF-8'e çöz
    const memo = Buffer.from(hexData, 'hex').toString('utf8');
    return memo;
  } catch {
    return null;
  }
}
```

## Adım 4: x402 Sistem Kullanıcısı

Bir x402 ödeme doğrulandığında, MERX ödemeyi krediye çekmeli ve siparişi yürütmelidir. Ancak x402 ödemeleri hesapsızdır -- alıcının MERX hesabı yoktur. Hesap olmadan defter girişleri nasıl oluşturursunuz?

Çözüm x402 sistem kullanıcısıdır. Bu, tüm x402 işlemlerini temsil eden özel bir dahili hesaptır:

```sql
INSERT INTO accounts (id, email, type)
VALUES (
  'x402-system-00000000-0000-0000-0000-000000000000',
  'x402@system.merx.exchange',
  'SYSTEM'
);
```

### Bakiye Kredisi

Bir x402 ödeme doğrulandığında, sistem:

1. x402 sistem hesabını krediye çeker (bakiye artar)
2. Hazine hesabını borçlandırır (alınan TRX)
3. x402 sistem hesabını hemen borçlandırır (sipariş ödemesi)
4. Sağlayıcı uzlaştırma hesabını krediye çeker (sağlayıcı ödemesi)

```sql
BEGIN;

-- x402 sistem hesabını krediye çek (ödeme alındı)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($x402_system_id, 'X402_PAYMENT', 1430000, 'CREDIT', $order_id);

-- Hazineyi borçlandır (blok zincir üzerinde alınan TRX)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($treasury_id, 'X402_PAYMENT', 1430000, 'DEBIT', $order_id);

-- x402 sistem hesabını borçlandır (sipariş ödemesi)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($x402_system_id, 'ORDER_PAYMENT', 1430000, 'DEBIT', $order_id);

-- Sağlayıcı uzlaştırmasını krediye çek (MERX sağlayıcıya borçlu)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($provider_settlement_id, 'ORDER_PAYMENT', 1430000, 'CREDIT', $order_id);

COMMIT;
```

x402 sistem hesabı bakiyesi her zaman sıfır veya sıfıra yakın olmalıdır: her kredi (alınan ödeme) hemen bir borç tarafından dengelenir (yürütülen sipariş). Bakiye büyürse, bunun anlamı ödemelerin alındığı ancak siparişlerin yürütülmediğidir -- bir uyarı koşuludur.

## Adım 5: Sipariş Yürütümü

Ödeme krediye çekildikten sonra, sipariş standart MERX sipariş ardışık düzeni aracılığıyla yürütülür:

```typescript
async function processValidPayment(
  invoice: X402Invoice,
  tx: TronTransaction
): Promise<void> {
  // Faturayı PAID olarak işaretle
  await updateInvoiceStatus(invoice.order_id, 'PAID', tx.txid);

  // Defter girişleri oluştur (yukarıda gösterildiği gibi)
  await createX402LedgerEntries(invoice, tx);

  // Energy siparişini yürüt
  const order = await executeOrder({
    energy_amount: invoice.energy_amount,
    duration_hours: invoice.duration_hours,
    target_address: invoice.target_address,
    source: 'x402',
    reference_tx: tx.txid
  });

  // Faturayı sipariş sonucu ile güncelle
  await updateInvoiceWithOrder(invoice.order_id, order);
}
```

Sipariş yürütümü, herhangi bir MERX siparişiyle aynı yolu izler: en iyi fiyat yönlendirmesi, sağlayıcı seçimi, devir ve blok zincir doğrulaması (önceki makalemizde açıklanan yarış durumu düzeltmesi dahil).

## Güvenlik: Memo Doğrulaması Çapraz Ödemeyi Engeller

Memo alanı x402 güvenliğinin mafsallı noktasıdır. Onsuz, kritik bir saldırı vektörü vardır:

### Çapraz Ödeme Saldırısı

Memosu olmayan x402'yi düşünün. İki kullanıcı eşzamanlı olarak fatura talep eder:

```
Alice, TAliceAddress için 65.000 energy talep eder. Fatura: 1,43 TRX TMerxTreasury'ye.
Bob, TBobAddress için 65.000 energy talep eder. Fatura: 1,43 TRX TMerxTreasury'ye.
```

Her iki fatura da aynı tutara ve aynı pay_to adresine sahiptir. Bob faturasını öderse, MERX energy'nin TAliceAddress'e mi yoksa TBobAddress'e mi devredileceğini nereden bilecek? Memo olmadan, ödeme belirsizdir.

Daha kötüsü: Bob bir kez ödeme yapabilir ve her iki faturayı da talep edebilir. Ya da Alice, Bob'un ödemesini kendi faturası için talep edebilir.

### Memoların Bunu Nasıl Engellediği

```
Alice'in faturası: memo = "merx_xpay_alice123"
Bob'un faturası:   memo = "merx_xpay_bob456"

Alice'in ödeme TX: 1,43 TRX TMerxTreasury'ye, memo = "merx_xpay_alice123"
Bob'un ödeme TX:   1,43 TRX TMerxTreasury'ye, memo = "merx_xpay_bob456"

Doğrulama:
  Alice'in TX memo, Alice'in faturası ile eşleşir -> TAliceAddress'e devreder
  Bob'un TX memo, Bob'un faturası ile eşleşir -> TBobAddress'e devreder
```

Her ödeme, belirli faturasına net bir şekilde bağlanır. Çapraz talep etmenin hiçbir yolu yoktur.

### Ek Güvenlik Kontrolleri

Memo eşleştirmesinin ötesinde, doğrulama ardışık düzeni şunları içerir:

**Çift ödeme önlemesi.** Bir fatura PAID olarak işaretlendikten sonra, aynı memo ile yapılan sonraki ödemeler reddedilir. Ödeyici destek ile iletişime geçerek geri ödeme talep etmelidir (veya tutar faturayı aşarsa sistem otomatik olarak fonları iade edebilir).

**Tutar kesinliği.** Ödeme fatura tutarı ile tam olarak eşleşmelidir. Bu, kısmi ödemeleri (karmaşık kısmi doldurma mantığı gerektiren) ve fazla ödemeleri (geri ödeme mantığı gerektiren) engeller.

**Son kullanma tarihi uygulanması.** Fatura süresi dolduktan sonra alınan ödemeler işlenmez. Bu, bir alıcı düşük fiyat döneminde bir fatura talep eden, fiyatlar yükselene kadar bekleyen ve eski faturayı ödeyen eski fiyat istismarını engeller.

**Adres doğrulaması.** Ödeme doğru hazine adresine gitmelidir. Bir kullanıcı bir şekilde farklı bir adrese ödeme yaparsa (yazım hatası, kimlik avı), ödeme monitor tarafından tespit edilmez.

## Hata İşleme

### Fatura Olmadan Ödeme

Hazine adresine, varolan hiçbir faturayla eşleşmeyen bir memo ile bir TRX transferi gelirse (yazım hatası, süresi dolmuş fatura, test işlemi), ödeme günlüğe kaydedilir ancak işlenmez. Fonlar hazinede kalır. Bir üretim sisteminde, bu manuel inceleme ve olası geri ödeme için bir destek uyarısını tetikler.

### Ödeme Sonrası Sağlayıcı Arızası

Doğrulanmış ödeme sonrasında energy sağlayıcısı devredemezse:

```typescript
try {
  const order = await executeOrder(invoice);
} catch (error) {
  // Sipariş başarısız -- x402 sistem hesabını geri öde
  await createRefundLedgerEntries(invoice);

  // Faturayı REFUND_REQUIRED olarak işaretle
  await updateInvoiceStatus(invoice.order_id, 'REFUND_REQUIRED');

  // Ops ekibini manuel TRX geri ödemesi için ödeyenin adresine uyar
  await alertOps({
    type: 'X402_REFUND_REQUIRED',
    invoice: invoice.order_id,
    payer_address: extractSenderFromTx(tx),
    amount_sun: invoice.amount_sun
  });
}
```

Geri ödeme yeni defter girişleri oluşturur (hiçbir zaman mevcut olanları değiştirmez) ve faturayı manuel geri ödeme işlemi için işaretler.

### Ağ Yoğunluğu

Yüksek ağ yoğunluğu sırasında, ödeme yayını ile ödeme onayı arasındaki boşluk 5 dakikalık fatura penceresini aşabilir. Sistem bunu, onay zaman damgası (bir bloka dahil edildiği zaman) yerine işlem zaman damgası (yayınlandığında) kontrol ederek işler. İşlem fatura süresi dolmadan önce yayınlanmışsa, onay süresi dolduktan sonra gelirse bile kabul edilir.

## Özet

MERX'te x402 uygulaması, güvensiz, hesapsız ödemelerin bugün pratik olduğunu gösterir. Temel tasarım kararları:

1. **Benzersiz memo ile fatura** -- net ödeme-sipariş bağlantısı
2. **Tam tutar eşleştirmesi** -- kısmi/fazla ödeme karmaşıklığını ortadan kaldırır
3. **5 dakikalık son kullanma tarihi** -- eski fiyat istismarını engeller
4. **Hex-to-base58 normalizasyonu** -- TronGrid adres biçimi sorununu çözer
5. **x402 sistem kullanıcısı** -- alıcı hesapları olmadan çift giriş muhasebesini mümkün kılar
6. **Değişmez defter girişleri** -- her x402 işlemi için tam denetim izi

Protokol HTTP 402'yi 29 yıllık bir yer tutucudan çalışan bir ödeme mekanizmasına dönüştürür. Hesap oluşturamayan veya API anahtarlarını yönetemeyen yapay zeka ajanları için x402, tek bir blok zincir işlemi aracılığıyla TRON energy'yi erişilebilir kılar.

Platform: [https://merx.exchange](https://merx.exchange)
Dokümantasyon: [https://merx.exchange/docs](https://merx.exchange/docs)
MCP sunucusu: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)


## Şu Anda Yapay Zeka ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin -- sıfır yükleme, salt okunur araçlar için API anahtarı gerekmez:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Yapay zeka ajanınıza sorun: "Şu anda en ucuz TRON energy nedir?" ve bağlı tüm sağlayıcılardan canlı fiyatlar alın.

Tam MCP dokümantasyonu: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)