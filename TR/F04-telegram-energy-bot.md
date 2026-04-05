# TRON Enerjisi Satın Alan Telegram Botu Oluşturma

Telegram, kripto topluluğunun fiili iletişim platformudur. TRON cüzdanlarını yönetiyorsanız, bir dApp çalıştırıyorsanız veya basitçe tarayıcı açmadan enerji satın almanın uygun bir yolunu istiyorsanız, bir Telegram botu pratik bir araçtır. Bu eğitim, enerji fiyatlarını kontrol eden, MERX aracılığıyla enerji satın alan ve siparişler dolduğunda bildirim gönderen eksiksiz bir Telegram botu oluşturmanın adımlarını açıklamaktadır.

Bu makalenin sonunda, üç komut (`/price`, `/buy` ve `/balance`) içeren ve gerçek zamanlı sipariş durumu güncellemeleri için webhook entegrasyonu sunan çalışan bir bota sahip olacaksınız.

## Ön Koşullar

- Node.js 18 veya sonrası
- Bir Telegram bot tokeni (@BotFather'dan)
- Bir MERX API anahtarı ([https://merx.exchange](https://merx.exchange) adresinden)
- TypeScript konusunda temel bilgi

## Proje Kurulumu

```bash
mkdir tron-energy-bot
cd tron-energy-bot
npm init -y
npm install telegraf merx-sdk dotenv express
npm install -D typescript @types/node @types/express ts-node
```

TypeScript yapılandırmasını oluşturun:

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"]
}
```

Ortam değişkenlerinizi ayarlayın:

```bash
# .env
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
MERX_API_KEY=your_merx_api_key
WEBHOOK_PORT=3001
WEBHOOK_URL=https://your-server.com/webhooks/merx
```

## Bot Mimarisi

Bot üç katmandan oluşur:

1. **Telegram komut işleyicileri** -- Kullanıcı komutlarını ayrıştırır ve yanıt verir
2. **MERX istemcisi** -- Enerji pazarı ile etkileşim kurar
3. **Webhook sunucusu** -- Asenkron sipariş bildirimlerini alır

```
Kullanıcı /price 65000 gönderir
       |
       v
[Telegram Bot] -- komutu ayrıştırır
       |
       v
[MERX İstemcisi] -- fiyatları sorgular
       |
       v
[Telegram Bot] -- yanıtı biçimlendirir ve gönderir
       |
       v
Kullanıcı fiyat tablosunu alır

Kullanıcı /buy 65000 1h TAdres gönderir
       |
       v
[Telegram Bot] -- girdiyi doğrular
       |
       v
[MERX İstemcisi] -- sipariş oluşturur
       |
       v
[Webhook Sunucusu] -- order.filled alır
       |
       v
[Telegram Bot] -- kullanıcıyı bilgilendirir
```

## Temel Uygulama

### Giriş Noktası

```typescript
// src/index.ts
import { Telegraf, Context } from 'telegraf';
import { MerxClient } from 'merx-sdk';
import express from 'express';
import dotenv from 'dotenv';

dotenv.config();

const bot = new Telegraf(process.env.TELEGRAM_BOT_TOKEN!);
const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY! });

// Sipariş bildirimleri için sohbet kimliklerini depolayın
const orderChatMap = new Map<string, number>();

// --- Komut İşleyicileri ---

bot.command('start', (ctx) => {
  ctx.reply(
    'TRON Enerji Botu\n\n' +
    'Komutlar:\n' +
    '/price <miktar> - Enerji fiyatlarını kontrol et\n' +
    '/buy <miktar> <süre> <adres> - Enerji satın al\n' +
    '/balance - MERX bakiyenizi kontrol et\n\n' +
    'Örnek:\n' +
    '/price 65000\n' +
    '/buy 65000 1h TAdresiniz123'
  );
});

bot.command('price', handlePrice);
bot.command('buy', handleBuy);
bot.command('balance', handleBalance);

// --- Webhook Sunucusu ---

const app = express();
app.use(express.json());

app.post('/webhooks/merx', handleMerxWebhook);

// --- Başlat ---

const PORT = parseInt(process.env.WEBHOOK_PORT || '3001');

app.listen(PORT, () => {
  console.log(`Webhook sunucusu ${PORT} portunda dinleniyor`);
});

bot.launch().then(() => {
  console.log('Telegram botu başlatıldı');
});

