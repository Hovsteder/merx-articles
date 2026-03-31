# Simulacion exacta de energia: como MERX sabe con precision lo que cuesta un intercambio

## The Problem with Guessing

Most TRON tools and wallets use hardcoded energy estimates. A USDT transfer? Budget 65,000 energy. A swap on SunSwap? Maybe 200,000. An approval? Around 50,000. These numbers are stored as constants somewhere in the codebase, and every transaction uses them regardless of the actual conditions.

This approach has two failure modes, and both cost you money.

If the estimate is too low, the transaction runs out of energy mid-execution. The entire transaction fails, but you still pay for the bandwidth consumed by the attempt. The energy you purchased is wasted on a transaction that produced no result. You have to buy more energy and try again.

If the estimate is too high, you purchase energy you never use. Delegated energy has a minimum rental period - typically one hour. Whatever is left over when the delegation expires is simply lost. Overshoot by 30% on every transaction, and over thousands of transactions, the waste adds up to serious money.

MERX eliminates guessing entirely. Every energy estimate is produced by simulating the exact transaction against the live blockchain state. The result is not an approximation or a range - it is the precise number of unidad de energias the transaction will consume.

## How triggerConstantContract Works

The TRON network provides a read-only simulation endpoint called `triggerConstantContract`. This endpoint executes a contrato inteligente call in a virtual environment that mirrors the current blockchain state but does not modify it. No transaction is broadcast. No resources are consumed. No state changes are committed.

The simulation runs the exact same bytecode that would execute in a real transaction. It uses the same storage slots, the same account balances, the same contract logic. The only difference is that the results are discarded instead of being written to the blockchain.

The key output is `energy_used` - the exact number of unidad de energias consumed by the simulated execution.

### The API Call

```javascript
const result = await tronWeb.transactionBuilder.triggerConstantContract(
  contractAddress,       // The contract being called
  functionSelector,      // e.g., "transfer(address,uint256)"
  { callValue: 0 },     // Options including any TRX value sent
  [                      // Function parameters
    { type: 'address', value: recipientAddress },
    { type: 'uint256', value: amount }
  ],
  senderAddress          // The address that would send the transaction
);

const energyUsed = result.energy_used;
```

This is not a gas estimation heuristic like Ethereum's `eth_estimateGas`, which returns an upper bound with a safety margin. `triggerConstantContract` returns the exact energy consumed by the execution trace. When MERX simulates a swap that returns 223,354 energy, the en cadena execution will consume 223,354 energy.

## Why the Sender Address Matters

A subtle but critical detail: the sender address affects the simulation result.

Consider a USDT transfer. If the sender has never interacted with the USDT contract before, the contract must allocate a new storage slot for the sender's balance. This allocation costs energy. If the sender already has a balance entry, the contract only needs to update an existing slot - cheaper by thousands of unidad de energias.

Similarly, the recipient address matters. Transferring USDT to an address that has never held USDT triggers a storage slot creation for the recipient. Transferring to an address that already has a USDT balance does not.

These differences can shift the costo de energia by 10,000-20,000 units for a simple transfer. For complex DeFi operations with multiple internal calls and storage modifications, the variance is even larger.

Es por esto que MERX simulates with the real sender and real recipient addresses for every transaction. A generic simulation with placeholder addresses would return a different energy value than the actual execution.

## Hardcoded Estimates vs Exact Simulation

Aqui esta a concrete comparison using real data from TRON mainnet.

### USDT Transfer Scenarios

| Scenario | Hardcoded Estimate | Exact Simulation |
|---|---|---|
| Sender has USDT, recipient has USDT | 65,000 | 29,631 |
| Sender has USDT, recipient never held USDT | 65,000 | 64,895 |
| Sender never approved, recipient new | 65,000 | 64,895 |
| Transfer to contract address | 65,000 | 47,222 |

