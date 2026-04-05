# Construir un Bot de Telegram que Compra Energía de TRON

Telegram es la plataforma de comunicación de facto para la comunidad cripto. Si administras billeteras TRON, ejecutas una dApp, o simplemente deseas una forma conveniente de comprar energía sin abrir un navegador, un bot de Telegram es una herramienta práctica. Este tutorial te guía a través de la construcción de un bot de Telegram completo que verifica precios de energía, compra energía a través de MERX, y envía notificaciones cuando las órdenes se completan.

Al final de este artículo, tendrás un bot funcional con tres comandos -- `/price`, `/buy`, y `/balance` -- más integración de webhooks para actualizaciones de estado de órdenes en tiempo real.

## Requisitos previos

- Node.js 18 o posterior
- Un token de bot de Telegram (desde @BotFather)
- Una clave API de MERX (desde [https://merx.exchange](https://merx.exchange))
- Familiaridad básica con TypeScript

## Configuración del proyecto

```bash
mkdir tron-energy-bot
cd tron-energy-bot
npm init -y
npm install telegraf merx-sdk dotenv express
npm install -D typescript @types/node @types/express ts-node
```

Crea la configuración de TypeScript:

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

Configura tus variables de entorno:

```bash
# .env
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
MERX_API_KEY=your_merx_api_key
WEBHOOK_PORT=3001
WEBHOOK_URL=https://your-server.com/webhooks/merx
```

## Arquitectura del bot

El bot tiene tres capas:

1. **Manejadores de comandos de Telegram** -- Analiza comandos de usuario y responde
2. **Cliente MERX** -- Interactúa con el mercado de energía
3. **Servidor de webhook** -- Recibe notificaciones de órdenes asincrónicas

```
El usuario envía /price 65000
       |
       v
[Bot de Telegram] -- analiza comando
       |
       v
[Cliente MERX] -- consulta precios
       |
       v
[Bot de Telegram] -- formatea y envía respuesta
       |
       v
El usuario recibe tabla de precios

El usuario envía /buy 65000 1h TAddress
       |
       v
[Bot de Telegram] -- valida entrada
       |
       v
[Cliente MERX] -- crea orden
       |
       v
[Servidor de webhook] -- recibe order.filled
       |
       v
[Bot de Telegram] -- notifica al usuario
```

## Implementación principal

### Punto de entrada

```typescript
// src/index.ts
import { Telegraf, Context } from 'telegraf';
import { MerxClient } from 'merx-sdk';
import express from 'express';
import dotenv from 'dotenv';

dotenv.config();

const bot = new Telegraf(process.env.TELEGRAM_BOT_TOKEN!);
const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY! });

// Almacena IDs de chat para notificaciones de órdenes
const orderChatMap = new Map<string, number>();

// --- Manejadores de comandos ---

bot.command('start', (ctx) => {
  ctx.reply(
    'Bot de Energía TRON\n\n' +
    'Comandos:\n' +
    '/price <amount> - Verifica precios de energía\n' +
    '/buy <amount> <duration> <address> - Compra energía\n' +
    '/balance - Verifica tu saldo de MERX\n\n' +
    'Ejemplo:\n' +
    '/price 65000\n' +
    '/buy 65000 1h TYourAddress123'
  );
});

bot.command('price', handlePrice);
bot.command('buy', handleBuy);
bot.command('balance', handleBalance);

// --- Servidor de webhook ---

const app = express();
app.use(express.json());

app.post('/webhooks/merx', handleMerxWebhook);

// --- Inicio ---

const PORT = parseInt(process.env.WEBHOOK_PORT || '3001');

app.listen(PORT, () => {
  console.log(`Servidor de webhook escuchando en puerto ${PORT}`);
});

bot.launch().then(() => {
  console.log('Bot de Telegram iniciado');
});

process.once('SIGINT', () => bot.stop('SIGINT'));
process.once('SIGTERM', () => bot.stop('SIGTERM'));
```

### El comando /price

El comando `/price` consulta MERX para obtener precios de energía actuales en todos los proveedores:

```typescript
// src/handlers/price.ts
async function handlePrice(ctx: Context): Promise<void> {
  const args = (ctx.message as any).text.split(' ').slice(1);

  if (args.length < 1) {
    await ctx.reply(
      'Uso: /price <energy_amount> [duration]\n' +
      'Ejemplo: /price 65000\n' +
      'Ejemplo: /price 65000 1h'
    );
    return;
  }

  const amount = parseInt(args[0]);
  if (isNaN(amount) || amount < 10000) {
    await ctx.reply('La cantidad de energía debe ser un número >= 10,000');
    return;
  }

  const duration = args[1] || '1h';

  try {
    await ctx.reply('Verificando precios en 7 proveedores...');

    const prices = await merx.getPrices({
      energy_amount: amount,
      duration: duration
    });

    let response = `Precios de energía para ${amount.toLocaleString()} unidades (${duration}):\n\n`;

    // Formatea precios de proveedores como una tabla
    for (const offer of prices.providers) {
      const totalTrx = (offer.price_sun * amount) / 1e6;
      const marker = offer.provider === prices.best.provider
        ? ' << MEJOR'
        : '';

      response +=
        `${offer.provider}: ${offer.price_sun} SUN ` +
        `(${totalTrx.toFixed(2)} TRX)${marker}\n`;
    }

    const bestTotal = (prices.best.price_sun * amount) / 1e6;
    response += `\nMejor precio: ${prices.best.price_sun} SUN `;
    response += `vía ${prices.best.provider}\n`;
    response += `Costo total: ${bestTotal.toFixed(2)} TRX`;

    await ctx.reply(response);
  } catch (error: any) {
    await ctx.reply(`Error al obtener precios: ${error.message}`);
  }
}
```

### El comando /buy

El comando `/buy` coloca una orden de energía a través de MERX:

```typescript
// src/handlers/buy.ts
async function handleBuy(ctx: Context): Promise<void> {
  const args = (ctx.message as any).text.split(' ').slice(1);

  if (args.length < 3) {
    await ctx.reply(
      'Uso: /buy <amount> <duration> <tron_address>\n' +
      'Ejemplo: /buy 65000 1h TYourAddress123\n\n' +
      'Duraciones: 5m, 10m, 30m, 1h, 3h, 6h, 12h, 1d, 3d, 14d'
    );
    return;
  }

  const amount = parseInt(args[0]);
  const duration = args[1];
  const targetAddress = args[2];

  // Valida entradas
  if (isNaN(amount) || amount < 10000) {
    await ctx.reply('La cantidad de energía debe ser un número >= 10,000');
    return;
  }

  if (!isValidTronAddress(targetAddress)) {
    await ctx.reply('Dirección TRON inválida. Debe comenzar con T.');
    return;
  }

  const validDurations = [
    '5m', '10m', '30m', '1h', '3h',
    '6h', '12h', '1d', '3d', '14d'
  ];
  if (!validDurations.includes(duration)) {
    await ctx.reply(
      `Duración inválida. Elige entre: ${validDurations.join(', ')}`
    );
    return;
  }

  try {
    // Obtiene el mejor precio primero
    const prices = await merx.getPrices({
      energy_amount: amount,
      duration: duration
    });

    const totalTrx =
      (prices.best.price_sun * amount) / 1e6;

    await ctx.reply(
      `Colocando orden:\n` +
      `Cantidad: ${amount.toLocaleString()} energía\n` +
      `Duración: ${duration}\n` +
      `Destino: ${targetAddress}\n` +
      `Precio: ${prices.best.price_sun} SUN ` +
      `(${totalTrx.toFixed(2)} TRX)\n` +
      `Proveedor: ${prices.best.provider}\n\n` +
      `Procesando...`
    );

    const order = await merx.createOrder({
      energy_amount: amount,
      duration: duration,
      target_address: targetAddress
    });

    // Almacena el ID de chat para notificación de webhook
    orderChatMap.set(order.id, ctx.chat!.id);

    await ctx.reply(
      `Orden colocada.\n` +
      `ID de orden: ${order.id}\n` +
      `Estado: ${order.status}\n\n` +
      `Serás notificado cuando la orden se complete.`
    );
  } catch (error: any) {
    await ctx.reply(`Error al colocar orden: ${error.message}`);
  }
}

function isValidTronAddress(address: string): boolean {
  return /^T[1-9A-HJ-NP-Za-km-z]{33}$/.test(address);
}
```

### El comando /balance

```typescript
// src/handlers/balance.ts
async function handleBalance(ctx: Context): Promise<void> {
  try {
    const balance = await merx.getBalance();

    await ctx.reply(
      `Saldo de cuenta MERX:\n\n` +
      `Disponible: ${balance.available_trx} TRX\n` +
      `Reservado (en órdenes): ${balance.reserved_trx} TRX\n` +
      `Total: ${balance.total_trx} TRX`
    );
  } catch (error: any) {
    await ctx.reply(`Error al obtener saldo: ${error.message}`);
  }
}
```

### Manejador de webhook

El manejador de webhook recibe notificaciones de estado de órdenes desde MERX y las reenvía al usuario vía Telegram:

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
            `Orden completada.\n\n` +
            `ID de orden: ${event.data.order_id}\n` +
            `Energía: ${event.data.energy_amount.toLocaleString()}\n` +
            `Proveedor: ${event.data.provider}\n` +
            `Precio: ${event.data.price_sun} SUN\n` +
            `Destino: ${event.data.target_address}\n\n` +
            `La energía ha sido delegada a la dirección de destino.`
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
            `Orden fallida.\n\n` +
            `ID de orden: ${event.data.order_id}\n` +
            `Razón: ${event.data.reason}\n\n` +
            `Tu saldo ha sido reembolsado.`
          );
          orderChatMap.delete(event.data.order_id);
        }
        break;
      }
    }
  } catch (error) {
    console.error('Error de procesamiento de webhook:', error);
  }

  res.status(200).json({ received: true });
}
```

## Agregar alertas de precio

Extiende el bot con un comando `/alert` que usa órdenes permanentes de MERX para notificar a los usuarios cuando los precios caen por debajo de un umbral:

```typescript
bot.command('alert', async (ctx) => {
  const args = (ctx.message as any).text.split(' ').slice(1);

  if (args.length < 2) {
    await ctx.reply(
      'Uso: /alert <energy_amount> <max_price_sun>\n' +
      'Ejemplo: /alert 65000 25\n\n' +
      'Serás notificado cuando el precio de energía caiga por debajo del objetivo.'
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
      `Alerta de precio establecida.\n\n` +
      `Monitoreando: ${amount.toLocaleString()} energía ` +
      `a o por debajo de ${maxPrice} SUN\n` +
      `ID de alerta: ${standing.id}\n\n` +
      `Serás notificado cuando este precio esté disponible.`
    );
  } catch (error: any) {
    await ctx.reply(`Error al establecer alerta: ${error.message}`);
  }
});
```

## Consideraciones de producción

### Almacenamiento persistente

El `orderChatMap` en memoria usado arriba es adecuado para desarrollo pero pierde datos al reiniciar. Para producción, usa Redis o una base de datos:

```typescript
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL);

