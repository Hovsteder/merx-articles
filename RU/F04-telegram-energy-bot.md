# Создание Telegram-бота для покупки TRON Energy

Telegram — это де-факто платформа общения для крипто-сообщества. Если вы управляете кошельками TRON, запускаете dApp или просто хотите удобный способ покупки энергии без открытия браузера, Telegram-бот — это практичный инструмент. Это руководство проведёт вас через создание полноценного Telegram-бота, который проверяет цены на энергию, покупает энергию через MERX и отправляет уведомления при исполнении заказов.

По окончании этой статьи у вас будет рабочий бот с тремя командами — `/price`, `/buy` и `/balance` — плюс интеграция webhook для обновления статуса заказов в реальном времени.

## Предварительные требования

- Node.js 18 или позже
- Токен Telegram-бота (от @BotFather)
- API ключ MERX (с [https://merx.exchange](https://merx.exchange))
- Базовое знакомство с TypeScript

## Настройка проекта

```bash
mkdir tron-energy-bot
cd tron-energy-bot
npm init -y
npm install telegraf merx-sdk dotenv express
npm install -D typescript @types/node @types/express ts-node
```

Создайте конфигурацию TypeScript:

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

Установите переменные окружения:

```bash
# .env
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
MERX_API_KEY=your_merx_api_key
WEBHOOK_PORT=3001
WEBHOOK_URL=https://your-server.com/webhooks/merx
```

## Архитектура бота

Бот состоит из трёх уровней:

1. **Обработчики команд Telegram** — анализируют команды пользователя и отвечают
2. **Клиент MERX** — взаимодействуют с рынком энергии
3. **Сервер webhook** — получают асинхронные уведомления о заказах

```
Пользователь отправляет /price 65000
       |
       v
[Telegram Bot] -- парсит команду
       |
       v
[MERX Client] -- запрашивает цены
       |
       v
[Telegram Bot] -- форматирует и отправляет ответ
       |
       v
Пользователь получает таблицу цен

Пользователь отправляет /buy 65000 1h TAddress
       |
       v
[Telegram Bot] -- проверяет входные данные
       |
       v
[MERX Client] -- создаёт заказ
       |
       v
[Webhook Server] -- получает order.filled
       |
       v
[Telegram Bot] -- уведомляет пользователя
```

## Основная реализация

### Точка входа

```typescript
// src/index.ts
import { Telegraf, Context } from 'telegraf';
import { MerxClient } from 'merx-sdk';
import express from 'express';
import dotenv from 'dotenv';

dotenv.config();

const bot = new Telegraf(process.env.TELEGRAM_BOT_TOKEN!);
const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY! });

// Сохраняем ID чатов для уведомлений о заказах
const orderChatMap = new Map<string, number>();

// --- Обработчики команд ---

bot.command('start', (ctx) => {
  ctx.reply(
    'TRON Energy Bot\n\n' +
    'Команды:\n' +
    '/price <amount> - Проверить цены на энергию\n' +
    '/buy <amount> <duration> <address> - Купить энергию\n' +
    '/balance - Проверить баланс MERX\n\n' +
    'Пример:\n' +
    '/price 65000\n' +
    '/buy 65000 1h TYourAddress123'
  );
});

bot.command('price', handlePrice);
bot.command('buy', handleBuy);
bot.command('balance', handleBalance);

// --- Сервер webhook ---

const app = express();
app.use(express.json());

app.post('/webhooks/merx', handleMerxWebhook);

// --- Запуск ---

const PORT = parseInt(process.env.WEBHOOK_PORT || '3001');

app.listen(PORT, () => {
  console.log(`Webhook сервер слушает на порту ${PORT}`);
});

bot.launch().then(() => {
  console.log('Telegram-бот запущен');
});

process.once('SIGINT', () => bot.stop('SIGINT'));
process.once('SIGTERM', () => bot.stop('SIGTERM'));
```

### Команда /price

Команда `/price` запрашивает текущие цены на энергию у MERX для всех поставщиков:

```typescript
// src/handlers/price.ts
async function handlePrice(ctx: Context): Promise<void> {
  const args = (ctx.message as any).text.split(' ').slice(1);

  if (args.length < 1) {
    await ctx.reply(
      'Использование: /price <energy_amount> [duration]\n' +
      'Пример: /price 65000\n' +
      'Пример: /price 65000 1h'
    );
    return;
  }

  const amount = parseInt(args[0]);
  if (isNaN(amount) || amount < 10000) {
    await ctx.reply('Количество энергии должно быть числом >= 10 000');
    return;
  }

  const duration = args[1] || '1h';

  try {
    await ctx.reply('Проверяю цены у 7 поставщиков...');

    const prices = await merx.getPrices({
      energy_amount: amount,
      duration: duration
    });

    let response = `Цены на энергию для ${amount.toLocaleString()} единиц (${duration}):\n\n`;

    // Форматируем цены поставщиков как таблицу
    for (const offer of prices.providers) {
      const totalTrx = (offer.price_sun * amount) / 1e6;
      const marker = offer.provider === prices.best.provider
        ? ' << ЛУЧШАЯ'
        : '';

      response +=
        `${offer.provider}: ${offer.price_sun} SUN ` +
        `(${totalTrx.toFixed(2)} TRX)${marker}\n`;
    }

    const bestTotal = (prices.best.price_sun * amount) / 1e6;
    response += `\nЛучшая цена: ${prices.best.price_sun} SUN `;
    response += `у ${prices.best.provider}\n`;
    response += `Итого: ${bestTotal.toFixed(2)} TRX`;

    await ctx.reply(response);
  } catch (error: any) {
    await ctx.reply(`Ошибка при получении цен: ${error.message}`);
  }
}
```

### Команда /buy

Команда `/buy` размещает заказ на энергию через MERX:

```typescript
// src/handlers/buy.ts
async function handleBuy(ctx: Context): Promise<void> {
  const args = (ctx.message as any).text.split(' ').slice(1);

  if (args.length < 3) {
    await ctx.reply(
      'Использование: /buy <amount> <duration> <tron_address>\n' +
      'Пример: /buy 65000 1h TYourAddress123\n\n' +
      'Доступные длительности: 5m, 10m, 30m, 1h, 3h, 6h, 12h, 1d, 3d, 14d'
    );
    return;
  }

  const amount = parseInt(args[0]);
  const duration = args[1];
  const targetAddress = args[2];

  // Проверяем входные данные
  if (isNaN(amount) || amount < 10000) {
    await ctx.reply('Количество энергии должно быть числом >= 10 000');
    return;
  }

  if (!isValidTronAddress(targetAddress)) {
    await ctx.reply('Неверный адрес TRON. Должен начинаться с T.');
    return;
  }

  const validDurations = [
    '5m', '10m', '30m', '1h', '3h',
    '6h', '12h', '1d', '3d', '14d'
  ];
  if (!validDurations.includes(duration)) {
    await ctx.reply(
      `Неверная длительность. Выберите из: ${validDurations.join(', ')}`
    );
    return;
  }

  try {
    // Сначала получаем лучшую цену
    const prices = await merx.getPrices({
      energy_amount: amount,
      duration: duration
    });

    const totalTrx =
      (prices.best.price_sun * amount) / 1e6;

    await ctx.reply(
      `Размещаю заказ:\n` +
      `Количество: ${amount.toLocaleString()} энергии\n` +
      `Длительность: ${duration}\n` +
      `Адрес назначения: ${targetAddress}\n` +
      `Цена: ${prices.best.price_sun} SUN ` +
      `(${totalTrx.toFixed(2)} TRX)\n` +
      `Поставщик: ${prices.best.provider}\n\n` +
      `Обработка...`
    );

    const order = await merx.createOrder({
      energy_amount: amount,
      duration: duration,
      target_address: targetAddress
    });

    // Сохраняем ID чата для уведомления webhook
    orderChatMap.set(order.id, ctx.chat!.id);

    await ctx.reply(
      `Заказ размещен.\n` +
      `ID заказа: ${order.id}\n` +
      `Статус: ${order.status}\n\n` +
      `Вы будете уведомлены при исполнении заказа.`
    );
  } catch (error: any) {
    await ctx.reply(`Ошибка при размещении заказа: ${error.message}`);
  }
}

function isValidTronAddress(address: string): boolean {
  return /^T[1-9A-HJ-NP-Za-km-z]{33}$/.test(address);
}
```

### Команда /balance

```typescript
// src/handlers/balance.ts
async function handleBalance(ctx: Context): Promise<void> {
  try {
    const balance = await merx.getBalance();

    await ctx.reply(
      `Баланс счёта MERX:\n\n` +
      `Доступно: ${balance.available_trx} TRX\n` +
      `Зарезервировано (в заказах): ${balance.reserved_trx} TRX\n` +
      `Всего: ${balance.total_trx} TRX`
    );
  } catch (error: any) {
    await ctx.reply(`Ошибка при получении баланса: ${error.message}`);
  }
}
```

### Обработчик webhook

Обработчик webhook получает уведомления о статусе заказов от MERX и пересылает их пользователю через Telegram:

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
            `Заказ исполнен.\n\n` +
            `ID заказа: ${event.data.order_id}\n` +
            `Энергия: ${event.data.energy_amount.toLocaleString()}\n` +
            `Поставщик: ${event.data.provider}\n` +
            `Цена: ${event.data.price_sun} SUN\n` +
            `Адрес назначения: ${event.data.target_address}\n\n` +
            `Энергия делегирована адресу назначения.`
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
            `Заказ не выполнен.\n\n` +
            `ID заказа: ${event.data.order_id}\n` +
            `Причина: ${event.data.reason}\n\n` +
            `Ваш баланс был возвращен.`
          );
          orderChatMap.delete(event.data.order_id);
        }
        break;
      }
    }
  } catch (error) {
    console.error('Ошибка обработки webhook:', error);
  }

  res.status(200).json({ received: true });
}
```

## Добавление оповещений о цене

Расширьте бот командой `/alert`, которая использует стоящие заказы MERX для уведомления пользователей при падении цен ниже порога:

```typescript
bot.command('alert', async (ctx) => {
  const args = (ctx.message as any).text.split(' ').slice(1);

  if (args.length < 2) {
    await ctx.reply(
      'Использование: /alert <energy_amount> <max_price_sun>\n' +
      'Пример: /alert 65000 25\n\n' +
      'Вы будете уведомлены при падении цены энергии ниже целевого значения.'
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
      `Оповещение о цене установлено.\n\n` +
      `Отслеживание: ${amount.toLocaleString()} энергии ` +
      `по цене ${maxPrice} SUN или ниже\n` +
      `ID оповещения: ${standing.id}\n\n` +
      `Вы будете уведомлены при доступности такой цены.`
    );
  } catch (error: any) {
    await ctx.reply(`Ошибка при установке оповещения: ${error.message}`);
  }
});
```

## Рассмотрения для production

### Постоянное хранилище

Использованное выше в памяти `orderChatMap` подходит для разработки, но теряет данные при перезагрузке. Для production используйте Redis или базу данных:

```typescript
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL);

