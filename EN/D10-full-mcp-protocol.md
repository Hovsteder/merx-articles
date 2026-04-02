# 30 Prompts and 21 Resources: Why MERX is the Only Full-Protocol MCP Server

## MCP Has Three Primitives, Not One

The Model Context Protocol defines three types of capabilities that a server can expose to an AI client:

1. **Tools** - executable actions (send a transaction, buy energy, execute a swap)
2. **Prompts** - pre-built templates that guide the model through complex workflows
3. **Resources** - structured data the model can read (price feeds, network parameters, documentation)

Most MCP servers implement only tools. They expose a handful of callable functions, call it a day, and market themselves as "MCP-compatible." This is technically correct in the same way that a car with an engine but no steering wheel is technically a vehicle.

Tools alone give the model actions but no guidance on when or how to use them. Without prompts, the model must figure out complex multi-step workflows from scratch every time. Without resources, the model has no structured data to reference - it must make tool calls just to read information that should be passively available.

MERX implements all three primitives: 54 tools, 30 prompts, and 21 resources. This article explains what each primitive does, why all three matter, and how MERX uses them to create an MCP server that is qualitatively different from tools-only implementations.

## Primitive 1: Tools (Actions)

Tools are the most intuitive primitive. A tool is a function the model can call to perform an action. It has a name, a description, typed parameters, and returns a structured response.

MERX exposes 54 tools covering the full scope of TRON blockchain operations:

### Wallet Operations
- `set_private_key` - Configure the wallet with a private key (derives address automatically)
- `get_trx_balance` - Check TRX balance of any address
- `get_trc20_balance` - Check TRC20 token balance
- `transfer_trx` - Send TRX to an address
- `transfer_trc20` - Send TRC20 tokens to an address
- `check_address_resources` - View energy and bandwidth allocation

### Energy Market
- `get_prices` - Current prices from all providers
- `get_best_price` - Find the cheapest energy offer
- `create_order` - Purchase energy from the market
- `create_paid_order` - Purchase energy via x402 (no account needed)
- `ensure_resources` - Automatically purchase needed resources
- `list_orders` - View order history
- `get_order` - Get details of a specific order

### DEX Trading
- `get_swap_quote` - Get a SunSwap V2 quote with exact simulation
- `execute_swap` - Execute a swap with automatic resource management
- `approve_trc20` - Set token spending approval

### Automation
- `create_standing_order` - Set up automatic energy purchases
- `list_standing_orders` - View active standing orders
- `create_monitor` - Set up delegation or balance monitors
- `list_monitors` - View active monitors

### Advanced
- `execute_intent` - Execute multi-step transaction plans
- `estimate_contract_call` - Simulate any contract call for energy estimation

Each tool has a JSON Schema definition for its parameters, a natural language description, and structured return types. The model knows exactly what each tool does, what parameters it needs, and what it will return.

But tools alone are not enough.

## Primitive 2: Prompts (Templates)

A prompt is a pre-built template that the model can use to handle a specific type of request. Think of it as a recipe: "When the user asks for X, here is the step-by-step approach, including which tools to call, in what order, and how to interpret the results."

MERX provides 30 prompts organized across 10 categories.

### Getting Started (3 prompts)

**setup_wallet**
```
Template: Walk the user through wallet configuration
Steps:
  1. Explain private key security
  2. Call set_private_key
  3. Call get_trx_balance to verify
  4. Call check_address_resources for baseline
  5. Recommend next steps based on balance
```

**first_energy_purchase**
```
Template: Guide first-time energy buyers
Steps:
  1. Explain what energy is and why it matters
  2. Call get_prices to show market overview
  3. Help choose amount and duration
  4. Call create_order or create_paid_order
  5. Verify delegation with check_address_resources
```

**platform_overview**
```
Template: Explain MERX capabilities
Content: Structured overview of all tools, prompts, and resources
  with examples of when to use each
```

### Energy Management (4 prompts)

**buy_cheapest_energy**
```
Template: Find and buy energy at the best price
Steps:
  1. Call get_best_price with user's parameters
  2. Show price comparison across providers
  3. Confirm purchase with user
  4. Call create_order
  5. Poll for delegation confirmation
```

**compare_providers**
```
Template: Detailed provider comparison
Steps:
  1. Call get_prices for all providers
  2. Calculate cost for user's specific needs
  3. Present comparison table
  4. Include reliability and speed metrics
  5. Recommend best option with reasoning
```

**optimize_costs**
```
Template: Analyze spending and suggest optimizations
Steps:
  1. Review order history
  2. Analyze price patterns
  3. Identify opportunities for standing orders
  4. Calculate potential savings
  5. Present optimization plan
```

**ensure_resources_for_tx**
```
Template: Prepare resources for a specific transaction
Steps:
  1. Estimate energy for the planned transaction
  2. Check current resources
  3. Calculate deficit
  4. Purchase if needed
  5. Confirm readiness
```

### Transaction Execution (4 prompts)

