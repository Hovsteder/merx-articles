# Кроссплатформенная интеграция: MERX в вашем существующем стеке

Every technology stack is different. Your backend might be Node.js, Go, Python, Ruby, or Java. Your architecture might be monolithic or microservices. Your communication patterns might be synchronous, event-driven, or a combination. The question is not whether MERX can fit into your stack -- it is which integration point gives you the most value with the least friction.

MERX offers five integration methods: REST API, WebSocket, webhooks, language SDKs, and an MCP server for AI agents. This article explains when to use each, how they interact, and how to choose the right approach for your specific architecture.

## Integration Methods Overview

| Method | Best For | Direction | Latency | Language |
|---|---|---|---|---|
| REST API | Request-response operations | Client -> MERX | ~200ms | Any |
| WebSocket | Real-time price feeds | MERX -> Client | Real-time | Any |
| Webhooks | Async notifications | MERX -> Client | Event-driven | Any |
| JS SDK | Node.js / browser apps | Bidirectional | ~200ms | JavaScript/TypeScript |
| Python SDK | Python backends, scripts | Bidirectional | ~200ms | Python |
| MCP Server | AI agent integration | Bidirectional | ~500ms | Any (via MCP protocol) |

## REST API: The Universal Integration

The REST API works from any programming language with HTTP support. It is the lowest-common-denominator integration -- if your language can make HTTP requests, it can talk to MERX.

### When to Use REST

- Your backend is in a language without an SDK (Go, Java, Ruby, Rust, PHP)
- You need occasional energy purchases, not continuous interaction
- Your architecture prefers explicit HTTP calls over persistent connections
- You are prototyping and want the simplest possible integration

### Implementation

The API follows standard REST conventions with JSON payloads:

```bash
# Check prices
curl -X POST https://merx.exchange/api/v1/prices \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"energy_amount": 65000, "duration": "1h"}'

# Place an order
curl -X POST https://merx.exchange/api/v1/orders \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: unique-request-id-123" \
  -d '{
    "energy_amount": 65000,
    "duration": "1h",
    "target_address": "TYourAddress..."
  }'

# Check order status
curl https://merx.exchange/api/v1/orders/ORDER_ID \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Go Example

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
)

type PriceRequest struct {
    EnergyAmount int    `json:"energy_amount"`
    Duration     string `json:"duration"`
}

type PriceResponse struct {
    Best struct {
        Provider string `json:"provider"`
        PriceSun int    `json:"price_sun"`
    } `json:"best"`
    Providers []struct {
        Provider string `json:"provider"`
        PriceSun int    `json:"price_sun"`
    } `json:"providers"`
}

func getBestPrice(amount int, duration string) (*PriceResponse, error) {
    body, _ := json.Marshal(PriceRequest{
        EnergyAmount: amount,
        Duration:     duration,
    })

    req, _ := http.NewRequest(
        "POST",
        "https://merx.exchange/api/v1/prices",
        bytes.NewBuffer(body),
    )
    req.Header.Set("Authorization", "Bearer "+apiKey)
    req.Header.Set("Content-Type", "application/json")

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var result PriceResponse
    json.NewDecoder(resp.Body).Decode(&result)
    return &result, nil
}
```

### Error Handling

All API errors follow a consistent format:

```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Account balance is insufficient for this order",
    "details": {
      "required": 1820000,
      "available": 500000
    }
  }
}
```

This consistency means your error handling logic works the same regardless of which endpoint you call or which provider is involved behind the scenes.

## WebSocket: Real-Time Price Feeds

WebSocket connections provide real-time price updates without polling. Prices stream to your application as they change across all seven providers.

### When to Use WebSocket

- You need live price displays (dashboards, trading interfaces)
- Your application makes purchasing decisions based on price movements
- You want to trigger actions when prices cross thresholds
- You are building real-time monitoring tools

### Implementation

```typescript
import WebSocket from 'ws';

const ws = new WebSocket(
  'wss://merx.exchange/ws',
  { headers: { 'Authorization': `Bearer ${API_KEY}` } }
);

ws.on('open', () => {
  // Subscribe to price updates for specific parameters
  ws.send(JSON.stringify({
    type: 'subscribe',
    channel: 'prices',
    params: {
      energy_amount: 65000,
      duration: '1h'
    }
  }));
});

ws.on('message', (data) => {
  const event = JSON.parse(data.toString());

  switch (event.type) {
    case 'price_update':
      console.log(
        `${event.provider}: ${event.price_sun} SUN`
      );
      break;

    case 'order_update':
      console.log(
        `Order ${event.order_id}: ${event.status}`
      );
      break;
  }
});
```

