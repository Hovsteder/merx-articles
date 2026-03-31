# SunSwap via MCP: AI Agents Trading on a DEX

## The Missing Bridge

Decentralized exchanges have been accessible through web interfaces, mobile wallets, and programmatic SDKs for years. What they have not been accessible through is natural language. An AI agent cannot click a swap button. It cannot navigate a web UI. And while an agent can technically call a smart contract through raw transaction construction, doing so requires the agent to understand ABI encoding, router contract addresses, liquidity pool mechanics, and the TRON resource model.

MERX bridges this gap. Through the MCP server, an AI agent can request a swap quote, understand the expected output and costs, and execute the trade - all through structured tool calls that abstract away the protocol-level complexity while preserving full transparency into what is happening on-chain.

This article covers the complete SunSwap integration: getting quotes, executing swaps, handling approvals, simulating energy costs, and understanding the real numbers from mainnet trades.

## Getting a Swap Quote

The first step in any trade is understanding what you will get. The `get_swap_quote` tool queries SunSwap V2's router contract to calculate the expected output for a given input:

```
Tool: get_swap_quote
Input: {
  "from_token": "TRX",
  "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "amount": "100000",
  "from_address": "TYourAddress..."
}

Response:
{
  "input_token": "TRX",
  "input_amount": "0.1",
  "input_amount_sun": 100000,
  "output_token": "USDT",
  "output_amount": "0.032847",
  "output_amount_raw": "32847",
  "minimum_received": "0.032519",
  "price_impact": "0.001%",
  "route": ["WTRX", "USDT"],
  "router": "TKzxdSv2FZKQrEkFPRQi4qeCokAFkrntVM",
  "energy_required": 223354,
  "energy_cost_estimate_trx": 11.76,
  "liquidity": {
    "pool_address": "TPool...",
    "trx_reserve": "45,234,891.23",
    "usdt_reserve": "14,832,445.67"
  }
}
```

Several important details in this response:

### Price Impact

The `price_impact` field shows how much your trade moves the pool price. For the 0.1 TRX trade above, the impact is 0.001% - negligible. For larger trades, this number grows:

```
0.1 TRX swap:      0.001% impact
1,000 TRX swap:    0.012% impact
100,000 TRX swap:  1.2% impact
1,000,000 TRX swap: 11.8% impact
```

Price impact is not a fee - it is a structural consequence of constant-product AMM mechanics. The larger your trade relative to the pool's liquidity, the worse your execution price.

### Minimum Received

The `minimum_received` field accounts for slippage protection. By default, MERX calculates a 1% slippage tolerance, meaning the swap will revert if the output is more than 1% below the quoted amount. This protects against front-running and rapid price movements between the quote and the execution.

### Route

The `route` field shows the token path. For TRX to USDT, the route passes through WTRX (Wrapped TRX) because SunSwap V2 pairs are between TRC20 tokens, and native TRX must be wrapped first. For token-to-token swaps without a direct pair, the route may include intermediate tokens:

```
Token A -> WTRX -> Token B  (two-hop route)
```

Multi-hop routes consume more energy due to additional contract interactions.

### Energy Required

This is the exact energy estimate from `triggerConstantContract` simulation. Not a hardcoded constant, not a range - the precise energy units this specific swap will consume with these specific parameters at the current blockchain state.

## Executing the Swap

Once the agent has reviewed the quote and decided to proceed, the `execute_swap` tool handles the entire execution pipeline:

```
Tool: execute_swap
Input: {
  "from_token": "TRX",
  "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "amount": "100000",
  "slippage": 1.0,
  "deadline_minutes": 5
}

Response:
{
  "status": "completed",
  "tx_hash": "9c7b3a2f...",
  "input": {
    "token": "TRX",
    "amount": "0.1"
  },
  "output": {
    "token": "USDT",
    "amount": "0.032847"
  },
  "energy_used": 223354,
  "energy_purchased": 225000,
  "energy_cost_trx": 11.84,
  "block": 58234892,
  "gas_saved_trx": 41.16
}
```

### What Happens Inside execute_swap

The `execute_swap` tool orchestrates the full resource-aware transaction pipeline:

1. **Simulate the swap** - `triggerConstantContract` with the exact swap parameters to get the precise energy requirement (223,354 in this case)

2. **Check current resources** - Query the sender's address for available energy and bandwidth

3. **Purchase energy deficit** - Find the best provider price, place the order, and wait for delegation confirmation

4. **Build the swap transaction** - Construct the SunSwap V2 router call with the correct function selector, parameters, and call value