The hardcoded approach uses 65,000 for all cases. Exact simulation reveals a 2x range in actual costs. In the first scenario, a hardcoded estimate causes you to purchase more than double the energy actually needed.

### SunSwap V2 Swap

| Scenario | Hardcoded Estimate | Exact Simulation |
|---|---|---|
| TRX -> USDT, direct pool | 200,000 | 223,354 |
| USDT -> TRX, direct pool | 200,000 | 218,847 |
| Token A -> Token B, multi-hop | 250,000 | 312,668 |
| Small amount, same pool | 200,000 | 223,354 |

For the direct TRX -> USDT swap, a hardcoded estimate of 200,000 would underestimate by 23,354 units. That transaction would fail. The alternative - bumping the hardcoded estimate to 250,000 to be safe - wastes 26,646 unidad de energias on every single swap.

## How MERX Uses Exact Simulation

The MERX MCP server exposes simulation through two tools that operate at different levels of abstraction.

### Low-Level: estimate_contract_call

This tool lets you simulate any arbitrary contract call:

```
Tool: estimate_contract_call
Input: {
  "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "function_selector": "transfer(address,uint256)",
  "parameters": [
    { "type": "address", "value": "TRecipient..." },
    { "type": "uint256", "value": "100000000" }
  ],
  "from_address": "TSender...",
  "call_value": 0
}

Response:
{
  "energy_used": 64895,
  "result": "0x0000...0001",
  "success": true
}
```

The response includes both the costo de energia and the return value of the simulated call. For a transfer, the return value indicates whether the transfer would succeed. For a swap, it returns the output amount.

### High-Level: get_swap_quote

For DEX operations, MERX provides a specialized tool that combines simulation with price quoting:

```
Tool: get_swap_quote
Input: {
  "from_token": "TRX",
  "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "amount": "100000",
  "from_address": "TSender..."
}

Response:
{
  "output_amount": "0.032847",
  "output_token": "USDT",
  "energy_required": 223354,
  "price_impact": "0.001%",
  "route": ["WTRX", "USDT"],
  "minimum_received": "0.032519"
}
```

The `energy_required` field comes from an exact simulation of the swap with the real parameters. The `output_amount` also comes from the simulation, so the quote reflects the actual output the swap would produce at the current block.

## Real Data: Simulation vs On-Chain Execution

Aqui esta a real swap executed through the MERX MCP server on TRON mainnet.

**Transaction: Swap 0.1 TRX for USDT via SunSwap V2**

Pre-execution simulation:

```
triggerConstantContract(
  contract: TKzxdSv2FZKQrEkFPRQi4qeCokAFkrntVM,  // SunSwap V2 Router
  function: swapExactETHForTokens(uint256,address[],address,uint256),
  parameters: [
    0,                                              // amountOutMin
    ["WTRX_address", "USDT_address"],              // path
    "TSender...",                                    // to
    1711814400                                       // deadline
  ],
  call_value: 100000,                               // 0.1 TRX in SUN
  from: "TSender..."
)

Result:
  energy_used: 223,354
  return_value: [32847]  // 0.032847 USDT
```

On-chain execution (after delegacion de energia):

```
Transaction: abc123...
  Status: SUCCESS
  Energy consumed: 223,354
  Bandwidth consumed: 420
  Output: 0.032847 USDT received
```

The simulation returned 223,354 energy. The en cadena execution consumed 223,354 energy. Exact match. Not an approximation. Not a range. The same number.

## Edge Cases and How MERX Handles Them

### State Changes Between Simulation and Execution

The blockchain state can change between the time you simulate and the time you broadcast. Another transaction might modify the same contract storage, changing the costo de energia. MERX mitigates this in three ways:

1. **Minimize the gap.** The pipeline simulates, purchases energy, polls for delegation, and broadcasts in the tightest possible sequence. Typical gap: 5-10 seconds.

2. **Add a small buffer for volatile operations.** For DEX swaps where liquidity pool state changes frequently, MERX adds a 5% buffer to the energy purchase. This buffer covers minor state variations without significantly increasing cost.

