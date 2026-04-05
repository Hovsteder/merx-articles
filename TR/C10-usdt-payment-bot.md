# MERX SDK ile USDT Ödeme Botu Oluşturma

TRON'da USDT göndermek basit olmalıdır. Alıcı adresiniz, miktar ve fonlar var. Ancak USDT'yi enerji olmadan gönderirseniz, TRON protokolü işlem maliyetini karşılamak için cüzdanınızdan yaklaşık 27 TRX yakar. Ölçekte - günde yüzler veya binlerce transfer - bu yakma maliyeti önemli bir harcama haline gelir.

Bu eğitim bu maliyeti ortadan kaldıran tam bir USDT ödeme botu oluşturur. Bot ödeme isteklerini alır, MERX aracılığıyla yakma fiyatının bir kısmına enerji kiralar, kiralanan enerji ile USDT gönderir ve tasarrufları raporlar. Hataları işler, yeniden denemeleri destekler ve üretim güvenilirliği için webhook bildirimlerini entegre eder.

Sonunda, her USDT transferinde yüzde 90'dan fazla tasarruf sağlayan çalışan bir ödeme botuna sahip olacaksınız.

## Mimari Özeti

Bot basit bir pipeline izler:

1. Ödeme isteği alır (alıcı adresi + miktar).
2. MERX hesap bakiyesini kontrol eder.
3. Transfer için gerekli enerjiyi tahmin eder.
4. MERX aracılığıyla enerji siparişi oluşturur.
5. Enerji delegasyonunun tamamlanmasını bekler.
6. Delegasyona verilen enerji kullanarak USDT transferini gönderir.
7. İşlemi ve tasarrufları raporlar.

Her adım izole ve yeniden denenebilir. Enerji siparişi başarısız olursa, USDT hiçbir zaman gönderilmez. USDT transferi başarısız olursa, yine de enerjiye sahip olursunuz (kiralama süresi sonra sona erer, ancak para kaybı olmaz).

## Ön Koşullar

Oluşturmadan önce şunlara ihtiyacınız vardır:

