# Webhook Integration: Get Notified When Energy Orders Fill

MERX webhooks deliver real-time HTTP notifications when events occur on your account - orders filling, orders failing, deposits arriving, withdrawals completing. This article covers all four event types, the payload format, HMAC-SHA256 signature verification using the `X-Merx-Signature` header, the retry policy with exponential backoff, auto-deactivation after repeated failures, and complete server implementations in Express.js and Flask with signature verification.

## Why Webhooks Instead of Polling

The alternative to webhooks is polling the `/api/v1/orders/:id` endpoint in a loop, waiting for the status to change from `PENDING` to `FILLED`. This works for simple cases but has clear drawbacks:

- Wasted requests. Most polls return the same unchanged status.
- Latency. Your application only discovers a state change at the next poll interval.
- Rate limits. With a 10-request-per-minute limit on the orders endpoint, aggressive polling quickly hits the ceiling.
- Complexity. Polling logic needs retry handling, timeout management, and state tracking.

Webhooks invert the model. Instead of asking MERX "has anything changed?", MERX tells you the moment something happens. Your server receives an HTTP POST with the full event payload, processes it, and moves on. No polling loops, no wasted requests, no artificial delays.

## Creating a Webhook

You can create webhooks via the REST API or the SDK.

### REST API

```bash
curl -X POST https://merx.exchange/api/v1/webhooks \
  -H "X-API-Key: sk_live_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-server.com/webhooks/merx",
    "events": ["order.filled", "order.failed", "deposit.received", "withdrawal.completed"]
  }'
```

Response:

```json
{
  "data": {
    "id": "wh_abc123",
    "url": "https://your-server.com/webhooks/merx",
    "events": ["order.filled", "order.failed", "deposit.received", "withdrawal.completed"],
    "secret": "a1b2c3d4e5f6...64-hex-characters...9876543210",
    "is_active": true,
    "created_at": "2026-03-30T12:00:00.000Z"
  }
}
```

The `secret` field is a 64-character hex string generated from 32 random bytes. It is returned only in the creation response. Store it securely - you will need it to verify incoming webhook signatures.

### JavaScript SDK

```javascript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY })

const webhook = await merx.webhooks.create({
  url: 'https://your-server.com/webhooks/merx',
  events: ['order.filled', 'order.failed', 'deposit.received'],
})

console.log(`Webhook ID: ${webhook.id}`)
console.log(`Secret: ${webhook.secret}`)  // Store this securely
```

### Python SDK

```python
from merx import MerxClient

client = MerxClient(api_key="sk_live_your_key_here")

webhook = client.webhooks.create(
    url="https://your-server.com/webhooks/merx",
    events=["order.filled", "order.failed", "deposit.received"],
)

print(f"Webhook ID: {webhook.id}")
print(f"Secret: {webhook.secret}")  # Store this securely
```

## Event Types

MERX supports four webhook event types. When creating a webhook, you choose which events to subscribe to. You can subscribe to all four or just the ones you need.

### order.filled

Sent when an order has been completely fulfilled. All provider delegations have been confirmed on-chain.

```json
{
  "event": "order.filled",
  "timestamp": "2026-03-30T12:05:00.000Z",
  "data": {
    "order_id": "ord_abc123",
    "resource_type": "ENERGY",
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

This is the most important event for automated systems. When you receive it, the energy has been delegated and the target address can proceed with its TRON transactions.

### order.failed

Sent when an order could not be fulfilled. This can happen when all providers are unavailable, capacity is exhausted, or a provider-side error occurs.

```json
{
  "event": "order.failed",
  "timestamp": "2026-03-30T12:05:00.000Z",
  "data": {
    "order_id": "ord_def456",
    "resource_type": "ENERGY",
    "amount": 500000,
    "target_address": "TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    "reason": "No provider could fulfill the order within the specified parameters",
    "refunded": true,
    "refund_amount_sun": 0
  }
}
```

When an order fails, any reserved balance is refunded. The `refunded` field confirms this, and `refund_amount_sun` shows the amount returned if payment was already deducted.

### deposit.received

Sent when a deposit to your MERX account has been detected and credited.

```json
{
  "event": "deposit.received",
  "timestamp": "2026-03-30T12:10:00.000Z",
  "data": {
    "deposit_id": "dep_ghi789",
    "amount_sun": 100000000,
    "amount_trx": "100.000000",
    "currency": "TRX",
    "tx_id": "789abc...",
    "new_balance_trx": 250.5
  }
}
```

### withdrawal.completed

Sent when a withdrawal request has been processed and the on-chain transaction is confirmed.

```json
{
  "event": "withdrawal.completed",
  "timestamp": "2026-03-30T12:15:00.000Z",
  "data": {
    "withdrawal_id": "wdr_jkl012",
    "amount": 50,
    "currency": "TRX",
    "address": "TExternalAddress...",
    "tx_id": "012def...",
    "tronscan_url": "https://tronscan.org/#/transaction/012def..."
  }
}
```

## Signature Verification

Every webhook delivery includes an `X-Merx-Signature` header containing an HMAC-SHA256 signature of the request body, computed using your webhook secret as the key.

The verification process:

1. Read the raw request body (before JSON parsing).
2. Compute HMAC-SHA256 of the raw body using your stored webhook secret.
3. Compare the computed signature with the `X-Merx-Signature` header value.
4. If they match, the request is authentic. If not, reject it.

This protects against:
- Forged requests from third parties who do not know your secret.
- Tampered payloads where the body has been modified in transit.
- Replay attacks (combined with timestamp validation).

Always use a constant-time comparison function when checking signatures. Standard string equality (`===` or `==`) is vulnerable to timing attacks.

## Express.js Webhook Handler

Here is a complete Express.js server that receives and verifies MERX webhooks:

```javascript
import express from 'express'
import crypto from 'node:crypto'

