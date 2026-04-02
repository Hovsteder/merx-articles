# Архитектура безопасности MERX: как мы защищаем ваши средства

When a platform handles financial transactions, security is not a feature - it is the foundation everything else sits on. A single vulnerability can destroy user trust permanently. At MERX, security considerations shaped every architectural decision from day one, not as an afterthought bolted on after the fact.

This article details the security architecture of MERX: how funds are protected, how data integrity is maintained, how the system defends against common attack vectors, and the design principles that make the platform resilient.

---

## Principle 1: No Custody of Private Keys

MERX never holds, stores, or has access to your TRON private keys. This is a fundamental design decision that eliminates an entire class of attacks.

### How It Works

When you use MERX to purchase energy, the delegation happens from the provider's address to your address. MERX orchestrates this transaction but never needs access to your wallet. Your private key stays on your device, in your hardware wallet, or wherever you manage it.

The flow:

```
1. You tell MERX: "Delegate 65,000 energy to TMyAddress"
2. MERX tells the provider: "Delegate 65,000 energy to TMyAddress"
3. The provider delegates from TProviderAddress to TMyAddress
4. MERX verifies the delegation on-chain
5. Your private key was never involved
```

### Why This Matters

If MERX were compromised, attackers could not steal your TRX or tokens because MERX does not have your keys. Compare this with platforms that require you to deposit tokens to a platform-controlled address - those platforms hold your keys (or keys to your funds), creating a single point of failure.

### The Treasury Exception

MERX does manage its own treasury address for receiving deposits and processing withdrawals. The treasury private key is stored as a Docker secret, accessible only to the `treasury-signer` service. It is never exposed to the API service, the web frontend, or any other component. More on this isolation below.

---

## Principle 2: Double-Entry Ledger

Every financial operation on MERX creates a paired ledger entry. This is the same accounting principle used by every bank and financial institution for the last 700 years. It works.

### How It Works

Every transaction creates two entries: a debit and a credit. The sum of all debits always equals the sum of all credits. If they do not, something is wrong, and the system detects it immediately.

```sql
-- Order payment example
INSERT INTO ledger (account_id, type, amount_sun, direction)
VALUES
  ($user_id, 'ORDER_PAYMENT', 5525000, 'DEBIT'),
  ($provider_settlement, 'ORDER_PAYMENT', 5525000, 'CREDIT');
```

### Immutability

Ledger records are never updated or deleted. If a transaction needs to be reversed (e.g., a refund), a new ledger entry is created with the opposite direction:

```sql
-- Refund: new entries, original entries remain
INSERT INTO ledger (account_id, type, amount_sun, direction, reference_id)
VALUES
  ($user_id, 'REFUND', 5525000, 'CREDIT', $original_order_id),
  ($provider_settlement, 'REFUND', 5525000, 'DEBIT', $original_order_id);
```

The original debit entry is never modified. The refund credit entry explicitly references the original, creating a complete audit trail.

### Why This Matters

If an attacker compromises the application layer and attempts to inflate a user's balance, the ledger entries will not balance. Regular reconciliation checks detect this immediately:

```sql
-- Reconciliation query: should always return 0
SELECT SUM(CASE direction
  WHEN 'DEBIT' THEN amount_sun
  WHEN 'CREDIT' THEN -amount_sun
END) as imbalance
FROM ledger;
```

Any non-zero result triggers an immediate alert and investigation.

---

## Principle 3: Atomic Balance Operations

Every balance mutation uses `SELECT FOR UPDATE` to prevent race conditions. This is not optional - it is enforced at the database level.

### The Race Condition Problem

Without proper locking, a user with a 10 TRX balance could submit two simultaneous orders for 8 TRX each:

```
Thread 1: SELECT balance WHERE user_id = 1    -> 10 TRX
Thread 2: SELECT balance WHERE user_id = 1    -> 10 TRX
Thread 1: balance (10) >= order (8)? YES       -> proceed
Thread 2: balance (10) >= order (8)? YES       -> proceed
Thread 1: UPDATE balance = 10 - 8 = 2 TRX
Thread 2: UPDATE balance = 10 - 8 = 2 TRX

Result: User spent 16 TRX with only 10 TRX balance
```