process.once('SIGINT', () => bot.stop('SIGINT'));
process.once('SIGTERM', () => bot.stop('SIGTERM'));
```

### /price Komutu

`/price` komutu, tüm sağlayıcılar arasında geçerli enerji fiyatları için MERX'i sorgular:

```typescript
// src/handlers/price.ts
async function handlePrice(ctx: Context): Promise<void> {
  const args = (ctx.message as any).text.split(' ').slice(1);

  if (args.length < 1) {
    await ctx.reply(
      'Kullanım: /price <enerji_miktarı> [süre]\n' +
      'Örnek: /price 65000\n' +
      'Örnek: /price 65000 1h'
    );
    return;
  }

  const amount = parseInt(args[0]);
  if (isNaN(amount) || amount < 10000) {
    await ctx.reply('Enerji miktarı 10.000 veya daha büyük bir sayı olmalıdır');
    return;
  }

  const duration = args[1] || '1h';

  try {
    await ctx.reply('7 sağlayıcı arasında fiyatlar kontrol ediliyor...');

    const prices = await merx.getPrices({
      energy_amount: amount,
      duration: duration
    });

    let response = `${amount.toLocaleString()} birim enerji fiyatları (${duration}):\n\n`;

    // Sağlayıcı fiyatlarını tablo olarak biçimlendir
    for (const offer of prices.providers) {
      const totalTrx = (offer.price_sun * amount) / 1e6;
      const marker = offer.provider === prices.best.provider
        ? ' << EN İYİSİ'
        : '';

      response +=
        `${offer.provider}: ${offer.price_sun} SUN ` +
        `(${totalTrx.toFixed(2)} TRX)${marker}\n`;
    }

    const bestTotal = (prices.best.price_sun * amount) / 1e6;
    response += `\nEn iyi fiyat: ${prices.best.price_sun} SUN `;
    response += `${prices.best.provider} aracılığıyla\n`;
    response += `Toplam maliyet: ${bestTotal.toFixed(2)} TRX`;

    await ctx.reply(response);
  } catch (error: any) {
    await ctx.reply(`Fiyatları alınırken hata: ${error.message}`);
  }
}
```

### /buy Komutu

`/buy` komutu MERX aracılığıyla bir enerji siparişi yerleştirir:

```typescript
// src/handlers/buy.ts
async function handleBuy(ctx: Context): Promise<void> {
  const args = (ctx.message as any).text.split(' ').slice(1);

  if (args.length < 3) {
    await ctx.reply(
      'Kullanım: /buy <miktar> <süre> <tron_adresi>\n' +
      'Örnek: /buy 65000 1h TAdresiniz123\n\n' +
      'Süreler: 5m, 10m, 30m, 1h, 3h, 6h, 12h, 1d, 3d, 14d'
    );
    return;
  }

  const amount = parseInt(args[0]);
  const duration = args[1];
  const targetAddress = args[2];

  // Girdiyi doğrula
  if (isNaN(amount) || amount < 10000) {
    await ctx.reply('Enerji miktarı 10.000 veya daha büyük bir sayı olmalıdır');
    return;
  }

  if (!isValidTronAddress(targetAddress)) {
    await ctx.reply('Geçersiz TRON adresi. T ile başlamalıdır.');
    return;
  }

  const validDurations = [
    '5m', '10m', '30m', '1h', '3h',
    '6h', '12h', '1d', '3d', '14d'
  ];
  if (!validDurations.includes(duration)) {
    await ctx.reply(
      `Geçersiz süre. Bunlardan seçin: ${validDurations.join(', ')}`
    );
    return;
  }

  try {
    // Önce en iyi fiyatı alın
    const prices = await merx.getPrices({
      energy_amount: amount,
      duration: duration
    });

    const totalTrx =
      (prices.best.price_sun * amount) / 1e6;

    await ctx.reply(
      `Sipariş yerleştiriliyor:\n` +
      `Miktar: ${amount.toLocaleString()} enerji\n` +
      `Süre: ${duration}\n` +
      `Hedef: ${targetAddress}\n` +
      `Fiyat: ${prices.best.price_sun} SUN ` +
      `(${totalTrx.toFixed(2)} TRX)\n` +
      `Sağlayıcı: ${prices.best.provider}\n\n` +
      `İşleniyor...`
    );

    const order = await merx.createOrder({
      energy_amount: amount,
      duration: duration,
      target_address: targetAddress
    });

    // Webhook bildirimi için sohbet kimliğini depolayın
    orderChatMap.set(order.id, ctx.chat!.id);

    await ctx.reply(
      `Sipariş yerleştirildi.\n` +
      `Sipariş Kimliği: ${order.id}\n` +
      `Durum: ${order.status}\n\n` +
      `Sipariş dolduğunda bilgilendirileceksiniz.`
    );
  } catch (error: any) {
    await ctx.reply(`Sipariş yerleştirilirken hata: ${error.message}`);
  }
}

