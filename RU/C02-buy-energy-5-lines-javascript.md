# Купить TRON Energy в 5 строк кода: MERX JavaScript SDK

MERX JavaScript SDK позволяет программно покупать TRON energy всего в пяти строк кода — установите пакет, инициализируйте клиент, проверьте цены, создайте заказ и подтвердите делегирование. Эта статья охватывает полный SDK от установки до развёртывания в production, включая TypeScript типы, обработку ошибок и все методы, доступные в четырёх модулях: prices, orders, balance и webhooks.

## Проблема прямого использования API поставщиков

У каждого поставщика TRON energy есть собственный API. Интеграция с одним поставщиком проста. Интеграция со всеми семью или восемью поставщиками, чтобы гарантировать лучшую цену — это серьёзный инженерный проект. Нужно обрабатывать различные схемы аутентификации, форматы ответов, коды ошибок, ограничения по частоте запросов и логику отказоустойчивости.

SDK MERX абстрагирует всё это в единого клиента с единообразным интерфейсом. За кулисами платформа MERX опрашивает всех поставщиков каждые 30 секунд, поддерживает индекс цен в реальном времени и маршрутизирует ваши заказы к самому дешёвому доступному источнику с автоматической отказоустойчивостью.

## Установка

SDK требует Node.js 18 или новее. Использует встроенный `fetch` без зависимостей во время выполнения.

```bash
npm install merx-sdk
```

```bash
yarn add merx-sdk
```

```bash
pnpm add merx-sdk
```

Пакет поставляется как ESM с полными TypeScript объявлениями типов и исходными картами.

## Пять строк для покупки energy

Вот полный процесс в минимальной форме:

```typescript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY })
const prices = await merx.prices.list()
const order = await merx.orders.create({
  resource_type: 'ENERGY',
  amount: 65000,
  target_address: 'TYourTargetAddressHere',
  duration_sec: 3600,
})
console.log(`Order ${order.id}: ${order.status}`)
```

Строка за строкой:

1. Импортируем класс клиента.
2. Инициализируем с вашим API ключом. Ключ начинается с `sk_live_` и создаётся в панели управления MERX на сайте [merx.exchange](https://merx.exchange).
3. Получаем текущие цены от всех подключённых поставщиков. Это опционально, но полезно для отображения состояния рынка перед заказом.
4. Создаём рыночный заказ на 65 000 единиц energy, делегированных на ваш адрес на один час. Платформа автоматически выбирает самого дешёвого поставщика.
5. Логируем ID заказа и статус. Заказ начинается в статусе `PENDING` и переходит в `FILLED` после подтверждения делегирования в блокчейне.

Это вся интеграция. Никакого выбора поставщика, никакой логики отказоустойчивости, никакого кода сравнения цен.

## Конфигурация клиента

```typescript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({
  apiKey: 'sk_live_your_key_here',   // Обязательно
  baseUrl: 'https://merx.exchange',  // Опционально, по умолчанию production
})
```

`apiKey` — это единственный обязательный параметр. `baseUrl` может быть переопределён для тестирования в staging окружении.

## Модуль Prices

Модуль `merx.prices` предоставляет пять методов для запроса данных рынка в реальном времени.

### prices.list()

Возвращает текущие цены от всех активных поставщиков.

```typescript
const prices = await merx.prices.list()

for (const p of prices) {
  const cheapest = p.energy_prices[0]
  if (cheapest) {
    console.log(`${p.provider}: ${cheapest.price_sun} SUN/energy, ${p.available_energy} available`)
  }
}
```

Каждый объект `ProviderPrice` включает:
- `provider` — идентификатор поставщика (например, "sohu", "catfee", "tronsave")
- `is_market` — поддерживает ли этот поставщик рыночные заказы с гибкими сроками
- `energy_prices` — массив уровней `{ duration_sec, price_sun }`
- `bandwidth_prices` — аналогичная структура для bandwidth
- `available_energy` — текущая мощность в единицах energy
- `available_bandwidth` — текущая мощность в единицах bandwidth
- `fetched_at` — Unix timestamp последнего успешного опроса

### prices.best(resource, amount?)

Возвращает единственную самую дешёвую точку цены для заданного типа ресурса.

```typescript
const best = await merx.prices.best('ENERGY')
console.log(`Cheapest: ${best.price_sun} SUN from ${best.provider}`)

// С фильтром минимального количества
const bestLarge = await merx.prices.best('ENERGY', 500000)
```

Опциональный параметр `amount` фильтрует поставщиков, у которых недостаточно мощности для выполнения вашего заказа.

### prices.history(params?)

Возвращает исторические данные цен для анализа и визуализации.

```typescript
const history = await merx.prices.history({
  provider: 'sohu',
  resource: 'ENERGY',
  period: '24h',
})

for (const entry of history) {
  console.log(`${entry.polled_at}: ${entry.price_sun} SUN, ${entry.available} available`)
}
```

Доступные периоды: `'1h'`, `'6h'`, `'24h'`, `'7d'`, `'30d'`. Все параметры фильтрации опциональны.

### prices.stats()

Возвращает статистику по всему рынку.

```typescript
const stats = await merx.prices.stats()
console.log(`Best price: ${stats.best_price_sun} SUN`)
console.log(`Average: ${stats.avg_price_sun} SUN`)
console.log(`Providers online: ${stats.total_providers}`)
console.log(`Cheapest-provider changes (24h): ${stats.cheapest_changes_24h}`)
```

### prices.preview(params)

Предварительно показывает стоимость заказа перед его размещением. Возвращает лучший подходящий поставщик и альтернативные варианты.

```typescript
const preview = await merx.prices.preview({
  resource: 'ENERGY',
  amount: 100000,
  duration: 86400,
  max_price_sun: 35,
})

if (preview.best) {
  console.log(`Best: ${preview.best.provider} at ${preview.best.cost_trx} TRX`)
  console.log(`Price: ${preview.best.price_sun} SUN/unit`)
}

for (const fb of preview.fallbacks) {
  console.log(`Fallback: ${fb.provider} at ${fb.cost_trx} TRX`)
}

if (preview.no_providers) {
  console.log('No providers available for this configuration')
}
```

Параметр `max_price_sun` фильтрует поставщиков, чьи цены превышают вашу максимальную цену.

## Модуль Orders

### orders.create(params)

Создаёт заказ на energy или bandwidth.

```typescript
const order = await merx.orders.create({
  resource_type: 'ENERGY',
  amount: 65000,
  target_address: 'TYourTargetAddress',
  duration_sec: 3600,
  order_type: 'MARKET',          // Опционально, по умолчанию 'MARKET'
  max_price_sun: 30,             // Опционально, требуется для LIMIT заказов
  idempotency_key: 'unique-id',  // Опционально, предотвращает дублирование
})

console.log(`Order ${order.id} created`)
console.log(`Status: ${order.status}`)
```

Типы заказов:
- `MARKET` — выполнить немедленно по лучшей доступной цене
- `LIMIT` — выполнить только если цена <= `max_price_sun`
- `PERIODIC` — повторяющийся заказ
- `BROADCAST` — отправить предписанную транзакцию делегирования

`idempotency_key` критичен для production систем. Если происходит ошибка сети и вы повторяете запрос с тем же ключом, API возвращает исходный заказ вместо создания дубликата.

### orders.list(limit?, offset?, status?)

Список заказов с постраничной навигацией.

```typescript
const { orders, total } = await merx.orders.list(10, 0, 'FILLED')
console.log(`${total} filled orders total`)

for (const o of orders) {
  console.log(`${o.id}: ${o.amount} ${o.resource_type}, cost: ${o.total_cost_sun} SUN`)
}
```

### orders.get(id)

Возвращает один заказ с деталями исполнения.

```typescript
const order = await merx.orders.get('ord_abc123')

console.log(`Status: ${order.status}`)
console.log(`Fills: ${order.fills.length}`)

for (const fill of order.fills) {
  console.log(`  ${fill.provider}: ${fill.amount} units at ${fill.price_sun} SUN`)
  console.log(`  Verified: ${fill.verified}`)
  if (fill.tronscan_url) {
    console.log(`  TX: ${fill.tronscan_url}`)
  }
}
```

Массив `fills` показывает точно, как был выполнен заказ. Каждое исполнение включает имя поставщика, выделённое количество, цену за единицу, стоимость, ID транзакции в блокчейне и то, было ли делегирование подтверждено в цепи.

## Модуль Balance

### balance.get()

Возвращает текущие балансы аккаунта.

```typescript
const balance = await merx.balance.get()
console.log(`TRX: ${balance.trx}`)
console.log(`USDT: ${balance.usdt}`)
console.log(`Locked: ${balance.trx_locked}`)
```

Поле `trx_locked` показывает TRX, зарезервированные для ожидающих заказов.

### balance.depositInfo()

Возвращает адрес депозита и памятку для пополнения вашего аккаунта MERX.

```typescript
const info = await merx.balance.depositInfo()
console.log(`Send TRX to: ${info.address}`)
console.log(`Include memo: ${info.memo}`)
console.log(`Minimum deposit: ${info.min_amount_trx} TRX`)
```

Памятка требуется для автоматического зачисления депозита. Депозиты без правильной памятки требуют ручной обработки.

### balance.withdraw(params)

Выводит TRX или USDT на внешний TRON адрес.

```typescript
const withdrawal = await merx.balance.withdraw({
  address: 'TYourExternalAddress',
  amount: 100,
  currency: 'TRX',
  idempotency_key: 'withdraw-unique-id',
})

console.log(`Withdrawal ${withdrawal.id}: ${withdrawal.status}`)
```

### balance.history(period?) и balance.summary()

```typescript
const history = await merx.balance.history('7D')
console.log(`${history.length} transactions in last 7 days`)

const summary = await merx.balance.summary()
console.log(`Total orders: ${summary.total_orders}`)
console.log(`Total energy: ${summary.total_energy}`)
console.log(`Average price: ${summary.avg_price_sun} SUN`)
```

## Модуль Webhooks

### webhooks.create(params)

Создаёт подписку на webhook. `secret` возвращается только при создании.

```typescript
const webhook = await merx.webhooks.create({
  url: 'https://your-server.com/merx-webhook',
  events: ['order.filled', 'order.failed', 'deposit.received'],
})

console.log(`Webhook ID: ${webhook.id}`)
console.log(`Secret: ${webhook.secret}`)
```

### webhooks.list() и webhooks.delete(id)

```typescript
const webhooks = await merx.webhooks.list()
console.log(`${webhooks.filter(w => w.is_active).length} active webhooks`)

await merx.webhooks.delete('wh_abc123')
```

## Обработка ошибок

Все ошибки API выбрасываются как экземпляры `MerxError` с машиночитаемым `code` и человекочитаемым `message`.

```typescript
import { MerxClient, MerxError } from 'merx-sdk'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY })

try {
  await merx.orders.create({
    resource_type: 'ENERGY',
    amount: 65000,
    target_address: 'TInvalidAddress',
    duration_sec: 3600,
  })
} catch (err) {
  if (err instanceof MerxError) {
    console.error(`[${err.code}]: ${err.message}`)
    // Output: [INVALID_ADDRESS]: Target address is not a valid TRON address
  }
}
```

Частые коды ошибок:

| Код                    | Значение                                       |
|------------------------|------------------------------------------------|
| `UNAUTHORIZED`         | Неверный или отсутствующий API ключ            |
| `INSUFFICIENT_FUNDS`   | Баланс аккаунта слишком низкий                 |
| `INVALID_ADDRESS`      | Адрес цели не является корректным TRON адресом |
| `ORDER_NOT_FOUND`      | ID заказа не существует                        |
| `INVALID_AMOUNT`       | Количество ниже минимума или превышает лимиты  |
| `DUPLICATE_REQUEST`    | Ключ идемпотентности уже использован           |
| `RATE_LIMITED`         | Слишком много запросов                         |
| `PROVIDER_UNAVAILABLE` | Нет доступных поставщиков                      |

## TypeScript типы

Все типы экспортируются из пакета и могут быть использованы для аннотаций типов во всём вашем приложении:

```typescript
import type {
  ProviderPrice,
  PricePoint,
  PriceHistoryEntry,
  PriceStats,
  OrderPreview,
  PreviewMatch,
  Order,
  OrderWithFills,
  Fill,
  CreateOrderParams,
  OrderType,
  OrderStatus,
  ResourceType,
  Balance,
  DepositInfo,
  Withdrawal,
  Webhook,
} from 'merx-sdk'
```

SDK совместим со strict режимом и не создаёт типы `any`. Каждый ответ полностью типизирован.

## Production пример: автоматизированный процесс передачи USDT

Вот полный production паттерн, который проверяет цены, создаёт заказ и опрашивает статус завершения:

```typescript
import { MerxClient, MerxError } from 'merx-sdk'
import { randomUUID } from 'node:crypto'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY! })

async function ensureEnergyAndTransfer(targetAddress: string) {
  // Предварительно проверяем стоимость
  const preview = await merx.prices.preview({
    resource: 'ENERGY',
    amount: 65000,
    duration: 3600,
  })

  if (preview.no_providers) {
    throw new Error('No energy providers available')
  }

  console.log(`Best price: ${preview.best!.cost_trx} TRX from ${preview.best!.provider}`)

  // Создаём заказ с идемпотентностью
  const order = await merx.orders.create({
    resource_type: 'ENERGY',
    amount: 65000,
    target_address: targetAddress,
    duration_sec: 3600,
    idempotency_key: randomUUID(),
  })

  // Опрашиваем пока не будет выполнено (в production используйте webhooks)
  let status = order.status
  while (status === 'PENDING' || status === 'EXECUTING') {
    await new Promise(r => setTimeout(r, 2000))
    const updated = await merx.orders.get(order.id)
    status = updated.status

    if (status === 'FILLED') {
      console.log(`Energy delegated. Fills:`)
      for (const fill of updated.fills) {
        console.log(`  ${fill.provider}: ${fill.amount} units, TX: ${fill.tronscan_url}`)
      }
    }
  }

  if (status === 'FAILED') {
    throw new Error(`Order ${order.id} failed`)
  }

  // Energy теперь доступен на адресе цели
  // Продолжаем с передачей USDT
}
```

В production замените цикл опроса на слушатель webhook. Создайте подписку на webhook для события `order.filled`, и ваш сервер будет уведомлён в момент подтверждения делегирования в блокчейне.

## Требования и совместимость

- Node.js 18+ (использует встроенный `fetch`)
- Нулевых зависимостей во время выполнения
- Только ESM (поставляется как ES модули)
- Полная поддержка TypeScript 5.4+
- Работает с Bun и Deno (любое окружение с поддержкой `fetch`)

## Ресурсы

SDK открытого исходного кода и доступен на GitHub. Приветствуются contributions и отчёты об ошибках.

- Платформа: [merx.exchange](https://merx.exchange)
- Документация: [merx.exchange/docs](https://merx.exchange/docs)
- GitHub: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- npm: [npmjs.com/package/merx-sdk](https://www.npmjs.com/package/merx-sdk)
- Python SDK: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- MCP Server для AI agents: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)

## Попробуйте прямо сейчас с AI

Добавьте MERX в Claude Desktop или любой MCP-совместимый клиент — никаких установок, API ключ не требуется для инструментов только для чтения:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Спросите у AI агента: "Какая сейчас самая дешёвая TRON energy?" и получите живые цены от всех подключённых поставщиков.

Полная документация MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)