### Combining WebSocket with REST

A common pattern: use WebSocket for price monitoring and REST for order placement:

```typescript
// WebSocket monitors prices
ws.on('message', async (data) => {
  const event = JSON.parse(data.toString());

  if (event.type === 'price_update' &&
      event.price_sun <= targetPrice) {
    // REST API places the order
    const response = await fetch(
      'https://merx.exchange/api/v1/orders',
      {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${API_KEY}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          energy_amount: 65000,
          duration: '1h',
          target_address: walletAddress
        })
      }
    );
  }
});
```

## Webhooks: Async Event Notifications

Webhooks push notifications to your server when events occur. Unlike WebSocket (which requires a persistent connection), webhooks work with any server that can receive HTTP POST requests.

### When to Use Webhooks

- Your architecture is event-driven (message queues, serverless functions)
- You cannot maintain persistent WebSocket connections
- You need reliable delivery with retries
- You want to decouple energy procurement from transaction execution

### Implementation

```typescript
import express from 'express';

const app = express();
app.use(express.json());

app.post('/webhooks/merx', (req, res) => {
  const event = req.body;

  switch (event.type) {
    case 'order.filled':
      handleOrderFilled(event.data);
      break;

    case 'order.failed':
      handleOrderFailed(event.data);
      break;

    case 'standing_order.triggered':
      handleStandingOrderTriggered(event.data);
      break;

    case 'auto_energy.purchased':
      handleAutoEnergyPurchased(event.data);
      break;
  }

  // Always respond 200 to acknowledge receipt
  res.status(200).json({ received: true });
});

async function handleOrderFilled(
  data: OrderFilledEvent
): Promise<void> {
  // Energy is available -- proceed with the transaction
  const pendingTx = await db.getPendingTransaction(
    data.order_id
  );
  if (pendingTx) {
    await executeTransaction(pendingTx);
  }
}
```

### Webhook Reliability

MERX retries failed webhook deliveries with exponential backoff. Your endpoint should:

1. Respond with 200 status quickly (within 5 seconds)
2. Process the event asynchronously if processing takes time
3. Handle duplicate deliveries idempotently (use the event ID for deduplication)

```typescript
const processedEvents = new Set<string>();

app.post('/webhooks/merx', async (req, res) => {
  const event = req.body;

  // Idempotency check
  if (processedEvents.has(event.id)) {
    res.status(200).json({ received: true });
    return;
  }

  processedEvents.add(event.id);

  // Acknowledge immediately
  res.status(200).json({ received: true });

  // Process asynchronously
  processEvent(event).catch(console.error);
});
```

## JavaScript SDK

The JavaScript/TypeScript SDK wraps the REST API and WebSocket connections with typed interfaces and convenience methods.

### When to Use the JS SDK

- Your backend is Node.js or your frontend is browser-based
- You want TypeScript types and IDE autocompletion
- You prefer method calls over raw HTTP requests
- You need both REST and WebSocket in one package

### Implementation

```typescript
import { MerxClient } from '@anthropic/merx-sdk';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY!
});

// Prices
const prices = await merx.getPrices({
  energy_amount: 65000,
  duration: '1h'
});

// Orders
const order = await merx.createOrder({
  energy_amount: 65000,
  duration: '1h',
  target_address: 'TAddress...'
});

// Standing orders
const standing = await merx.createStandingOrder({
  energy_amount: 65000,
  max_price_sun: 25,
  duration: '1h',
  repeat: true
});

// Energy estimation
const estimate = await merx.estimateEnergy({
  contract_address: 'TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t',
  function_selector: 'transfer(address,uint256)',
  parameter: [recipient, amount],
  owner_address: sender
});

// WebSocket (built into the SDK)
const ws = merx.connectWebSocket();
ws.on('price_update', (data) => {
  console.log(`${data.provider}: ${data.price_sun} SUN`);
});
```

The SDK handles authentication, request serialization, error parsing, and type validation automatically.

## Python SDK

The Python SDK provides the same functionality for Python backends, data analysis pipelines, and automation scripts.

### When to Use the Python SDK

