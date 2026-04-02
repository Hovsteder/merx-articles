# Three Protocols, One TRON Agent: MCP, A2A, and ACP on MERX

The AI agent ecosystem is fragmenting across protocols. Anthropic has MCP. Google launched A2A. BeeAI built ACP. Each protocol solves agent communication differently, and each has its own ecosystem of frameworks and orchestrators. For developers building TRON applications, this means choosing a protocol also means choosing which frameworks can access TRON resources. MERX eliminates that choice by supporting all three protocols from a single platform -- the first and only TRON agent to do so.

## The Protocol Landscape in 2026

Three protocols now dominate the AI agent infrastructure:

**MCP (Model Context Protocol)** -- created by Anthropic. A tool-based protocol where agents discover and call functions. 55 tools, 30 prompts, 21 resources. Used by Claude, Cursor, Windsurf, and hundreds of MCP-compatible clients. This is the most mature protocol for direct agent-to-tool interaction.

**A2A (Agent-to-Agent Protocol)** -- created by Google, now under the Linux Foundation. A task-based protocol where orchestrators submit tasks to specialist agents and receive results asynchronously. Used by LangChain, CrewAI, Vertex AI Agent Builder, AutoGen, and Mastra. Designed for multi-agent systems where one agent delegates work to another.

**ACP (Agent Communication Protocol)** -- created by BeeAI (IBM). A run-based protocol for enterprise orchestrators. ACP is now merging into A2A under the Linux Foundation, but the protocol endpoint remains useful for existing ACP clients.

Each protocol has a different discovery mechanism, a different execution model, and a different set of compatible frameworks. A TRON agent that only speaks MCP is invisible to LangChain. An agent that only speaks A2A is invisible to Claude. Until now, no TRON project supported more than one of these protocols.

## What MERX Now Supports

As of April 2026, MERX supports all three protocols from a single deployment:

| Protocol | Discovery | Execution | Compatible Frameworks |
|----------|-----------|-----------|----------------------|
| MCP | `merx.exchange/mcp/sse` | Tool calls (request-response) | Claude, Cursor, Windsurf, any MCP client |
| A2A | `merx.exchange/.well-known/agent.json` | Tasks (async, SSE streaming) | LangChain, CrewAI, Vertex AI, AutoGen, Mastra |
| ACP | `merx.exchange/.well-known/agent-manifest.json` | Runs (async, long-polling) | BeeAI, IBM watsonx, ACP frameworks |

All three protocols share the same backend. When an A2A task calls `buy_energy`, it executes the exact same order routing logic as an MCP `create_order` tool call. The protocols are different entry points to the same MERX aggregation engine.

## MCP: 53 Tools for Direct Integration

MCP is the deepest integration point. The MERX MCP server provides 55 tools organized into 15 categories:

- **Price Intelligence** (5 tools): real-time prices from all 7 providers, best-price routing, historical data, market analysis
- **Resource Trading** (4 tools): create orders, list orders, get order details, ensure resources
- **Token Operations** (4 tools): send TRX, send TRC-20 tokens, approve allowances, query token info
- **DEX Swaps** (3 tools): get SunSwap quotes, execute swaps, check token prices
- **Standing Orders** (4 tools): server-side automation with price triggers, schedules, balance alerts
- **On-chain Queries** (5 tools): account info, balances, transactions, block data
- Plus 28 more tools across estimation, convenience, contracts, network, onboarding, payments, intent execution, and session management

The MCP server also provides 30 prompts for guided workflows and 21 resources for structured data access.

### Connect in one line

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

No installation. No API keys for read-only tools. 22 tools available immediately.

## A2A: 6 Skills for Orchestrator Frameworks

A2A exposes MERX as a specialist agent that orchestrators can delegate tasks to. The Agent Card at `/.well-known/agent.json` advertises 8 skills:

| Skill | Description | Auth Required |
|-------|-------------|---------------|
| `buy_energy` | Purchase delegated energy from the aggregated market | Yes |
| `get_prices` | Current energy prices from all 7 providers | No |
| `analyze_prices` | Market analysis with trends and recommendations | No |
| `check_balance` | Account balance and on-chain resource allocation | Optional |
| `ensure_resources` | Declarative resource provisioning (buy only the deficit) | Yes |
| `create_standing_order` | Server-side automation rules | Yes |

### How A2A works

The A2A protocol uses a task-based model. An orchestrator submits a task, MERX processes it asynchronously, and the orchestrator retrieves the result.

**Step 1: Discover the agent**

```bash
curl https://merx.exchange/.well-known/agent.json
```

This returns the Agent Card with all 8 skills, their input schemas, supported modes, and authentication requirements.

**Step 2: Submit a task**

```bash
curl -X POST https://merx.exchange/a2a/tasks/send \
  -H "Content-Type: application/json" \
  -d '{
    "id": "task-001",
    "message": {
      "role": "user",
      "parts": [{
        "type": "data",
        "data": { "action": "get_prices" }
      }]
    }
  }'
```

The response returns immediately with status `submitted`. The task processes in the background.

**Step 3: Get the result**

```bash
curl https://merx.exchange/a2a/tasks/task-001
```

