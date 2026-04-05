# MERX Sipariş Yönlendirmesi: Siparişiniz En İyi Fiyatı Nasıl Alır

MERX'e bir energy siparişi gönderdiğinizde, milisaniyeler içinde bir dizi karar alınır: hangi sağlayıcının en iyi fiyatı var, bu siparişi doldurabilir mi, başarısız olursa ne olur, delegasyonun on-chain'de gerçekleştiğini nasıl doğrularız. Bu, sipariş yönlendirme motoru - basit bir API çağrısını optimize edilmiş, hata toleranslı, doğrulanmış energy teslimatına dönüştüren bileşendir.

Bu makale, `createOrder` çağrısından delege edilen energy'nin TRON adresinizde görünmesine kadar MERX siparişinin tam yaşam döngüsünü açıklamaktadır.

---

## Sipariş Yaşam Döngüsü

Her sipariş altı aşamadan geçer:

```
1. Doğrulama    -> Sipariş iyi biçimlenmiş mi?
2. Fiyatlandırma -> Şu an en iyi fiyat nedir?
3. Yönlendirme  -> Hangi sağlayıcı(lar) bunu dolduracak?
4. Yürütme      -> Sağlayıcı(lara) gönder
5. Doğrulama    -> On-chain delegasyonu onayla
6. Yerleşim     -> Bakiye debit et, defteri kaydet
```

Her aşamayı adım adım ele alalım.

---

## Aşama 1: Doğrulama

Herhangi bir yönlendirme mantığı çalışmadan önce, sipariş doğrulanır:

```typescript
// Zod ile giriş doğrulaması
const OrderSchema = z.object({
  energy: z.number().int().min(10000).max(100000000),
  targetAddress: z.string().refine(isValidTronAddress),
  duration: z.enum(['1h', '1d', '3d', '7d', '14d', '30d']),
  maxPrice: z.number().optional(),  // energy birimi başına SUN
  idempotencyKey: z.string().optional()
});
```

Anahtar doğrulamalar:

- **Energy miktarı**: desteklenen aralık içinde pozitif bir tamsayı olmalıdır.
- **Hedef adres**: geçerli, etkinleştirilmiş bir TRON adresi olmalıdır. Sistem adres biçimini doğrular ve isteğe bağlı olarak hesabın on-chain'de var olup olmadığını kontrol eder.
- **Süre**: desteklenen delegasyon dönemlerinden biri olmalıdır.
- **Maksimum fiyat**: isteğe bağlı bir üst sınır. Ayarlanırsa, sipariş yalnızca mevcut en iyi fiyat bu eşik değerin altında veya eşitse yürütülür.
- **İdempotency anahtarı**: sağlanırsa, aynı anahtarla yinelenen gönderimleri yeni bir tane oluşturmak yerine orijinal siparişin sonucunu döndürür.

### İdempotency

İdempotency anahtarı, üretim entegrasyonları için kritiktir. Ağ sorunları, bir istemciyi isteği yeniden denemeye zorlayabilir ve potansiyel olarak yinelenen siparişler oluşturabilir. İdempotency anahtarı ile, ikinci istek ilkinin sonucunu döndürür:

```typescript
const order = await client.createOrder({
  energy: 65000,
  targetAddress: 'TBuyerAddress...',
  duration: '1h',
  idempotencyKey: 'payment-123-energy'
});

// Aynı anahtarla yeniden çağrılırsa, aynı siparişi döndürür
// Yinelenen delegasyon yok, çift ücret yok
```

---

## Aşama 2: Fiyatlandırma

Sipariş yürütücüsü, Redis önbelleğinden mevcut fiyatları okur. Fiyat monitörü, bunları her 30 saniyede bir günceller, bu nedenle veriler en fazla 30 saniye eski olur.

```typescript
async function getBestPrices(
  energyAmount: number,
  duration: string
): Promise<ProviderPrice[]> {

  const allPrices = await redis.keys('prices:*');
  const validPrices = [];

  for (const key of allPrices) {
    const price = JSON.parse(await redis.get(key));

    // Filtre: istenen süreyi desteklemeli
    if (!price.durations.includes(duration)) continue;

    // Filtre: yeterli kullanılabilirliğe sahip olmalı
    if (price.availableEnergy < energyAmount) continue;

    // Filtre: sağlık eşiğini geçmeli
    const health = await getProviderHealth(price.provider);
    if (health.fillRate < 0.90) continue;

    validPrices.push(price);
  }

  // Etkili fiyata göre sırala (güvenilirliği hesaba kat)
  return validPrices.sort((a, b) => {
    const effectiveA = a.energyPricePerUnit / a.fillRate;
    const effectiveB = b.energyPricePerUnit / b.fillRate;
    return effectiveA - effectiveB;
  });
}
```