function isValidTronAddress(address: string): boolean {
  return /^T[1-9A-HJ-NP-Za-km-z]{33}$/.test(address);
}
```

### /balance Komutu

```typescript
// src/handlers/balance.ts
async function handleBalance(ctx: Context): Promise<void> {
  try {
    const balance = await merx.getBalance();

    await ctx.reply(
      `MERX Hesap Bakiyesi:\n\n` +
      `Mevcut: ${balance.available_trx} TRX\n` +
      `Ayrılmış (siparişlerde): ${balance.reserved_trx} TRX\n` +
      `Toplam: ${balance.total_trx} TRX`
    );
  } catch (error: any) {
    await ctx.reply(`Bakiye alınırken hata: ${error.message}`);
  }
}
```

### Webhook İşleyici

Webhook işleyici MERX'ten sipariş durumu bildirimlerini alır ve Telegram aracılığıyla kullanıcıya iletir:

```typescript
// src/handlers/webhook.ts
import { Request, Response } from 'express';

async function handleMerxWebhook(
  req: Request,
  res: Response
): Promise<void> {
  const event = req.body;

  try {
    switch (event.type) {
      case 'order.filled': {
        const chatId = orderChatMap.get(event.data.order_id);
        if (chatId) {
          await bot.telegram.sendMessage(
            chatId,
            `Sipariş dolduruldu.\n\n` +
            `Sipariş Kimliği: ${event.data.order_id}\n` +
            `Enerji: ${event.data.energy_amount.toLocaleString()}\n` +
            `Sağlayıcı: ${event.data.provider}\n` +
            `Fiyat: ${event.data.price_sun} SUN\n` +
            `Hedef: ${event.data.target_address}\n\n` +
            `Enerji hedef adrese verildi.`
          );
          orderChatMap.delete(event.data.order_id);
        }
        break;
      }

      case 'order.failed': {
        const chatId = orderChatMap.get(event.data.order_id);
        if (chatId) {
          await bot.telegram.sendMessage(
            chatId,
            `Sipariş başarısız oldu.\n\n` +
            `Sipariş Kimliği: ${event.data.order_id}\n` +
            `Neden: ${event.data.reason}\n\n` +
            `Bakiyeniz iade edildi.`
          );
          orderChatMap.delete(event.data.order_id);
        }
        break;
      }
    }
  } catch (error) {
    console.error('Webhook işleme hatası:', error);
  }

  res.status(200).json({ received: true });
}
```

## Fiyat Uyarıları Ekleme

Botu, kullanıcıları fiyatlar bir eşiğin altına düştüğünde bilgilendirmek için MERX kalıcı siparişlerini kullanan bir `/alert` komutuyla genişletin:

```typescript
bot.command('alert', async (ctx) => {
  const args = (ctx.message as any).text.split(' ').slice(1);

  if (args.length < 2) {
    await ctx.reply(
      'Kullanım: /alert <enerji_miktarı> <maksimum_fiyat_sun>\n' +
      'Örnek: /alert 65000 25\n\n' +
      'Enerji fiyatı hedefinizin altına düştüğünde bilgilendirileceksiniz.'
    );
    return;
  }

  const amount = parseInt(args[0]);
  const maxPrice = parseInt(args[1]);

  try {
    const standing = await merx.createStandingOrder({
      energy_amount: amount,
      max_price_sun: maxPrice,
      duration: '1h',
      repeat: false
    });

    orderChatMap.set(standing.id, ctx.chat!.id);

    await ctx.reply(
      `Fiyat uyarısı ayarlandı.\n\n` +
      `İzlenen: ${amount.toLocaleString()} enerji ` +
      `${maxPrice} SUN veya altında\n` +
      `Uyarı Kimliği: ${standing.id}\n\n` +
      `Bu fiyat mevcut olduğunda bilgilendirileceksiniz.`
    );
  } catch (error: any) {
    await ctx.reply(`Uyarı ayarlanırken hata: ${error.message}`);
  }
});
```

## Üretim Konuları

### Kalıcı Depolama

Yukarıda kullanılan bellek içi `orderChatMap` geliştirme için iyidir ancak yeniden başlatmada verileri kaybeder. Üretim için Redis veya bir veritabanı kullanın:

```typescript
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL);

async function storeOrderChat(
  orderId: string,
  chatId: number
): Promise<void> {
  // 24 saatlik TTL ile depolayın
  await redis.set(
    `order:${orderId}:chat`,
    chatId.toString(),
    'EX',
    86400
  );
}