const app = express()
const WEBHOOK_SECRET = process.env.MERX_WEBHOOK_SECRET

// Use raw body for signature verification
app.post('/webhooks/merx', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-merx-signature']
  if (!signature || !WEBHOOK_SECRET) {
    res.status(401).json({ error: 'Missing signature or secret' })
    return
  }

  // Compute expected signature
  const expected = crypto
    .createHmac('sha256', WEBHOOK_SECRET)
    .update(req.body)
    .digest('hex')

  // Constant-time comparison
  if (!crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected))) {
    console.warn('Webhook signature mismatch')
    res.status(401).json({ error: 'Invalid signature' })
    return
  }

  // Signature verified - parse and handle the event
  const event = JSON.parse(req.body.toString())
  console.log(`Received event: ${event.event}`)

  switch (event.event) {
    case 'order.filled':
      handleOrderFilled(event.data)
      break
    case 'order.failed':
      handleOrderFailed(event.data)
      break
    case 'deposit.received':
      handleDepositReceived(event.data)
      break
    case 'withdrawal.completed':
      handleWithdrawalCompleted(event.data)
      break
    default:
      console.warn(`Unknown event type: ${event.event}`)
  }

  // Always respond with 200 quickly to acknowledge receipt
  res.status(200).json({ received: true })
})

function handleOrderFilled(data) {
  console.log(`Order ${data.order_id} filled`)
  console.log(`  Amount: ${data.amount} ${data.resource_type}`)
  console.log(`  Cost: ${(data.total_cost_sun / 1_000_000).toFixed(3)} TRX`)
  console.log(`  Fills: ${data.fills.length}`)

  for (const fill of data.fills) {
    console.log(`  ${fill.provider}: ${fill.amount} at ${fill.price_sun} SUN`)
    if (fill.tronscan_url) {
      console.log(`    TX: ${fill.tronscan_url}`)
    }
  }

  // Your business logic here:
  // - Mark the transaction as ready to proceed
  // - Trigger the USDT transfer
  // - Update your database
}

function handleOrderFailed(data) {
  console.log(`Order ${data.order_id} failed: ${data.reason}`)
  if (data.refunded) {
    console.log(`  Refunded: ${data.refund_amount_sun} SUN`)
  }
  // Alert your operations team
}

function handleDepositReceived(data) {
  console.log(`Deposit received: ${data.amount_trx} ${data.currency}`)
  console.log(`  New balance: ${data.new_balance_trx} TRX`)
  // Update local balance cache
}

function handleWithdrawalCompleted(data) {
  console.log(`Withdrawal ${data.withdrawal_id} completed`)
  console.log(`  ${data.amount} ${data.currency} to ${data.address}`)
  // Update withdrawal status in your system
}

app.listen(3000, () => {
  console.log('Webhook server listening on port 3000')
})
```

Key implementation details:

- Use `express.raw({ type: 'application/json' })` instead of `express.json()` for the webhook route. You need the raw bytes for signature computation.
- Use `crypto.timingSafeEqual()` for constant-time signature comparison.
- Respond with HTTP 200 as quickly as possible. Do heavy processing asynchronously after acknowledging receipt.

## Flask Webhook Handler

Here is the equivalent implementation in Python using Flask:

```python
import hashlib
import hmac
import json
import os

from flask import Flask, request, jsonify

app = Flask(__name__)
WEBHOOK_SECRET = os.environ.get("MERX_WEBHOOK_SECRET", "")


def verify_signature(payload: bytes, signature: str, secret: str) -> bool:
    """Verify HMAC-SHA256 signature using constant-time comparison."""
    expected = hmac.new(
        secret.encode("utf-8"),
        payload,
        hashlib.sha256,
    ).hexdigest()
    return hmac.compare_digest(signature, expected)


@app.route("/webhooks/merx", methods=["POST"])
def handle_webhook():
    signature = request.headers.get("X-Merx-Signature", "")
    raw_body = request.get_data()

    if not signature or not WEBHOOK_SECRET:
        return jsonify({"error": "Missing signature or secret"}), 401

    if not verify_signature(raw_body, signature, WEBHOOK_SECRET):
        app.logger.warning("Webhook signature mismatch")
        return jsonify({"error": "Invalid signature"}), 401

    event = json.loads(raw_body)
    event_type = event.get("event", "")
    data = event.get("data", {})

    app.logger.info(f"Received event: {event_type}")

    if event_type == "order.filled":
        handle_order_filled(data)
    elif event_type == "order.failed":
        handle_order_failed(data)
    elif event_type == "deposit.received":
        handle_deposit_received(data)
    elif event_type == "withdrawal.completed":
        handle_withdrawal_completed(data)
    else:
        app.logger.warning(f"Unknown event type: {event_type}")

    return jsonify({"received": True}), 200


