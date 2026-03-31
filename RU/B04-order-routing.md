# Маршрутизация ордеров MERX: как ваш заказ получает лучшую цену

When you submit an energy order to MERX, a sequence of decisions happens in milliseconds: which provider has the best price, can they fill this order, what happens if they fail, how do we verify the delegation landed on-chain. This is the order routing engine - the component that turns a simple API call into an optimized, fault-tolerant, verified energy delivery.

This article walks through the complete lifecycle of a MERX order, from the moment you call `createOrder` to the moment delegated energy appears at your TRON address.

---

## The Order Lifecycle

Every order passes through six stages:

```
1. Validation    -> Is the order well-formed?
2. Pricing       -> What is the best price right now?
3. Routing       -> Which provider(s) will fill it?
4. Execution     -> Submit to provider(s)
5. Verification  -> Confirm on-chain delegation
6. Settlement    -> Debit balance, record in ledger
```

Let us walk through each stage.

---

## Stage 1: Validation

Before any routing logic runs, the order is validated:

```typescript
// Input validation with Zod
const OrderSchema = z.object({
  energy: z.number().int().min(10000).max(100000000),
  targetAddress: z.string().refine(isValidTronAddress),
  duration: z.enum(['1h', '1d', '3d', '7d', '14d', '30d']),
  maxPrice: z.number().optional(),  // SUN per energy unit
  idempotencyKey: z.string().optional()
});
```

Key validations:

- **Energy amount**: must be a positive integer within supported range.
- **Target address**: must be a valid, activated TRON address. The system validates the address format and optionally checks on-chain that the account exists.
- **Duration**: must be one of the supported delegation periods.
- **Max price**: optional ceiling. If set, the order only executes if the best available price is at or below this threshold.
- **Idempotency key**: if provided, duplicate submissions with the same key return the original order instead of creating a new one.

### Idempotency

The idempotency key is critical for production integrations. Network issues can cause a client to retry a request, potentially creating duplicate orders. With an idempotency key, the second request returns the result of the first:

```typescript
const order = await client.createOrder({
  energy: 65000,
  targetAddress: 'TBuyerAddress...',
  duration: '1h',
  idempotencyKey: 'payment-123-energy'
});

// If called again with the same key, returns the same order
// No duplicate delegation, no double charge
```

---

## Stage 2: Pricing

The order executor reads current prices from the Redis cache. The price monitor updates these every 30 seconds, so the data is at most 30 seconds old.

```typescript
async function getBestPrices(
  energyAmount: number,
  duration: string
): Promise<ProviderPrice[]> {

  const allPrices = await redis.keys('prices:*');
  const validPrices = [];

  for (const key of allPrices) {
    const price = JSON.parse(await redis.get(key));

    // Filter: must support requested duration
    if (!price.durations.includes(duration)) continue;

    // Filter: must have sufficient availability
    if (price.availableEnergy < energyAmount) continue;

    // Filter: must pass health threshold
    const health = await getProviderHealth(price.provider);
    if (health.fillRate < 0.90) continue;

    validPrices.push(price);
  }

  // Sort by effective price (accounting for reliability)
  return validPrices.sort((a, b) => {
    const effectiveA = a.energyPricePerUnit / a.fillRate;
    const effectiveB = b.energyPricePerUnit / b.fillRate;
    return effectiveA - effectiveB;
  });
}
```

### Max Price Enforcement

If the buyer specified a `maxPrice`, the order is rejected if no provider can meet it:

```typescript
if (maxPrice && bestPrice.energyPricePerUnit > maxPrice) {
  throw new OrderError({
    code: 'PRICE_EXCEEDED',
    message: `Best available price (${bestPrice.energyPricePerUnit} SUN) exceeds your maximum (${maxPrice} SUN)`,
    details: { bestAvailable: bestPrice.energyPricePerUnit, maxPrice }
  });
}
```

This prevents unexpected charges during price spikes.

---

## Stage 3: Routing

The routing engine decides which provider(s) will fill the order. This is the core intelligence of the system.

### Simple Case: Single Provider Fill

For orders within a single provider's capacity:

```
Order: 65,000 energy
Best provider: itrx at 85 SUN/unit, 500,000 available

Route: 100% to itrx
```

### Split Case: Multi-Provider Fill

For large orders or when the cheapest provider has limited stock:

```
Order: 500,000 energy

Provider A: 200,000 available at 85 SUN
Provider B: 180,000 available at 87 SUN
Provider C: 300,000 available at 92 SUN

Routing plan:
  Leg 1: Provider A -> 200,000 energy at 85 SUN
  Leg 2: Provider B -> 180,000 energy at 87 SUN
  Leg 3: Provider C -> 120,000 energy at 92 SUN

Blended rate: (200K*85 + 180K*87 + 120K*92) / 500K = 87.28 SUN
```

The router fills from cheapest to most expensive, taking as much as possible from each provider before moving to the next.

### The Failover Chain

Every routing plan includes a failover chain - an ordered list of alternative providers to try if the primary fails:

```
Primary:   Provider A (85 SUN)
Failover 1: Provider B (87 SUN)
Failover 2: Provider C (92 SUN)
Failover 3: Provider D (95 SUN)
```

If Provider A fails to execute (API error, timeout, insufficient funds), the executor automatically moves to Provider B without any action from the buyer.

---

## Stage 4: Execution

The executor submits the order to the selected provider(s) and monitors for completion.

### Execution Flow

```typescript
async function executeOrder(
  order: Order,
  routingPlan: RoutingPlan
): Promise<ExecutionResult> {

  const results: LegResult[] = [];

  for (const leg of routingPlan.legs) {
    try {
      const result = await executeLeg(leg, order);
      results.push(result);
    } catch (error) {
      // Primary provider failed - try failover
      const failoverResult = await executeWithFailover(
        leg,
        order,
        routingPlan.failoverChain
      );
      results.push(failoverResult);
    }
  }

  return {
    orderId: order.id,
    legs: results,
    totalEnergy: results.reduce((sum, r) => sum + r.energy, 0),
    totalCostSun: results.reduce((sum, r) => sum + r.costSun, 0)
  };
}

async function executeWithFailover(
  failedLeg: RoutingLeg,
  order: Order,
  failoverChain: Provider[]
): Promise<LegResult> {

  for (const provider of failoverChain) {
    try {
      const result = await executeLeg(
        { ...failedLeg, provider: provider.name },
        order
      );
      return result;
    } catch (error) {
      // Log and continue to next failover
      continue;
    }
  }

  throw new OrderError({
    code: 'ALL_PROVIDERS_FAILED',
    message: 'Order could not be filled by any available provider'
  });
}
```

### Timeout Handling

Each provider execution has a strict timeout. If the provider does not acknowledge the order within the timeout window (typically 30 seconds), the executor moves to the failover chain:

```
T+0:    Submit order to Provider A
T+30s:  No response -> timeout, failover to Provider B
T+31s:  Submit order to Provider B
T+35s:  Provider B acknowledges -> proceed to verification
```

---

## Stage 5: Verification

Acknowledgment from the provider is not enough. MERX verifies that the energy delegation actually appears on the TRON blockchain.

### On-Chain Verification Process

```typescript
async function verifyDelegation(
  targetAddress: string,
  expectedEnergy: number,
  delegationTxHash: string
): Promise<VerificationResult> {

  // Step 1: Verify the delegation transaction exists
  const tx = await tronWeb.trx.getTransaction(delegationTxHash);
  if (!tx || tx.ret[0].contractRet !== 'SUCCESS') {
    return { verified: false, reason: 'Transaction not found or failed' };
  }

  // Step 2: Check target address resources
  const resources = await tronWeb.trx.getAccountResources(targetAddress);
  const currentEnergy = resources.EnergyLimit || 0;

  // Step 3: Verify energy increased by expected amount (with tolerance)
  const tolerance = expectedEnergy * 0.02; // 2% tolerance
  if (currentEnergy < expectedEnergy - tolerance) {
    return { verified: false, reason: 'Energy amount below expected' };
  }

  return {
    verified: true,
    txHash: delegationTxHash,
    energyDelivered: currentEnergy,
    verifiedAt: new Date()
  };
}
```

### Why 2% Tolerance

Energy delegation amounts are based on TRX-to-energy conversion ratios that can shift slightly between order placement and delegation confirmation. A 2% tolerance accounts for this without accepting grossly incorrect amounts.

### Verification Timing

```
T+0:    Provider acknowledges order
T+3-6s: Delegation transaction confirms on TRON (1 block)
T+10s:  MERX queries target address resources
T+10s:  Verification complete, buyer notified
```

The entire process from order submission to verified delivery typically takes 15-45 seconds.

---

## Stage 6: Settlement

After verification, the financial settlement occurs:

### Balance Deduction

```sql
-- Atomic balance check and deduction
BEGIN;

SELECT balance_sun FROM accounts
WHERE user_id = $1
FOR UPDATE;

-- Verify sufficient balance
-- If insufficient, ROLLBACK and return error

UPDATE accounts
SET balance_sun = balance_sun - $2
WHERE user_id = $1;

COMMIT;
```

The `SELECT FOR UPDATE` ensures no race condition between balance check and deduction. If two orders are processed simultaneously, the second one will wait for the first to complete before checking the balance.

### Ledger Entry

Every settlement creates an immutable ledger entry:

```sql
INSERT INTO ledger (
  user_id, type, amount_sun,
  reference_type, reference_id,
  balance_before, balance_after,
  created_at
) VALUES (
  $1, 'ORDER_PAYMENT', $2,
  'order', $3,
  $4, $5,
  NOW()
);
```

Ledger entries are append-only. They are never updated or deleted. This creates a complete, auditable history of every financial operation.

---

## Tracking Your Order

The MERX API provides real-time order status through both REST and WebSocket:

```typescript
import { MerxClient } from 'merx-sdk';

const client = new MerxClient({ apiKey: 'your-key' });

// REST: poll order status
const order = await client.getOrder('ord_abc123');
console.log(order.status);
// 'pending' | 'executing' | 'verifying' | 'completed' | 'failed'

// WebSocket: real-time updates
client.onOrderUpdate('ord_abc123', (update) => {
  console.log(`Order ${update.orderId}: ${update.status}`);
  if (update.status === 'completed') {
    console.log(`Delegation TX: ${update.delegationTxHash}`);
  }
});
```

### Order Status Flow

```
pending -> executing -> verifying -> completed
                |                       |
                v                       v
              failed              partially_filled
```

- **pending**: Order received, awaiting execution.
- **executing**: Submitted to provider, awaiting acknowledgment.
- **verifying**: Provider acknowledged, awaiting on-chain confirmation.
- **completed**: Energy verified at target address.
- **failed**: All providers failed. Balance not charged.
- **partially_filled**: Some legs completed, others failed. Balance charged for filled portion.

---

## Обработка ошибок

### Provider Errors

Every provider can fail in unique ways. The order executor normalizes all provider errors into standard error codes:

```json
{
  "error": {
    "code": "PROVIDER_UNAVAILABLE",
    "message": "Primary provider could not fill the order. Failover to secondary provider.",
    "details": {
      "primaryProvider": "tronsave",
      "primaryError": "timeout",
      "filledBy": "feee",
      "priceImpact": "+2 SUN/unit"
    }
  }
}
```

### Partial Fills

For multi-leg orders, some legs may succeed while others fail. MERX handles this by:

1. Completing successful legs normally.
2. Attempting failover for failed legs.
3. If failover also fails, returning a partial fill result.
4. Charging the buyer only for the energy that was actually delivered.

---

## Производительность

Typical order execution times:

```
Validation:    < 5ms
Pricing:       < 10ms (Redis read)
Routing:       < 5ms
Execution:     5-30 seconds (provider API + blockchain)
Verification:  3-10 seconds (blockchain confirmation)

Total: 10-45 seconds from API call to verified delivery
```

The bottleneck is always the blockchain. MERX's internal processing adds less than 50ms of overhead. The rest is waiting for the provider and the TRON network.

---

## Заключение

Order routing is where MERX's value is most tangible. A single API call triggers a cascade of optimizations: best-price selection, multi-provider splitting, automatic failover, on-chain verification, and atomic settlement. Every step is designed to ensure you get the cheapest energy available, delivered reliably, with full auditability.

The complexity is real, but it is MERX's complexity to manage, not yours. Your integration remains a single API call.

Start routing orders at [https://merx.exchange](https://merx.exchange). Full API reference at [https://merx.exchange/docs](https://merx.exchange/docs).

---

*This article is part of the MERX technical series. MERX is the first blockchain resource exchange. SDKs: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) and [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python). MCP server: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp).*
