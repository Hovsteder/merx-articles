# De la idea a produccion: construyendo MERX en 30 dias

MERX went from concept to live production system in 30 days. Not a landing page. Not a prototype. A fully operational blockchain resource exchange with seven provider integrations, tiempo real price aggregation, en cadena ejecucion de ordenes, partida doble accounting, comprehensive documentation, SDKs in two languages, and an MCP server with 54 tools for AI agent integration.

This article is the technical story of how it happened - the architecture decisions, the problems we solved, the shortcuts we deliberately did not take, and the lessons from building a financial platform at speed without compromising on the things that matter.

---

## Day 0: The Problem Statement

The TRON mercado de energia is fragmented. Seven or more providers offer delegacion de energia services, each with their own API, pricing, and reliability characteristics. If you want el mejor precio, you need to integrate with all of them. If you want respaldo, you need to build routing logic. If you want transparency, you need to build monitoring.

Every business sending USDT on TRON faces this integration tax. The solution is an aggregation layer - a una sola API that handles multi-provider routing, best-price selection, and respaldo automatico.

No one had built it yet. We decided to.

---

## Week 1: Foundation

### Architecture First

Before writing a single line of code, we spent two days on architecture. The result was a 40-section architecture document covering everything from database schema to API formato de errors to color hex codes. This document became the single fuente de verdad for every implementation decision.

Key architecture decisions made in those two days:

**Decision 1: Microservices from day one.**

Not because microservices are trendy, but because financial systems need isolation. The treasury signer must not be accessible from the API service. The monitor de precios must not have write access to user balances. Docker containers provide this isolation naturally.

```
services/
  api/              HTTP/WebSocket API
  price-monitor/    Provider price polling
  order-executor/   Order routing and execution
  ledger/           Double-entry accounting
  deposit-monitor/  Incoming payment detection
  treasury-signer/  Transaction signing (isolated)
```

**Decision 2: PostgreSQL + Redis, no exotic databases.**

PostgreSQL for everything that needs ACID guarantees (balances, orders, entradas del libro mayor). Redis for everything that needs speed (price cache, pub/sub, limite de velocidading). Both are battle-tested, well-documented, and operationally simple.

**Decision 3: All amounts in SUN.**

Every financial value stored as an integer in SUN (1 TRX = 1,000,000 SUN). No floating-point anywhere in the financial path. This eliminated an entire category of bugs before we wrote our first function.

**Decision 4: Node.js + TypeScript for services, Go for the matching engine.**

TypeScript for the bulk of the system - fast development, strong typing, excellent async I/O for API and monitoring workloads. Go reserved for the matching engine where raw performance matters.

### Database Schema

The database migrations were written on day 3. Every table was designed with financial integrity in mind:

```sql
-- Core principle: every balance mutation creates a ledger entry
CREATE TABLE ledger (
  id BIGSERIAL PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(id),
  type VARCHAR(50) NOT NULL,
  amount_sun BIGINT NOT NULL,
  direction VARCHAR(6) NOT NULL CHECK (direction IN ('DEBIT', 'CREDIT')),
  reference_type VARCHAR(50),
  reference_id UUID,
  balance_before BIGINT NOT NULL,
  balance_after BIGINT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- No UPDATE or DELETE triggers - ledger is append-only
```

One file per migration, named `YYYYMMDD_description.sql`. By the end of the 30 days, there were 14 migration files, each one additive, none destructive.

### Provider Interface

The `IEnergyProvider` interface was defined on day 4. This was the contract every provider adapter would implement:

```typescript
interface IEnergyProvider {
  name: string;
  getPrices(): Promise<ProviderPriceResponse>;
  createOrder(params: OrderParams): Promise<OrderResult>;
  getOrderStatus(orderId: string): Promise<OrderStatus>;
  healthCheck(): Promise<boolean>;
}
```

This interface never changed. Seven providers were integrated against it over the following weeks, each in its own file, none requiring changes to the core system.

---

## Week 2: Core Services

### Price Monitor

The monitor de precios was the first service to go live. It polls every provider every 30 seconds, normalizes prices, publishes to Redis, and stores history in PostgreSQL. The implementation is roughly 180 lines of TypeScript across three files.

