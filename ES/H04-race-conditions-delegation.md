# Condiciones de carrera en la delegacion de energia: como las resolvimos

This is a story about a bug that cost 19 TRX per transaction instead of saving 1.43 TRX. It is a story about a race condition that looked correct in tests, worked on testnet, and only revealed itself under mainnet conditions. And it is a story about the fix -- a polling mechanism that waits for en cadena confirmation before allowing a transaction to proceed.

## The Setup

MERX buys energy from providers on behalf of users. The flow, in theory, is straightforward:

1. User requests energy for their address
2. MERX selects el proveedor mas barato
3. Provider delegates energy to the user's address
4. User executes their transaction with the delegated energy
5. Transaction consumes the rented energy instead of burning TRX

The critical word is "before." The user's transaction must execute after the delegation is confirmed en cadena. If the transaction broadcasts before the delegation arrives, the transaction will still succeed -- but it will burn TRX at the protocol tasa de quema instead of consuming the rented energy. The user pays twice: once for the rental, and once through the TRX burn.

## The Bug

The initial implementation followed a sequential pattern:

```typescript
async function executeWithEnergy(
  targetAddress: string,
  energyAmount: number,
  transactionFn: () => Promise<string>
): Promise<string> {
  // Step 1: Buy energy
  const order = await merx.createOrder({
    energy_amount: energyAmount,
    target_address: targetAddress,
    duration: '1h'
  });

  // Step 2: Wait for order confirmation from MERX
  await waitForOrderStatus(order.id, 'FILLED');

  // Step 3: Execute the transaction
  const txHash = await transactionFn();

  return txHash;
}
```

This looks correct. Buy energy, wait for the order to be filled, then execute the transaction. The `waitForOrderStatus` function polled the MERX API until the estado de la orden changed to `FILLED`.

The problem is what "FILLED" means. In the MERX system, an order is marked `FILLED` when the provider confirms that it has initiated the delegation. The provider returns a success response: "I have submitted the delegation transaction to the TRON network."

But "submitted" is not "confirmed." The delegation is a TRON transaction that must be included in a block and processed by the network. Between submission and confirmation, there is a window -- typically 3 to 6 seconds, but occasionally longer during network congestion.

During that window, the target address does not yet have the delegated energy.

### What Happened on Mainnet

The bug manifested exactly as predicted by the race condition:

```
Timeline:

  T+0.0s    Order created, sent to provider
  T+0.3s    Provider responds: delegation TX submitted
  T+0.4s    Order status -> FILLED
  T+0.5s    User transaction broadcast (USDT transfer)
  T+0.6s    User TX included in block N
  T+3.2s    Delegation TX included in block N+1

  Result:
    - User TX in block N: no delegation exists yet
    - Energy consumed: 0 (delegation not active)
    - TRX burned: 65,000 * 420 SUN = 27,300,000 SUN = 27.3 TRX
    - Energy rental cost: 1,820,000 SUN = 1.82 TRX
    - Total cost: 29.12 TRX (rental + burn)
    - Expected cost: 1.82 TRX (rental only)
```

The user paid 29.12 TRX instead of 1.82 TRX. The alquiler de energia was wasted because the delegation arrived one block too late.

### The Numbers

On TRON mainnet, each block takes approximately 3 seconds. A delegation transaction submitted by the provider goes through the same block inclusion process as any other transaction. If the user's transaction and the delegation transaction are submitted within a few seconds of each other, they may end up in different blocks -- with no guarantee of ordering.

The cost difference is stark:

```
Without delegation (TRX burn):
  65,000 energy * 420 SUN/energy = 27,300,000 SUN = 27.3 TRX

With delegation (rental):
  65,000 energy * 28 SUN/energy = 1,820,000 SUN = 1.82 TRX

Money wasted per race condition occurrence:
  27.3 + 1.82 = 29.12 TRX total cost
  vs.
  1.82 TRX expected cost

  Overpayment: 27.3 TRX per incident
```

