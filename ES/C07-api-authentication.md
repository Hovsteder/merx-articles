# Autenticacion de la API de MERX: claves, permisos y limites de velocidad

Every API integration starts with authentication. Get it right, and your automated energy trading runs smoothly around the clock. Get it wrong, and you are dealing with leaked credentials, unexplained 403 errors, or limite de velocidad bans that halt your production systems at the worst possible moment.

MERX provides two authentication methods, each designed for a different caso de uso. This article covers both in detail - how they work, when to use each, how to manage permissions granularly, and what limite de velocidads apply across the API surface.

## Two Authentication Methods

MERX supports two ways to authenticate API requests: clave de APIs and JWT tokens. They serve different purposes and are not interchangeable.

### API Key Authentication

clave de APIs are long-lived credentials designed for server-to-server communication. You create them through the API or admin panel, assign specific permissions, and include them in every request via the `X-API-Key` header.

```bash
curl https://merx.exchange/api/v1/prices \
  -H "X-API-Key: merx_live_k7x9m2p4..."
```

clave de APIs are the right choice when:

- Your backend service calls MERX on behalf of your users.
- You run scheduled jobs that create orders or check balances.
- You want fine-grained permission control (an order-creation key that cannot withdraw funds).
- You need credentials that work without an interactive login flow.

clave de APIs never expire on their own. They remain valid until you explicitly revoke them. This makes them convenient for long-running services but demands careful management.

### JWT Token Authentication

JWT tokens are short-lived credentials issued after a login. You authenticate with your email and password (or via OAuth), receive a JWT, and include it in requests via the `Authorization` header with a `Bearer` prefix.

```bash
curl https://merx.exchange/api/v1/balance \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

JWTs are the right choice when:

- A human user interacts with a frontend that calls the API.
- You want time-limited access that expires automatically.
- You are building a web or mobile application with a login flow.

JWT tokens issued by MERX expire after 24 hours. After expiration, the client must re-authenticate to get a new token. Refresh tokens extend sessions without requiring the user to log in again.

### Which One to Use

For programmatic integrations - payment bots, automated trading, backend services - use clave de APIs. They are simpler to manage, support granular permissions, and do not require a login flow.

For user-facing applications where a human logs in, use JWT tokens. They provide session-based access with automatic expiration, reducing the risk of credential leakage from client-side code.

You can use both in the same system. A common pattern: JWT authentication for your admin panel (human operators log in to manage settings), clave de API authentication for your backend services (automated order creation, balance monitoring).

## Creating and Managing API Keys

### Creating a Key

Create clave de APIs via the API itself or through the MERX web dashboard. The API endpoint is `POST /api/v1/keys`:

```bash
curl -X POST https://merx.exchange/api/v1/keys \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": "production-order-bot",
    "permissions": ["create_orders", "view_orders", "view_balance"]
  }'
```

The response includes the full clave de API. This is the only time the complete key is returned. Store it immediately in your secrets manager.

```json
{
  "id": "key_8f3k2m9x",
  "name": "production-order-bot",
  "key": "merx_live_k7x9m2p4q8r1s5t3u7v2w6x0y4z...",
  "permissions": ["create_orders", "view_orders", "view_balance"],
  "created_at": "2026-03-30T10:00:00Z"
}
```

Note: the key creation endpoint requires JWT authentication. You must log in first to create clave de APIs. This is a deliberate security decision - clave de APIs cannot create other clave de APIs.

### Using the JavaScript SDK

```javascript
import { MerxClient } from '@anthropic/merx-sdk';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
  baseUrl: 'https://merx.exchange/api/v1',
});

// The SDK automatically includes the X-API-Key header
const prices = await merx.prices.list();
const balance = await merx.account.getBalance();
```

### Using the Python SDK

```python
from merx_sdk import MerxClient

client = MerxClient(
    api_key="merx_live_k7x9m2p4...",
    base_url="https://merx.exchange/api/v1",
)

prices = client.get_prices()
balance = client.get_balance()
```

### Listing and Revoking Keys

List all active keys for your account:

```bash
curl https://merx.exchange/api/v1/keys \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

Revoke a key immediately:

```bash
curl -X DELETE https://merx.exchange/api/v1/keys/key_8f3k2m9x \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

Revocation is instant. Any request using the revoked key returns 401 immediately. Hay no grace period.

## Permission Types

MERX clave de APIs support granular permissions. When creating a key, you specify exactly which operations it can perform. A key without a required permission receives a 403 Forbidden response.

### Available Permissions

| Permission | Description | Typical Use Case |
|-----------|-------------|-----------------|
| `view_balance` | Read account balance and transaction history | Monitoring dashboards, alerting |
| `view_orders` | Read estado de la orden and order history | Order tracking, reporting |
| `create_orders` | Create new energy and bandwidth orders | Automated bot de tradings |
| `broadcast` | Submit signed transactions for broadcast | Custom transaction workflows |

### Permission Design Principles

**Least privilege.** Give each key only the permissions it needs. A monitoring dashboard does not need `create_orders`. A price display widget does not need `view_balance`.

**Separate keys for separate concerns.** Use one key for your order bot (`create_orders`, `view_orders`, `view_balance`) and a different key for your monitoring system (`view_balance`, `view_orders`). If the monitoring key leaks, the attacker cannot create orders.

**No withdrawal permission on clave de APIs.** Withdrawals require JWT authentication. This is intentional. An clave de API, even with full permissions, cannot withdraw funds from your account. This adds a layer of protection for the highest-risk operation.

### Example: Minimal Key for a Price Widget

A public-facing price widget only needs to fetch prices. It does not need authentication at all since the prices endpoint is public, but if you want to track usage:

```bash
curl -X POST https://merx.exchange/api/v1/keys \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "website-price-widget",
    "permissions": []
  }'