The hardest part was not the polling logic - it was the normalization. Each provider returns prices in slightly different formats:

- Provider A: SUN per unidad de energia
- Provider B: total TRX for a fixed cantidad de energia
- Provider C: SUN per unidad de energia, but with a different minimum order
- Provider D: tiered pricing based on volume

Each adapter translates its provider's format into the standard `ProviderPriceResponse`. The monitor de precios does not care about provider quirks; it only sees normalized data.

### Order Executor

The ejecutor de ordenes is the most complex service. It reads prices from Redis, determines optimal routing, submits orders to providers, monitors for en cadena confirmation, and publishes settlement events.

The cadena de respaldo was the critical design element. If Provider A fails, try Provider B. If B fails, try C. The buyer's API call succeeds as long as any provider is operational.

```
Order received -> Read prices -> Select cheapest
  -> Execute at Provider A
    -> Success? Verify on-chain -> Settle
    -> Failure? Try Provider B
      -> Success? Verify on-chain -> Settle
      -> Failure? Try Provider C
        -> ... and so on
```

### Ledger Service

The ledger service enforces the partida doble constraint. Every mutacion de saldo creates paired entries. The service runs a reconciliation check every hour:

```sql
SELECT SUM(CASE direction
  WHEN 'DEBIT' THEN amount_sun
  WHEN 'CREDIT' THEN -amount_sun
END) FROM ledger;
-- Must be 0. If not: alert immediately.
```

In 30 days of development and testing, this check never fired. The constraint was never violated because the architecture made violations structurally impossible, not just unlikely.

---

## Week 3: API, Frontend, and On-Chain Verification

### API Design

The API follows REST conventions with strict versioning (`/api/v1/...`). Every endpoint was designed before implementation:

```
GET    /api/v1/prices          Current prices from all providers
GET    /api/v1/prices/best     Best current price
POST   /api/v1/orders          Create a new order
GET    /api/v1/orders/:id      Get order status
GET    /api/v1/balance         Get account balance
POST   /api/v1/deposit         Get deposit address
POST   /api/v1/withdraw        Request withdrawal
```

Error responses use a consistent format:

```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Account balance (5.2 TRX) is insufficient for this order (8.1 TRX)",
    "details": {
      "balance": 5200000,
      "required": 8100000
    }
  }
}
```

No endpoint was published without Zod validation on all inputs.

### Frontend

The frontend is a Next.js application with a strict design system: dark theme only, no rounded corners over 2px, no gradients, no shadows, Cormorant Garamond for headings, IBM Plex Mono for everything else. The visual identity was defined in the architecture document and implemented faithfully.

### On-Chain Verification

Every order is verified on the TRON blockchain. The verification service watches for delegation transactions and confirms that energy arrived at the target address. This was the most challenging integration because blockchain confirmation times are variable and provider transaction formats differ.

Eight mainnet transactions were verified during the testing phase, confirming that the end-to-end flow - from API call to en cadena delegation - worked correctly with real TRX and real providers.

---

## Week 4: SDKs, MCP Server, and Documentation

### JavaScript SDK

The JavaScript SDK was built for Node.js and browser environments:

```typescript
import { MerxClient } from 'merx-sdk';

const client = new MerxClient({ apiKey: 'your-key' });
const prices = await client.getPrices({ energy: 65000 });
const order = await client.createOrder({
  energy: 65000,
  targetAddress: 'TAddress...',
  duration: '1h'
});
```

Fuente: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)

### Python SDK

The Python SDK mirrors the JavaScript SDK's API surface:

```python
from merx import MerxClient

client = MerxClient(api_key='your-key')
prices = client.get_prices(energy=65000)
order = client.create_order(
    energy=65000,
    target_address='TAddress...',
    duration='1h'
)
```

Fuente: [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)

### Servidor MCP: 52 Tools

The MCP (Model Context Protocol) server was perhaps the most forward-looking component. It exposes MERX functionality as tools that AI agents can use directly.

The MCP server grew from 7 tools in its initial version to 54 tools by the end of the 30 days:

```
Account management:    create_account, login, get_balance, get_deposit_info
Price data:            get_prices, get_best_price, compare_providers, analyze_prices
Order management:      create_order, get_order, list_orders, create_standing_order
Resource monitoring:   check_address_resources, estimate_transaction_cost
TRON utilities:        validate_address, convert_address, get_trx_balance
On-chain operations:   transfer_trx, transfer_trc20, approve_trc20
Analytics:             calculate_savings, get_price_history, suggest_duration
... and 30 more
```

Fuente: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)

### Documentation

Documentation was rebuilt from 5 pages to 36 pages, covering the complete API reference, SDK guides, TRON concepts, and integration tutorials. The documentation lives at [https://merx.exchange/docs](https://merx.exchange/docs).

Additionally, 4 SEO guide pages and 7 provider comparison pages were published, bringing the sitemap to 53 URLs.

---

## What We Did Not Compromise On

Speed creates pressure to cut corners. Aqui estan the corners we explicitly did not cut:

### No Floating-Point for Money

Using integers (SUN) for all financial values added complexity in display formatting but eliminated rounding errors entirely. Every test case matched expected values exactly.

### No String Concatenation for SQL

Every database query uses parameterized statements. This was a non-negotiable rule from day one. SQL injection is a solved problem, and we kept it solved.

### No Hardcoded Secrets

Environment variables from day one. Docker secrets for the treasury key. `.gitignore` set up before the first commit.

### No Services Sharing State Directly

Services communicate via Redis pub/sub or REST API calls. No direct imports between services. This made independent deployment possible and prevented cascade failures.

### No Ledger Mutations

Append-only ledger from the first migration. No UPDATE or DELETE on ledger tables. Corrections create new entries, not modifications.

---

## What We Learned

### Lesson 1: Architecture Documents Pay for Themselves

The two days spent on architecture saved weeks of rework. Every developer question was answered by the document. Every design disagreement was resolved by referencing the spec. The 40 sections were not bureaucratic overhead; they were a forcing function for thinking through problems before they became bugs.

### Lesson 2: Provider APIs Are Unreliable

Of the seven providers integrated, at least two experienced downtime during the 30-day build period. The cadena de respaldo was not a theoretical nicety - it was exercised within the first week of testing.

### Lesson 3: The Adapter Pattern Is Worth the Boilerplate

Writing seven adapters that all implement the same interface felt repetitive. But when Provider C changed their API response format on day 22, we updated one file and nothing else changed. The 10 minutes spent updating the adapter versus the days we would have spent updating every call site made the pattern's value obvious.

### Lesson 4: MCP Is the Future of Service Integration

The MCP server was initially an experiment. But watching AI agents use MERX tools to autonomously manage adquisicion de energia was a revelation. This is how services will be consumed in the future - not through human developers writing integration code, but through AI agents calling tool APIs directly.

### Lesson 5: 200-Line File Limit Is a Feature

We enforced a strict 200-line-per-file limit throughout the project. This forced constant decomposition. Functions stayed small. Responsibilities stayed clear. When a file approached 200 lines, it was time to split, and the split always improved clarity.

---

## By the Numbers

```
Architecture document:     40 sections
Services:                  9 Docker containers
Provider integrations:     7
Database migrations:       14
API endpoints:             20+
MCP tools:                 52 (from initial 7)
SDK languages:             2 (JavaScript, Python)
Documentation pages:       36 (from initial 5)
Sitemap URLs:              53
Mainnet transactions:      8 verified
Commission rate:           0%
Days to production:        30
```

---

## What Comes Next

The platform is live at [https://merx.exchange](https://merx.exchange). The immediate focus is testing, optimization, and onboarding the first production users. The foundation is solid - the architecture supports horizontal scaling, new providers can be added in hours, and the cero comision model removes adoption friction.

The energy aggregation market on TRON is waiting for a platform that makes it simple. MERX is that platform.

---

*MERX es el primer exchange de recursos blockchain. Explore la plataforma en [https://merx.exchange](https://merx.exchange). Documentacion en [https://merx.exchange/docs](https://merx.exchange/docs). SDKs de codigo abierto y servidor MCP en GitHub.*

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