- Your backend is Python (Django, Flask, FastAPI)
- You are building data analysis or reporting tools
- Your DevOps scripts are in Python
- You are integrating with Python-based trading bots

### Implementation

```python
from merx import MerxClient

merx = MerxClient(api_key="your-api-key")

# Get prices
prices = merx.get_prices(
    energy_amount=65000,
    duration="1h"
)
print(f"Best: {prices.best.price_sun} SUN "
      f"via {prices.best.provider}")

# Place order
order = merx.create_order(
    energy_amount=65000,
    duration="1h",
    target_address="TAddress..."
)

# Estimate energy for a contract call
estimate = merx.estimate_energy(
    contract_address="TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    function_selector="transfer(address,uint256)",
    parameter=[recipient, amount],
    owner_address=sender
)
print(f"Energy needed: {estimate.energy_required}")
```

### Data Analysis Example

```python
import pandas as pd
from merx import MerxClient

merx = MerxClient(api_key="your-api-key")

# Pull price history for analysis
history = merx.get_price_history(
    energy_amount=65000,
    duration="1h",
    period="30d"
)

df = pd.DataFrame(history.prices)
print(f"Average price: {df['price_sun'].mean():.1f} SUN")
print(f"Min price: {df['price_sun'].min()} SUN")
print(f"Max price: {df['price_sun'].max()} SUN")
print(f"Std dev: {df['price_sun'].std():.1f} SUN")
```

## MCP Server: AI Agent Integration

The MERX MCP (Model Context Protocol) server allows AI agents to interact with the TRON energy market directly. This is the newest integration point and enables a fundamentally different interaction model.

### When to Use MCP

- You are building AI agents that manage TRON operations
- You want conversational energy management
- You are using Claude, ChatGPT, or other LLMs with tool-use capabilities
- You want to prototype energy strategies quickly through natural language

### How It Works

The MCP server at [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp) exposes MERX capabilities as tools that AI agents can call:

- `get_prices` -- Check current energy prices
- `create_order` -- Purchase energy
- `analyze_prices` -- Get price statistics
- `estimate_energy` -- Simulate transaction energy needs
- `check_resources` -- Check wallet energy balance

An AI agent can use these tools to answer questions like "What is the cheapest energy available right now?" or execute commands like "Buy 65,000 energy for my wallet at the best price."

### Integration

```json
{
  "mcpServers": {
    "merx": {
      "command": "npx",
      "args": ["@anthropic/merx-mcp"],
      "env": {
        "MERX_API_KEY": "your-api-key"
      }
    }
  }
}
```

## Choosing the Right Integration

### Decision Matrix

| Your Situation | Recommended Integration |
|---|---|
| Quick prototype, any language | REST API |
| Node.js backend | JS SDK |
| Python backend | Python SDK |
| Real-time price display | WebSocket |
| Event-driven architecture | Webhooks |
| Serverless (Lambda, Cloud Functions) | REST API + Webhooks |
| AI agent building | MCP Server |
| Go / Java / Ruby backend | REST API |
| Full-featured application | JS/Python SDK + Webhooks |

### Combining Integration Points

Most production systems use multiple integration methods:

- **SDK + Webhooks**: Use the SDK for outbound requests (prices, orders) and webhooks for inbound notifications (order filled, price alerts)
- **WebSocket + REST**: Use WebSocket for monitoring and REST for actions
- **REST + Webhooks**: The language-agnostic full-featured stack
- **MCP + SDK**: AI agent for strategy, SDK for execution

## Migration Path

If you are currently using a single provider's API, migrating to MERX is straightforward:

1. **Add the MERX SDK** to your project
2. **Replace price checks** with `merx.getPrices()` -- same data, more providers
3. **Replace order placement** with `merx.createOrder()` -- same flow, best price routing
4. **Add webhooks** for order notifications (if not already event-driven)
5. **Remove old provider code** once MERX integration is verified

The migration can be done incrementally. Run both systems in parallel during testing, comparing prices and order results before fully switching.

## Заключение

MERX is designed to fit into any technology stack through the integration method that matches your architecture. REST for universality, WebSocket for real-time, webhooks for events, SDKs for developer experience, and MCP for AI agents.

The choice is not exclusive -- combine integration points to match your needs. Start with the simplest option that solves your immediate problem, and expand as your requirements grow.

API documentation at [https://merx.exchange/docs](https://merx.exchange/docs). MCP server at [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp). Platform at [https://merx.exchange](https://merx.exchange).
