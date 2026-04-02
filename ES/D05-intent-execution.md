# Ejecucion de intenciones: planes de multiples pasos para agentes de IA en TRON

## The Problem with One-at-a-Time

AI agents interacting with TRON face a structural problem when tasks involve multiple en cadena operations. Consider a simple scenario: an agent needs to send 100 USDT to Alice and then swap 50 TRX for USDT. Each operation requires its own energy purchase, its own delegation wait, and its own broadcast cycle.

With the one-at-a-time approach, the agent makes at least 8 tool calls:

1. Estimate energy for USDT transfer
2. Buy energy for USDT transfer
3. Wait for delegation
4. Execute USDT transfer
5. Estimate energy for swap
6. Buy energy for swap
7. Wait for delegation
8. Execute swap

Each energy purchase is a separate market interaction with its own transaction overhead. Each delegation wait adds 3-6 seconds of latency. The total wall-clock time can exceed 30 seconds for what should be a simple two-step task.

MERX solves this with intent execution - a system that takes a multi-step plan from an AI agent, simulates every step, optimizes resource purchases across all steps, and executes the entire plan in sequence.

## What Is an Intent

In the MERX system, an intent is a declarative description of what the agent wants to accomplish, expressed as an ordered list of actions. The agent specifies the desired outcome, and MERX handles the execution mechanics.

An intent differs from a sequence of tool calls in three important ways:

1. **Resource optimization** - MERX can batch energy purchases across all steps, buying the total energy needed in a single order rather than step by step.

2. **Pre-validation** - Every step is simulated before any step executes. If step 3 of a 5-step plan would fail, the agent knows before step 1 is broadcast.

3. **Atomic planning** - The agent submits the entire plan at once, giving MERX visibility into the full scope of work. This enables optimizations that are impossible when steps are submitted individually.

## The execute_intent Tool

The MCP server exposes intent execution through the `execute_intent` tool:

```
Tool: execute_intent
Input: {
  "steps": [
    {
      "action": "transfer_trc20",
      "params": {
        "to": "TAliceAddress...",
        "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "100000000"
      }
    },
    {
      "action": "swap",
      "params": {
        "from_token": "TRX",
        "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "50000000",
        "slippage": 0.5
      }
    }
  ],
  "resource_strategy": "batch_cheapest"
}
```

The response includes the simulation results for each step, the total resource cost, and the execution status of each step after completion.

## Supported Actions

The intent system supports the following action types:

### transfer_trx

Send TRX to an address. This is a native transfer that consumes bandwidth but no energy.

```json
{
  "action": "transfer_trx",
  "params": {
    "to": "TRecipient...",
    "amount_sun": 1000000
  }
}
```

### transfer_trc20

Send a TRC20 token (USDT, USDC, etc.) to an address. Consumes energy for the contrato inteligente call.

```json
{
  "action": "transfer_trc20",
  "params": {
    "to": "TRecipient...",
    "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "amount": "100000000"
  }
}
```

### swap

Execute a token swap on SunSwap V2. Includes exact energy simulation for the specific swap parameters.

```json
{
  "action": "swap",
  "params": {
    "from_token": "TRX",
    "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "amount": "100000000",
    "slippage": 0.5
  }
}
```

### approve

Set a spending approval for a TRC20 token. Required before swapping tokens (not needed for swapping TRX).

```json
{
  "action": "approve",
  "params": {
    "token_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "spender": "TRouterAddress...",
    "amount": "unlimited"
  }
}
```

### call_contract

Execute an arbitrary contrato inteligente call. This is the escape hatch for operations not covered by the specific action types.

```json
{
  "action": "call_contract",
  "params": {
    "contract_address": "TContractAddress...",
    "function_selector": "stake(uint256)",
    "parameters": [{ "type": "uint256", "value": "1000000" }],
    "call_value": 0
  }
}
```

### buy_resource

Purchase energy or bandwidth as a step in the plan. Useful when the agent wants explicit control over resource timing.