The response includes the task status (`completed`, `failed`, etc.) and the result artifacts with price data from all 7 providers.

**Step 4: Stream events (optional)**

```bash
curl -N https://merx.exchange/a2a/tasks/task-001/events
```

SSE stream delivers real-time state transitions: `submitted` to `working` to `completed`.

### Skill routing

A2A tasks can use structured data or natural language. The task processor routes automatically:

- **Structured**: `{ "action": "buy_energy", "energy_amount": 65000, "target_address": "T..." }` routes directly to the buy_energy skill
- **Natural language**: "What is the current energy price?" matches the keyword pattern and routes to get_prices

## ACP: Run-Based Execution for Enterprise

ACP uses a run-based model similar to A2A but with a different API surface. The manifest at `/.well-known/agent-manifest.json` declares the same 8 capabilities.

```bash
# Create a run
curl -X POST https://merx.exchange/acp/v1/agents/merx-tron-agent/runs \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "merx-tron-agent",
    "input": [{
      "role": "user",
      "parts": [{
        "contentType": "application/json",
        "content": "{\"action\":\"get_prices\"}"
      }]
    }]
  }'

# Poll result (with long-polling)
curl "https://merx.exchange/acp/v1/runs/{runId}?wait=true"
```

The `?wait=true` parameter enables long-polling: the request blocks for up to 30 seconds waiting for the run to complete, reducing the need for repeated polling.

Note: ACP is merging into A2A under the Linux Foundation. The endpoint will continue to operate for existing clients, but new integrations should use A2A.

## Architecture: One Backend, Three Entry Points

All three protocols share the same execution path:

```
MCP Tool Call ─┐
               ├──► MERX API ──► Provider Router ──► 7 Energy Providers
A2A Task ──────┤                                     (Netts, CatFee, TEM,
               ├──► MERX API                          ITRX, TronSave, Feee,
ACP Run ───────┘                                      PowerSun)
```

The A2A and ACP handlers run inside the existing API service (`services/api/src/agent-protocols/`). They make internal HTTP calls to the same REST endpoints that the MCP server and web dashboard use. This means:

- **Same prices**: all protocols see the same real-time provider data
- **Same routing**: orders go through the same cheapest-provider logic
- **Same auth**: X-API-Key works across all protocols
- **Same reliability**: failover and retry logic applies equally

Task and run state is stored in Redis with 24-hour TTL. No database writes required for protocol operations.

## Why Multi-Protocol Matters

### For developers

You are building a TRON integration. Your orchestration framework uses LangChain (A2A). Your teammate's bot uses Claude (MCP). Your enterprise client requires BeeAI (ACP). With a single-protocol agent, you need three different integrations to the same underlying service.

With MERX, all three connect to the same platform. One API key. One set of documentation. One support channel.

### For agent builders

Multi-agent systems are becoming standard. A planning agent coordinates with a trading agent, a monitoring agent, and a reporting agent. These agents may run on different frameworks. A CrewAI crew might delegate energy purchases to MERX via A2A while a Claude agent monitors prices via MCP.

MERX handles both without the agents knowing about each other's protocol choice.

### For the TRON ecosystem

More protocol coverage means more potential integrations. Every AI framework that supports A2A can now access TRON energy markets. Every MCP client can optimize TRON transaction costs. The total addressable market for TRON energy services expands with each supported protocol.

## Where MERX Is Listed

MERX is the only TRON project with presence across both MCP and A2A directories:

**MCP registries:**
- [Glama](https://glama.ai/mcp/servers/Hovsteder/merx-mcp)
- [Smithery](https://smithery.ai/servers/powersun/merx)
- [Official MCP Registry](https://registry.modelcontextprotocol.io/servers/exchange.merx/mcp)
- mcp.so
- PulseMCP

**A2A directories:**
- [awesome-a2a](https://github.com/pab1it0/awesome-a2a) (Financial Services section)
- [a2aregistry.in](https://a2aregistry.in)

## Getting Started

### MCP (Claude, Cursor, Windsurf)

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

### A2A (LangChain, CrewAI, Vertex AI, AutoGen)

Discovery URL: `https://merx.exchange/.well-known/agent.json`

### ACP (BeeAI)

Discovery URL: `https://merx.exchange/.well-known/agent-manifest.json`

### Documentation

- [Agent Protocols overview](https://merx.exchange/agents)
- [MCP Server (55 tools)](https://merx.exchange/mcp)
- [A2A Protocol docs](https://merx.exchange/docs/tools/a2a)
- [ACP Protocol docs](https://merx.exchange/docs/tools/acp)
- [GitHub](https://github.com/Hovsteder/merx-mcp)

---

*Tags: tron mcp server, a2a protocol, acp protocol, tron ai agent, ai agent tron energy, langchain tron, crewai tron, multi-protocol agent, merx exchange*

---

**Try it now with AI**

Add to your MCP client:

```json
{ "merx": { "url": "https://merx.exchange/mcp/sse" } }
```

Or discover via A2A:

```bash
curl https://merx.exchange/.well-known/agent.json
```

Then ask: "What are the current TRON energy prices across all providers?"