def handle_order_filled(data):
    order_id = data["order_id"]
    amount = data["amount"]
    resource = data["resource_type"]
    cost_trx = data["total_cost_sun"] / 1_000_000

    app.logger.info(f"Order {order_id} filled: {amount} {resource}, {cost_trx:.3f} TRX")

    for fill in data.get("fills", []):
        app.logger.info(
            f"  {fill['provider']}: {fill['amount']} at {fill['price_sun']} SUN"
        )

    # Your business logic here


def handle_order_failed(data):
    app.logger.warning(
        f"Order {data['order_id']} failed: {data.get('reason', 'unknown')}"
    )


def handle_deposit_received(data):
    app.logger.info(
        f"Deposit: {data['amount_trx']} {data['currency']}, "
        f"new balance: {data.get('new_balance_trx', 'N/A')} TRX"
    )


def handle_withdrawal_completed(data):
    app.logger.info(
        f"Withdrawal {data['withdrawal_id']}: "
        f"{data['amount']} {data['currency']} to {data['address']}"
    )


if __name__ == "__main__":
    app.run(port=3000)
```

Key Python-specific details:

- Use `request.get_data()` to get the raw request body as bytes.
- Use `hmac.compare_digest()` for constant-time string comparison. Python's `==` operator is not constant-time.
- Use `hmac.new()` with `hashlib.sha256` to compute the HMAC.

## Retry Policy

MERX retries failed webhook deliveries using an exponential backoff schedule:

| Attempt | Delay after failure |
|---------|---------------------|
| 1       | Immediate           |
| 2       | 30 seconds          |
| 3       | 5 minutes           |

A delivery is considered failed if:
- Your server does not respond within 10 seconds.
- Your server returns an HTTP status code outside the 2xx range.
- The connection cannot be established (DNS failure, connection refused, TLS error).

After 3 failed attempts for a single event, the event is dropped. No further retries are made for that specific delivery.

Your webhook handler should:
- Respond with HTTP 200 within a few seconds. Do heavy processing asynchronously.
- Be idempotent. The same event may be delivered more than once in edge cases.
- Log the event payload for debugging if processing fails.

## Auto-Deactivation

If a webhook endpoint consistently fails, MERX automatically deactivates it to prevent wasting resources on a dead endpoint.

The deactivation threshold is based on consecutive failures across multiple events. If your endpoint fails to accept deliveries repeatedly, the webhook's `is_active` flag is set to `false`.

When a webhook is deactivated:
- No further events are sent to that URL.
- The webhook still appears in your list with `is_active: false`.
- You can fix the endpoint issue and create a new webhook.

Monitor your webhook status periodically:

```javascript
const webhooks = await merx.webhooks.list()
for (const wh of webhooks) {
  if (!wh.is_active) {
    console.warn(`Webhook ${wh.id} (${wh.url}) is deactivated`)
  }
}
```

```python
webhooks = client.webhooks.list()
for wh in webhooks:
    if not wh.is_active:
        print(f"Webhook {wh.id} ({wh.url}) is deactivated")
```

## Managing Webhooks

### Listing Webhooks

```bash
curl -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/webhooks
```

### Deleting a Webhook

```bash
curl -X DELETE -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/webhooks/wh_abc123
```

Note that the webhook secret cannot be retrieved after creation. If you lose the secret, delete the webhook and create a new one.

## Testing Webhooks Locally

During development, your webhook URL needs to be publicly accessible. Tools like ngrok can expose a local server:

```bash
# Terminal 1: Start your webhook server
node webhook-server.js

# Terminal 2: Expose it publicly
ngrok http 3000
```

Use the ngrok URL (e.g., `https://abc123.ngrok.io/webhooks/merx`) when creating the webhook. Once you verify everything works, replace it with your production URL.

## Best Practices

1. Always verify signatures. Never trust webhook payloads without checking the `X-Merx-Signature` header.

2. Respond quickly. Return HTTP 200 within 2-3 seconds. Queue heavy processing for background workers.

3. Be idempotent. Use the `order_id` or `deposit_id` as a deduplication key. If you receive the same event twice, the second processing should be a no-op.

4. Store the raw payload. Log the full JSON body before processing. If your handler has a bug, you can replay the events from logs.

5. Monitor webhook health. Check `is_active` status regularly. Set up alerts if webhooks are deactivated.

6. Use HTTPS. MERX requires webhook URLs to use HTTPS. Self-signed certificates are not accepted.

7. Subscribe selectively. Only subscribe to the events you actually handle. Unnecessary events waste bandwidth and processing time.

## Resources

- Platform: [merx.exchange](https://merx.exchange)
- Documentation: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- Python SDK: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- MCP Server: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