### Maksimum Fiyat Uygulaması

Alıcı bir `maxPrice` belirtirse, hiçbir sağlayıcı bunu karşılayamıyorsa sipariş reddedilir:

```typescript
if (maxPrice && bestPrice.energyPricePerUnit > maxPrice) {
  throw new OrderError({
    code: 'PRICE_EXCEEDED',
    message: `Best available price (${bestPrice.energyPricePerUnit} SUN) exceeds your maximum (${maxPrice} SUN)`,
    details: { bestAvailable: bestPrice.energyPricePerUnit, maxPrice }
  });
}
```

Bu, fiyat dalgalanmaları sırasında beklenmedik ücretleri önler.

---

## Aşama 3: Yönlendirme

Yönlendirme motoru, hangi sağlayıcı(ların) siparişi dolduracağına karar verir. Bu, sistemin çekirdek zekasıdır.

### Basit Durum: Tek Sağlayıcı Doldurması

Tek sağlayıcının kapasitesi içindeki siparişler için:

```
Sipariş: 65.000 energy
En iyi sağlayıcı: itrx at 85 SUN/birim, 500.000 kullanılabilir

Yönlendirme: itrx'e %100
```

### Bölünmüş Durum: Çok Sağlayıcı Doldurması

Büyük siparişler veya en ucuz sağlayıcının sınırlı stoku olduğunda:

```
Sipariş: 500.000 energy

Sağlayıcı A: 85 SUN'da 200.000 kullanılabilir
Sağlayıcı B: 87 SUN'da 180.000 kullanılabilir
Sağlayıcı C: 92 SUN'da 300.000 kullanılabilir

Yönlendirme planı:
  Bacak 1: Sağlayıcı A -> 85 SUN'da 200.000 energy
  Bacak 2: Sağlayıcı B -> 87 SUN'da 180.000 energy
  Bacak 3: Sağlayıcı C -> 92 SUN'da 120.000 energy

Karışık oran: (200K*85 + 180K*87 + 120K*92) / 500K = 87,28 SUN
```

Yönlendirici, en ucuzdan en pahalısına doğru doldurur, bir sonrakine geçmeden önce her sağlayıcıdan mümkün olduğunca çok alır.

### Failover Zinciri

Her yönlendirme planı bir failover zinciri içerir - birincil başarısız olursa denenmesi gereken alternatif sağlayıcıların sıralı listesi:

```
Birincil:   Sağlayıcı A (85 SUN)
Failover 1: Sağlayıcı B (87 SUN)
Failover 2: Sağlayıcı C (92 SUN)
Failover 3: Sağlayıcı D (95 SUN)
```

Sağlayıcı A yürütülmezse (API hatası, zaman aşımı, yetersiz fon), yürütücü alıcıdan herhangi bir işlem olmadan otomatik olarak Sağlayıcı B'ye geçer.

---

## Aşama 4: Yürütme

Yürütücü, siparişi seçilen sağlayıcı(lara) gönderir ve tamamlanmayı izler.

### Yürütme Akışı

```typescript
async function executeOrder(
  order: Order,
  routingPlan: RoutingPlan
): Promise<ExecutionResult> {

  const results: LegResult[] = [];

  for (const leg of routingPlan.legs) {
    try {
      const result = await executeLeg(leg, order);
      results.push(result);
    } catch (error) {
      // Birincil sağlayıcı başarısız - failover dene
      const failoverResult = await executeWithFailover(
        leg,
        order,
        routingPlan.failoverChain
      );
      results.push(failoverResult);
    }
  }

  return {
    orderId: order.id,
    legs: results,
    totalEnergy: results.reduce((sum, r) => sum + r.energy, 0),
    totalCostSun: results.reduce((sum, r) => sum + r.costSun, 0)
  };
}

async function executeWithFailover(
  failedLeg: RoutingLeg,
  order: Order,
  failoverChain: Provider[]
): Promise<LegResult> {

  for (const provider of failoverChain) {
    try {
      const result = await executeLeg(
        { ...failedLeg, provider: provider.name },
        order
      );
      return result;
    } catch (error) {
      // Kaydet ve sonraki failover'a devam et
      continue;
    }
  }

  throw new OrderError({
    code: 'ALL_PROVIDERS_FAILED',
    message: 'Sipariş hiçbir mevcut sağlayıcı tarafından doldurulmadı'
  });
}
```