5. **Sign locally** - The private key signs the transaction on the agent's machine

6. **Broadcast** - The signed transaction is sent to the TRON network

7. **Verify** - Poll for transaction confirmation and parse the result

The agent calls one tool. Seven steps execute behind the scenes. The agent receives a single response with the complete result.

## Token Approvals

When swapping TRC20 tokens (as opposed to swapping native TRX), the SunSwap router needs permission to spend tokens on your behalf. This is the standard TRC20 `approve` mechanism.

MERX handles this automatically. Before executing a TRC20-to-TRC20 or TRC20-to-TRX swap, the execute_swap tool checks whether the router contract has sufficient allowance for the input token:

```
Approval check:
  Token: USDT (TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t)
  Spender: SunSwap V2 Router (TKzxdSv2FZKQrEkFPRQi4qeCokAFkrntVM)
  Current allowance: 0
  Required: 100,000,000 (100 USDT)

  -> Approval needed
```

If approval is needed, MERX:

1. Simulates the approval transaction to get its energy cost
2. Adds the approval energy to the swap energy for a combined purchase
3. Executes the approval transaction first
4. Waits for confirmation
5. Then executes the swap

```
Combined energy requirement:
  Approval: 46,312 energy
  Swap: 223,354 energy
  Total: 269,666 energy

  Purchased: 270,000 energy at 14.20 TRX
```

The agent does not need to know about approvals. If one is needed, it happens. If the token is already approved, the step is skipped.

### Approval Amount

By default, MERX approves the maximum uint256 value (unlimited approval). This means future swaps of the same token will not require another approval transaction, saving energy and time. If the agent prefers exact approvals (approving only the specific amount needed), this can be specified:

```
Tool: approve_trc20
Input: {
  "token_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "spender": "TKzxdSv2FZKQrEkFPRQi4qeCokAFkrntVM",
  "amount": "100000000"
}
```

Exact approvals are more secure (limit exposure if the router contract is compromised) but cost energy on every swap.

## Real Swap: 0.1 TRX to USDT

Here are the complete details of a real mainnet swap executed through the MERX MCP server:

### Pre-Swap State

```
Address: TWallet...
TRX Balance: 523.456789 TRX
USDT Balance: 0.00 USDT
Energy Available: 0
Bandwidth Available: 1,420 (free)
```

### Quote

```
get_swap_quote:
  Input: 0.1 TRX (100,000 SUN)
  Output: 0.032847 USDT
  Price impact: 0.001%
  Energy required: 223,354
  Route: WTRX -> USDT
```

### Execution

```
execute_swap:
  Energy purchased: 225,000 at 11.84 TRX (provider: sohu)
  Delegation confirmed: 4.1 seconds
  Approval: Not needed (TRX -> USDT, native TRX does not require approval)
  Swap TX broadcast: 9c7b3a2f...
  Swap TX confirmed: Block 58,234,892
```

### Post-Swap State

```
Address: TWallet...
TRX Balance: 511.516789 TRX  (523.456789 - 0.1 swap - 11.84 energy)
USDT Balance: 0.032847 USDT
Energy Available: 1,646  (225,000 purchased - 223,354 used)
Bandwidth Available: 1,000 (free, partially consumed by TX)
```

### Cost Breakdown

```
Energy purchase:      11.84 TRX
Swap input:            0.10 TRX
Total spent:          11.94 TRX

Without energy purchase (TRX burn):
  Swap energy burn:   ~53.00 TRX
  Swap input:           0.10 TRX
  Total:              ~53.10 TRX

Savings:              41.16 TRX (77.5%)
```

## Price Impact Estimation

For agents making larger trades, understanding price impact is critical. The `get_swap_quote` tool provides this estimate, but understanding how it is calculated helps agents make better decisions.

SunSwap V2 uses the constant-product formula: `x * y = k`, where x and y are the token reserves in the pool. When you swap dx of token X for dy of token Y:

```
dy = y * dx / (x + dx)
```

The effective price you receive (dy/dx) is always worse than the spot price (y/x) because your trade increases the supply of X and decreases the supply of Y.

For a pool with 45 million TRX and 14.8 million USDT:

```
Spot price: 14,800,000 / 45,000,000 = 0.328889 USDT/TRX

Swap 100 TRX:
  Output: 14,800,000 * 100 / (45,000,000 + 100) = 0.032888 USDT/TRX
  Impact: 0.0003%

Swap 100,000 TRX:
  Output: 14,800,000 * 100,000 / (45,000,000 + 100,000) = 32.797 USDT
  Effective price: 0.32797 USDT/TRX
  Impact: 0.28%

Swap 1,000,000 TRX:
  Output: 14,800,000 * 1,000,000 / (45,000,000 + 1,000,000) = 321,739 USDT
  Effective price: 0.32174 USDT/TRX
  Impact: 2.17%
```

