# Building a Telegram Bot That Buys TRON Energy

Telegram is the de facto communication platform for the crypto community. If you manage TRON wallets, run a dApp, or simply want a convenient way to purchase energy without opening a browser, a Telegram bot is a practical tool. This tutorial walks through building a complete Telegram bot that checks energy prices, purchases energy through MERX, and sends notifications when orders fill.

By the end of this article, you will have a working bot with three commands -- `/price`, `/buy`, and `/balance` -- plus webhook integration for real-time order status updates.

## Prerequisites

- Node.js 18 or later
- A Telegram bot token (from @BotFather)
- A MERX API key (from [https://merx.exchange](https://merx.exchange))
- Basic familiarity with TypeScript

## Project Setup

```bash
mkdir tron-energy-bot
cd tron-energy-bot
npm init -y
npm install telegraf merx-sdk dotenv express
npm install -D typescript @types/node @types/express ts-node
```

Create the TypeScript configuration:

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

Set up your environment variables:

```bash
# .env
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
MERX_API_KEY=your_merx_api_key
WEBHOOK_PORT=3001
WEBHOOK_URL=https://your-server.com/webhooks/merx
```

## Bot Architecture

The bot has three layers:

1. **Telegram command handlers** -- Parse user commands and respond
2. **MERX client** -- Interact with the energy market
3. **Webhook server** -- Receive asynchronous order notifications

```
User sends /price 65000
       |
       v
[Telegram Bot] -- parses command
       |
       v
[MERX Client] -- queries prices
       |
       v
[Telegram Bot] -- formats and sends response
       |
       v
User receives price table

User sends /buy 65000 1h TAddress
       |
       v
[Telegram Bot] -- validates input
       |
       v
[MERX Client] -- creates order
       |
       v
[Webhook Server] -- receives order.filled
       |
       v
[Telegram Bot] -- notifies user
```

## Core Implementation

### Entry Point

```typescript
// src/index.ts
import { Telegraf, Context } from 'telegraf';
import { MerxClient } from 'merx-sdk';
import express from 'express';
import dotenv from 'dotenv';

dotenv.config();

const bot = new Telegraf(process.env.TELEGRAM_BOT_TOKEN!);
const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY! });

// Store chat IDs for order notifications
const orderChatMap = new Map<string, number>();

// --- Command Handlers ---

bot.command('start', (ctx) => {
  ctx.reply(
    'TRON Energy Bot\n\n' +
    'Commands:\n' +
    '/price <amount> - Check energy prices\n' +
    '/buy <amount> <duration> <address> - Buy energy\n' +
    '/balance - Check your MERX balance\n\n' +
    'Example:\n' +
    '/price 65000\n' +
    '/buy 65000 1h TYourAddress123'
  );
});

bot.command('price', handlePrice);
bot.command('buy', handleBuy);
bot.command('balance', handleBalance);

// --- Webhook Server ---

const app = express();
app.use(express.json());

app.post('/webhooks/merx', handleMerxWebhook);

// --- Start ---

const PORT = parseInt(process.env.WEBHOOK_PORT || '3001');

app.listen(PORT, () => {
  console.log(`Webhook server listening on port ${PORT}`);
});

bot.launch().then(() => {
  console.log('Telegram bot started');
});

process.once('SIGINT', () => bot.stop('SIGINT'));
process.once('SIGTERM', () => bot.stop('SIGTERM'));
```

### The /price Command

The `/price` command queries MERX for current energy prices across all providers:

```typescript
// src/handlers/price.ts
async function handlePrice(ctx: Context): Promise<void> {
  const args = (ctx.message as any).text.split(' ').slice(1);

  if (args.length < 1) {
    await ctx.reply(
      'Usage: /price <energy_amount> [duration]\n' +
      'Example: /price 65000\n' +
      'Example: /price 65000 1h'
    );
    return;
  }

  const amount = parseInt(args[0]);
  if (isNaN(amount) || amount < 10000) {
    await ctx.reply('Energy amount must be a number >= 10,000');
    return;
  }

  const duration = args[1] || '1h';

  try {
    await ctx.reply('Checking prices across 7 providers...');

    const prices = await merx.getPrices({
      energy_amount: amount,
      duration: duration
    });

    let response = `Energy prices for ${amount.toLocaleString()} units (${duration}):\n\n`;

    // Format provider prices as a table
    for (const offer of prices.providers) {
      const totalTrx = (offer.price_sun * amount) / 1e6;
      const marker = offer.provider === prices.best.provider
        ? ' << BEST'
        : '';

      response +=
        `${offer.provider}: ${offer.price_sun} SUN ` +
        `(${totalTrx.toFixed(2)} TRX)${marker}\n`;
    }

    const bestTotal = (prices.best.price_sun * amount) / 1e6;
    response += `\nBest price: ${prices.best.price_sun} SUN `;
    response += `via ${prices.best.provider}\n`;
    response += `Total cost: ${bestTotal.toFixed(2)} TRX`;

    await ctx.reply(response);
  } catch (error: any) {
    await ctx.reply(`Error fetching prices: ${error.message}`);
  }
}
```

### The /buy Command

The `/buy` command places an energy order through MERX:

```typescript
// src/handlers/buy.ts
async function handleBuy(ctx: Context): Promise<void> {
  const args = (ctx.message as any).text.split(' ').slice(1);

  if (args.length < 3) {
    await ctx.reply(
      'Usage: /buy <amount> <duration> <tron_address>\n' +
      'Example: /buy 65000 1h TYourAddress123\n\n' +
      'Durations: 5m, 10m, 30m, 1h, 3h, 6h, 12h, 1d, 3d, 14d'
    );
    return;
  }

  const amount = parseInt(args[0]);
  const duration = args[1];
  const targetAddress = args[2];

  // Validate inputs
  if (isNaN(amount) || amount < 10000) {
    await ctx.reply('Energy amount must be a number >= 10,000');
    return;
  }

  if (!isValidTronAddress(targetAddress)) {
    await ctx.reply('Invalid TRON address. Must start with T.');
    return;
  }

  const validDurations = [
    '5m', '10m', '30m', '1h', '3h',
    '6h', '12h', '1d', '3d', '14d'
  ];
  if (!validDurations.includes(duration)) {
    await ctx.reply(
      `Invalid duration. Choose from: ${validDurations.join(', ')}`
    );
    return;
  }

  try {
    // Get the best price first
    const prices = await merx.getPrices({
      energy_amount: amount,
      duration: duration
    });

    const totalTrx =
      (prices.best.price_sun * amount) / 1e6;

    await ctx.reply(
      `Placing order:\n` +
      `Amount: ${amount.toLocaleString()} energy\n` +
      `Duration: ${duration}\n` +
      `Target: ${targetAddress}\n` +
      `Price: ${prices.best.price_sun} SUN ` +
      `(${totalTrx.toFixed(2)} TRX)\n` +
      `Provider: ${prices.best.provider}\n\n` +
      `Processing...`
    );

    const order = await merx.createOrder({
      energy_amount: amount,
      duration: duration,
      target_address: targetAddress
    });

    // Store the chat ID for webhook notification
    orderChatMap.set(order.id, ctx.chat!.id);

    await ctx.reply(
      `Order placed.\n` +
      `Order ID: ${order.id}\n` +
      `Status: ${order.status}\n\n` +
      `You will be notified when the order fills.`
    );
  } catch (error: any) {
    await ctx.reply(`Error placing order: ${error.message}`);
  }
}

function isValidTronAddress(address: string): boolean {
  return /^T[1-9A-HJ-NP-Za-km-z]{33}$/.test(address);
}
```

### The /balance Command

```typescript
// src/handlers/balance.ts
async function handleBalance(ctx: Context): Promise<void> {
  try {
    const balance = await merx.getBalance();

    await ctx.reply(
      `MERX Account Balance:\n\n` +
      `Available: ${balance.available_trx} TRX\n` +
      `Reserved (in orders): ${balance.reserved_trx} TRX\n` +
      `Total: ${balance.total_trx} TRX`
    );
  } catch (error: any) {
    await ctx.reply(`Error fetching balance: ${error.message}`);
  }
}
```

### Webhook Handler

The webhook handler receives order status notifications from MERX and forwards them to the user via Telegram:

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
            `Order filled.\n\n` +
            `Order ID: ${event.data.order_id}\n` +
            `Energy: ${event.data.energy_amount.toLocaleString()}\n` +
            `Provider: ${event.data.provider}\n` +
            `Price: ${event.data.price_sun} SUN\n` +
            `Target: ${event.data.target_address}\n\n` +
            `Energy has been delegated to the target address.`
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
            `Order failed.\n\n` +
            `Order ID: ${event.data.order_id}\n` +
            `Reason: ${event.data.reason}\n\n` +
            `Your balance has been refunded.`
          );
          orderChatMap.delete(event.data.order_id);
        }
        break;
      }
    }
  } catch (error) {
    console.error('Webhook processing error:', error);
  }

  res.status(200).json({ received: true });
}
```

## Adding Price Alerts

Extend the bot with a `/alert` command that uses MERX standing orders to notify users when prices drop below a threshold:

```typescript
bot.command('alert', async (ctx) => {
  const args = (ctx.message as any).text.split(' ').slice(1);

  if (args.length < 2) {
    await ctx.reply(
      'Usage: /alert <energy_amount> <max_price_sun>\n' +
      'Example: /alert 65000 25\n\n' +
      'You will be notified when energy price drops below the target.'
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
      `Price alert set.\n\n` +
      `Watching for: ${amount.toLocaleString()} energy ` +
      `at or below ${maxPrice} SUN\n` +
      `Alert ID: ${standing.id}\n\n` +
      `You will be notified when this price is available.`
    );
  } catch (error: any) {
    await ctx.reply(`Error setting alert: ${error.message}`);
  }
});
```

## Production Considerations

### Persistent Storage

The in-memory `orderChatMap` used above is fine for development but loses data on restart. For production, use Redis or a database:

```typescript
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL);

