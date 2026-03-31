# 构建购买 TRON 能量的 Telegram 机器人

Telegram 是加密社区的事实标准通信平台。如果你管理 TRON 钱包、运行 dApp,或只是想要一种方便的方式在不打开浏览器的情况下购买能量,Telegram 机器人是一个实用工具。本教程将引导你构建一个完整的 Telegram 机器人,它可以查看能量价格、通过 MERX 购买能量,并在订单完成时发送通知。

## 前提条件

- Node.js 18 或更高版本
- Telegram 机器人令牌(来自 @BotFather)
- MERX API 密钥(来自 [https://merx.exchange](https://merx.exchange))

## 项目设置

```bash
mkdir tron-energy-bot
cd tron-energy-bot
npm init -y
npm install telegraf merx-sdk dotenv express
npm install -D typescript @types/node @types/express ts-node
```

环境变量:

```bash
# .env
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
MERX_API_KEY=your_merx_api_key
WEBHOOK_PORT=3001
WEBHOOK_URL=https://your-server.com/webhooks/merx
```

## 核心实现

```typescript
// src/index.ts
import { Telegraf, Context } from 'telegraf';
import { MerxClient } from 'merx-sdk';
import express from 'express';
import dotenv from 'dotenv';

dotenv.config();

const bot = new Telegraf(process.env.TELEGRAM_BOT_TOKEN!);
const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY! });
const orderChatMap = new Map<string, number>();

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

const app = express();
app.use(express.json());
app.post('/webhooks/merx', handleMerxWebhook);

const PORT = parseInt(process.env.WEBHOOK_PORT || '3001');
app.listen(PORT, () => console.log(`Webhook server listening on port ${PORT}`));
bot.launch().then(() => console.log('Telegram bot started'));
```

### /price 命令

```typescript
async function handlePrice(ctx: Context): Promise<void> {
  const args = (ctx.message as any).text.split(' ').slice(1);
  const amount = parseInt(args[0]);
  const duration = args[1] || '1h';

  const prices = await merx.getPrices({ energy_amount: amount, duration });

  let response = `Energy prices for ${amount.toLocaleString()} units (${duration}):\n\n`;
  for (const offer of prices.providers) {
    const totalTrx = (offer.price_sun * amount) / 1e6;
    const marker = offer.provider === prices.best.provider ? ' << BEST' : '';
    response += `${offer.provider}: ${offer.price_sun} SUN (${totalTrx.toFixed(2)} TRX)${marker}\n`;
  }
  await ctx.reply(response);
}
```

### /buy 命令

```typescript
async function handleBuy(ctx: Context): Promise<void> {
  const args = (ctx.message as any).text.split(' ').slice(1);
  const amount = parseInt(args[0]);
  const duration = args[1];
  const targetAddress = args[2];

  const order = await merx.createOrder({
    energy_amount: amount, duration, target_address: targetAddress
  });

  orderChatMap.set(order.id, ctx.chat!.id);
  await ctx.reply(`Order placed.\nOrder ID: ${order.id}\nYou will be notified when the order fills.`);
}
```

### Webhook 处理器

```typescript
async function handleMerxWebhook(req: Request, res: Response): Promise<void> {
  const event = req.body;
  if (event.type === 'order.filled') {
    const chatId = orderChatMap.get(event.data.order_id);
    if (chatId) {
      await bot.telegram.sendMessage(chatId,
        `Order filled.\nEnergy: ${event.data.energy_amount}\nProvider: ${event.data.provider}`
      );
      orderChatMap.delete(event.data.order_id);
    }
  }
  res.status(200).json({ received: true });
}
```

## 生产考虑

### 持久存储

生产环境使用 Redis 替代内存中的 `orderChatMap`:

```typescript
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL);

async function storeOrderChat(orderId: string, chatId: number): Promise<void> {
  await redis.set(`order:${orderId}:chat`, chatId.toString(), 'EX', 86400);
}
```

### 频率限制

```typescript
const RATE_LIMIT_MS = 5000;
bot.use(async (ctx, next) => {
  const userId = ctx.from?.id;
  if (userId && isRateLimited(userId)) {
    await ctx.reply('Please wait a few seconds between commands.');
    return;
  }
  return next();
});
```

## 部署

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY dist/ ./dist/
CMD ["node", "dist/index.js"]
```

## 结论

用三个核心命令和 webhook 集成的 Telegram 机器人,用户可以在不离开 Telegram 的情况下查价格、买能量并接收通知。实现利用 MERX SDK 进行所有能量市场交互,自动获得跨七个供应商的最优价格路由。

API 文档: [https://merx.exchange/docs](https://merx.exchange/docs)。MCP 服务器: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)。