```json
{
  "action": "buy_resource",
  "params": {
    "resource_type": "energy",
    "amount": 130000,
    "duration_hours": 1
  }
}
```

## Resource Strategies

The `resource_strategy` parameter controls how MERX handles energy purchases across the intent's steps.

### batch_cheapest

This is the default and recommended strategy. MERX simulates all steps, sums the total energy required, subtracts available resources, and makes a single energy purchase for the entire intent.

```
Step 1 (transfer_trc20): 64,895 energy
Step 2 (swap):           223,354 energy
Total needed:            288,249 energy
Currently available:     0 energy
Purchase:                290,000 energy (rounded to order unit)
```

One purchase. One delegation wait. Then all steps execute sequentially using the pooled energy.

Benefits:
- Single market interaction (lower overhead)
- Single delegation wait (lower latency)
- Potential volume discount on larger orders
- Simpler failure handling

### per_step

Each step purchases its own energy independently. Use this when steps are conditional or when you need to minimize risk (if step 1 fails, you haven't purchased energy for step 2).

```
Step 1: buy 65,000 energy -> wait -> execute transfer
Step 2: buy 225,000 energy -> wait -> execute swap
```

This strategy is slower (two delegation waits) but wastes less energy if execution is halted mid-plan.

## Stateful Simulation

The intent system's simulation engine maintains state across steps. This is critical for plans where later steps depend on the results of earlier steps.

Consider this intent: "Swap 50 TRX for USDT, then send the received USDT to Alice."

The simulation engine:

1. Simulates step 1 (swap). Result: agent receives 16.42 USDT.
2. Updates the simulated state to reflect the new USDT balance.
3. Simulates step 2 (transfer 16.42 USDT to Alice) against the updated state.
4. Confirms step 2 would succeed with the balance from step 1.

Without stateful simulation, step 2 would be simulated against the agent's current balance (which might not include the USDT from the swap). The simulation would incorrectly report that step 2 would fail due to insufficient balance.

```
Tool: execute_intent
Input: {
  "steps": [
    {
      "action": "swap",
      "params": {
        "from_token": "TRX",
        "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "50000000",
        "slippage": 0.5
      }
    },
    {
      "action": "transfer_trc20",
      "params": {
        "to": "TAliceAddress...",
        "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "use_previous_output"
      }
    }
  ],
  "resource_strategy": "batch_cheapest"
}
```

The `use_previous_output` parameter tells the intent system to use the output amount from the preceding step as the input amount for this step.

## Simulation Response

Before execution begins, the intent system returns a simulation summary:

```json
{
  "simulation": {
    "steps": [
      {
        "action": "transfer_trc20",
        "energy_required": 64895,
        "bandwidth_required": 345,
        "simulated_success": true,
        "estimated_cost_trx": 3.42
      },
      {
        "action": "swap",
        "energy_required": 223354,
        "bandwidth_required": 420,
        "simulated_success": true,
        "estimated_output": "16.42 USDT",
        "estimated_cost_trx": 11.76
      }
    ],
    "total_energy": 288249,
    "total_bandwidth": 765,
    "total_cost_trx": 15.18,
    "resource_purchase": {
      "energy": 290000,
      "price": 15.24,
      "provider": "sohu"
    }
  },
  "status": "ready_to_execute"
}
```

The agent sees the full plan with costs before any en cadena action is taken. If the costs are unacceptable or a step would fail, the agent can modify the plan without spending anything.

## Execution and Error Handling

Once the agent confirms the plan (or if auto-execution is enabled), the intent executes step by step:

```json
{
  "execution": {
    "steps": [
      {
        "action": "transfer_trc20",
        "status": "completed",
        "tx_hash": "abc123...",
        "energy_used": 64895,
        "block": 58234567
      },
      {
        "action": "swap",
        "status": "completed",
        "tx_hash": "def456...",
        "energy_used": 223354,
        "output_amount": "16.42",
        "block": 58234568
      }
    ],
    "total_energy_used": 288249,
    "total_energy_purchased": 290000,
    "energy_wasted": 1751,
    "status": "all_steps_completed"
  }
}
```

### Failure Mid-Execution

If a step fails during execution (not during simulation), the intent system stops and reports the failure:

```json
{
  "execution": {
    "steps": [
      {
        "action": "transfer_trc20",
        "status": "completed",
        "tx_hash": "abc123..."
      },
      {
        "action": "swap",
        "status": "failed",
        "error": "SLIPPAGE_EXCEEDED",
        "message": "Output 15.89 USDT below minimum 16.34 USDT"
      }
    ],
    "status": "partial_execution",
    "completed_steps": 1,
    "failed_step": 2,
    "remaining_energy": 223354
  }
}
```

Step 1 has already been committed en cadena and cannot be reversed. The agent receives the remaining energy balance and can decide how to proceed - retry the failed step with adjusted parameters, execute a different action, or let the energy expire.

## Real-World Example: Treasury Rebalancing

Aqui esta a realistic multi-step intent that an agent might execute for treasury management:

"Swap 1,000 TRX for USDT, send 300 USDT to the operations wallet, send 200 USDT to the marketing wallet, keep the rest."

```
Tool: execute_intent
Input: {
  "steps": [
    {
      "action": "swap",
      "params": {
        "from_token": "TRX",
        "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "1000000000",
        "slippage": 1.0
      }
    },
    {
      "action": "transfer_trc20",
      "params": {
        "to": "TOpsWallet...",
        "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "300000000"
      }
    },
    {
      "action": "transfer_trc20",
      "params": {
        "to": "TMarketingWallet...",
        "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "200000000"
      }
    }
  ],
  "resource_strategy": "batch_cheapest"
}
```

Simulation:

```
Step 1 (swap):     223,354 energy
Step 2 (transfer): 29,631 energy  (OpsWallet already has USDT)
Step 3 (transfer): 64,895 energy  (MarketingWallet is new to USDT)
Total:             317,880 energy

Batch purchase: 320,000 energy at 16.83 TRX from catfee

Without intent batching: 3 separate purchases = ~18.20 TRX
With intent batching: 1 purchase = 16.83 TRX
Savings from batching: 1.37 TRX + reduced latency (1 wait vs 3)
```

## When to Use Intents vs Individual Tools

Use `execute_intent` when:

- The task involves two or more en cadena operations
- Steps have dependencies (step 2 uses the output of step 1)
- You want to minimize total resource costs through batching
- You need pre-validation of the entire plan before committing

Use individual tools when:

- The task is a single operation
- The agent needs to make decisions between steps based on external input
- Steps are separated by significant time gaps
- The agent wants maximum control over each stage of execution

## Intents and Agent Autonomy

The intent system is designed for agent autonomy. An agent that receives a high-level instruction like "rebalance the treasury" can decompose it into concrete steps, construct an intent, simulate it, review the costs, and execute - all without human intervention at any stage.

The simulation step serves as the agent's safety check. Before committing any funds, the agent can verify that every step will succeed, the total cost is within budget, and the expected outputs match the desired outcome. This is the equivalent of a human reviewing a transaction before clicking "confirm," but executed programmatically by the agent itself.

Combined with orden permanentes for recurring resource purchases and monitors for balance alerts, the intent system enables fully autonomous en cadena operations that run 24/7 without human oversight.

## Conclusion

Single-step execution is the training wheels of blockchain automation. Real agent workflows involve multiple operations, dependencies between steps, and resource optimization across the full plan.

MERX intent execution gives AI agents the ability to think in plans rather than individual actions. Simulate everything. Optimize resources across the full scope. Execute with confidence that every step has been pre-validated.

The blockchain is not a single-operation environment. Your agent should not be either.

---

**Enlaces:**
- MERX Plataforma: [https://merx.exchange](https://merx.exchange)
- Servidor MCP (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- Servidor MCP (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)

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