async function storeOrderChat(
  orderId: string,
  chatId: number
): Promise<void> {
  // Store with 24-hour TTL
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

### User Authentication

For multi-user bots, associate Telegram user IDs with MERX accounts:

```typescript
// Store user API keys securely
async function setUserApiKey(
  telegramId: number,
  apiKey: string
): Promise<void> {
  // Encrypt before storing
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

### Rate Limiting

Prevent abuse by limiting command frequency:

```typescript
const rateLimits = new Map<number, number>();
const RATE_LIMIT_MS = 5000; // 5 seconds between commands

function isRateLimited(userId: number): boolean {
  const lastCommand = rateLimits.get(userId) || 0;
  const now = Date.now();

  if (now - lastCommand < RATE_LIMIT_MS) {
    return true;
  }

  rateLimits.set(userId, now);
  return false;
}

// Apply to all commands
bot.use(async (ctx, next) => {
  const userId = ctx.from?.id;
  if (userId && isRateLimited(userId)) {
    await ctx.reply('Please wait a few seconds between commands.');
    return;
  }
  return next();
});
```

### Error Handling

Wrap all command handlers with consistent error handling:

```typescript
function withErrorHandling(
  handler: (ctx: Context) => Promise<void>
) {
  return async (ctx: Context) => {
    try {
      await handler(ctx);
    } catch (error: any) {
      console.error('Command error:', error);
      await ctx.reply(
        `An error occurred: ${error.message}\n` +
        `Please try again or contact support.`
      );
    }
  };
}

bot.command('price', withErrorHandling(handlePrice));
bot.command('buy', withErrorHandling(handleBuy));
bot.command('balance', withErrorHandling(handleBalance));
```

## Deployment

### Running with PM2

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

## Testing the Bot

1. Start the bot: `npx ts-node src/index.ts`
2. Open Telegram and find your bot
3. Send `/start` to see available commands
4. Send `/price 65000` to check current energy prices
5. Send `/buy 65000 1h TYourAddress` to place an order
6. Wait for the webhook notification confirming the order filled

## Conclusion

A Telegram bot for TRON energy purchasing turns a web-based workflow into a conversational interface. With three core commands and webhook integration, users can check prices, buy energy, and receive fill notifications without leaving Telegram.

The implementation leverages the MERX SDK for all energy market interactions, which means the bot automatically gets best-price routing across seven providers, standing order support for price alerts, and reliable order execution with failover.

The complete source code in this article is production-ready with the addition of persistent storage and proper authentication. The total implementation is under 300 lines of TypeScript.

For API documentation, visit [https://merx.exchange/docs](https://merx.exchange/docs). For the MCP server integration, see [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp).


## Try It Now with AI

Add MERX to Claude Desktop or any MCP-compatible client -- zero install, no API key needed for read-only tools:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Ask your AI agent: "What is the cheapest TRON energy right now?" and get live prices from all connected providers.

Full MCP documentation: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)
