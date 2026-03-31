# MERX REST API: TRON Energy Ticareti Icin 46 Endpoint

The MERX REST API provides 46 endpoints that cover the complete lifecycle of TRON energy and bandwidth trading - from real-time price discovery across eight providers to order execution, account management, on-chain queries, and automated standing orders. This article walks through the API architecture, authentication model, endpoint groups, rate limits, error handling, and practical code examples in curl, JavaScript, and Python.

## Why a Unified API Matters

The TRON energy market is fragmented across multiple providers, each with its own API format, authentication scheme, and pricing model. A developer who wants the best price on a USDT transfer has to integrate with every provider individually, handle failover logic, and monitor prices continuously.

MERX consolidates all of that into a single REST API. One API key, one authentication header, one error format, one set of SDKs. The platform polls all connected providers every 30 seconds, routes orders to the cheapest available source, and verifies delegations on-chain.

The API is versioned at `/api/v1/` and will maintain backward compatibility. All endpoints return JSON in a consistent envelope format.

## Kimlik Dogrulama

MERX uses API key authentication. Every authenticated request must include the `X-API-Key` header.

API keys are created in the MERX dashboard at [merx.exchange](https://merx.exchange) or programmatically via the `/api/v1/keys` endpoint. Each key has a permission set that controls what operations it can perform.

```bash
curl -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/balance
```

Keys follow the format `sk_live_` followed by 64 hex characters. The raw key is shown exactly once at creation time. MERX stores only the bcrypt hash, so lost keys cannot be recovered - they must be revoked and replaced.

### Key Permissions

When creating an API key, you assign one or more permissions:

| Permission       | Grants access to                        |
|------------------|-----------------------------------------|
| `create_orders`  | POST /orders, POST /ensure              |
| `view_orders`    | GET /orders, GET /orders/:id            |
| `view_balance`   | GET /balance, GET /history              |
| `broadcast`      | POST /chain/broadcast                   |

This allows you to create read-only keys for monitoring dashboards and restricted keys for automated trading systems.

## Response Envelope

Every response follows the same structure:

```json
{
  "data": { ... }
}
```

On error:

```json
{
  "error": {
    "code": "INSUFFICIENT_FUNDS",
    "message": "Account balance too low for the requested operation",
    "details": { "required": 150000000, "available": 42000000 }
  }
}
```

Error codes are machine-readable strings. The `details` field is optional and provides context for debugging.

## Endpoint Groups

The 46 endpoints are organized into nine groups. Here is the complete map.

### Prices (6 endpoints)

These endpoints are public - no API key required.

| Method | Path                        | Description                              |
|--------|-----------------------------|------------------------------------------|
| GET    | /api/v1/prices              | Current prices from all providers        |
| GET    | /api/v1/prices/best         | Cheapest provider for a resource type    |
| GET    | /api/v1/prices/history      | Historical price data                    |
| GET    | /api/v1/prices/stats        | Aggregate market statistics              |
| GET    | /api/v1/prices/analysis     | Trend analysis and buy recommendation    |
| GET    | /api/v1/orders/preview      | Cost preview before placing an order     |

### Orders (3 endpoints)

| Method | Path                        | Description                              |
|--------|-----------------------------|------------------------------------------|
| POST   | /api/v1/orders              | Create a new energy or bandwidth order   |
| GET    | /api/v1/orders              | List orders with pagination and filters  |
| GET    | /api/v1/orders/:id          | Get order details with fill breakdown    |

### Account (7 endpoints)

| Method | Path                        | Description                              |
|--------|-----------------------------|------------------------------------------|
| GET    | /api/v1/balance             | Current TRX and USDT balances            |
| GET    | /api/v1/deposit/info        | Deposit address and memo                 |
| POST   | /api/v1/deposit/prepare     | Prepare a deposit transaction            |
| POST   | /api/v1/deposit/submit      | Submit a deposit proof                   |
| POST   | /api/v1/withdraw            | Withdraw TRX or USDT                     |
| GET    | /api/v1/history             | Order execution history                  |
| GET    | /api/v1/history/summary     | Aggregate account statistics             |

### API Keys (3 endpoints)

| Method | Path                        | Description                              |
|--------|-----------------------------|------------------------------------------|
| GET    | /api/v1/keys                | List all API keys                        |
| POST   | /api/v1/keys                | Create a new API key                     |
| DELETE | /api/v1/keys/:id            | Revoke an API key                        |

### Kimlik Dogrulama (2 endpoints)

| Method | Path                        | Description                              |
|--------|-----------------------------|------------------------------------------|
| POST   | /api/v1/auth/register       | Create a new account                     |
| POST   | /api/v1/auth/login          | Authenticate and receive a JWT token     |

### Estimation (2 endpoints)

| Method | Path                        | Description                              |
|--------|-----------------------------|------------------------------------------|
| POST   | /api/v1/estimate            | Estimate energy and cost for a transaction|
| POST   | /api/v1/ensure              | Ensure minimum resources on an address   |

### Webhooks (3 endpoints)

| Method | Path                        | Description                              |
|--------|-----------------------------|------------------------------------------|
| POST   | /api/v1/webhooks            | Create a webhook subscription            |
| GET    | /api/v1/webhooks            | List webhook subscriptions               |
| DELETE | /api/v1/webhooks/:id        | Delete a webhook                         |

### Standing Orders and Monitors (7 endpoints)

| Method | Path                             | Description                           |
|--------|----------------------------------|---------------------------------------|
| POST   | /api/v1/standing-orders          | Create a standing order               |
| GET    | /api/v1/standing-orders          | List standing orders                  |
| GET    | /api/v1/standing-orders/:id      | Get standing order details            |
| DELETE | /api/v1/standing-orders/:id      | Cancel a standing order               |
| POST   | /api/v1/monitors                 | Create a resource monitor             |
| GET    | /api/v1/monitors                 | List active monitors                  |
| DELETE | /api/v1/monitors/:id             | Cancel a monitor                      |

### Chain Proxy (10 endpoints)

These endpoints proxy TRON network queries through MERX, eliminating the need for clients to call TronGrid directly.

| Method | Path                             | Description                           |
|--------|----------------------------------|---------------------------------------|
| GET    | /api/v1/chain/account/:address   | Account info and resources            |
| GET    | /api/v1/chain/balance/:address   | TRX balance                           |
| GET    | /api/v1/chain/resources/:address | Energy and bandwidth breakdown        |
| GET    | /api/v1/chain/transaction/:txid  | Transaction details                   |
| GET    | /api/v1/chain/block/:number      | Block by number (or latest)           |
| GET    | /api/v1/chain/parameters         | Chain parameters                      |
| GET    | /api/v1/chain/history/:address   | Address transaction history           |
| POST   | /api/v1/chain/read-contract      | Call a constant contract function     |
| POST   | /api/v1/chain/broadcast          | Broadcast a signed transaction        |
| GET    | /api/v1/address/:addr/resources  | Address resource summary              |

### x402 Pay-Per-Use (3 endpoints)

| Method | Path                        | Description                              |
|--------|-----------------------------|------------------------------------------|
| POST   | /api/v1/x402/invoice        | Create a payment invoice                 |
| GET    | /api/v1/x402/invoice/:id    | Check invoice status                     |
| POST   | /api/v1/x402/verify         | Verify payment and execute order         |

## Key Endpoints in Detail

### GET /api/v1/prices

Returns current pricing from all active providers. No authentication required. This is the endpoint you call to see the full market at a glance.

```bash
curl https://merx.exchange/api/v1/prices
```

Response (abbreviated):

```json
{
  "data": [
    {
      "provider": "sohu",
      "is_market": false,
      "energy_prices": [
        { "duration_sec": 3600, "price_sun": 24 },
        { "duration_sec": 86400, "price_sun": 30 }
      ],
      "bandwidth_prices": [],
      "available_energy": 5000000,
      "available_bandwidth": 0,
      "fetched_at": 1743292800
    }
  ]
}
```

Each provider entry includes the price tiers (by duration), available capacity, and the timestamp of the last successful poll.

### POST /api/v1/orders

Creates an energy or bandwidth order. The platform matches the order against available providers and routes to the cheapest one that can fulfill it.

```bash
curl -X POST https://merx.exchange/api/v1/orders \
  -H "X-API-Key: sk_live_your_key_here" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: unique-request-id-123" \
  -d '{
    "resource_type": "ENERGY",
    "order_type": "MARKET",
    "amount": 65000,
    "target_address": "TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    "duration_sec": 3600
  }'
```

Response:

```json
{
  "data": {
    "id": "ord_a1b2c3d4",
    "status": "PENDING",
    "created_at": "2026-03-30T12:00:00.000Z"
  }
}
```

The `Idempotency-Key` header prevents duplicate orders if the same request is retried. If the key has been seen before, the API returns the original order instead of creating a new one.

Order types:
- `MARKET` - execute immediately at best available price
- `LIMIT` - execute only if price is at or below `max_price_sun`
- `PERIODIC` - recurring order on a schedule
- `BROADCAST` - broadcast a pre-signed delegation transaction

### GET /api/v1/orders/:id

Returns the order with its fill details - which providers fulfilled the order, at what price, and the on-chain transaction IDs.

```bash
curl -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/orders/ord_a1b2c3d4
```

```json
{
  "data": {
    "id": "ord_a1b2c3d4",
    "resource_type": "ENERGY",
    "order_type": "MARKET",
    "status": "FILLED",
    "amount": 65000,
    "target_address": "TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    "duration_sec": 3600,
    "total_cost_sun": 1560000,
    "fills": [
      {
        "provider": "sohu",
        "amount": 65000,
        "price_sun": 24,
        "cost_sun": 1560000,
        "delegation_tx": "abc123def456...",
        "verified": true,
        "tronscan_url": "https://tronscan.org/#/transaction/abc123def456..."
      }
    ]
  }
}
```

### POST /api/v1/estimate

Estimates the energy and bandwidth required for a TRON operation, then compares the rental cost against the TRX burn cost.

```bash
curl -X POST https://merx.exchange/api/v1/estimate \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "trc20_transfer",
    "to_address": "TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    "amount": "1000000"
  }'
```

```json
{
  "data": {
    "energy_required": 65000,
    "bandwidth_required": 345,
    "rental_cost": {
      "energy": {
        "best_price_sun": 24,
        "best_provider": "sohu",
        "cost_trx": "1.560"
      }
    },
    "total_rental_trx": "1.905",
    "total_burn_trx": "27.645",
    "savings_percent": 93.1
  }
}
```

This endpoint is useful for showing users exactly how much they save by renting energy through MERX versus burning TRX.

## Rate Limits

Rate limits are applied per IP address using sliding windows.

| Endpoint group      | Limit              | Window  |
|---------------------|--------------------|---------|
| Prices (public)     | 300 requests       | 1 min   |
| Default (general)   | 100 requests       | 1 min   |
| Balance             | 60 requests        | 1 min   |
| History             | 60 requests        | 1 min   |
| Orders              | 10 requests        | 1 min   |
| Withdrawals         | 5 requests         | 1 min   |
| Broadcast           | 20 requests        | 1 min   |
| Registration        | 5 requests         | 1 hour  |

When a rate limit is exceeded, the API returns HTTP 429 with the standard error format:

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded"
  }
}
```

Rate limit headers (`RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`) are included in all responses following the IETF draft standard.

## Error Codes

| Code                   | HTTP | Description                                        |
|------------------------|------|----------------------------------------------------|
| `UNAUTHORIZED`         | 401  | Invalid or missing API key                         |
| `RATE_LIMITED`         | 429  | Too many requests                                  |
| `VALIDATION_ERROR`    | 400  | Request body or parameters failed validation       |
| `INVALID_ADDRESS`     | 400  | Not a valid TRON address                           |
| `INSUFFICIENT_FUNDS`  | 400  | Account balance too low                            |
| `BELOW_MINIMUM_ORDER` | 400  | Order amount below provider minimum                |
| `DUPLICATE_REQUEST`   | 409  | Idempotency key already used                       |
| `ORDER_NOT_FOUND`     | 404  | Order or resource does not exist                   |
| `PROVIDER_UNAVAILABLE`| 404  | No provider can fulfill the request                |
| `INTERNAL_ERROR`      | 500  | Server-side error                                  |

## Hizli Baslangic with SDKs

While the REST API can be called directly with any HTTP client, MERX provides official SDKs for JavaScript and Python that handle authentication, error parsing, and type safety.

### JavaScript / TypeScript

```bash
npm install merx-sdk
```

```javascript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({ apiKey: 'sk_live_your_key_here' })

