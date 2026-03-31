# Ключи идемпотентности: безопасные повторные попытки для ордеров на energy TRON

Network requests fail. Connections time out. Load balancers drop packets. Mobile clients lose signal mid-request. In any distributed system, the question is not whether failures will happen but how you handle them when they do.

For most API calls, the answer is straightforward: retry the request. But when the request involves money - creating an order that debits your balance, or initiating a withdrawal that moves funds on-chain - a naive retry can be catastrophic. You send the request, the server processes it, but the response never reaches you. You retry. The server processes it again. You just paid twice for the same order, or worse, triggered two on-chain withdrawals.

This is the problem that idempotency keys solve. MERX implements idempotency on every financial endpoint, making it safe to retry any request without risk of duplicate execution.

## What Idempotency Means in Practice

An operation is idempotent if performing it multiple times produces the same result as performing it once. HTTP GET is naturally idempotent - fetching the same URL ten times returns the same data. HTTP DELETE is idempotent by convention - deleting an already-deleted resource is a no-op.

HTTP POST is not idempotent. Each POST to `/api/v1/orders` creates a new order. Send it three times, get three orders, pay three times.

Idempotency keys make POST requests behave idempotently. The client generates a unique identifier and sends it with the request. The server uses this key to detect duplicates. If the same key arrives again, the server returns the result from the first execution instead of processing the request again.

The distinction matters: the server does not simply reject duplicates. It returns the original response. From the client's perspective, the retry behaves exactly like a successful first attempt. This is critical for automation - your code does not need to distinguish between "first successful call" and "successful retry."

## Почему это важно for TRON Energy Trading

TRON energy orders involve real money moving through real systems. When you create an order through MERX, several things happen in sequence:

1. Your account balance is debited.
2. The order is routed to the cheapest available provider.
3. The provider delegates energy on-chain to your target address.
4. The order status is updated and webhooks fire.

If the connection drops after step 1 but before you receive confirmation, you have no way to know whether the order was created. Without idempotency, your options are bad:

- **Do not retry.** You might have a pending order you do not know about, or you might have lost a request that never reached the server. You have to poll the orders list to find out, adding complexity.
- **Retry blindly.** If the first request did go through, you now have two orders and two balance debits. For an automated system processing hundreds of orders per day, this adds up fast.
- **Retry with idempotency key.** The server recognizes the duplicate, returns the existing order, and your code continues as normal. No double-spend. No manual reconciliation.

The same logic applies to withdrawals. A duplicate withdrawal request could move funds on-chain twice. With idempotency keys, the retry returns the existing withdrawal record.

## How MERX Implements Idempotency

MERX supports the `Idempotency-Key` header on two endpoints:

- `POST /api/v1/orders` - energy and bandwidth order creation
- `POST /api/v1/withdraw` - balance withdrawals

The implementation follows a straightforward lifecycle.

### Key Format and Generation

The idempotency key is a client-generated string, up to 64 characters. MERX does not enforce a specific format, but best practice is to use UUIDs (v4) or structured identifiers that encode context:

```
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

Or with business context:

```
Idempotency-Key: order-user42-2026-03-30-batch7-item3
```

The key must be unique per distinct operation. Reusing a key with different request parameters (different amount, different target address) returns an error rather than silently ignoring the new parameters.

### Server-Side Behavior

When MERX receives a request with an `Idempotency-Key` header, the following logic runs:

1. **First request with this key.** The server processes the request normally - validates parameters, debits balance, creates the order. It stores the key, the request hash, and the response. Returns the response with HTTP 201.

2. **Duplicate request with the same key and same parameters.** The server skips all processing and returns the stored response from the first execution. The response is identical, including the same order ID, status, and timestamps. Returns HTTP 200 (not 201) so the client can distinguish if needed.

3. **Duplicate request with the same key but different parameters.** The server returns HTTP 409 Conflict. This prevents a subtle bug where a key collision causes an unrelated order to be returned.

4. **Request while the first is still processing.** The server returns HTTP 409 with a message indicating the original request is still in progress. This handles the race condition where a retry arrives before the first request finishes.

### Key Expiration

Idempotency keys are stored for 24 hours. After that, the same key can be reused for a new request. In practice, retries happen within seconds or minutes, not days. The 24-hour window is generous enough to cover any realistic retry scenario while preventing unbounded storage growth.

## Code Examples

### JavaScript SDK - Retry-Safe Order Creation

The MERX JavaScript SDK handles idempotency keys automatically when you pass the `idempotencyKey` option. Here is a complete example with retry logic:

```javascript
import { MerxClient } from '@anthropic/merx-sdk';
import { randomUUID } from 'crypto';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
  baseUrl: 'https://merx.exchange/api/v1',
});

async function createOrderSafe(params, maxRetries = 3) {
  const idempotencyKey = randomUUID();

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const order = await merx.orders.create(
        {
          energy_amount: params.energyAmount,
          target_address: params.targetAddress,
          duration_hours: params.durationHours,
        },
        {
          idempotencyKey,
        }
      );

      console.log(`Order created: ${order.id}, status: ${order.status}`);
      return order;
    } catch (err) {
      if (err.status === 409 && err.code === 'IDEMPOTENCY_CONFLICT') {
        // Same key with different params - this is a bug in our code
        throw new Error('Idempotency key conflict - parameters mismatch');
      }

      if (attempt === maxRetries) {
        throw err;
      }

      const delay = Math.min(1000 * Math.pow(2, attempt - 1), 10000);
      console.log(`Attempt ${attempt} failed, retrying in ${delay}ms...`);
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }
}