On a bad day, this race condition could trigger on 10-20% of orders where the client executed immediately after receiving the FILLED status.

## Why Testing Did Not Catch It

### Testnet Behavior

On Shasta testnet, block times are similar to mainnet, but network congestion is minimal. Delegation transactions are typically included in the very next block. The window between "submitted" and "confirmed" was consistently under 3 seconds on testnet, and our test harness had a built-in 2-second delay between steps that masked the race condition.

### Sequential Testing

Our integration tests were sequential. One test would buy energy, wait, execute a transaction, and verify. There was never concurrent load, never a race between delegation and execution, never the timing pressure that mainnet produced.

### The Order Status Trap

The most insidious aspect: the estado de la orden was technically correct. The order was FILLED -- the provider had accepted and initiated the delegation. The bug was not in the status tracking. It was in the assumption that FILLED meant "energy is available en cadena."

## The Fix

The fix has two parts: a verification step that checks the target address's en cadena resources, and a polling loop that waits until the delegation is actually confirmed.

### Part 1: check_address_resources

TRON provides an API to check the resources (energy and bandwidth) currently available to any address:

```
GET https://api.trongrid.io/wallet/getaccountresource
```

This returns the current energy limit, energy used, bandwidth limit, and bandwidth used for an address. Critically, this reflects the en cadena state -- if a delegation has been confirmed, the energy limit will reflect it. If the delegation is still pending, the energy limit will not include it.

### Part 2: Poll Until Confirmed

The fix replaces the single "wait for FILLED" check with a polling loop that verifies en cadena resources:

```typescript
async function executeWithEnergy(
  targetAddress: string,
  energyAmount: number,
  transactionFn: () => Promise<string>
): Promise<string> {
  // Step 1: Check baseline resources
  const baseline = await checkAddressResources(targetAddress);
  const baselineEnergy = baseline.energy_limit - baseline.energy_used;

  // Step 2: Buy energy
  const order = await merx.createOrder({
    energy_amount: energyAmount,
    target_address: targetAddress,
    duration: '1h'
  });

  // Step 3: Wait for order to be filled by the provider
  await waitForOrderStatus(order.id, 'FILLED');

  // Step 4: Poll on-chain resources until delegation is confirmed
  const confirmed = await pollUntilDelegationConfirmed(
    targetAddress,
    baselineEnergy,
    energyAmount,
    { maxAttempts: 15, intervalMs: 2000 }
  );

  if (!confirmed) {
    throw new Error(
      'Delegation not confirmed on-chain within timeout. ' +
      'Do not execute transaction -- energy may not be available.'
    );
  }

  // Step 5: Execute the transaction (delegation is confirmed on-chain)
  const txHash = await transactionFn();

  return txHash;
}

async function pollUntilDelegationConfirmed(
  address: string,
  baselineEnergy: number,
  expectedIncrease: number,
  options: { maxAttempts: number; intervalMs: number }
): Promise<boolean> {
  for (let attempt = 0; attempt < options.maxAttempts; attempt++) {
    const resources = await checkAddressResources(address);
    const currentEnergy = resources.energy_limit - resources.energy_used;
    const increase = currentEnergy - baselineEnergy;

    if (increase >= expectedIncrease * 0.95) {
      // Allow 5% tolerance for rounding
      return true;
    }

    await sleep(options.intervalMs);
  }

  return false;
}

async function checkAddressResources(
  address: string
): Promise<{ energy_limit: number; energy_used: number }> {
  const response = await fetch(
    'https://api.trongrid.io/wallet/getaccountresource',
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        address: address,
        visible: true
      })
    }
  );

  const data = await response.json();
  return {
    energy_limit: data.EnergyLimit || 0,
    energy_used: data.EnergyUsed || 0
  };
}
```

### Why 2-Second Intervals

The intervalo de sondeo is 2 seconds. This is chosen based on TRON's 3-second block time:

- Polling every 1 second would result in many redundant checks between blocks
- Polling every 3 seconds would miss some blocks (clock drift, network latency)
- Polling every 2 seconds ensures we check within every block cycle

### Why 15 Attempts

15 attempts at 2-second intervals = 30-second maximum wait time. En la practica, the delegation confirms within 1-2 polls (3-6 seconds). The 30-second timeout handles extreme cases:

- Network congestion delaying block inclusion
- Provider submitting the delegation transaction late
- Temporary RPC node lag

If the delegation is not confirmed after 30 seconds, something is genuinely wrong, and it is safer to fail loudly than to proceed and burn TRX.

## Real Mainnet Data

After deploying the fix, we measured the polling behavior over one week of production traffic:

```
Delegation confirmation timing:

  Confirmed on first poll (0-2s):     12%
  Confirmed on second poll (2-4s):    61%
  Confirmed on third poll (4-6s):     22%
  Confirmed on fourth poll (6-8s):    4%
  Confirmed on fifth+ poll (8s+):     1%
  Timeout (not confirmed in 30s):     0.04%

Average time from order FILLED to on-chain confirmation: 3.1 seconds
Median: 2.8 seconds
P99: 8.2 seconds
```

The data confirms the race condition window: 3.1 seconds on average between the provider's "FILLED" response and en cadena confirmation. Without the polling fix, any transaction executed within that window would have burned TRX.

### Cost Impact

```
Before fix (30 days):
  Orders affected by race condition:  ~180
  Average overpayment per incident:   ~19 TRX
  Total cost of race conditions:      ~3,420 TRX

After fix (30 days):
  Race condition incidents:           0
  Timeout failures (delegation never confirmed): 2
    Both caught by the timeout, transaction not executed
    Orders refunded automatically
```

The fix eliminated the race condition entirely. The two timeout failures were genuine provider-side issues where the delegation transaction was never submitted -- exactly the cases where you want to fail rather than proceed.

## Lessons Learned

### "Submitted" Is Not "Confirmed"

This is the central lesson. In any system that interacts with a blockchain, there is always a gap between submitting a transaction and that transaction being confirmed en cadena. Any logic that treats submission as confirmation will eventually fail.

### Check the Chain, Not the Service

The provider says the delegation is done. MERX says the order is filled. But neither of these statements means the delegation exists en cadena right now. The only authoritative source is the blockchain itself. Check the chain.

### Testnet Hides Timing Bugs

Testnets have lower load, faster inclusion, and more predictable behavior. Timing-sensitive bugs that never trigger on testnet will appear on mainnet. If your logic depends on timing between two en cadena events, test it under realistic load conditions.

### Fail Loudly

When the polling loop times out, the fix throws an error and prevents the transaction from executing. This is the correct behavior. The alternative -- executing anyway and hoping the delegation arrived -- costs 19 TRX per failure. An error message costs nothing.

## The MERX Implementation

The `ensure_resources` tool in the MERX MCP server implements this pattern. When an AI agent calls `ensure_resources` before executing a contract call, the tool:

1. Checks current en cadena resources
2. Calculates the deficit
3. Purchases exactly the needed energy
4. Polls until the delegation is confirmed en cadena
5. Returns success only when resources are verified

The agent never has to implement polling logic itself. The race condition is handled at the platform level.

```
Tool: ensure_resources
Input: {
  "address": "TYourAddress...",
  "energy_needed": 65000
}

Response: {
  "status": "confirmed",
  "energy_available": 65000,
  "confirmation_time_ms": 4200,
  "order_id": "ord_abc123"
}
```

The `confirmation_time_ms` field tells the agent how long the polling took. The `status: "confirmed"` means the energy is en cadena and safe to use.

Plataforma: [https://merx.exchange](https://merx.exchange)
Documentacion: [https://merx.exchange/docs](https://merx.exchange/docs)
Servidor MCP: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
