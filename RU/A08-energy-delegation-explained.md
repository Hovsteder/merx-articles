# Как работает делегирование energy в TRON

Energy delegation is the mechanism that makes the entire TRON energy rental market possible. Without it, energy would be non-transferable - stakers could only use their own energy, and there would be no way to sell or share it. Understanding how delegation works at the protocol level is essential for anyone building on TRON or evaluating energy providers.

This article explains the Stake 2.0 delegation mechanism in detail: how providers delegate energy to buyers, what happens during and after the delegation period, and how MERX manages the full delegation lifecycle across multiple providers.

---

## The Stake 2.0 Foundation

TRON introduced Stake 2.0 (also called Stake v2) as a replacement for the original resource freezing mechanism. The key innovation of Stake 2.0 is the separation of staking from resource usage. Under the original model (Stake 1.0), frozen TRX produced energy that could only be used by the freezer's own address. Stake 2.0 introduced delegation, allowing one address to direct its staked resources to any other address on the network.

### The Three Operations

Stake 2.0 delegation involves three on-chain operations:

**1. Stake (freezeBalanceV2)**

The provider locks TRX in the staking contract. This produces energy proportional to the amount staked and the total network stake. The energy regenerates continuously over a 24-hour window.

```
Provider stakes 36,000 TRX
Network allocates ~65,000 energy/day to provider
```

**2. Delegate (delegateResource)**

The provider directs a portion of their energy to a target address. This is the actual delegation operation. After this transaction confirms, the target address can use the delegated energy as if it were their own.

```json
{
  "owner_address": "TProviderAddress...",
  "receiver_address": "TBuyerAddress...",
  "resource": "ENERGY",
  "balance": 36000000000,
  "lock": true,
  "lock_period": 3
}
```

Key parameters:
- `owner_address`: the provider's address (the delegator)
- `receiver_address`: the buyer's address (the recipient)
- `resource`: ENERGY or BANDWIDTH
- `balance`: the amount of TRX (in SUN) backing this delegation
- `lock`: whether the delegation is locked for a minimum period
- `lock_period`: the lock duration in days (if lock is true)

**3. Undelegate (undelegateResource)**

When the delegation period expires, the provider reclaims the resource by undelegating. After this transaction confirms, the energy stops flowing to the buyer's address and returns to the provider's pool.

```json
{
  "owner_address": "TProviderAddress...",
  "receiver_address": "TBuyerAddress...",
  "resource": "ENERGY",
  "balance": 36000000000
}
```

---

## The Delegation Lifecycle

A complete delegation goes through several phases. Understanding each phase is critical for building reliable systems around energy rental.

### Phase 1: Order Placement

The buyer requests energy for a specific address and duration. This happens off-chain - the buyer pays the provider (directly or through an aggregator like MERX), and the provider queues the delegation.

### Phase 2: On-Chain Delegation

The provider broadcasts the `delegateResource` transaction. This typically confirms within 3-6 seconds on TRON (one block). Once confirmed, the energy is immediately available at the buyer's address.

```
Buyer's effective energy = own_staked_energy + all_delegated_energy
```

Important: the buyer can use delegated energy immediately. There is no warm-up period. As soon as the delegation transaction is confirmed, the buyer's next smart contract call will consume delegated energy.

### Phase 3: Active Delegation Period

During the delegation period, the buyer uses the energy for their operations. Key behaviors during this phase:

- **Regeneration**: delegated energy regenerates at the same rate as self-staked energy - continuously over 24 hours.
- **Consumption priority**: when a buyer has both self-staked and delegated energy, TRON does not distinguish between them. All energy is pooled.
- **Multiple delegations**: a buyer can receive delegations from multiple providers simultaneously. The energy amounts are additive.

### Phase 4: Expiry and Reclamation

When the lock period expires, the provider can execute the `undelegateResource` transaction to reclaim their resources. This is not automatic - the provider must actively call this function.