3. **Fail safely.** If the transaction runs out of energy despite the buffer, it fails before modifying state. The energy is consumed, but no tokens are incorrectly transferred. The agent can retry with a fresh simulation.

### Reverted Simulations

Sometimes `triggerConstantContract` returns a revert. Esto significa the transaction would fail en cadena. Common causes:

- Insufficient token balance
- Swap output below minimum (slippage exceeded)
- Approval not set for token spending
- Contract paused or restricted

MERX surfaces these reverts before any energy is purchased:

```
Response:
{
  "success": false,
  "revert_reason": "INSUFFICIENT_OUTPUT_AMOUNT",
  "energy_used": 0,
  "message": "Swap would fail: output below minimum. Adjust slippage or amount."
}
```

This prevents el mas caro type of failure: buying energy for a transaction that was never going to succeed.

### Approval Transactions

TRC20 token swaps on SunSwap require an approval transaction before the swap. This is a separate contract call with its own costo de energia. MERX detects when approval is needed and simulates both transactions:

```
Simulation results:
  1. approve(router, MAX_UINT256): 46,312 energy
  2. swapExactTokensForTokens(...): 223,354 energy
  Total: 269,666 energy
```

The agent can then purchase energy for both operations in a single order, avoiding the overhead of two separate mercado de energia interactions.

## Building Simulation Into Your Workflow

If you are building on TRON and not using exact simulation, you are leaving money on the table or exposing yourself to transaction failures. Aqui esta how to integrate it:

### For MCP Agent Developers

Use the MERX MCP server. The `ensure_resources` and `execute_swap` tools run simulation automatically. You do not need to call `estimate_contract_call` separately unless you want to inspect the results.

### For API Integrators

Call the MERX estimation endpoint before every transaction:

```bash
curl -X POST https://merx.exchange/api/v1/estimate \
  -H "Content-Type: application/json" \
  -d '{
    "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "function_selector": "transfer(address,uint256)",
    "parameters": ["TRecipient...", 100000000],
    "from_address": "TSender...",
    "call_value": 0
  }'
```

### For Direct TronWeb Users

Call `triggerConstantContract` yourself before every contrato inteligente interaction:

```javascript
const simulation = await tronWeb.transactionBuilder.triggerConstantContract(
  contractAddress,
  functionSelector,
  options,
  parameters,
  senderAddress
);

if (!simulation.result.result) {
  console.error('Transaction would fail:', simulation.result.message);
  return;
}

const energyNeeded = simulation.energy_used;
// Now purchase exactly this much energy before broadcasting
```

## The Economics of Precision

Consider an application that processes 1,000 USDT transfers per day.

With hardcoded estimates (65,000 energy per transfer):

```
Daily energy purchased: 65,000,000
Average actual usage: 47,000,000  (varies by scenario)
Daily waste: 18,000,000 energy
Daily waste cost: ~94 TRX (~$24)
Annual waste: ~$8,760
```

With exact simulation:

```
Daily energy purchased: 47,200,000  (actual + 0.4% rounding)
Daily waste: 200,000  (rounding to minimum order units)
Daily waste cost: ~1 TRX (~$0.26)
Annual waste: ~$95
```

The difference is $8,665 per year for a single operation type. For applications that run multiple transaction types at higher volumes, the savings scale linearly.

## Conclusion

Exact energy simulation is not a feature. It is a requirement for any serious TRON application. The difference between a hardcoded estimate and an exact simulation is the difference between guessing and knowing. MERX chooses knowing.

Every transaction simulated. Every unidad de energia accounted for. Every swap quoted precisely. The simulation says 223,354, and the chain confirms 223,354.

That is not an approximation. That is engineering.

---

**Enlaces:**
- MERX Plataforma: [https://merx.exchange](https://merx.exchange)
- Servidor MCP (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- Servidor MCP (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)