### Zaman Aşımı Yönetimi

Her sağlayıcı yürütmesinin katı bir zaman aşımı vardır. Sağlayıcı, zaman aşımı penceresinde (genellikle 30 saniye) siparişi onaylamazsa, yürütücü failover zincirine geçer:

```
T+0:    Siparişi Sağlayıcı A'ya gönder
T+30s:  Cevap yok -> zaman aşımı, Sağlayıcı B'ye failover
T+31s:  Siparişi Sağlayıcı B'ye gönder
T+35s:  Sağlayıcı B onayladı -> doğrulamaya geç
```

---

## Aşama 5: Doğrulama

Sağlayıcıdan onay almak yeterli değildir. MERX, energy delegasyonunun aslında TRON blokzincirinde göründüğünü doğrular.

### On-Chain Doğrulama Süreci

```typescript
async function verifyDelegation(
  targetAddress: string,
  expectedEnergy: number,
  delegationTxHash: string
): Promise<VerificationResult> {

  // Adım 1: Delegasyon işleminin var olduğunu doğrula
  const tx = await tronWeb.trx.getTransaction(delegationTxHash);
  if (!tx || tx.ret[0].contractRet !== 'SUCCESS') {
    return { verified: false, reason: 'İşlem bulunamadı veya başarısız' };
  }

  // Adım 2: Hedef adres kaynaklarını kontrol et
  const resources = await tronWeb.trx.getAccountResources(targetAddress);
  const currentEnergy = resources.EnergyLimit || 0;

  // Adım 3: Energy'nin beklenen miktar kadar arttığını doğrula (toleransla)
  const tolerance = expectedEnergy * 0.02; // %2 tolerans
  if (currentEnergy < expectedEnergy - tolerance) {
    return { verified: false, reason: 'Energy miktarı beklenenin altında' };
  }

  return {
    verified: true,
    txHash: delegationTxHash,
    energyDelivered: currentEnergy,
    verifiedAt: new Date()
  };
}
```

### Neden %2 Tolerans

Energy delegasyonu miktarları, sipariş yerleştirilmesi ile delegasyon onayı arasında biraz değişebilen TRX-to-energy dönüştürme oranlarına dayanır. %2 tolerans, bunu kabul etmeyen büyük ölçüde yanlış miktarları hesaba katmadan hesaba katar.

### Doğrulama Zamanlaması

```
T+0:    Sağlayıcı siparişi onayladı
T+3-6s: Delegasyon işlemi TRON'da onaylandı (1 blok)
T+10s:  MERX hedef adres kaynaklarını sorguladı
T+10s:  Doğrulama tamamlandı, alıcı bilgilendirildi
```

Sipariş gönderiminden doğrulanmış teslimatına kadar olan tüm süreç tipik olarak 15-45 saniye sürer.

---

## Aşama 6: Yerleşim

Doğrulamadan sonra, finansal yerleşim gerçekleşir:

### Bakiye Kesintisi

```sql
-- Atomik bakiye kontrolü ve kesinti
BEGIN;

SELECT balance_sun FROM accounts
WHERE user_id = $1
FOR UPDATE;

-- Yeterli bakiye doğrula
-- Yetersiz ise, ROLLBACK ve hata döndür

UPDATE accounts
SET balance_sun = balance_sun - $2
WHERE user_id = $1;

COMMIT;
```

`SELECT FOR UPDATE`, bakiye kontrolü ile kesinti arasında yarış koşulu olmadığını sağlar. İki sipariş eşzamanlı olarak işlenirse, ikincisi birincisinin tamamlanmasını bekledikten sonra bakiyeyi kontrol eder.

### Defter Girişi

Her yerleşim, değişmez bir defter girişi oluşturur:

```sql
INSERT INTO ledger (
  user_id, type, amount_sun,
  reference_type, reference_id,
  balance_before, balance_after,
  created_at
) VALUES (
  $1, 'ORDER_PAYMENT', $2,
  'order', $3,
  $4, $5,
  NOW()
);
```

Defter girişleri ekleyen. Hiçbir zaman güncellenmez veya silinmez. Bu, her finansal işlemin tamamen, denetlenebilir bir geçmişini oluşturur.

---