// Get all prices
const prices = await merx.prices.list()

// Create an order
const order = await merx.orders.create({
  resource_type: 'ENERGY',
  amount: 65000,
  target_address: 'TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb',
  duration_sec: 3600,
})

// Check order status
const details = await merx.orders.get(order.id)
```

### Python

```bash
pip install merx-sdk
```

```python
from merx import MerxClient

client = MerxClient(api_key="sk_live_your_key_here")

# Get all prices
prices = client.prices.list()

# Create an order
order = client.orders.create(
    resource_type="ENERGY",
    amount=65000,
    target_address="TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    duration_sec=3600,
)

# Check order status
details = client.orders.get(order.id)
```

## WebSocket for Real-Time Data

In addition to the REST API, MERX provides a WebSocket endpoint at `wss://merx.exchange/ws` for real-time price updates. Price changes are pushed to connected clients as they happen, with updates arriving every 30 seconds per provider.

The WebSocket connection supports provider filtering - subscribe only to the providers you care about, and ignore the rest.

## Standing Orders

Standing orders automate energy purchases based on triggers. You can set a price threshold, a schedule, or a balance condition, and the platform executes orders automatically within your specified budget.

Trigger types include `price_below`, `price_above`, `schedule`, `balance_below`, and `provider_available`. Action types include `buy_resource`, `ensure_resources`, `deposit_trx`, and `notify_only`.

This makes MERX suitable for fully automated infrastructure management - set your rules once, and the platform handles execution.

## Sirada Ne Var

The MERX API is designed for developers and businesses that need reliable, cost-effective access to TRON network resources. Whether you are building a payment processor, a DeFi application, or an exchange, the API provides the building blocks to manage energy and bandwidth programmatically.

Tam API dokumantasyonu su adreste mevcuttur: [merx.exchange/docs](https://merx.exchange/docs). The JavaScript SDK is on [GitHub](https://github.com/Hovsteder/merx-sdk-js) and [npm](https://www.npmjs.com/package/merx-sdk). The Python SDK is on [PyPI](https://pypi.org/project/merx-sdk/).

---

**Baglantilar:**
- Platform: [merx.exchange](https://merx.exchange)
- Dokumantasyon: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- Python SDK: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- MCP Sunucusu: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp) | [npm](https://www.npmjs.com/package/merx-mcp)