MERX includes this impact calculation in every quote so the agent can assess whether the trade size is appropriate for the available liquidity.

## Multi-Hop Swaps

Not all token pairs have direct liquidity pools. For tokens that trade against WTRX but not against each other, SunSwap V2 routes through an intermediate token:

```
Token A -> WTRX -> Token B
```

This requires two swap operations in a single transaction (handled by the router contract). Energy consumption is higher:

```
Direct swap (e.g., TRX -> USDT): ~223,000 energy
Two-hop swap (e.g., USDC -> USDT via WTRX): ~340,000 energy
Three-hop swap: ~460,000 energy
```

The `get_swap_quote` tool automatically finds the optimal route and reports the total energy requirement for the complete path.

## Swap Strategies for Agents

### Dollar-Cost Averaging

An agent tasked with accumulating USDT can use standing orders combined with swap execution:

```
Standing order:
  Trigger: schedule "0 */4 * * *" (every 4 hours)
  Action: Execute swap 100 TRX -> USDT at market price
  Constraint: max 600 TRX/day
```

Six swaps per day, each for 100 TRX, smoothing out price volatility.

### Threshold-Based Trading

```
Standing order:
  Trigger: TRX/USDT price above 0.35
  Action: Swap 1000 TRX -> USDT
  Constraint: max 1 execution/day
```

The agent sells TRX when the price is favorable, accumulating USDT for operational expenses.

### Rebalancing

An agent managing a portfolio can use intents to rebalance:

```
execute_intent:
  Step 1: Swap 500 TRX -> USDT (if TRX allocation > 60%)
  Step 2: Swap 200 USDT -> TRX (if TRX allocation < 40%)
  Resource strategy: batch_cheapest
```

## Comparison: Raw TronWeb vs MERX MCP

### Raw TronWeb (without MERX)

```javascript
// 1. Estimate energy (manual)
const estimation = await tronWeb.transactionBuilder.triggerConstantContract(
  routerAddress, 'swapExactETHForTokens(uint256,address[],address,uint256)',
  { callValue: 100000 },
  [{ type: 'uint256', value: 0 },
   { type: 'address[]', value: [wtrxAddr, usdtAddr] },
   { type: 'address', value: myAddr },
   { type: 'uint256', value: deadline }],
  myAddr
);
// 2. Buy energy somewhere (separate integration)
// 3. Wait for delegation (manual polling)
// 4. Build swap transaction
// 5. Sign
// 6. Broadcast
// 7. Verify
// ~50 lines of code, multiple API integrations
```

### MERX MCP (one tool call)

```
Tool: execute_swap
Input: {
  "from_token": "TRX",
  "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "amount": "100000",
  "slippage": 1.0
}
// One call. Everything handled internally.
```

The agent does not need to know about router addresses, WTRX wrapping, ABI encoding, energy markets, delegation mechanics, or transaction construction. It expresses intent ("swap TRX for USDT") and receives a result.

## Risks and Limitations

### Slippage

Even with slippage protection, rapid price movements can cause swaps to revert. If the swap reverts, the energy used for the reverted transaction is lost (though the tokens are not). The agent can retry with a higher slippage tolerance or a smaller amount.

### MEV and Front-Running

TRON's block production model is different from Ethereum's, and the MEV landscape is less developed. However, large swaps on SunSwap can still be front-run by sophisticated actors monitoring the mempool. For large trades, consider splitting into multiple smaller swaps across different blocks.

### Liquidity

SunSwap V2 liquidity varies significantly between token pairs. Major pairs (TRX/USDT, TRX/USDC) have deep liquidity. Smaller tokens may have thin pools where even moderate trades cause significant price impact. Always check the quote before executing.

## Conclusion

DEX trading through an AI agent is no longer a theoretical capability. MERX makes it operational with two tool calls: one to quote, one to execute. The energy simulation is exact. The resource purchasing is automatic. The token approvals are handled transparently.

For AI agents operating on TRON, SunSwap via MCP is the fastest path to on-chain trading. No web UI. No manual transaction construction. No resource management overhead.

Quote. Execute. Done.

---

**Links:**
- MERX Platform: [https://merx.exchange](https://merx.exchange)
- MCP Server (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP Server (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)