### The Solution

```sql
BEGIN;

-- Lock the row - second transaction waits here
SELECT balance_sun FROM accounts
WHERE user_id = $1
FOR UPDATE;

-- Check balance
-- If insufficient: ROLLBACK
-- If sufficient: proceed

UPDATE accounts
SET balance_sun = balance_sun - $order_amount
WHERE user_id = $1
  AND balance_sun >= $order_amount;  -- Double-check in UPDATE

COMMIT;
```

`FOR UPDATE` acquires a row-level lock. The second transaction blocks until the first commits or rolls back. After the first transaction commits (reducing balance to 2 TRX), the second transaction reads the updated balance (2 TRX) and correctly rejects the insufficient-funds order.

---

## Principle 4: Input Validation

All inputs are validated with Zod schemas before processing. This includes API requests, webhook payloads, provider responses, and internal service messages.

### API Input Validation

```typescript
const CreateOrderSchema = z.object({
  energy: z.number()
    .int('Energy must be an integer')
    .min(10000, 'Minimum order is 10,000 energy')
    .max(100000000, 'Maximum order is 100,000,000 energy'),

  targetAddress: z.string()
    .regex(/^T[1-9A-HJ-NP-Za-km-z]{33}$/, 'Invalid TRON address format')
    .refine(isValidTronAddress, 'Invalid TRON address checksum'),

  duration: z.enum(['1h', '1d', '3d', '7d', '14d', '30d']),

  maxPrice: z.number()
    .positive()
    .optional(),

  idempotencyKey: z.string()
    .max(255)
    .optional()
});
```

Every field is typed, bounded, and validated. No raw user input reaches business logic or database queries.

### SQL Injection Prevention

All database queries use parameterized statements. String concatenation is never used to build SQL:

```typescript
// Never this:
const query = `SELECT * FROM users WHERE id = '${userId}'`;  // SQL injection

// Always this:
const query = 'SELECT * FROM users WHERE id = $1';
const result = await db.query(query, [userId]);
```

This is enforced by code review and linting rules. There is no mechanism for raw SQL string interpolation in the codebase.

### TRON Address Validation

TRON addresses are validated at multiple levels:

1. **Format check**: must match the TRON address regex (starts with T, 34 characters, base58).
2. **Checksum verification**: the address includes a checksum that detects typos.
3. **On-chain verification** (optional): confirm the address exists and is activated.

Sending energy to an invalid address wastes resources and cannot be reversed on-chain. Strict validation prevents this.

---

## Principle 5: Service Isolation

MERX runs as a set of isolated Docker containers, each with minimal permissions and no unnecessary access.

### Container Architecture

```
Docker network:
  |
  |-- api           (port 3000, public-facing)
  |-- price-monitor (no external ports)
  |-- order-executor (no external ports)
  |-- ledger        (no external ports)
  |-- treasury-signer (no external ports, Docker secret access)
  |-- deposit-monitor (no external ports)
  |-- withdrawal-executor (no external ports)
  |
  |-- postgresql    (port 5432, internal only)
  |-- redis         (port 6379, internal only)
```

### Key Isolation Properties

- **The API service cannot access the treasury private key.** Only the `treasury-signer` container can read the Docker secret containing the key.
- **The price monitor cannot modify balances.** It only has read access to provider APIs and write access to Redis price channels.
- **The order executor cannot directly modify the ledger.** It publishes settlement events to Redis, which the ledger service consumes.
- **PostgreSQL and Redis are not exposed externally.** They are only accessible from within the Docker network.

### Why This Matters

If an attacker compromises the API service (the most exposed component), they cannot:
- Access the treasury private key (different container, Docker secret).
- Directly modify ledger entries (different service, no database write access to ledger tables).
- Bypass balance checks (enforced at database level with FOR UPDATE).

The blast radius of any single-service compromise is limited by design.

---

## Principle 6: Rate Limiting and Abuse Prevention

### API Rate Limits

Every API endpoint has rate limits appropriate to its purpose:

```
Public endpoints (prices, health):     100 requests/minute
Authenticated reads (orders, balance): 60 requests/minute
Authenticated writes (create order):   30 requests/minute
Withdrawals:                           5 requests/minute
```

Rate limits are enforced per API key, tracked in Redis with sliding windows.

### Withdrawal Safeguards

Withdrawals are the highest-risk operation (moving real assets off-platform). Additional safeguards include:

- **Rate limiting**: maximum 5 withdrawal requests per minute.
- **Amount limits**: daily withdrawal limits per account.
- **Confirmation delay**: large withdrawals trigger a cooldown period.
- **Balance verification**: `SELECT FOR UPDATE` ensures sufficient balance.
- **Idempotency**: duplicate withdrawal requests (same idempotency key) return the original result.

---

## Principle 7: Webhook Security

MERX sends webhook notifications for order status updates, deposits, and other events. Webhooks are signed with HMAC-SHA256 to prevent forgery.

### How HMAC Webhooks Work

```
1. MERX computes: HMAC-SHA256(webhook_body, your_webhook_secret)
2. MERX includes the signature in the X-Merx-Signature header
3. Your server recomputes the HMAC with the same secret
4. If signatures match: genuine webhook. If not: forged, discard.
```

### Verification in Code

```typescript
import crypto from 'crypto';

function verifyWebhook(body: string, signature: string, secret: string): boolean {
  const computed = crypto
    .createHmac('sha256', secret)
    .update(body)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(computed)
  );
}
```

Note the use of `timingSafeEqual` to prevent timing attacks. A naive string comparison (`===`) would leak information about the correct signature through response time variations.

---

## Principle 8: Secrets Management

No secret is ever hardcoded in source code. All sensitive values are managed through environment variables and Docker secrets.

### Environment Variables

```
# .env (never committed to git)
DATABASE_URL=postgresql://...
REDIS_URL=redis://...
API_JWT_SECRET=...
WEBHOOK_SIGNING_SECRET=...
TRON_API_KEY=...
```

### Docker Secrets (for High-Sensitivity Values)

The treasury private key is too sensitive for environment variables (which can be logged or leaked through process inspection). It is stored as a Docker secret:

```yaml
# docker-compose.yml
services:
  treasury-signer:
    secrets:
      - treasury_private_key

secrets:
  treasury_private_key:
    file: /run/secrets/treasury_key
```

Docker secrets are mounted as files inside the container, readable only by the service process. They do not appear in environment variable listings, container inspect output, or logs.

### Git Protection

The `.gitignore` file excludes all sensitive files:

```
.env
*.key
*.pem
secrets/
```

This is set up before the first commit, not after.

---

## Monitoring and Incident Response

### Automated Alerts

The following conditions trigger immediate alerts:

- Ledger imbalance detected (debit-credit mismatch).
- Treasury balance drops below threshold.
- Failed authentication attempts exceed threshold (10/minute per IP).
- Provider API returns unexpected errors.
- Order execution failure rate exceeds 5%.

### Audit Logging

Every security-relevant operation is logged with structured data:

```json
{
  "event": "withdrawal_requested",
  "user_id": "usr_abc123",
  "amount_sun": 10000000,
  "destination": "TAddress...",
  "ip": "203.0.113.45",
  "timestamp": "2026-03-30T12:00:00Z"
}
```

Logs are retained for forensic analysis and compliance. They are append-only and stored separately from application data.

---

## Заключение

Security at MERX is not a single feature but a set of interlocking principles: no key custody, double-entry accounting, atomic balance operations, strict input validation, service isolation, rate limiting, signed webhooks, and proper secrets management. Each principle addresses a specific threat vector, and together they create a defense-in-depth architecture where compromising any single component does not compromise the system.

No system is invulnerable. But by designing security into the architecture from the start - rather than patching it on later - MERX minimizes the attack surface and maximizes the cost an attacker must pay to cause harm.

Review the open-source components: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js), [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python), [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp).

Start using MERX at [https://merx.exchange](https://merx.exchange).

---

*This article is part of the MERX technical series. MERX is the first blockchain resource exchange, built with security as a foundational requirement, not an afterthought.*

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
