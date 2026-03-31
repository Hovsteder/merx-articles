# Building a USDT Payment Bot with MERX SDK

Sending USDT on TRON should be simple. You have a recipient address, an amount, and a wallet with funds. But if you send USDT without energy, the TRON protocol burns approximately 27 TRX from your wallet to cover the transaction cost. At scale - hundreds or thousands of transfers per day - that burn cost becomes a significant expense.

This tutorial builds a complete USDT payment bot that eliminates that cost. The bot receives payment requests, rents energy through MERX at a fraction of the burn price, sends the USDT with the rented energy, and reports the savings. It handles errors, supports retries, and integrates webhook notifications for production reliability.

By the end, you will have a working payment bot that saves over 90 percent on every USDT transfer.

## Architecture Overview

The bot follows a straightforward pipeline:

1. Receive a payment request (recipient address + amount).
2. Check your MERX account balance.
3. Estimate the energy required for the transfer.
4. Create an energy order through MERX.
5. Wait for the energy delegation to complete.
6. Send the USDT transfer using the delegated energy.
7. Report the transaction and savings.

Each step is isolated and retryable. If the energy order fails, the USDT is never sent. If the USDT transfer fails, you still have the energy (it expires after the rental period, but no funds are lost).

## Prerequisites

Before building, you need:

- **Node.js 18 or later** - the bot uses ES modules and modern JavaScript features.
- **A MERX account with balance** - sign up at [merx.exchange](https://merx.exchange) and deposit TRX.
- **A MERX API key** - create one with `create_orders`, `view_orders`, and `view_balance` permissions.
- **A TRON wallet** - with USDT balance for the payments and a small amount of TRX for bandwidth.
- **TronWeb** - for signing and broadcasting the USDT transfer transaction.

### Project Setup

```bash
mkdir usdt-payment-bot
cd usdt-payment-bot
npm init -y
npm install merx-sdk tronweb dotenv uuid
```

Create a `.env` file (never commit this to version control):

```bash
MERX_API_KEY=merx_live_your_key_here
TRON_PRIVATE_KEY=your_tron_wallet_private_key
TRON_FULL_HOST=https://api.trongrid.io
USDT_CONTRACT=TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t
MIN_SAVINGS_PERCENT=50
```

## Step 1: Initialize the Clients

Create the MERX client and TronWeb instance:

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

## Step 2: Check MERX Balance

Before doing anything, verify you have sufficient balance to rent energy:

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

## Step 3: Estimate Energy Cost

Use the MERX estimation API to determine how much energy the transfer needs and what it will cost:

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

## Step 4: Create the Energy Order

Order energy through MERX with an idempotency key for safe retries:

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

## Step 5: Wait for Energy Delegation

After creating the order, wait for the energy to be delegated on-chain. The bot polls the order status with a timeout:

```javascript
// src/wait.js
import { merx } from './clients.js';

export async function waitForFill(orderId, timeoutMs = 60000) {
  const startTime = Date.now();
  const pollInterval = 2000; // 2 seconds

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

    // Still pending or processing - wait and poll again
    await new Promise((resolve) => setTimeout(resolve, pollInterval));
  }

  throw new Error(`Order ${orderId} timed out after ${timeoutMs / 1000}s`);
}
```

For production systems, consider using webhooks instead of polling (covered later in this article).

## Step 6: Send the USDT Transfer

With energy delegated to your address, send the USDT transfer. The energy is consumed instead of burning TRX:

```javascript
// src/transfer.js
import { tronWeb, USDT_CONTRACT, SENDER_ADDRESS } from './clients.js';

export async function sendUSDT(recipientAddress, amountUSDT) {
  // USDT has 6 decimals on TRON
  const amountSun = Math.floor(amountUSDT * 1_000_000);

  // Validate recipient address
  if (!tronWeb.isAddress(recipientAddress)) {
    throw new Error(`Invalid TRON address: ${recipientAddress}`);
  }

  // Build the TRC-20 transfer transaction
  const contract = await tronWeb.contract().at(USDT_CONTRACT);

  const tx = await contract.methods
    .transfer(recipientAddress, amountSun)
    .send({
      from: SENDER_ADDRESS,
      feeLimit: 100_000_000, // 100 TRX fee limit (safety cap)
    });

  return {
    txHash: tx,
    recipient: recipientAddress,
    amount: amountUSDT,
    amountSun,
  };
}
```

## Step 7: Putting It All Together

The main bot function orchestrates all steps:

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

  // Step 1: Estimate costs
  console.log('\n[1/5] Estimating energy cost...');
  const estimate = await estimateTransferCost();
  console.log(`  Energy required: ${estimate.energyRequired}`);
  console.log(`  Burn cost: ${estimate.burnCostTRX}`);
  console.log(`  Rental cost: ${estimate.rentalCostTRX}`);
  console.log(`  Savings: ${estimate.savingsPercent}%`);

  // Step 2: Decide whether renting is worthwhile
  if (estimate.savingsPercent < MIN_SAVINGS_PERCENT) {
    console.log(`  Savings below threshold (${MIN_SAVINGS_PERCENT}%). Skipping energy rental.`);
    const tx = await sendUSDT(recipientAddress, amountUSDT);
    return { ...tx, energyRented: false, savings: null };
  }

  // Step 3: Check MERX balance
  console.log('\n[2/5] Checking MERX balance...');
  const balance = await checkMerxBalance(estimate.rentalCostSun);
  console.log(`  Available: ${balance.availableTRX} TRX`);

  // Step 4: Create energy order
  console.log('\n[3/5] Creating energy order...');
  const order = await createEnergyOrder(
    estimate.energyRequired,
    SENDER_ADDRESS,
    estimate.durationHours
  );
  console.log(`  Order ID: ${order.orderId}`);
  console.log(`  Provider: ${order.provider}`);
  console.log(`  Status: ${order.status}`);

  // Step 5: Wait for energy delegation
  console.log('\n[4/5] Waiting for energy delegation...');
  const fill = await waitForFill(order.orderId);
  console.log(`  Filled by: ${fill.provider}`);
  console.log(`  Delegation TX: ${fill.txHash}`);

  // Step 6: Send USDT with delegated energy
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

### Running the Bot

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

Sample output:

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

## Error Handling

Production payment systems need robust error handling. Here are the failure modes and how to handle each.

### Insufficient MERX Balance

If your MERX account runs low, the bot should alert you and optionally fall back to burning TRX:

```javascript
try {
  await checkMerxBalance(estimate.rentalCostSun);
} catch (err) {
  if (err.message.includes('Insufficient MERX balance')) {
    console.warn('MERX balance low. Sending without energy rental.');
    // Optionally: send alert to ops team
    // await alertOps('MERX balance low', err.message);
    const tx = await sendUSDT(recipientAddress, amountUSDT);
    return { ...tx, energyRented: false, fallbackReason: 'low_balance' };
  }
  throw err;
}
```

### Order Fill Timeout

If the energy provider takes too long to delegate, cancel and retry with a different provider or fall back:

```javascript
try {
  const fill = await waitForFill(order.orderId, 30000); // 30-second timeout
} catch (err) {
  if (err.message.includes('timed out')) {
    console.warn('Energy delegation timed out. Sending without energy.');
    const tx = await sendUSDT(recipientAddress, amountUSDT);
    return { ...tx, energyRented: false, fallbackReason: 'delegation_timeout' };
  }
  throw err;
}
```

### USDT Transfer Failure

If the USDT transfer itself fails (insufficient balance, contract error), the energy is not wasted - it remains delegated for the rental duration. You can retry the transfer:

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

## Webhook Notifications

Polling for order status works for simple bots, but production systems should use webhooks. MERX sends HTTP POST notifications to your webhook URL when order status changes.

### Setting Up Webhooks

Configure your webhook endpoint in the MERX dashboard or via the API:

```bash
curl -X POST https://merx.exchange/api/v1/webhooks \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-server.com/webhooks/merx",
    "events": ["order.filled", "order.failed"]
  }'
```

### Handling Webhook Events

```javascript
// webhook-handler.js
import express from 'express';

const app = express();
app.use(express.json());

// Store pending payment callbacks
const pendingPayments = new Map();

app.post('/webhooks/merx', (req, res) => {
  const event = req.body;

  // Verify webhook signature (production requirement)
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

// Replace polling with webhook-based waiting
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

Webhooks eliminate the polling loop, reduce API calls, and respond faster to order fills (sub-second notification versus 2-second poll intervals).

## Production Considerations

### Concurrency

If your bot processes multiple payments simultaneously, each payment should have its own idempotency key and operate independently. Use a queue (Bull, BullMQ, or similar) to manage concurrent payments:

```javascript
import Queue from 'bull';

const paymentQueue = new Queue('payments', {
  redis: { host: '127.0.0.1', port: 6379 },
});

paymentQueue.process(5, async (job) => {
  // Process up to 5 payments concurrently
  const { recipient, amount } = job.data;
  return await processPayment(recipient, amount);
});

// Add a payment to the queue
await paymentQueue.add({
  recipient: 'TRecipientAddressHere',
  amount: 100,
});
```

### Monitoring and Alerting

Track key metrics for operational visibility:

- **Payments per hour** - throughput tracking.
- **Average savings percent** - if savings drop below 80 percent, investigate provider pricing.
- **MERX balance** - alert when balance drops below a threshold (e.g., enough for 100 transfers).
- **Fill time** - if energy delegation takes longer than 30 seconds, providers might be congested.
- **Failure rate** - percentage of payment attempts that fail at any step.

Track these in your preferred monitoring tool (Prometheus, Datadog, or even a simple JSON log).

### Rate Limits

The MERX API limits order creation to 10 requests per minute. If your bot needs to send more than 10 payments per minute, you have two options:

1. **Queue payments** and process them at a rate within the limit.
2. **Batch energy purchases** - buy enough energy for multiple transfers in a single order, then send the transfers sequentially while the energy is active.

Option 2 is more efficient for high-volume scenarios:

```javascript
async function processBatch(payments) {
  // Calculate total energy for all payments
  const totalEnergy = payments.length * 65000; // ~65K per USDT transfer

  // Single energy order for the entire batch
  const order = await createEnergyOrder(totalEnergy, SENDER_ADDRESS, 1);
  await waitForFill(order.orderId);

  // Send all USDT transfers while energy is active
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

### Security Checklist

Before deploying to production:

- Store the TRON private key in a secrets manager, not in `.env` on disk.
- Run the bot in a restricted environment with no inbound network access except the webhook endpoint.
- Use a dedicated TRON wallet for the bot with only the USDT needed for near-term payments. Do not keep your entire treasury in the bot wallet.
- Implement withdrawal limits - the bot should not be able to drain the wallet in a single run.
- Log every payment with full details for audit trails.
- Set up alerts for unusual activity (payment amounts above threshold, rapid successive failures).

## Complete Project Structure

```
usdt-payment-bot/
  .env                  # Never commit
  .gitignore
  package.json
  index.js              # Entry point
  src/
    clients.js          # MERX + TronWeb initialization
    balance.js          # Balance checking
    estimate.js         # Cost estimation
    order.js            # Energy order creation
    wait.js             # Order fill polling
    transfer.js         # USDT transfer execution
    bot.js              # Main orchestration
    webhook-handler.js  # Webhook receiver (optional)
```

Each file has a single responsibility. The total codebase is under 300 lines. Mock the MERX client for unit tests, use the Shasta testnet for integration tests.

## Conclusion

This payment bot demonstrates the core MERX integration pattern: estimate, order, wait, transact. The same pattern applies whether you are building a payment processor, a wallet, an exchange withdrawal system, or any application that sends TRC-20 tokens.

The key takeaway is the savings math. At 94 percent savings per transfer, a bot processing 1,000 USDT transfers per day saves approximately 26,000 TRX daily - over 750,000 TRX per month. That is not an optimization. That is a fundamental cost structure change.

Start on the Shasta testnet (`TRON_FULL_HOST=https://api.shasta.trongrid.io`) to validate the flow without risking real funds, then switch to mainnet when everything works.

- MERX platform: [merx.exchange](https://merx.exchange)
- API documentation: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- Python SDK: [github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)
- MCP server for AI agents: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
