# Создание бота для платежей USDT с MERX SDK

Отправка USDT в TRON должна быть простой. У вас есть адрес получателя, сумма и кошелек с средствами. Но если отправить USDT без energy, протокол TRON сжигает примерно 27 TRX из вашего кошелька для покрытия стоимости транзакции. В масштабе - сотни или тысячи переводов в день - стоимость сжигания становится значительной тратой.

Это руководство создает полнофункциональный бот для платежей USDT, который исключает эту стоимость. Бот получает запросы на платежи, берет в аренду energy через MERX по цене ниже стоимости сжигания, отправляет USDT с помощью арендованной energy и сообщает об экономии. Он обрабатывает ошибки, поддерживает повторные попытки и интегрирует webhook-уведомления для надежности в production.

К концу у вас будет работающий бот для платежей, который экономит более 90 процентов на каждом переводе USDT.

## Обзор архитектуры

Бот следует простому конвейеру:

1. Получить запрос на платеж (адрес получателя + сумма).
2. Проверить баланс вашего аккаунта MERX.
3. Оценить energy, необходимую для перевода.
4. Создать заказ energy через MERX.
5. Дождаться завершения делегирования energy.
6. Отправить перевод USDT, используя делегированную energy.
7. Сообщить о транзакции и экономии.

Каждый этап изолирован и допускает повторение. Если заказ energy не срабатывает, USDT никогда не отправляется. Если перевод USDT не срабатывает, energy у вас все еще есть (она истекает после периода аренды, но средства не теряются).

## Предусловия

Перед началом вам нужны:

- **Node.js 18 или позже** - бот использует ES модули и современные возможности JavaScript.
- **Аккаунт MERX с балансом** - зарегистрируйтесь на [merx.exchange](https://merx.exchange) и пополните TRX.
- **API ключ MERX** - создайте его с разрешениями `create_orders`, `view_orders` и `view_balance`.
- **Кошелек TRON** - с балансом USDT для платежей и небольшим количеством TRX для bandwidth.
- **TronWeb** - для подписания и трансляции транзакции перевода USDT.

### Инициализация проекта

```bash
mkdir usdt-payment-bot
cd usdt-payment-bot
npm init -y
npm install merx-sdk tronweb dotenv uuid
```

Создайте файл `.env` (никогда не коммитьте это в систему контроля версий):

```bash
MERX_API_KEY=merx_live_your_key_here
TRON_PRIVATE_KEY=your_tron_wallet_private_key
TRON_FULL_HOST=https://api.trongrid.io
USDT_CONTRACT=TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t
MIN_SAVINGS_PERCENT=50
```

## Шаг 1: Инициализация клиентов

Создайте клиент MERX и экземпляр TronWeb:

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

## Шаг 2: Проверка баланса MERX

Перед началом проверьте, достаточно ли у вас средств для аренды energy:

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

## Шаг 3: Оценка стоимости energy

Используйте API оценки MERX, чтобы определить, сколько energy потребуется для перевода и какова будет его стоимость:

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

## Шаг 4: Создание заказа energy

Закажите energy через MERX с ключом идемпотентности для безопасного повтора:

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

## Шаг 5: Ожидание делегирования energy

После создания заказа дождитесь, пока energy будет делегирована в сети. Бот опрашивает статус заказа с истечением времени:

```javascript
// src/wait.js
import { merx } from './clients.js';

export async function waitForFill(orderId, timeoutMs = 60000) {
  const startTime = Date.now();
  const pollInterval = 2000; // 2 секунды

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

    // Все еще ожидание или обработка - подождите и опросите снова
    await new Promise((resolve) => setTimeout(resolve, pollInterval));
  }

  throw new Error(`Order ${orderId} timed out after ${timeoutMs / 1000}s`);
}
```

Для production-систем рассмотрите использование webhooks вместо polling (рассмотрено позже в этой статье).

## Шаг 6: Отправка перевода USDT

С energy, делегированной на ваш адрес, отправьте перевод USDT. Energy расходуется вместо сжигания TRX:

```javascript
// src/transfer.js
import { tronWeb, USDT_CONTRACT, SENDER_ADDRESS } from './clients.js';

export async function sendUSDT(recipientAddress, amountUSDT) {
  // USDT имеет 6 десятичных знаков на TRON
  const amountSun = Math.floor(amountUSDT * 1_000_000);

  // Валидация адреса получателя
  if (!tronWeb.isAddress(recipientAddress)) {
    throw new Error(`Invalid TRON address: ${recipientAddress}`);
  }

  // Создайте транзакцию перевода TRC-20
  const contract = await tronWeb.contract().at(USDT_CONTRACT);

  const tx = await contract.methods
    .transfer(recipientAddress, amountSun)
    .send({
      from: SENDER_ADDRESS,
      feeLimit: 100_000_000, // 100 TRX лимит комиссии (безопасный лимит)
    });

  return {
    txHash: tx,
    recipient: recipientAddress,
    amount: amountUSDT,
    amountSun,
  };
}
```

## Шаг 7: Объединение всего вместе

Основная функция бота организует все этапы:

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

  // Шаг 1: Оценить стоимость
  console.log('\n[1/5] Estimating energy cost...');
  const estimate = await estimateTransferCost();
  console.log(`  Energy required: ${estimate.energyRequired}`);
  console.log(`  Burn cost: ${estimate.burnCostTRX}`);
  console.log(`  Rental cost: ${estimate.rentalCostTRX}`);
  console.log(`  Savings: ${estimate.savingsPercent}%`);

  // Шаг 2: Решить, стоит ли аренда
  if (estimate.savingsPercent < MIN_SAVINGS_PERCENT) {
    console.log(`  Savings below threshold (${MIN_SAVINGS_PERCENT}%). Skipping energy rental.`);
    const tx = await sendUSDT(recipientAddress, amountUSDT);
    return { ...tx, energyRented: false, savings: null };
  }

  // Шаг 3: Проверить баланс MERX
  console.log('\n[2/5] Checking MERX balance...');
  const balance = await checkMerxBalance(estimate.rentalCostSun);
  console.log(`  Available: ${balance.availableTRX} TRX`);

  // Шаг 4: Создать заказ energy
  console.log('\n[3/5] Creating energy order...');
  const order = await createEnergyOrder(
    estimate.energyRequired,
    SENDER_ADDRESS,
    estimate.durationHours
  );
  console.log(`  Order ID: ${order.orderId}`);
  console.log(`  Provider: ${order.provider}`);
  console.log(`  Status: ${order.status}`);

  // Шаг 5: Ждать делегирования energy
  console.log('\n[4/5] Waiting for energy delegation...');
  const fill = await waitForFill(order.orderId);
  console.log(`  Filled by: ${fill.provider}`);
  console.log(`  Delegation TX: ${fill.txHash}`);

  // Шаг 6: Отправить USDT с делегированной energy
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

### Запуск бота

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

Пример вывода:

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

## Обработка ошибок

Production-системы платежей требуют надежной обработки ошибок. Вот режимы отказов и способы обработки каждого из них.

### Недостаточный баланс MERX

Если баланс вашего аккаунта MERX низкий, бот должен предупредить вас и опционально вернуться к сжиганию TRX:

```javascript
try {
  await checkMerxBalance(estimate.rentalCostSun);
} catch (err) {
  if (err.message.includes('Insufficient MERX balance')) {
    console.warn('MERX balance low. Sending without energy rental.');
    // Опционально: отправить оповещение команде ops
    // await alertOps('MERX balance low', err.message);
    const tx = await sendUSDT(recipientAddress, amountUSDT);
    return { ...tx, energyRented: false, fallbackReason: 'low_balance' };
  }
  throw err;
}
```

### Истечение времени заполнения заказа

Если поставщик energy слишком долго делегирует, отмените и повторите попытку с другим поставщиком или вернитесь назад:

```javascript
try {
  const fill = await waitForFill(order.orderId, 30000); // 30-секундный таймаут
} catch (err) {
  if (err.message.includes('timed out')) {
    console.warn('Energy delegation timed out. Sending without energy.');
    const tx = await sendUSDT(recipientAddress, amountUSDT);
    return { ...tx, energyRented: false, fallbackReason: 'delegation_timeout' };
  }
  throw err;
}
```

### Ошибка перевода USDT

Если сам перевод USDT не срабатывает (недостаточно средств, ошибка контракта), energy не теряется - она остается делегированной на время аренды. Вы можете повторить попытку перевода:

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

## Webhook-уведомления

Polling для проверки статуса заказа работает для простых ботов, но production-системы должны использовать webhooks. MERX отправляет HTTP POST-уведомления на ваш webhook URL при изменении статуса заказа.

### Настройка Webhooks

Настройте вашу webhook-конечную точку в панели управления MERX или через API:

```bash
curl -X POST https://merx.exchange/api/v1/webhooks \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-server.com/webhooks/merx",
    "events": ["order.filled", "order.failed"]
  }'
```

### Обработка Webhook-событий

```javascript
// webhook-handler.js
import express from 'express';

const app = express();
app.use(express.json());

// Храните обратные вызовы ожидающих платежей
const pendingPayments = new Map();

app.post('/webhooks/merx', (req, res) => {
  const event = req.body;

  // Проверьте подпись webhook (требование production)
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

// Замените polling на webhook-ожидание
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

Webhooks исключают цикл polling, уменьшают вызовы API и быстрее реагируют на заполнение заказов (уведомление менее чем за секунду против 2-секундных интервалов polling).

## Соображения Production

### Параллелизм

Если ваш бот обрабатывает несколько платежей одновременно, каждый платеж должен иметь свой собственный ключ идемпотентности и работать независимо. Используйте очередь (Bull, BullMQ или подобное) для управления одновременными платежами:

```javascript
import Queue from 'bull';

const paymentQueue = new Queue('payments', {
  redis: { host: '127.0.0.1', port: 6379 },
});

paymentQueue.process(5, async (job) => {
  // Обрабатывайте до 5 платежей одновременно
  const { recipient, amount } = job.data;
  return await processPayment(recipient, amount);
});

// Добавьте платеж в очередь
await paymentQueue.add({
  recipient: 'TRecipientAddressHere',
  amount: 100,
});
```

### Мониторинг и оповещения

Отслеживайте ключевые метрики для видимости операций:

- **Платежи в час** - отслеживание пропускной способности.
- **Средний процент экономии** - если экономия упадет ниже 80 процентов, исследуйте цены поставщиков.
- **Баланс MERX** - оповещение, когда баланс упадет ниже порога (например, достаточно для 100 переводов).
- **Время заполнения** - если делегирование energy занимает более 30 секунд, поставщики могут быть перегружены.
- **Частота отказов** - процент попыток платежа, которые не срабатывают на любом этапе.

Отслеживайте их в вашем предпочтительном инструменте мониторинга (Prometheus, Datadog или даже простом JSON логе).

### Ограничения скорости

API MERX ограничивает создание заказов до 10 запросов в минуту. Если вашему боту нужно отправлять более 10 платежей в минуту, у вас есть два варианта:

1. **Поставьте платежи в очередь** и обрабатывайте их с учетом лимита.
2. **Групповая покупка energy** - купите достаточно energy для нескольких переводов в одном заказе, затем отправляйте переводы последовательно, пока energy активна.

Вариант 2 более эффективен для высокообъемных сценариев:

```javascript
async function processBatch(payments) {
  // Рассчитайте общую energy для всех платежей
  const totalEnergy = payments.length * 65000; // ~65K за перевод USDT

  // Единственный заказ energy для всей партии
  const order = await createEnergyOrder(totalEnergy, SENDER_ADDRESS, 1);
  await waitForFill(order.orderId);

  // Отправьте все переводы USDT, пока energy активна
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

### Контрольный список безопасности

Перед развертыванием на production:

- Сохраняйте приватный ключ TRON в менеджере секретов, а не в `.env` на диске.
- Запускайте бота в ограниченной среде без входящего сетевого доступа, кроме webhook-конечной точки.
- Используйте выделенный кошелек TRON для бота только с USDT, необходимым для ближайших платежей. Не храните весь казначейство в кошельке бота.
- Реализуйте лимиты вывода - бот не должен иметь возможность опустошить кошелек за один запуск.
- Логируйте каждый платеж со всеми деталями для аудита.
- Установите оповещения для необычной активности (суммы платежей выше порога, быстрые последовательные отказы).

## Полная структура проекта

```
usdt-payment-bot/
  .env                  # Никогда не коммитьте
  .gitignore
  package.json
  index.js              # Точка входа
  src/
    clients.js          # Инициализация MERX + TronWeb
    balance.js          # Проверка баланса
    estimate.js         # Оценка стоимости
    order.js            # Создание заказа energy
    wait.js             # Polling заполнения заказа
    transfer.js         # Выполнение перевода USDT
    bot.js              # Главная оркестрация
    webhook-handler.js  # Приемник webhook (опционально)
```

Каждый файл имеет одну ответственность. Полная кодовая база составляет менее 300 строк. Имитируйте клиент MERX для unit-тестов, используйте Shasta testnet для интеграционных тестов.

## Заключение

Этот бот для платежей демонстрирует основной паттерн интеграции MERX: оценить, заказать, ждать, проводить транзакцию. Тот же паттерн применяется независимо от того, создаете ли вы процессор платежей, кошелек, систему вывода биржи или любое приложение, отправляющее токены TRC-20.

Ключевой вывод - математика экономии. При 94 процентах экономии на каждый трансфер, бот, обрабатывающий 1000 переводов USDT в день, экономит примерно 26 000 TRX ежедневно - более 750 000 TRX в месяц. Это не оптимизация. Это фундаментальное изменение структуры затрат.

Начните на Shasta testnet (`TRON_FULL_HOST=https://api.shasta.trongrid.io`), чтобы проверить процесс без риска реальных средств, затем переключитесь на mainnet, когда все будет работать.

- Платформа MERX: [merx.exchange](https://merx.exchange)
- Документация API: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- Python SDK: [github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)
- MCP сервер для AI агентов: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)


## Попробуйте прямо сейчас с AI

Добавьте MERX в Claude Desktop или любого MCP-совместимого клиента -- ноль установки, API ключ не требуется для инструментов только для чтения:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Спросите вашего AI агента: "Какая самая дешевая TRON energy прямо сейчас?" и получите живые цены от всех подключенных поставщиков.

Полная документация MCP: [merx.exchange/docs/tools/mcp-server