```
Timeline:
  T+0:   delegateResource confirmed (energy active)
  T+3d:  Lock period expires
  T+3d+: Provider calls undelegateResource (energy removed)
```

If the provider does not undelegate, the energy continues to flow to the buyer indefinitely. Providers are incentivized to reclaim promptly because the staked TRX backing the delegation cannot be used for other customers until it is undelegated.

### Phase 5: Post-Undelegation

After undelegation confirms, the provider's TRX returns to their "undelegatable" pool. They can then delegate it to the next customer. The buyer's available energy decreases by the undelegated amount.

---

## Lock Period Mechanics

The `lock` parameter and `lock_period` in the delegation transaction deserve special attention.

### Locked Delegations

When `lock: true`, the provider commits to maintaining the delegation for at least `lock_period` days. During this time, the provider cannot undelegate. This gives the buyer a guarantee of resource availability.

```
lock_period = 3 days

Day 0: Delegation created, lock starts
Day 1: Provider CANNOT undelegate
Day 2: Provider CANNOT undelegate
Day 3: Lock expires, provider CAN undelegate
```

### Unlocked Delegations

When `lock: false`, the provider can undelegate at any time - even immediately after delegating. This is risky for buyers because the provider could reclaim the energy before the buyer has used it.

In practice, reputable providers always use locked delegations. MERX verifies that all delegations are properly locked for the purchased duration.

### Lock Period Granularity

TRON's lock period is specified in days (blocks, technically, but the protocol maps to approximately 24-hour periods). The minimum practical lock is 1 day. Common durations in the market:

| Lock Period | Typical Use Case |
|-------------|-----------------|
| 0 (unlocked) | Not recommended |
| 1 day | Short-term / testing |
| 3 days | Standard rental |
| 7 days | Weekly operations |
| 14 days | Bi-weekly coverage |
| 30 days | Monthly contracts |

---

## Energy Delivery Verification

How do you know the delegation actually happened? There are several verification methods.

### On-Chain Verification

Query the TRON network directly:

```typescript
// Using TronWeb
const accountResources = await tronWeb.trx.getAccountResources(buyerAddress);
console.log('Total energy limit:', accountResources.EnergyLimit);
console.log('Energy used:', accountResources.EnergyUsed);
console.log('Available:', accountResources.EnergyLimit - accountResources.EnergyUsed);
```

The `EnergyLimit` field reflects total energy from all sources: self-staked and delegated. An increase in `EnergyLimit` after a delegation confirms delivery.

### Delegation Record Lookup

TRON provides APIs to query delegation records for an address:

```typescript
// Get all delegations received by an address
const delegations = await tronWeb.trx.getDelegatedResourceV2(
  buyerAddress,
  providerAddress
);
```

This returns the specific delegation amounts and lock periods from a given provider to the buyer.

### MERX Verification

MERX performs automated verification after every order:

1. Order is placed and paid.
2. Provider executes delegation transaction.
3. MERX monitors the blockchain for the delegation transaction confirmation.
4. MERX queries the buyer's account resources to verify the energy increase.
5. If verification fails (energy not received within timeout), the order is flagged and the buyer is refunded.

```typescript
import { MerxClient } from '@merx/sdk';

const client = new MerxClient({ apiKey: 'your-key' });

// Place order
const order = await client.createOrder({
  energy: 65000,
  targetAddress: 'TBuyerAddress...',
  duration: '1d'
});

// Check order status (includes delegation verification)
const status = await client.getOrder(order.id);
console.log(status.delegationStatus);
// 'pending' | 'confirmed' | 'verified' | 'failed'
```

---

## Multi-Provider Delegation

A single address can receive delegations from multiple providers simultaneously. This is how aggregators like MERX can split large orders across multiple providers.

### How It Works