async function getOrderChat(
  orderId: string
): Promise<number | null> {
  const chatId = await redis.get(`order:${orderId}:chat`);
  return chatId ? parseInt(chatId) : null;
}
```

### Kullanıcı Kimlik Doğrulaması

Çok kullanıcılı botlar için Telegram kullanıcı kimliklerini MERX hesaplarıyla ilişkilendirin:

```typescript
// Kullanıcı API anahtarlarını güvenli şekilde depolayın
async function setUserApiKey(
  telegramId: number,
  apiKey: string
): Promise<void> {
  // Depolamadan önce şifreleyin
  const encrypted = encrypt(apiKey);
  await redis.set(
    `user:${telegramId}:apikey`,
    encrypted
  );
}

async function getUserClient(
  telegramId: number
): Promise<MerxClient | null> {
  const encrypted = await redis.get(
    `user:${telegramId}:apikey`
  );
  if (!encrypted) return null;

  const apiKey = decrypt(encrypted);
  return new MerxClient({ apiKey });
}
```

### Hız Sınırlaması

Komut sıklığını sınırlandırarak kötüye kullanımı önleyin:

```typescript
const rateLimits = new Map<number, number>();
const RATE_LIMIT_MS = 5000; // Komutlar arasında 5 saniye

function isRateLimited(userId: number): boolean {
  const lastCommand = rateLimits.get(userId) || 0;
  const now = Date.now();

  if (now - lastCommand < RATE_LIMIT_MS) {
    return true;
  }

  rateLimits.set(userId, now);
  return false;
}

// Tüm komutlara uygulayın
bot.use(async (ctx, next) => {
  const userId = ctx.from?.id;
  if (userId && isRateLimited(userId)) {
    await ctx.reply('Lütfen komutlar arasında birkaç saniye bekleyin.');
    return;
  }
  return next();
});
```

### Hata Yönetimi

Tüm komut işleyicilerini tutarlı hata yönetimiyle sarın:

```typescript
function withErrorHandling(
  handler: (ctx: Context) => Promise<void>
) {
  return async (ctx: Context) => {
    try {
      await handler(ctx);
    } catch (error: any) {
      console.error('Komut hatası:', error);
      await ctx.reply(
        `Bir hata oluştu: ${error.message}\n` +
        `Lütfen tekrar deneyin veya desteğe başvurun.`
      );
    }
  };
}

bot.command('price', withErrorHandling(handlePrice));
bot.command('buy', withErrorHandling(handleBuy));
bot.command('balance', withErrorHandling(handleBalance));
```

## Dağıtım

### PM2 ile Çalıştırma

```bash
npm run build
pm2 start dist/index.js --name "tron-energy-bot"
pm2 save
```

### Docker

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY dist/ ./dist/
CMD ["node", "dist/index.js"]
```

```bash
docker build -t tron-energy-bot .
docker run -d \
  --name tron-energy-bot \
  --env-file .env \
  -p 3001:3001 \
  tron-energy-bot
```

## Botu Test Etme

1. Botu başlatın: `npx ts-node src/index.ts`
2. Telegram'ı açın ve botunuzu bulun
3. Kullanılabilir komutları görmek için `/start` gönderin
4. Geçerli enerji fiyatlarını kontrol etmek için `/price 65000` gönderin
5. Sipariş yerleştirmek için `/buy 65000 1h TAdresiniz` gönderin
6. Siparişin doldurulduğunu doğrulayan webhook bildirimini bekleyin

## Sonuç

TRON enerji satın alınması için bir Telegram botu, web tabanlı bir iş akışını sohbetli bir arayüze dönüştürür. Üç temel komut ve webhook entegrasyonu ile, kullanıcılar Telegram'dan ayrılmadan fiyatları kontrol edebilir, enerji satın alabilir ve doldurma bildirimlerini alabilir.

Uygulama, tüm enerji pazarı etkileşimleri için MERX SDK'sını kullanır; bu, botun otomatik olarak yedi sağlayıcı arasında en iyi fiyat yönlendirmesini, fiyat uyarıları için kalıcı sipariş desteğini ve yük devretmeli güvenilir sipariş yürütmesini elde ettiği anlamına gelir.

Bu makalede yer alan tam kaynak kodu, kalıcı depolama ve uygun kimlik doğrulaması eklenmesiyle üretim için hazırdır. Toplam uygulama 300 satırdan az TypeScript'tir.

API belgeleri için [https://merx.exchange/docs](https://merx.exchange/docs) adresini ziyaret edin. MCP sunucusu entegrasyonu için [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp) adresine bakın.

## Yapay Zeka ile Şimdi Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin -- sıfır kurulum, salt okuma araçları için API anahtarı gerekmez:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Yapay zeka aracınıza şunu sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve tüm bağlı sağlayıcılardan gerçek zamanlı fiyatlar alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)