async function storeOrderChat(
  orderId: string,
  chatId: number
): Promise<void> {
  // Сохраняем с TTL 24 часа
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

### Аутентификация пользователей

Для ботов с несколькими пользователями привязывайте ID пользователей Telegram к учётным записям MERX:

```typescript
// Сохраняем API ключи пользователей в защищённом виде
async function setUserApiKey(
  telegramId: number,
  apiKey: string
): Promise<void> {
  // Шифруем перед сохранением
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

### Ограничение частоты запросов

Предотвратите злоупотребления ограничением частоты выполнения команд:

```typescript
const rateLimits = new Map<number, number>();
const RATE_LIMIT_MS = 5000; // 5 секунд между командами

function isRateLimited(userId: number): boolean {
  const lastCommand = rateLimits.get(userId) || 0;
  const now = Date.now();

  if (now - lastCommand < RATE_LIMIT_MS) {
    return true;
  }

  rateLimits.set(userId, now);
  return false;
}

// Применяем ко всем командам
bot.use(async (ctx, next) => {
  const userId = ctx.from?.id;
  if (userId && isRateLimited(userId)) {
    await ctx.reply('Пожалуйста, подождите несколько секунд перед следующей командой.');
    return;
  }
  return next();
});
```

### Обработка ошибок

Оборачивайте все обработчики команд последовательной обработкой ошибок:

```typescript
function withErrorHandling(
  handler: (ctx: Context) => Promise<void>
) {
  return async (ctx: Context) => {
    try {
      await handler(ctx);
    } catch (error: any) {
      console.error('Ошибка команды:', error);
      await ctx.reply(
        `Произошла ошибка: ${error.message}\n` +
        `Пожалуйста, попробуйте ещё раз или свяжитесь со службой поддержки.`
      );
    }
  };
}

bot.command('price', withErrorHandling(handlePrice));
bot.command('buy', withErrorHandling(handleBuy));
bot.command('balance', withErrorHandling(handleBalance));
```

## Развёртывание

### Запуск с PM2

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

## Тестирование бота

1. Запустите бот: `npx ts-node src/index.ts`
2. Откройте Telegram и найдите вашего бота
3. Отправьте `/start` для просмотра доступных команд
4. Отправьте `/price 65000` для проверки текущих цен на энергию
5. Отправьте `/buy 65000 1h TYourAddress` для размещения заказа
6. Ждите уведомления webhook об исполнении заказа

## Заключение

Telegram-бот для покупки TRON energy превращает веб-ориентированный рабочий процесс в интерфейс для разговора. С тремя основными командами и интеграцией webhook пользователи могут проверять цены, покупать энергию и получать уведомления об исполнении, не выходя из Telegram.

Реализация использует SDK MERX для всех взаимодействий с рынком энергии, что означает, что бот автоматически получает маршрутизацию по лучшей цене среди семи поставщиков, поддержку стоящих заказов для оповещений о цене и надёжное исполнение заказов с резервированием.

Полный исходный код в этой статье готов к production с добавлением постоянного хранилища и надлежащей аутентификации. Общая реализация составляет менее 300 строк TypeScript.

Для документации API посетите [https://merx.exchange/docs](https://merx.exchange/docs). Для интеграции MCP сервера см. [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp).


## Попробуйте сейчас с AI

Добавьте MERX в Claude Desktop или любой совместимый с MCP клиент — нет установки, не нужен API ключ для инструментов только для чтения:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Спросите вашего AI помощника: «Какова сейчас самая дешёвая TRON energy?» и получите живые цены от всех подключённых поставщиков.

Полная документация MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)