// Usage
const order = await createOrderSafe({
  energyAmount: 65000,
  targetAddress: 'TTargetAddressHere',
  durationHours: 1,
});
```

Key points in this implementation:

- The idempotency key is generated once, before the retry loop. Every retry sends the same key.
- Exponential backoff prevents hammering the server during transient failures.
- The 409 conflict on parameter mismatch is treated as a non-retryable error because it indicates a logic bug, not a network issue.

### Python SDK - Retry-Safe Withdrawal

The Python SDK follows the same pattern. Here is an example for withdrawals:

```python
import uuid
import time
from merx_sdk import MerxClient, MerxAPIError

client = MerxClient(
    api_key="your_api_key",
    base_url="https://merx.exchange/api/v1",
)


def withdraw_safe(address: str, amount_sun: int, max_retries: int = 3):
    idempotency_key = str(uuid.uuid4())

    for attempt in range(1, max_retries + 1):
        try:
            result = client.withdraw(
                address=address,
                amount=amount_sun,
                idempotency_key=idempotency_key,
            )
            print(f"Withdrawal initiated: {result['id']}")
            return result

        except MerxAPIError as e:
            if e.status_code == 409:
                raise ValueError(
                    "Idempotency conflict - check parameters"
                ) from e

            if attempt == max_retries:
                raise

            delay = min(2 ** (attempt - 1), 10)
            print(f"Attempt {attempt} failed: {e}. Retrying in {delay}s...")
            time.sleep(delay)

    raise RuntimeError("All retry attempts exhausted")


# Usage
result = withdraw_safe(
    address="TWithdrawalAddressHere",
    amount_sun=100_000_000,  # 100 TRX
)
```

### Raw HTTP - Direct API Call with curl

If you are not using an SDK, the `Idempotency-Key` header is a standard HTTP header:

```bash
IDEMPOTENCY_KEY=$(uuidgen)

curl -X POST https://merx.exchange/api/v1/orders \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $IDEMPOTENCY_KEY" \
  -d '{
    "energy_amount": 65000,
    "target_address": "TTargetAddressHere",
    "duration_hours": 1
  }'
```

Retry the exact same curl command (with the same `IDEMPOTENCY_KEY` value) and you will get the original order back instead of a new one.

## Лучшие практики for Production Systems

### Generate Keys at the Right Level

The idempotency key should represent the business intent, not the HTTP request. If a user clicks "Buy Energy" once, that is one business operation with one key - regardless of how many HTTP retries it takes.

Do not generate a new key on each retry. That defeats the purpose entirely.

### Store Keys Before Sending

In a system that processes orders in a queue, write the idempotency key to your database before making the API call. If your process crashes and restarts, it picks up the same key from the database and retries safely.

```javascript
// Write the intent to your database first
const intent = await db.orderIntents.create({
  idempotencyKey: randomUUID(),
  energyAmount: 65000,
  targetAddress: 'TTargetAddressHere',
  status: 'pending',
});

// Now make the API call with the stored key
const order = await merx.orders.create(
  { energy_amount: intent.energyAmount, target_address: intent.targetAddress },
  { idempotencyKey: intent.idempotencyKey }
);

// Update the intent with the result
await db.orderIntents.update(intent.id, {
  status: 'completed',
  orderId: order.id,
});
```

### Handle All Response Codes

Your retry logic should handle these cases:

| HTTP Status | Meaning | Action |
|------------|---------|--------|
| 201 | First successful creation | Store result, continue |
| 200 | Duplicate key, same params | Treat as success (same result) |
| 409 | Key conflict or still processing | Do not retry - investigate |
| 429 | Rate limited | Retry after delay |
| 500+ | Server error | Retry with backoff |

### Use Structured Keys for Debugging

While UUIDs work perfectly, structured keys make debugging easier:

```
order-{userId}-{date}-{sequenceNumber}
withdraw-{userId}-{timestamp}-{nonce}
```

When investigating a support case, a structured key immediately tells you who initiated the operation and when.

### Set Reasonable Retry Limits

Three to five retries with exponential backoff covers the vast majority of transient failures. If the server is genuinely down, retrying 50 times will not help and will only generate noise in your logs.

A sensible ceiling: retry up to 3 times, with delays of 1 second, 2 seconds, and 4 seconds. If all three fail, surface the error to a monitoring system rather than retrying indefinitely.

## Типичные ошибки

**Generating a new key on each retry.** This is the most common mistake. If each retry has a new key, the server treats each as a unique request. You get duplicate orders.

**Sharing keys across different operations.** Each distinct business operation needs its own key. If you use the same key for two different orders (different amounts, different addresses), the second will fail with a 409.

**Not handling the 200 vs 201 distinction.** While both indicate success, the status code tells you whether this was the first execution or a replay. This can be useful for logging and metrics - knowing how often retries hit duplicates tells you something about your network reliability.

**Ignoring idempotency for webhooks.** MERX sends webhooks on order status changes. Your webhook handler should be idempotent too - if you receive the same `order.filled` event twice, processing it twice should not cause problems. Use the order ID as a natural idempotency key for your own processing.

## Заключение

Idempotency keys are a small addition to your API calls - one header, one UUID - but they eliminate an entire class of financial bugs. For any system that creates TRON energy orders or initiates withdrawals programmatically, they are not optional. They are the difference between a system that handles network failures gracefully and one that creates support tickets every time a connection drops.

MERX supports idempotency keys on all financial endpoints. The JavaScript SDK and Python SDK both provide built-in support. Start using them from day one - retrofitting idempotency into a production system that has already experienced duplicate orders is considerably more painful than building it in from the start.

- Платформа MERX: [merx.exchange](https://merx.exchange)
- Документация API: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- Python SDK: [github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)