## Siparişinizi İzleme

MERX API, REST ve WebSocket aracılığıyla gerçek zamanlı sipariş durumu sağlar:

```typescript
import { MerxClient } from 'merx-sdk';

const client = new MerxClient({ apiKey: 'your-key' });

// REST: sipariş durumunu yokla
const order = await client.getOrder('ord_abc123');
console.log(order.status);
// 'pending' | 'executing' | 'verifying' | 'completed' | 'failed'

// WebSocket: gerçek zamanlı güncellemeler
client.onOrderUpdate('ord_abc123', (update) => {
  console.log(`Sipariş ${update.orderId}: ${update.status}`);
  if (update.status === 'completed') {
    console.log(`Delegasyon TX: ${update.delegationTxHash}`);
  }
});
```

### Sipariş Durumu Akışı

```
pending -> executing -> verifying -> completed
              |                        |
              v                        v
            failed              partially_filled
```

- **pending**: Sipariş alındı, yürütmeyi bekliyor.
- **executing**: Sağlayıcıya gönderildi, onayı bekliyor.
- **verifying**: Sağlayıcı onayladı, on-chain onayını bekliyor.
- **completed**: Energy hedef adresinde doğrulandı.
- **failed**: Tüm sağlayıcılar başarısız. Bakiye ücretlendirilmedi.
- **partially_filled**: Bazı bacaklar tamamlandı, diğerleri başarısız. Bakiye doldurulmuş kısım için ücretlendirildi.

---

## Hata Yönetimi

### Sağlayıcı Hataları

Her sağlayıcı benzersiz şekillerde başarısız olabilir. Sipariş yürütücüsü, tüm sağlayıcı hatalarını standart hata kodlarına normalleştirir:

```json
{
  "error": {
    "code": "PROVIDER_UNAVAILABLE",
    "message": "Birincil sağlayıcı siparişi dolduramazdı. İkincil sağlayıcıya failover.",
    "details": {
      "primaryProvider": "tronsave",
      "primaryError": "timeout",
      "filledBy": "feee",
      "priceImpact": "+2 SUN/birim"
    }
  }
}
```

### Kısmi Doldurmalar

Çok bacaklı siparişler için, bazı bacaklar başarılı olabilirken diğerleri başarısız olabilir. MERX bunu şu şekilde yönetir:

1. Başarılı bacakları normal olarak tamamlar.
2. Başarısız bacaklar için failover'i dener.
3. Failover da başarısız olursa, kısmi doldurma sonucu döndürür.
4. Alıcıyı yalnızca aslında teslim edilen energy için ücretlendirir.

---

## Performans

Tipik sipariş yürütme süreleri:

```
Doğrulama:    < 5ms
Fiyatlandırma: < 10ms (Redis okuma)
Yönlendirme:   < 5ms
Yürütme:       5-30 saniye (sağlayıcı API'si + blokzincir)
Doğrulama:     3-10 saniye (blokzincir onayı)

Toplam: API çağrısından doğrulanmış teslimatına 10-45 saniye
```

Darboğaz her zaman blokzincirdir. MERX'in iç işlemesi 50ms'den daha az ek yük ekler. Geri kalan, sağlayıcı ve TRON ağını beklemeye harcanır.

---

## Sonuç

Sipariş yönlendirmesi, MERX'in değerinin en somut olduğu yerdir. Tek bir API çağrısı, bir dizi optimizasyonu tetikler: en iyi fiyat seçimi, çok sağlayıcı bölünmesi, otomatik failover, on-chain doğrulaması ve atomik yerleşim. Her adım, mevcut en ucuz energy'yi güvenilir bir şekilde sunmanızı ve tam denetlenebilirliği sağlamak için tasarlanmıştır.

Karmaşıklık gerçektir, ancak yönetmek için MERX'indir, sizin değil. Entegrasyonunuz basit bir API çağrısı olarak kalır.

[https://merx.exchange](https://merx.exchange) adresinde sipariş yönlendirmesini başlatın. Tam API başvurusu [https://merx.exchange/docs](https://merx.exchange/docs) adresinde.

---

*Bu makale MERX teknik serisinin bir parçasıdır. MERX ilk blokzincir kaynağı değişimidir. SDK'lar: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) ve [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python). MCP sunucusu: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp).*


## Şimdi AI ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin -- kurulum yok, salt okunur araçlar için API anahtarı yok:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI aracınıza şunu sorun: "Şu an en ucuz TRON energy nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)