```
Buyer address: TBuyer123...

Delegation 1: Provider A -> 200,000 energy (65,000 x 3)
Delegation 2: Provider B -> 130,000 energy (65,000 x 2)
Delegation 3: Provider C ->  65,000 energy (65,000 x 1)

Total delegated energy: 395,000/day
Plus buyer's own staked energy: 0
Total available energy: 395,000/day
```

All delegations are independent. Provider A can undelegate without affecting Provider B's or C's delegation. The buyer's total energy is simply the sum of all active delegations plus their own staked energy.

### Why This Matters for Aggregation

When a buyer orders 500,000 energy through MERX, no single provider may have that much available. MERX can split the order:

```
Order: 500,000 energy for TBuyer123...

Routing:
  Provider A: 200,000 energy at 82 SUN/unit  = 16,400,000 SUN
  Provider B: 180,000 energy at 85 SUN/unit  = 15,300,000 SUN
  Provider C: 120,000 energy at 88 SUN/unit  = 10,560,000 SUN

Total cost: 42,260,000 SUN
Effective rate: 84.52 SUN/unit
```

The buyer sees a single order with a blended rate. Behind the scenes, three separate delegation transactions execute on-chain.

---

## How MERX Manages the Delegation Lifecycle

MERX handles the full lifecycle of every delegation, from order to expiry, across all integrated providers.

### Order Execution

1. **Price check**: Poll all providers for current best price.
2. **Routing**: Select cheapest provider(s) that can fill the order.
3. **Execution**: Submit order to selected provider(s) via their API.
4. **Monitoring**: Watch for on-chain delegation transaction.
5. **Verification**: Confirm energy arrived at target address.
6. **Notification**: Inform buyer via webhook or WebSocket.

### During Active Period

- **Health monitoring**: Periodically verify delegations are still active.
- **Resource tracking**: Monitor buyer's energy usage to detect issues.
- **Alerts**: Notify buyer if energy is running low or if usage patterns suggest they need more.

### At Expiry

- **Countdown notification**: Alert buyer before delegation expires.
- **Renewal option**: Offer automatic renewal at current best price.
- **Post-expiry verification**: Confirm energy was properly reclaimed by provider.

### Error Handling

- **Delegation failure**: If provider fails to delegate within the expected timeframe, MERX routes to the next cheapest provider automatically.
- **Partial fill**: If a provider can only fill part of an order, MERX fills the remainder from other providers.
- **Provider downtime**: If a provider is unreachable, their prices are removed from the order book and orders are routed to available providers.

---

## Common Edge Cases

### Energy Used Before Delegation Expires

The buyer is not obligated to "return" unused energy. When the delegation expires and the provider undelegates, the energy simply stops being available. There is nothing to return.

### Provider Delegates More Than Purchased

Occasionally, a provider may delegate more energy than the buyer paid for (due to internal accounting differences). The buyer benefits from the extra energy during the delegation period. MERX tracks the exact amounts ordered and delivered.

### Multiple Orders to Same Address

If a buyer places multiple orders for the same target address at different times, they accumulate. Each delegation is independent and expires according to its own lock period.

### Address Does Not Exist

TRON allows delegation to any valid address, even if it has never been activated. The energy will be available when the address is activated. However, most providers and MERX validate that the target address exists and is activated before processing orders.

---

## Заключение

Energy delegation is the foundation of TRON's resource economy. The Stake 2.0 mechanism transforms energy from a non-transferable resource into a tradeable commodity, enabling the entire provider ecosystem. Understanding how delegations work - the lock mechanics, the verification process, the multi-provider composition - is essential for anyone building production systems on TRON.

MERX abstracts the complexity of managing delegations across multiple providers into a single API call, but knowing what happens under the hood helps you make better decisions about duration, volume, and provider selection.

Explore the MERX API and start managing delegations programmatically at [https://merx.exchange/docs](https://merx.exchange/docs).

---

*This article is part of the MERX knowledge series on TRON infrastructure. MERX is the first blockchain resource exchange. GitHub: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js).*