- **Node.js 18 veya üstü** - bot ES modüllerini ve modern JavaScript özelliklerini kullanır.
- **MERX hesabı ve bakiye ile** - [merx.exchange](https://merx.exchange) adresinde kaydolun ve TRX yatırın.
- **MERX API anahtarı** - `create_orders`, `view_orders` ve `view_balance` izinleriyle bir tane oluşturun.
- **TRON cüzdanı** - ödemeler için USDT bakiyesi ve bandwidth için küçük bir TRX miktarı ile.
- **TronWeb** - USDT transfer işlemini imzalamak ve yayınlamak için.

### Proje Kurulumu

```bash
mkdir usdt-payment-bot
cd usdt-payment-bot
npm init -y
npm install merx-sdk tronweb dotenv uuid
```

`.env` dosyası oluşturun (bunu asla sürüm kontrolüne kaydetmeyin):

```bash
MERX_API_KEY=merx_live_your_key_here
TRON_PRIVATE_KEY=your_tron_wallet_private_key
TRON_FULL_HOST=https://api.trongrid.io
USDT_CONTRACT=TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t
MIN_SAVINGS_PERCENT=50
```

## Adım 1: İstemcileri Başlatın

MERX istemcisini ve TronWeb örneğini oluşturun:

```javascript
// src/clients.js
import { MerxClient } from 'merx-sdk';
import TronWeb from 'tronweb';
import dotenv from 'dotenv';

dotenv.config();

export const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
  baseUrl: 'https://merx.exchange/api/v1',
});

export const tronWeb = new TronWeb({
  fullHost: process.env.TRON_FULL_HOST,
  privateKey: process.env.TRON_PRIVATE_KEY,
});

export const USDT_CONTRACT = process.env.USDT_CONTRACT;
export const SENDER_ADDRESS = tronWeb.defaultAddress.base58;
```

## Adım 2: MERX Bakiyesini Kontrol Edin

Herhangi bir şey yapmadan önce, enerji kiralamak için yeterli bakiyeniz olduğunu doğrulayın:

```javascript
// src/balance.js
import { merx } from './clients.js';

export async function checkMerxBalance(requiredSun) {
  const balance = await merx.account.getBalance();

  const availableSun = balance.available_sun;

  if (availableSun < requiredSun) {
    throw new Error(
      `Insufficient MERX balance. ` +
        `Available: ${(availableSun / 1_000_000).toFixed(2)} TRX, ` +
        `Required: ${(requiredSun / 1_000_000).toFixed(2)} TRX`
    );
  }

  return {
    available: availableSun,
    availableTRX: (availableSun / 1_000_000).toFixed(2),
  };
}
```

## Adım 3: Enerji Maliyetini Tahmin Edin

Transfer için gereken enerji miktarını ve maliyetini belirlemek için MERX tahmin API'sini kullanın:

```javascript
// src/estimate.js
import { merx, SENDER_ADDRESS } from './clients.js';

export async function estimateTransferCost() {
  const estimate = await merx.estimate({
    operation: 'trc20_transfer',
    target_address: SENDER_ADDRESS,
  });

  const savings = estimate.costs.savings;
  const rental = estimate.costs.rental;
  const burn = estimate.costs.burn;

  return {
    energyRequired: estimate.energy_required,
    bandwidthRequired: estimate.bandwidth_required,
    burnCostTRX: burn.trx_cost_readable,
    burnCostSun: burn.trx_cost,
    rentalCostTRX: rental.total_cost_trx,
    rentalCostSun: rental.total_cost_sun,
    bestProvider: rental.best_provider,
    savingsPercent: savings.percent,
    savingsTRX: savings.trx_saved,
    durationHours: rental.duration_hours,
  };
}
```

## Adım 4: Enerji Siparişi Oluşturun

Güvenli yeniden denemeler için bir eşsizlik anahtarıyla MERX aracılığıyla enerji sipariş edin:

```javascript
// src/order.js
import { merx } from './clients.js';
import { randomUUID } from 'crypto';

export async function createEnergyOrder(energyAmount, targetAddress, durationHours = 1) {
  const idempotencyKey = randomUUID();

  const order = await merx.orders.create(
    {
      energy_amount: energyAmount,
      target_address: targetAddress,
      duration_hours: durationHours,
    },
    {
      idempotencyKey,
    }
  );

  return {
    orderId: order.id,
    status: order.status,
    provider: order.provider,
    totalCostSun: order.total_cost_sun,
    idempotencyKey,
  };
}
```

## Adım 5: Enerji Delegasyonunu Bekleyin

Siparişi oluşturduktan sonra, enerji zincir üzerinde delegasyonu bekleyin. Bot siparişin durumunu bir zaman aşımıyla yoklar:

```javascript
// src/wait.js
import { merx } from './clients.js';

export async function waitForFill(orderId, timeoutMs = 60000) {
  const startTime = Date.now();
  const pollInterval = 2000; // 2 saniye

  while (Date.now() - startTime < timeoutMs) {
    const order = await merx.orders.get(orderId);

    if (order.status === 'filled') {
      return {
        status: 'filled',
        provider: order.provider,
        txHash: order.delegation_tx_hash,
        filledAt: order.filled_at,
      };
    }

    if (order.status === 'failed' || order.status === 'cancelled') {
      throw new Error(
        `Order ${orderId} ${order.status}: ${order.failure_reason || 'unknown reason'}`
      );
    }

    // Hâlâ beklemede veya işleniyor - bekleyin ve tekrar yoklayın
    await new Promise((resolve) => setTimeout(resolve, pollInterval));
  }

  throw new Error(`Order ${orderId} timed out after ${timeoutMs / 1000}s`);
}
```

Üretim sistemleri için, yoklamak yerine webhook kullanmayı düşünün (bu makalenin ilerleyen kısmında ele alınmıştır).

## Adım 6: USDT Transferini Gönderin

Adresinize enerji delegasyonu ile, USDT transferini gönderin. Enerji TRX yakmak yerine tüketilir:

```javascript
// src/transfer.js
import { tronWeb, USDT_CONTRACT, SENDER_ADDRESS } from './clients.js';

export async function sendUSDT(recipientAddress, amountUSDT) {
  // USDT TRON'da 6 ondalık basamağa sahiptir
  const amountSun = Math.floor(amountUSDT * 1_000_000);

  // Alıcı adresini doğrulayın
  if (!tronWeb.isAddress(recipientAddress)) {
    throw new Error(`Invalid TRON address: ${recipientAddress}`);
  }

  // TRC-20 transfer işlemini oluşturun
  const contract = await tronWeb.contract().at(USDT_CONTRACT);

  const tx = await contract.methods
    .transfer(recipientAddress, amountSun)
    .send({
      from: SENDER_ADDRESS,
      feeLimit: 100_000_000, // 100 TRX ücret sınırı (güvenlik sınırı)
    });

  return {
    txHash: tx,
    recipient: recipientAddress,
    amount: amountUSDT,
    amountSun,
  };
}
```

## Adım 7: Hepsini Bir Araya Getirin

Ana bot işlevi tüm adımları düzenler:

```javascript
// src/bot.js
import { checkMerxBalance } from './balance.js';
import { estimateTransferCost } from './estimate.js';
import { createEnergyOrder } from './order.js';
import { waitForFill } from './wait.js';
import { sendUSDT } from './transfer.js';
import { SENDER_ADDRESS } from './clients.js';

const MIN_SAVINGS_PERCENT = parseFloat(process.env.MIN_SAVINGS_PERCENT || '50');

export async function processPayment(recipientAddress, amountUSDT) {
  const startTime = Date.now();

  console.log(`--- Payment Request ---`);
  console.log(`Recipient: ${recipientAddress}`);
  console.log(`Amount: ${amountUSDT} USDT`);

  // Adım 1: Maliyetleri tahmin edin
  console.log('\n[1/5] Estimating energy cost...');
  const estimate = await estimateTransferCost();
  console.log(`  Energy required: ${estimate.energyRequired}`);
  console.log(`  Burn cost: ${estimate.burnCostTRX}`);
  console.log(`  Rental cost: ${estimate.rentalCostTRX}`);
  console.log(`  Savings: ${estimate.savingsPercent}%`);

  // Adım 2: Kiralamaya değer olup olmadığına karar verin
  if (estimate.savingsPercent < MIN_SAVINGS_PERCENT) {
    console.log(`  Savings below threshold (${MIN_SAVINGS_PERCENT}%). Skipping energy rental.`);
    const tx = await sendUSDT(recipientAddress, amountUSDT);
    return { ...tx, energyRented: false, savings: null };
  }

  // Adım 3: MERX bakiyesini kontrol edin
  console.log('\n[2/5] Checking MERX balance...');
  const balance = await checkMerxBalance(estimate.rentalCostSun);
  console.log(`  Available: ${balance.availableTRX} TRX`);

  // Adım 4: Enerji siparişi oluşturun
  console.log('\n[3/5] Creating energy order...');
  const order = await createEnergyOrder(
    estimate.energyRequired,
    SENDER_ADDRESS,
    estimate.durationHours
  );
  console.log(`  Order ID: ${order.orderId}`);
  console.log(`  Provider: ${order.provider}`);
  console.log(`  Status: ${order.status}`);

  // Adım 5: Enerji delegasyonunu bekleyin
  console.log('\n[4/5] Waiting for energy delegation...');
  const fill = await waitForFill(order.orderId);
  console.log(`  Filled by: ${fill.provider}`);
  console.log(`  Delegation TX: ${fill.txHash}`);

  // Adım 6: Delegasyona verilen enerji ile USDT gönderin
  console.log('\n[5/5] Sending USDT...');
  const tx = await sendUSDT(recipientAddress, amountUSDT);
  console.log(`  TX Hash: ${tx.txHash}`);

  const elapsed = ((Date.now() - startTime) / 1000).toFixed(1);

  console.log(`\n--- Payment Complete ---`);
  console.log(`  Total time: ${elapsed}s`);
  console.log(`  Energy cost: ${estimate.rentalCostTRX}`);
  console.log(`  Saved: ${estimate.savingsTRX} (${estimate.savingsPercent}%)`);

  return {
    txHash: tx.txHash,
    recipient: recipientAddress,
    amount: amountUSDT,
    energyRented: true,
    energyOrderId: order.orderId,
    rentalCost: estimate.rentalCostTRX,
    burnCostAvoided: estimate.burnCostTRX,
    savingsPercent: estimate.savingsPercent,
    savingsTRX: estimate.savingsTRX,
    elapsedSeconds: parseFloat(elapsed),
  };
}
```

### Botu Çalıştırın

```javascript
// index.js
import { processPayment } from './src/bot.js';

const recipient = process.argv[2];
const amount = parseFloat(process.argv[3]);

if (!recipient || !amount) {
  console.error('Usage: node index.js <recipient_address> <amount_usdt>');
  process.exit(1);
}

try {
  const result = await processPayment(recipient, amount);
  console.log('\nResult:', JSON.stringify(result, null, 2));
} catch (err) {
  console.error('\nPayment failed:', err.message);
  process.exit(1);
}
```

```bash
node index.js TRecipientAddressHere 100
```

Örnek çıktı:

```
--- Payment Request ---
Recipient: TRecipientAddressHere
Amount: 100 USDT

[1/5] Estimating energy cost...
  Energy required: 64895
  Burn cost: 27.37 TRX
  Rental cost: 1.43 TRX
  Savings: 94.8%

[2/5] Checking MERX balance...
  Available: 50.00 TRX

[3/5] Creating energy order...
  Order ID: ord_7k3m9x2p
  Provider: sohu
  Status: pending

[4/5] Waiting for energy delegation...
  Filled by: sohu
  Delegation TX: 5a8f2c1e...

[5/5] Sending USDT...
  TX Hash: 3d7b4e9a...

--- Payment Complete ---
  Total time: 8.2s
  Energy cost: 1.43 TRX
  Saved: 25.94 TRX (94.8%)
```

## Hata İşleme

Üretim ödeme sistemleri sağlam hata işlemeye gerek duyar. Her başarısızlık modu ve nasıl ele alınacağı:

### Yetersiz MERX Bakiyesi

MERX hesabınız düşükse, bot sizi uyarmalı ve isteğe bağlı olarak TRX yakmaya geri dönmelidir:

```javascript
try {
  await checkMerxBalance(estimate.rentalCostSun);
} catch (err) {
  if (err.message.includes('Insufficient MERX balance')) {
    console.warn('MERX balance low. Sending without energy rental.');
    // İsteğe bağlı: ops ekibine uyarı gönderin
    // await alertOps('MERX balance low', err.message);
    const tx = await sendUSDT(recipientAddress, amountUSDT);
    return { ...tx, energyRented: false, fallbackReason: 'low_balance' };
  }
  throw err;
}
```

### Sipariş Doldurma Zaman Aşımı

Enerji sağlayıcı delegasyonu çok uzun sürerse, iptal edin ve farklı bir sağlayıcı ile yeniden deneyin veya geri dönün:

```javascript
try {
  const fill = await waitForFill(order.orderId, 30000); // 30 saniye zaman aşımı
} catch (err) {
  if (err.message.includes('timed out')) {
    console.warn('Energy delegation timed out. Sending without energy.');
    const tx = await sendUSDT(recipientAddress, amountUSDT);
    return { ...tx, energyRented: false, fallbackReason: 'delegation_timeout' };
  }
  throw err;
}
```

### USDT Transfer Hatası

USDT transferinin kendisi başarısız olursa (yetersiz bakiye, kontrat hatası), enerji boşa harcanmaz - kiralama süresi boyunca delegasyona kalır. Transferi yeniden deneyebilirsiniz:

```javascript
async function sendUSDTWithRetry(recipientAddress, amountUSDT, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await sendUSDT(recipientAddress, amountUSDT);
    } catch (err) {
      if (attempt === maxRetries) throw err;
      console.warn(`Transfer attempt ${attempt} failed: ${err.message}`);
      await new Promise((r) => setTimeout(r, 2000 * attempt));
    }
  }
}
```

## Webhook Bildirimleri

Siparişin durumunu yoklamak basit botlar için işe yarar, ancak üretim sistemleri webhook kullanmalıdır. MERX, siparişin durumu değiştiğinde webhook URL'nize HTTP POST bildirimleri gönderir.

### Webhook Kurulumu

MERX panosunda veya API aracılığıyla webhook uç noktanızı yapılandırın:

```bash
curl -X POST https://merx.exchange/api/v1/webhooks \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-server.com/webhooks/merx",
    "events": ["order.filled", "order.failed"]
  }'
```

### Webhook Olaylarını İşle

```javascript
// webhook-handler.js
import express from 'express';

const app = express();
app.use(express.json());

// Beklemede olan ödeme geri çağrılarını depola
const pendingPayments = new Map();

app.post('/webhooks/merx', (req, res) => {
  const event = req.body;

  // Webhook imzasını doğrulayın (üretim gereksinimi)
  // const isValid = verifyWebhookSignature(req);
  // if (!isValid) return res.status(401).json({ error: 'Invalid signature' });

  if (event.type === 'order.filled') {
    const callback = pendingPayments.get(event.data.order_id);
    if (callback) {
      callback.resolve(event.data);
      pendingPayments.delete(event.data.order_id);
    }
  }

  if (event.type === 'order.failed') {
    const callback = pendingPayments.get(event.data.order_id);
    if (callback) {
      callback.reject(new Error(`Order failed: ${event.data.reason}`));
      pendingPayments.delete(event.data.order_id);
    }
  }

  res.status(200).json({ received: true });
});

// Yoklamayı webhook tabanlı beklemeyle değiştirin
export function waitForFillWebhook(orderId, timeoutMs = 60000) {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => {
      pendingPayments.delete(orderId);
      reject(new Error(`Order ${orderId} webhook timed out`));
    }, timeoutMs);

    pendingPayments.set(orderId, {
      resolve: (data) => {
        clearTimeout(timer);
        resolve(data);
      },
      reject: (err) => {
        clearTimeout(timer);
        reject(err);
      },
    });
  });
}
```

Webhook'ler yoklama döngüsünü ortadan kaldırır, API çağrılarını azaltır ve siparişin doldurulmasına daha hızlı yanıt verir (2 saniyeli yoklama aralıklarına karşı saniyenin altında bildirim).

## Üretim Hususları

### Eşzamanlılık

Botunuz aynı anda birden fazla ödeme işlerse, her ödemenin kendi eşsizlik anahtarı olmalı ve bağımsız olarak çalışmalıdır. Eşzamanlı ödemeleri yönetmek için bir kuyruk (Bull, BullMQ veya benzer) kullanın:

```javascript
import Queue from 'bull';

const paymentQueue = new Queue('payments', {
  redis: { host: '127.0.0.1', port: 6379 },
});

paymentQueue.process(5, async (job) => {
  // Aynı anda en fazla 5 ödemeyi işleyin
  const { recipient, amount } = job.data;
  return await processPayment(recipient, amount);
});

// Kuyruğa bir ödeme ekleyin
await paymentQueue.add({
  recipient: 'TRecipientAddressHere',
  amount: 100,
});
```

### İzleme ve Uyarı

Operasyonel görünürlük için önemli metrikleri izleyin:

- **Saatlik ödemeler** - verim izleme.
- **Ortalama tasarruf yüzdesi** - tasarruf yüzde 80'in altına düşerse, sağlayıcı fiyatlandırmasını araştırın.
- **MERX bakiyesi** - bakiye bir eşiğin altına düşünce uyarı verin (örneğin 100 transfer için yeterli).
- **Doldurma zamanı** - enerji delegasyonu 30 saniyeden uzun sürerse, sağlayıcılar tıkış olabilir.
- **Başarısızlık oranı** - herhangi bir adımda başarısız olan ödeme denemelerinin yüzdesi.

Bunları tercih ettiğiniz izleme aracında izleyin (Prometheus, Datadog veya basit bir JSON günlüğü).

### Hız Sınırları

MERX API, siparişin oluşturulmasını dakikada 10 istekle sınırlandırır. Botunuzun dakikada 10'dan fazla ödeme göndermesi gerekiyorsa, iki seçeneğiniz vardır:

1. **Ödemeleri kuyruğa alın** ve sınırı içinde işleyin.
2. **Toplu enerji satın alın** - birden fazla transfer için tek bir siparişte yeterli enerji satın alın, ardından enerji aktif iken transferleri sıra ile gönderin.

Seçenek 2, yüksek hacimli senaryolar için daha verimlidir:

```javascript
async function processBatch(payments) {
  // Tüm ödemeler için toplam enerjiyi hesaplayın
  const totalEnergy = payments.length * 65000; // USDT transferi başına ~65K

  // Tüm toplu iş için tek enerji siparişi
  const order = await createEnergyOrder(totalEnergy, SENDER_ADDRESS, 1);
  await waitForFill(order.orderId);

  // Enerji aktif iken tüm USDT transferlerini gönderin
  const results = [];
  for (const payment of payments) {
    try {
      const tx = await sendUSDT(payment.recipient, payment.amount);
      results.push({ success: true, ...tx });
    } catch (err) {
      results.push({ success: false, error: err.message, ...payment });
    }
  }

  return results;
}
```

### Güvenlik Kontrol Listesi

Üretim ortamına dağıtmadan önce:

- TRON özel anahtarını bir sırlar yöneticisinde depolayın, disk üzerinde `.env` dosyasında değil.
- Botu, webhook uç noktası hariç gelen ağ erişimi olmayan kısıtlı bir ortamda çalıştırın.
- Bot cüzdanı için ayrılmış bir TRON cüzdanı kullanın, sadece yakın vadeli ödemeler için gereken USDT ile. Tüm hazinesini bot cüzdanında tutmayın.
- Para çekme limitleri uygulayın - bot tek bir çalışmada cüzdanı tükeyememelidir.
- Denetim izleri için tüm ayrıntılarla her ödemeyi günlüğe kaydedin.
- Olağan dışı aktivite için uyarılar ayarlayın (eşikten yüksek ödeme tutarları, hızlı ardışık başarısızlıklar).

## Tam Proje Yapısı

```
usdt-payment-bot/
  .env                  # Asla kaydetmeyin
  .gitignore
  package.json
  index.js              # Giriş noktası
  src/
    clients.js          # MERX + TronWeb başlatması
    balance.js          # Bakiye kontrolü
    estimate.js         # Maliyet tahmini
    order.js            # Enerji siparişi oluşturma
    wait.js             # Sipariş doldurma yoklaması
    transfer.js         # USDT transfer yürütme
    bot.js              # Ana düzenleme
    webhook-handler.js  # Webhook alıcı (isteğe bağlı)
```

Her dosya tek bir sorumluluğa sahiptir. Toplam kod tabanı 300 satırdan azdır. Birim testleri için MERX istemcisini simüle edin, entegrasyon testleri için Shasta testnetini kullanın.

## Sonuç

Bu ödeme botu temel MERX entegrasyon modelini gösterir: tahmin et, sipariş ver, bekle, işlem yap. Aynı model, bir ödeme işlemcisi, cüzdan, borsanın para çekme sistemi veya TRC-20 token gönderen herhangi bir uygulama oluştururken geçerlidir.

Önemli sonuç, tasarruf matematiktedir. Transfer başına yüzde 94 tasarrufla, günde 1.000 USDT transfer işleyen bir bot günde yaklaşık 26.000 TRX tasarruf eder - ayda 750.000 TRX'den fazla. Bu bir optimizasyon değildir. Bu, temel bir maliyet yapısı değişikliğidir.

Gerçek fon risksiz bir şekilde doğrulamak için Shasta testnetinde başlayın (`TRON_FULL_HOST=https://api.shasta.trongrid.io`), her şey çalıştığında mainnet'e geçin.

- MERX platformu: [merx.exchange](https://merx.exchange)
- API dokümantasyonu: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- Python SDK: [github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)
- AI aracıları için MCP sunucusu: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx