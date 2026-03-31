# Provider Reliability: Uptime, Speed, and Fill Rates Compared

Price is the most visible metric when choosing a TRON energy provider. But price alone does not tell the full story. A provider quoting 22 SUN is worthless if the order takes 10 minutes to fill, fails 15% of the time, or the delegation drops before the stated duration expires.

This article examines the reliability dimensions that matter beyond price: uptime, fill speed, fill rates, and consistency. It explains how MERX tracks these metrics across all seven providers and how aggregation fundamentally improves reliability compared to single-provider dependence.

## Why Reliability Matters

For a one-off energy purchase, reliability is a minor concern. If your order fails, you try again. If it takes 5 minutes instead of 30 seconds, you wait.

For automated systems -- payment processors, trading bots, distribution services -- reliability is a critical operational parameter. A failed energy order can cascade into a failed transaction, which cascades into a failed payment, which costs real money and erodes user trust.

### The True Cost of Unreliability

Consider a payment processor handling 500 USDT transfers per day. Each transfer requires energy. If the energy provider has a 95% fill rate (which sounds high), 5% of orders fail. That is 25 failed energy purchases per day.

Each failure triggers a fallback: either retry (adding latency), buy from an alternative (requiring multi-provider integration), or fall back to TRX burn (paying 5-10x more for that transaction).

At 25 failures per day, the annual cost of a "95% reliable" provider includes:

- 9,125 failed orders requiring manual or automated intervention
- Additional latency on affected transactions
- Higher cost on fallback transactions
- Engineering time to build and maintain retry/fallback logic

A 99.5% fill rate reduces those 25 daily failures to 2.5 -- a 10x improvement in operational smoothness.

## Reliability Dimensions

### Uptime

Uptime measures the percentage of time a provider's API is responsive and accepting orders. This is the most basic reliability metric -- if the API is down, nothing else matters.

Causes of downtime include:

- **Planned maintenance**: Scheduled API updates or infrastructure changes
- **Infrastructure failures**: Server crashes, network issues, database problems
- **Supply exhaustion**: Some providers go offline when their energy supply is depleted rather than returning "unavailable" responses
- **Rate limiting**: Aggressive rate limits can effectively create downtime for high-volume users

An individual provider might maintain 98-99% uptime, which sounds excellent until you calculate the implications: 1% downtime is 87 hours per year, or roughly 15 minutes per day.

### Fill Speed

Fill speed measures the time from order placement to energy delegation appearing on the target address. This varies significantly across providers:

- **Fast providers**: 10-30 seconds. The order is processed, the delegation transaction is broadcast, and the target address receives energy within half a minute.
- **Moderate providers**: 30-120 seconds. Processing takes longer, possibly due to batch delegation or manual approval steps.
- **Slow providers**: 2-10 minutes. Some providers, particularly P2P marketplaces, require matching with a seller before the delegation can occur.

For time-sensitive operations (user-facing payments, trading bots), the difference between 15-second and 5-minute fills is operationally significant.

### Fill Rate

Fill rate measures the percentage of orders that successfully complete. An order can fail for several reasons:

- **Insufficient supply**: The provider accepted the order but cannot fulfill it
- **Delegation failure**: The on-chain delegation transaction fails
- **Timeout**: The order is not filled within the expected timeframe
- **Payment issues**: Internal payment processing fails

Fill rates vary by provider and by order parameters. A provider might have a 99% fill rate for 65,000-energy orders but only 85% for 5,000,000-energy orders due to supply constraints.

### Delegation Consistency

Consistency measures whether the energy delegation persists for the full stated duration. A provider selling "1-hour" energy should maintain the delegation for a full 60 minutes, not 45 minutes.

Some providers have been observed to:

- End delegations early (particularly during supply crunches)
- Fail to extend delegations on longer-duration orders
- Reduce delegated amounts mid-duration

These consistency issues are difficult for individual buyers to detect but have real cost implications -- if your 1-hour energy disappears after 40 minutes, transactions in the remaining 20 minutes burn TRX.

## How MERX Tracks Provider Health

MERX maintains continuous monitoring across all seven providers, tracking metrics that individual buyers cannot practically measure on their own.

### Health Monitoring

```typescript
import { MerxClient } from '@anthropic/merx-sdk';

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY });

// Compare providers for your specific order profile
const comparison = await merx.compareProviders({
  energy_amount: 65000,
  duration: '1h'
});

for (const provider of comparison.providers) {
  console.log(`${provider.name}:`);
  console.log(`  Price: ${provider.price_sun} SUN`);
  console.log(`  Available: ${provider.available}`);
  console.log(`  Avg fill time: ${provider.avg_fill_seconds}s`);
  console.log(`  Fill rate: ${provider.fill_rate}%`);
}
```

### What MERX Measures

For each provider, MERX tracks:

- **API response time**: How quickly the provider's API responds to queries
- **Order fill time**: Time from order placement to confirmed delegation
- **Fill rate**: Percentage of orders that complete successfully
- **Price accuracy**: Whether the filled price matches the quoted price
- **Duration compliance**: Whether delegations persist for the stated period
- **Error patterns**: Types and frequency of errors

This data feeds into MERX's routing algorithm. When prices are equal between two providers, the more reliable one gets the order.

## Aggregation and Reliability

The most powerful reliability benefit of aggregation is not any single metric improvement -- it is the elimination of single-provider dependency.

### Single Provider Reliability Model

With one provider at 99% uptime and 97% fill rate:

- Effective success rate: 99% x 97% = 96.03%
- Annual failed orders (at 500 orders/day): 7,244
- Monthly failed orders: 604

### Aggregated Reliability Model (7 providers)

With MERX routing across seven providers, the system fails only when all providers simultaneously fail. Even if each provider individually has 99% uptime:

- Probability of all 7 being down simultaneously: 0.01^7 = 10^-14 (effectively zero)
- Effective uptime: essentially 100% (limited only by MERX's own infrastructure)

For fill rate, the aggregated model means that if the primary provider cannot fill an order, it routes to the next available provider automatically:

```
Order placed
  |
  v
Provider 1 (cheapest): order failed
  |
  v
Provider 2: order filled at slightly higher price
  |
  v
Energy delegated to target address
```

The buyer experiences a slightly higher price (the second-cheapest instead of the cheapest) but the order fills. Without aggregation, the same scenario results in a complete failure requiring manual intervention.

### Failover Transparency

MERX's failover is transparent to the buyer. The API response indicates which provider filled the order, but the buyer's code does not need to handle provider-specific failure cases:

```typescript
const order = await merx.createOrder({
  energy_amount: 65000,
  duration: '1h',
  target_address: wallet
});

// order.provider tells you who filled it
// Your code never needs to handle provider failures
console.log(`Filled by: ${order.provider}`);
```

Compare this with manual failover:

```typescript
// Without aggregation: manual failover is complex
let filled = false;
for (const provider of [providerA, providerB, providerC]) {
  try {
    const order = await provider.buyEnergy(65000, '1h', wallet);
    filled = true;
    break;
  } catch (error) {
    // Handle provider-specific error
    // Different error format for each provider
    // Different retry logic for each provider
    continue;
  }
}
if (!filled) {
  // All providers failed -- handle the crisis
}
```

The aggregated approach eliminates this entire failover codebase.

## Provider Reliability Characteristics

Based on general market observations (specific metrics vary over time):

### P2P Providers (TronSave)

- **Uptime**: Generally high (99%+)
- **Fill speed**: Variable (30 seconds to several minutes depending on seller matching)
- **Fill rate**: Lower for large orders (supply depends on active sellers)
- **Consistency**: Generally good once delegation is established

### Fixed-Price Providers (PowerSun)

- **Uptime**: High (99%+)
- **Fill speed**: Typically fast (15-60 seconds)
- **Fill rate**: High for standard orders within supply limits
- **Consistency**: Excellent -- fixed model incentivizes reliable delivery

### Dynamic Providers (Feee, Catfee, Netts, iTRX, Sohu)

- **Uptime**: Varies by provider (97-99.5%)
- **Fill speed**: Generally moderate (15-90 seconds)
- **Fill rate**: Varies, generally 95-99% for standard orders
- **Consistency**: Generally good, occasional early delegation termination

## Building Reliability-Aware Systems

For systems where reliability is paramount, combine MERX aggregation with application-level resilience:

```typescript
async function reliableEnergyPurchase(
  amount: number,
  wallet: string,
  maxAttempts: number = 3
): Promise<Order> {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      const order = await merx.createOrder({
        energy_amount: amount,
        duration: '5m',
        target_address: wallet
      });

      // Wait for fill confirmation
      const filled = await waitForFill(order.id, {
        timeout: 60000
      });

      if (filled) {
        return order;
      }

      // Order timed out -- MERX may have already routed
      // to another provider internally

    } catch (error) {
      if (attempt === maxAttempts) {
        // Final fallback: accept TRX burn
        console.warn(
          'Energy purchase failed after all attempts. ' +
          'Transaction will burn TRX.'
        );
        throw error;
      }
      // Brief pause before retry
      await delay(2000 * attempt);
    }
  }

  throw new Error('Energy purchase failed');
}
```

Note that even the retry logic here is simpler than manual multi-provider failover because MERX handles the provider routing internally. Your retry logic only needs to handle the rare case where the aggregation layer itself encounters issues.

## Measuring Your Own Reliability

Track these metrics for your specific operations:

```typescript
interface ReliabilityMetrics {
  totalOrders: number;
  successfulFills: number;
  failedOrders: number;
  averageFillTimeMs: number;
  medianFillTimeMs: number;
  p95FillTimeMs: number;
  trxBurnEvents: number; // Times energy was insufficient
  providerDistribution: Record<string, number>;
}
```

Monitor these over time. If your fill rate drops or fill times increase, it may indicate market-wide supply issues, and you should adjust your purchasing strategy (higher price targets, earlier purchasing, larger buffers).

## Conclusion

Provider reliability encompasses far more than whether the API responds. Fill speed, fill rate, delegation consistency, and failover capability all determine whether your energy procurement actually supports your operations or introduces failure points.

No single provider guarantees perfect reliability. The aggregation model does not guarantee perfection either, but it achieves near-perfect practical reliability by eliminating single-provider dependency. When seven providers back your energy supply, the probability of complete failure drops to essentially zero.

For any automated system where transaction throughput matters, the reliability improvement from aggregation is as valuable as the price optimization -- and often more valuable, because a single critical failure at the wrong moment can cost more than years of price savings.

Explore MERX's provider comparison tools at [https://merx.exchange/docs](https://merx.exchange/docs) or test the platform at [https://merx.exchange](https://merx.exchange).