**send_usdt**
```
Template: Complete USDT transfer with resource optimization
Steps:
  1. Validate recipient address
  2. Estimate energy
  3. Ensure resources
  4. Execute transfer
  5. Verify on-chain
```

**send_trx**
```
Template: TRX transfer (simpler - no energy needed)
Steps:
  1. Validate recipient address
  2. Check bandwidth
  3. Execute transfer
  4. Verify on-chain
```

**swap_tokens**
```
Template: DEX swap with full pipeline
Steps:
  1. Get quote
  2. Review price impact and slippage
  3. Check if approval needed
  4. Ensure resources for approval + swap
  5. Execute swap
  6. Verify result
```

**execute_complex_intent**
```
Template: Multi-step transaction plan
Steps:
  1. Help user define steps
  2. Simulate all steps
  3. Review total costs
  4. Choose resource strategy
  5. Execute intent
  6. Report results
```

### Automation (4 prompts)

**setup_standing_order**
```
Template: Configure automatic energy purchases
Steps:
  1. Understand user's needs (volume, timing, budget)
  2. Recommend trigger type
  3. Set appropriate constraints
  4. Create standing order
  5. Explain monitoring and management
```

**setup_delegation_monitor**
```
Template: Configure auto-renewal for energy delegations
Steps:
  1. Review current delegations
  2. Set renewal window
  3. Configure price protection
  4. Set notification channels
  5. Create monitor
```

**setup_balance_monitor**
```
Template: Configure balance-based alerts and actions
Steps:
  1. Analyze current resource usage patterns
  2. Recommend threshold levels
  3. Configure action (buy, ensure, or notify)
  4. Set budget constraints
  5. Create monitor
```

**review_automation**
```
Template: Audit and optimize existing automation
Steps:
  1. List all standing orders and monitors
  2. Review execution history
  3. Identify underperforming rules
  4. Suggest improvements
  5. Apply changes if approved
```

### Analysis (4 prompts)

**analyze_prices**
```
Template: Market price analysis
Steps:
  1. Pull current prices from all providers
  2. Pull price history
  3. Identify trends and patterns
  4. Calculate optimal buy times
  5. Present analysis with charts
```

**calculate_savings**
```
Template: Calculate savings from using delegated energy
Steps:
  1. Estimate energy for user's transaction types
  2. Calculate cost with TRX burn
  3. Calculate cost with energy purchase
  4. Show savings percentage
  5. Project annual savings at given volume
```

**estimate_transaction_cost**
```
Template: Cost estimation for any transaction type
Steps:
  1. Identify transaction type
  2. Simulate with exact parameters
  3. Show energy and bandwidth requirements
  4. Show cost with and without delegation
  5. Recommend optimal approach
```

**portfolio_overview**
```
Template: Complete account overview
Steps:
  1. Check all balances (TRX, USDT, other tokens)
  2. Check resource allocation
  3. Review active delegations
  4. Summarize active automation rules
  5. Present financial overview
```

### Account Management (3 prompts)

**account_setup**, **deposit_guide**, **withdrawal_guide** - Step-by-step templates for account lifecycle operations.

### x402 (2 prompts)

**x402_purchase** - Guide through the zero-registration purchase flow.
**x402_explain** - Explain how x402 works and when to use it.

### Network Information (2 prompts)

**explain_energy**, **explain_bandwidth** - Educational templates that explain TRON's resource model using data from resources.

### Troubleshooting (2 prompts)

**transaction_failed**, **delegation_not_arrived** - Diagnostic templates that help identify and resolve common issues.

### Integration (2 prompts)

**sdk_quickstart**, **api_integration** - Templates for developers integrating MERX into their applications.

### Why Prompts Matter

Without prompts, a model receiving the request "help me buy energy" would need to:
1. Figure out which tools are relevant
2. Determine the correct order of operations
3. Decide what information to gather first
4. Handle edge cases it may not know about

With the `buy_cheapest_energy` prompt, the model has a tested, optimized workflow that handles edge cases, presents information in the right format, and follows the correct sequence of operations. The difference is the same as handing someone a set of tools versus handing them tools plus a manual.

## Primitive 3: Resources (Data)

Resources are structured data that the model can read without making an action call. Unlike tools, which do something, resources provide something - they make information passively available.

MERX provides 21 resources: 14 static and 7 dynamic templates.

### Static Resources (14)

Static resources provide fixed reference data that does not change between sessions:

```
merx://docs/getting-started
merx://docs/energy-explained
merx://docs/bandwidth-explained
merx://docs/resource-model
merx://docs/providers-overview
merx://docs/x402-protocol
merx://docs/standing-orders-guide
merx://docs/monitors-guide
merx://docs/intent-execution-guide
merx://docs/api-reference
merx://docs/sdk-js
merx://docs/sdk-python
merx://docs/faq
merx://docs/troubleshooting
```

When the model reads `merx://docs/energy-explained`, it receives a structured document explaining TRON's energy model - what energy is, how it is consumed, how delegation works, and how it relates to TRX costs. This information is available immediately, without making any API calls or tool invocations.

### Dynamic Template Resources (7)

Template resources accept parameters and return data specific to the request:

```
merx://prices/current
merx://prices/history/{period}
merx://providers/{provider_name}
merx://account/{address}/resources
merx://account/{address}/balances
merx://network/parameters
merx://network/chain-info
```

For example, `merx://prices/current` returns the current energy prices from all providers in a structured format:

```json
{
  "timestamp": "2026-03-30T12:00:00Z",
  "prices": [
    {
      "provider": "sohu",
      "energy_1h": 0.0000526,
      "energy_24h": 0.0000498,
      "available": true,
      "min_amount": 65000
    },
    {
      "provider": "catfee",
      "energy_1h": 0.0000540,
      "energy_24h": 0.0000510,
      "available": true,
      "min_amount": 65000
    }
  ]
}
```

And `merx://account/{address}/resources` returns the current resource allocation for any address:

```json
{
  "address": "TYourAddress...",
  "energy": {
    "available": 487231,
    "total": 500000,
    "used": 12769,
    "delegated_from": [
      {
        "amount": 500000,
        "provider": "sohu",
        "expires_at": "2026-03-31T02:00:00Z"
      }
    ]
  },
  "bandwidth": {
    "available": 1342,
    "total": 1500,
    "free_remaining": 1342
  }
}
```

### Why Resources Matter

Resources give the model context without the overhead of tool calls. When a user asks "what is my energy balance?", the model can check the resource `merx://account/{address}/resources` directly. When the model needs to reference documentation about standing orders before creating one, it reads `merx://docs/standing-orders-guide`.

Without resources, every piece of information requires a tool call. The model would need to call `get_prices` to see current prices, even though that information could be passively available. The distinction is between a model that must actively request every piece of data (tools-only) and a model that has a rich information environment available for reference (tools + resources).

## The Full-Protocol Advantage

When all three primitives work together, the model operates at a fundamentally different level:

### Scenario: User asks "Help me set up energy automation"

**Tools-only MCP server:**
1. Model guesses which tools exist for automation
2. Calls tools in a trial-and-error sequence
3. May miss important configuration options
4. No guidance on best practices
5. No background reference material

**Full-protocol MERX MCP server:**
1. Model reads `merx://docs/standing-orders-guide` for reference material
2. Model activates `setup_standing_order` prompt for step-by-step workflow
3. Model reads `merx://prices/history/7d` to recommend price thresholds
4. Model calls `create_standing_order` tool with optimized parameters
5. Model reads `merx://docs/monitors-guide` and suggests complementary monitors
6. Model calls `create_monitor` tool for delegation expiry protection
7. Result: Complete, well-configured automation with documentation-backed decisions

The full-protocol approach is not just marginally better - it produces qualitatively different outcomes because the model has access to guidance (prompts), knowledge (resources), and capability (tools) simultaneously.

## What Other MCP Servers Are Missing

A survey of blockchain-related MCP servers as of early 2026 shows a consistent pattern:

| Server | Tools | Prompts | Resources | Full Protocol |
|---|---|---|---|---|
| Generic ETH MCP | 5-8 | 0 | 0 | No |
| Solana MCP | 10-12 | 0 | 0 | No |
| Bitcoin MCP | 3-4 | 0 | 0 | No |
| Multi-chain MCP | 15-20 | 0 | 0 | No |
| MERX | 52 | 30 | 21 | Yes |

No other blockchain MCP server implements prompts or resources. They are all tools-only, which means:

- No workflow guidance for multi-step operations
- No documentation available at model inference time
- No passive data access for context and reference
- Every interaction starts from scratch with no structural knowledge

MERX is the only MCP server where the model has a complete operational environment - the ability to act (tools), the knowledge of how to act (prompts), and the data to inform decisions (resources).

## The Numbers

MERX by the numbers:

- **54 tools** across 5 categories (wallet, energy market, DEX, automation, advanced)
- **30 prompts** across 10 categories (getting started, energy, transactions, automation, analysis, accounts, x402, network, troubleshooting, integration)
- **21 resources** (14 static documentation, 7 dynamic data templates)
- **72 total capabilities** exposed through a single MCP server

For an AI agent connected to MERX, this means:
- It can perform any TRON operation (tools)
- It knows the best approach for any scenario (prompts)
- It has real-time data and documentation always available (resources)

## Conclusion

MCP is a protocol with three primitives, not one. Implementing only tools is like building an API with endpoints but no documentation and no data feeds. It works, technically, but it forces the consumer to figure out everything on their own.

MERX implements the full protocol because the full protocol is how MCP was designed to be used. Tools for action. Prompts for guidance. Resources for knowledge. All three working together to create an environment where an AI agent can operate on the TRON blockchain with the same depth of capability as a human developer with years of experience.

Thirty prompts. Twenty-one resources. Twenty-one tools. No other blockchain MCP server comes close.

---

**Links:**
- MERX Platform: [https://merx.exchange](https://merx.exchange)
- MCP Server (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP Server (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)


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