async function storeOrderChat(
  orderId: string,
  chatId: number
): Promise<void> {
  // Almacena con TTL de 24 horas
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

### Autenticación de usuario

Para bots multiusuario, asocia IDs de usuario de Telegram con cuentas de MERX:

```typescript
// Almacena claves API de usuario de forma segura
async function setUserApiKey(
  telegramId: number,
  apiKey: string
): Promise<void> {
  // Encripta antes de almacenar
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

### Limitación de velocidad

Previene abusos limitando la frecuencia de comandos:

```typescript
const rateLimits = new Map<number, number>();
const RATE_LIMIT_MS = 5000; // 5 segundos entre comandos

function isRateLimited(userId: number): boolean {
  const lastCommand = rateLimits.get(userId) || 0;
  const now = Date.now();

  if (now - lastCommand < RATE_LIMIT_MS) {
    return true;
  }

  rateLimits.set(userId, now);
  return false;
}

// Aplica a todos los comandos
bot.use(async (ctx, next) => {
  const userId = ctx.from?.id;
  if (userId && isRateLimited(userId)) {
    await ctx.reply('Por favor espera unos segundos entre comandos.');
    return;
  }
  return next();
});
```

### Manejo de errores

Envuelve todos los manejadores de comandos con manejo de errores consistente:

```typescript
function withErrorHandling(
  handler: (ctx: Context) => Promise<void>
) {
  return async (ctx: Context) => {
    try {
      await handler(ctx);
    } catch (error: any) {
      console.error('Error de comando:', error);
      await ctx.reply(
        `Ocurrió un error: ${error.message}\n` +
        `Por favor intenta de nuevo o contacta soporte.`
      );
    }
  };
}

bot.command('price', withErrorHandling(handlePrice));
bot.command('buy', withErrorHandling(handleBuy));
bot.command('balance', withErrorHandling(handleBalance));
```

## Implementación

### Ejecutar con PM2

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

## Prueba el bot

1. Inicia el bot: `npx ts-node src/index.ts`
2. Abre Telegram y encuentra tu bot
3. Envía `/start` para ver los comandos disponibles
4. Envía `/price 65000` para verificar precios actuales de energía
5. Envía `/buy 65000 1h TYourAddress` para colocar una orden
6. Espera la notificación del webhook confirmando que la orden se completó

## Conclusión

Un bot de Telegram para compra de energía TRON convierte un flujo de trabajo basado en web en una interfaz conversacional. Con tres comandos principales e integración de webhooks, los usuarios pueden verificar precios, comprar energía, y recibir notificaciones de finalización sin salir de Telegram.

La implementación aprovecha el SDK de MERX para todas las interacciones del mercado de energía, lo que significa que el bot obtiene automáticamente enrutamiento de mejor precio en siete proveedores, soporte de órdenes permanentes para alertas de precio, y ejecución de orden confiable con conmutación por error.

El código fuente completo en este artículo está listo para producción con la adición de almacenamiento persistente y autenticación adecuada. La implementación total es menos de 300 líneas de TypeScript.

Para documentación de API, visita [https://merx.exchange/docs](https://merx.exchange/docs). Para la integración del servidor MCP, consulta [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp).


## Pruébalo ahora con IA

Agrega MERX a Claude Desktop o cualquier cliente compatible con MCP -- sin instalación, sin clave API necesaria para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregunta a tu agente de IA: "¿Cuál es la energía TRON más barata en este momento?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)