# 5 Satir Kodla TRON Energy Satin Alin: MERX JavaScript SDK

The MERX JavaScript SDK lets you buy TRON energy programmatically in five lines of code - install the package, initialize the client, check prices, create an order, and verify the delegation. This article covers the complete SDK from installation through production deployment, including TypeScript types, error handling, and every method available across the four modules: prices, orders, balance, and webhooks.

## The Problem with Direct Provider APIs

Every TRON energy provider has its own API. Integrating with one is straightforward. Integrating with all seven or eight providers to guarantee the best price is an engineering project. You need to handle different authentication schemes, response formats, error codes, rate limits, and failover logic.

The MERX SDK abstracts all of that into a single client with a consistent interface. Behind the scenes, the MERX platform polls all providers every 30 seconds, maintains a real-time price index, and routes your orders to the cheapest available source with automatic failover.

## Kurulum

The SDK requires Node.js 18 or later. It uses native `fetch` with zero runtime dependencies.

```bash
npm install merx-sdk
```

```bash
yarn add merx-sdk
```

```bash
pnpm add merx-sdk
```

The package ships as ESM with full TypeScript declarations and source maps.

## Five Lines to Buy Energy

Here is the complete flow in its minimal form:

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

Line by line:

1. Import the client class.
2. Initialize with your API key. The key starts with `sk_live_` and is created in the MERX dashboard at [merx.exchange](https://merx.exchange).
3. Fetch current prices from all connected providers. This is optional but useful for displaying the market state before ordering.
4. Create a market order for 65,000 energy units delegated to your target address for one hour. The platform automatically selects the cheapest provider.
5. Log the order ID and status. The order begins in `PENDING` status and transitions to `FILLED` once the on-chain delegation is confirmed.

That is the entire integration. No provider selection, no failover logic, no price comparison code.

## Client Configuration

```typescript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({
  apiKey: 'sk_live_your_key_here',   // Required
  baseUrl: 'https://merx.exchange',  // Optional, defaults to production
})
```

The `apiKey` is the only required option. The `baseUrl` can be overridden for testing against a staging environment.

## The Prices Module

The `merx.prices` module provides five methods for querying real-time market data.

### prices.list()

Returns current pricing from all active providers.

```typescript
const prices = await merx.prices.list()

for (const p of prices) {
  const cheapest = p.energy_prices[0]
  if (cheapest) {
    console.log(`${p.provider}: ${cheapest.price_sun} SUN/energy, ${p.available_energy} available`)
  }
}
```

Each `ProviderPrice` object includes:
- `provider` - provider identifier (e.g., "sohu", "catfee", "tronsave")
- `is_market` - whether this provider supports market orders with flexible durations
- `energy_prices` - array of `{ duration_sec, price_sun }` tiers
- `bandwidth_prices` - same structure for bandwidth
- `available_energy` - current capacity in energy units
- `available_bandwidth` - current capacity in bandwidth units
- `fetched_at` - Unix timestamp of the last successful poll

### prices.best(resource, amount?)

Returns the single cheapest price point for a given resource type.

```typescript
const best = await merx.prices.best('ENERGY')
console.log(`Cheapest: ${best.price_sun} SUN from ${best.provider}`)

// With minimum amount filter
const bestLarge = await merx.prices.best('ENERGY', 500000)
```

The optional `amount` parameter filters out providers that do not have enough capacity to fulfill your order.

### prices.history(params?)

Returns historical price data for analysis and charting.

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

Available periods: `'1h'`, `'6h'`, `'24h'`, `'7d'`, `'30d'`. All filter parameters are optional.

### prices.stats()

Returns aggregate statistics across the entire market.

```typescript
const stats = await merx.prices.stats()
console.log(`Best price: ${stats.best_price_sun} SUN`)
console.log(`Average: ${stats.avg_price_sun} SUN`)
console.log(`Providers online: ${stats.total_providers}`)
console.log(`Cheapest-provider changes (24h): ${stats.cheapest_changes_24h}`)
```

### prices.preview(params)

Previews what an order would cost before placing it. Returns the best matching provider and fallback options.

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

The `max_price_sun` parameter filters out providers that exceed your price ceiling.

## The Orders Module

### orders.create(params)

Creates an energy or bandwidth order.

```typescript
const order = await merx.orders.create({
  resource_type: 'ENERGY',
  amount: 65000,
  target_address: 'TYourTargetAddress',
  duration_sec: 3600,
  order_type: 'MARKET',          // Optional, defaults to 'MARKET'
  max_price_sun: 30,             // Optional, required for LIMIT orders
  idempotency_key: 'unique-id',  // Optional, prevents duplicate orders
})

console.log(`Order ${order.id} created`)
console.log(`Status: ${order.status}`)
```

Order types:
- `MARKET` - execute immediately at the best available price
- `LIMIT` - execute only if price is at or below `max_price_sun`
- `PERIODIC` - recurring order
- `BROADCAST` - broadcast a pre-signed delegation transaction

The `idempotency_key` is critical for production systems. If a network error occurs and you retry the request with the same key, the API returns the original order instead of creating a duplicate.

### orders.list(limit?, offset?, status?)

Lists orders with pagination.

```typescript
const { orders, total } = await merx.orders.list(10, 0, 'FILLED')
console.log(`${total} filled orders total`)

for (const o of orders) {
  console.log(`${o.id}: ${o.amount} ${o.resource_type}, cost: ${o.total_cost_sun} SUN`)
}
```

### orders.get(id)

Returns a single order with fill details.

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

The `fills` array shows exactly how the order was fulfilled. Each fill includes the provider name, the amount allocated, the price per unit, the cost, the on-chain transaction ID, and whether the delegation has been verified on-chain.

## The Balance Module

### balance.get()

Returns current account balances.

```typescript
const balance = await merx.balance.get()
console.log(`TRX: ${balance.trx}`)
console.log(`USDT: ${balance.usdt}`)
console.log(`Locked: ${balance.trx_locked}`)
```

The `trx_locked` field shows TRX that is currently reserved for pending orders.

### balance.depositInfo()

Returns the deposit address and memo for funding your MERX account.

```typescript
const info = await merx.balance.depositInfo()
console.log(`Send TRX to: ${info.address}`)
console.log(`Include memo: ${info.memo}`)
console.log(`Minimum deposit: ${info.min_amount_trx} TRX`)
```

The memo is required for automated deposit crediting. Deposits without the correct memo require manual processing.

### balance.withdraw(params)

Withdraws TRX or USDT to an external TRON address.

```typescript
const withdrawal = await merx.balance.withdraw({
  address: 'TYourExternalAddress',
  amount: 100,
  currency: 'TRX',
  idempotency_key: 'withdraw-unique-id',
})

console.log(`Withdrawal ${withdrawal.id}: ${withdrawal.status}`)
```

### balance.history(period?) and balance.summary()

```typescript
const history = await merx.balance.history('7D')
console.log(`${history.length} transactions in last 7 days`)

const summary = await merx.balance.summary()
console.log(`Total orders: ${summary.total_orders}`)
console.log(`Total energy: ${summary.total_energy}`)
console.log(`Average price: ${summary.avg_price_sun} SUN`)
```

## The Webhooks Module

### webhooks.create(params)

Creates a webhook subscription. The `secret` is returned only at creation time.

```typescript
const webhook = await merx.webhooks.create({
  url: 'https://your-server.com/merx-webhook',
  events: ['order.filled', 'order.failed', 'deposit.received'],
})

console.log(`Webhook ID: ${webhook.id}`)
console.log(`Secret: ${webhook.secret}`)
```

### webhooks.list() and webhooks.delete(id)

```typescript
const webhooks = await merx.webhooks.list()
console.log(`${webhooks.filter(w => w.is_active).length} active webhooks`)

await merx.webhooks.delete('wh_abc123')
```

## Hata Yonetimi

All API errors are thrown as `MerxError` instances with a machine-readable `code` and a human-readable `message`.

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

Common error codes:

| Code                   | Meaning                                          |
|------------------------|--------------------------------------------------|
| `UNAUTHORIZED`         | Invalid or missing API key                       |
| `INSUFFICIENT_FUNDS`   | Account balance too low                          |
| `INVALID_ADDRESS`      | Target address is not a valid TRON address        |
| `ORDER_NOT_FOUND`      | Order ID does not exist                          |
| `INVALID_AMOUNT`       | Amount below minimum or exceeds limits           |
| `DUPLICATE_REQUEST`    | Idempotency key already used                     |
| `RATE_LIMITED`         | Too many requests                                |
| `PROVIDER_UNAVAILABLE` | No providers available                           |

## TypeScript Types

All types are exported from the package and can be used for type annotations throughout your application:

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

The SDK is strict-mode compatible and produces no `any` types. Every response is fully typed.

## Production Example: Automated USDT Transfer Flow

Here is a complete production pattern that checks prices, creates an order, and polls for completion:

```typescript
import { MerxClient, MerxError } from 'merx-sdk'
import { randomUUID } from 'node:crypto'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY! })

async function ensureEnergyAndTransfer(targetAddress: string) {
  // Preview the cost
  const preview = await merx.prices.preview({
    resource: 'ENERGY',
    amount: 65000,
    duration: 3600,
  })

  if (preview.no_providers) {
    throw new Error('No energy providers available')
  }

  console.log(`Best price: ${preview.best!.cost_trx} TRX from ${preview.best!.provider}`)

  // Create the order with idempotency
  const order = await merx.orders.create({
    resource_type: 'ENERGY',
    amount: 65000,
    target_address: targetAddress,
    duration_sec: 3600,
    idempotency_key: randomUUID(),
  })

  // Poll until filled (in production, use webhooks instead)
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

  // Energy is now available on the target address
  // Proceed with your USDT transfer
}
```

In production, replace the polling loop with a webhook listener. Create a webhook subscription for the `order.filled` event, and your server will be notified the moment the delegation is confirmed on-chain.

## Gereksinimler ve Uyumluluk

- Node.js 18+ (uses native `fetch`)
- Zero runtime dependencies
- ESM only (ships as ES modules)
- Full TypeScript 5.4+ support
- Works with Bun and Deno (any runtime that supports `fetch`)

## Kaynaklar

The SDK is open source and available on GitHub. Contributions and bug reports are welcome.

- Platform: [merx.exchange](https://merx.exchange)
- Dokumantasyon: [merx.exchange/docs](https://merx.exchange/docs)
- GitHub: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- npm: [npmjs.com/package/merx-sdk](https://www.npmjs.com/package/merx-sdk)
- Python SDK: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- Yapay zeka ajanlari icin MCP Sunucusu: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)

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