```

A key with an empty permissions array can access public endpoints (prices, provider list) while allowing you to track request volume and apply limite de velocidads per key.

### Example: Full Trading Bot Key

```bash
curl -X POST https://merx.exchange/api/v1/keys \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "trading-bot-prod",
    "permissions": ["create_orders", "view_orders", "view_balance"]
  }'
```

## Rate Limits

MERX applies limite de velocidads per endpoint category, not per key. All limite de velocidads are measured in requests per minute from the same authenticated identity (clave de API or JWT session).

### Rate Limit Table

| Endpoint Category | Rate Limit | Examples |
|------------------|-----------|---------|
| Price data | 300/min | `GET /prices`, `GET /prices/history` |
| Order creation | 10/min | `POST /orders` |
| Order queries | 60/min | `GET /orders`, `GET /orders/:id` |
| Withdrawals | 5/min | `POST /withdraw` |
| Account data | 60/min | `GET /balance`, `GET /keys` |
| Key management | 10/min | `POST /keys`, `DELETE /keys/:id` |

### Rate Limit Headers

Every response includes limite de velocidad information in HTTP headers:

```
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 287
X-RateLimit-Reset: 1711785660
```

- `X-RateLimit-Limit` - maximum requests allowed in the current window.
- `X-RateLimit-Remaining` - requests remaining before the limit is reached.
- `X-RateLimit-Reset` - Unix timestamp when the window resets.

### When You Hit the Limit

Exceeding the limite de velocidad returns HTTP 429 Too Many Requests with a `Retry-After` header indicating how many seconds to wait:

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Retry after 12 seconds.",
    "details": {
      "limit": 10,
      "window": "1m",
      "retry_after": 12
    }
  }
}
```

### Handling Rate Limits in Code

The SDKs handle limite de velocidads automatically with configurable retry behavior:

```javascript
const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
  maxRetries: 3,
  retryOnRateLimit: true, // Automatically wait and retry on 429
});
```

For raw HTTP clients, implement backoff based on the `Retry-After` header:

```python
import requests
import time

def merx_request(method, path, **kwargs):
    url = f"https://merx.exchange/api/v1{path}"
    headers = {"X-API-Key": API_KEY}

    for attempt in range(3):
        response = requests.request(method, url, headers=headers, **kwargs)

        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", 10))
            print(f"Rate limited. Waiting {retry_after}s...")
            time.sleep(retry_after)
            continue

        response.raise_for_status()
        return response.json()

    raise Exception("Rate limit retries exhausted")
```

## Security Best Practices

### Store Keys in Environment Variables

Never hardcode clave de APIs in source code. Use environment variables or a secrets manager:

```bash
# .env file (never commit this)
MERX_API_KEY=merx_live_k7x9m2p4q8r1s5t3u7v2w6x0y4z...
```

```javascript
// Load from environment
const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
});
```

### Rotate Keys Periodically

Create a new key, update your services to use it, then revoke the old key. MERX supports multiple active keys simultaneously, so you can rotate without downtime:

1. Create a new key with the same permissions.
2. Deploy the new key to your services.
3. Verify the new key works in production.
4. Revoke the old key.

### Monitor Key Usage

Review your clave de API list periodically. Revoke keys that are no longer in use. Each key has a `last_used_at` timestamp - if a key has not been used in months, it is a candidate for revocation.

### Never Expose Keys in Client-Side Code

clave de APIs should never appear in JavaScript running in a browser, mobile app bundles, or any code that end users can inspect. If you need to call MERX from a frontend, proxy the requests through your backend, which holds the clave de API server-side.

```
Browser -> Your Backend (holds API key) -> MERX API
```

### Use Separate Keys for Separate Environments

Maintain distinct keys for development, staging, and production. If a development key leaks, it cannot affect production. MERX does not currently have environment-scoped keys, but naming conventions help:

```
dev-price-monitor
staging-order-bot
prod-order-bot
prod-balance-alerter
```

### Audit on Suspicion

If you suspect a key has been compromised, revoke it immediately and create a replacement. Check your recent order and withdrawal history for unauthorized activity. MERX logs all clave de API usage with IP addresses, which can help identify the source of unauthorized access.

## Putting It All Together

A complete authentication setup for a production system typically looks like this:

1. Log in via the web dashboard to get a JWT session.
2. Create separate clave de APIs for each service: order bot, monitoring, reporting.
3. Assign minimal permissions to each key.
4. Store keys in your secrets manager or environment variables.
5. Implement limite de velocidad handling with automatic retry.
6. Set up key rotation on a regular schedule (quarterly is reasonable).
7. Monitor key usage and revoke unused keys.

Authentication is the foundation of every MERX integration. Spending an hour getting it right saves days of debugging and security incidents down the line.

- Plataforma y panel: [merx.exchange](https://merx.exchange)
- Full API reference: [merx.exchange/docs](https://merx.exchange/docs)
- SDK de JavaScript: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- SDK de Python: [github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)
