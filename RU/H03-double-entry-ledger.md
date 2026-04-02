# Двойная запись для блокчейна: архитектура бухгалтерского учета MERX

Double-entry bookkeeping was invented in 13th-century Italy. It has survived every financial innovation since -- paper currency, stock exchanges, central banking, electronic funds transfer, and now blockchain. There is a reason it has endured: it works. Every transaction produces a balanced pair of entries, and any imbalance signals an error immediately.

When we built the accounting system for MERX -- a platform that handles TRX deposits, energy purchases, provider settlements, and withdrawals -- we chose double-entry ledger design without debate. The alternative, single-entry tracking with running balance updates, is how most crypto platforms handle accounting. It is also how most crypto platforms end up with unexplainable balance discrepancies.

This article explains the ledger architecture behind MERX: why double-entry matters for blockchain platforms, how the ledger table is designed, why records are immutable, how debit and credit accounts interact, and how we reconcile balances using SELECT FOR UPDATE.

## Why Double-Entry for Crypto

A single-entry system works like a bank statement: one column, one running total. When a user deposits 100 TRX, you add 100 to their balance. When they spend 5 TRX, you subtract 5. Simple.

The problems with single-entry appear under stress:

**Concurrent modifications.** Two orders execute simultaneously. Both read the user's balance as 100 TRX. Both deduct 5 TRX. Both write 95 TRX. The user has been charged once instead of twice. Or worse: one write overwrites the other, and the platform loses track of a transaction entirely.

**Missing audit trail.** A user's balance is 47.3 TRX. How did it get there? With single-entry, you have to reconstruct the balance from individual transaction records -- which may or may not be complete, and which have no built-in integrity check.

**Reconciliation failure.** The sum of all user balances should equal the platform's TRX holdings. With single-entry, verifying this requires aggregating every user's balance and comparing it to the treasury. If the numbers do not match, there is no systematic way to find the discrepancy.

Double-entry solves all of these problems structurally. Every balance mutation creates two entries that must sum to zero. The integrity check is built into every operation.

## The Ledger Table

The MERX ledger is a single PostgreSQL table:

```sql
CREATE TABLE ledger (
  id            BIGSERIAL PRIMARY KEY,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  account_id    UUID NOT NULL REFERENCES accounts(id),
  entry_type    TEXT NOT NULL,
  amount_sun    BIGINT NOT NULL,
  direction     TEXT NOT NULL CHECK (direction IN ('DEBIT', 'CREDIT')),
  reference_id  UUID,
  reference_type TEXT,
  description   TEXT,
  idempotency_key TEXT UNIQUE
);

CREATE INDEX idx_ledger_account ON ledger(account_id);
CREATE INDEX idx_ledger_reference ON ledger(reference_id);
CREATE INDEX idx_ledger_created ON ledger(created_at);
```

### Column Design

**amount_sun**: All amounts stored in SUN (1 TRX = 1,000,000 SUN). Using the smallest unit eliminates floating-point arithmetic entirely. There are no decimal amounts, no rounding errors, no precision loss. Every calculation is integer arithmetic.

**direction**: Either DEBIT or CREDIT. The meaning depends on the account type:

- For user accounts: CREDIT increases the balance, DEBIT decreases it
- For settlement accounts: DEBIT increases the balance, CREDIT decreases it
- For revenue accounts: CREDIT increases the balance

**entry_type**: Categorizes the ledger entry. Examples:

```
DEPOSIT              User deposits TRX to their MERX account
WITHDRAWAL           User withdraws TRX from their MERX account
ORDER_PAYMENT        User pays for an energy order
ORDER_REFUND         Order fails, payment returned to user
PROVIDER_SETTLEMENT  Payment to energy provider for fulfilled order
X402_PAYMENT         On-chain payment received via x402 protocol
```

**reference_id** and **reference_type**: Link the ledger entry to the business object that caused it (an order, a deposit, a withdrawal). This creates a bidirectional audit trail: from the ledger entry you can find the order, and from the order you can find its ledger entries.

**idempotency_key**: Prevents duplicate entries. If the same operation is processed twice (due to a retry, a network timeout, a duplicate webhook), the unique constraint on idempotency_key ensures only one entry is created.

## The Immutability Rule

Ledger records are never updated. They are never deleted. This is enforced at the database level:

```sql
-- Prevent any updates to ledger records
CREATE OR REPLACE FUNCTION prevent_ledger_update()
RETURNS TRIGGER AS $$
BEGIN
  RAISE EXCEPTION 'Ledger records cannot be updated';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_ledger_update
  BEFORE UPDATE ON ledger
  FOR EACH ROW
  EXECUTE FUNCTION prevent_ledger_update();

-- Prevent any deletes from ledger
CREATE OR REPLACE FUNCTION prevent_ledger_delete()
RETURNS TRIGGER AS $$
BEGIN
  RAISE EXCEPTION 'Ledger records cannot be deleted';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_ledger_delete
  BEFORE DELETE ON ledger
  FOR EACH ROW
  EXECUTE FUNCTION prevent_ledger_delete();
```

These triggers make the immutability rule unbreakable at the database level. Not even an administrator running direct SQL can modify or remove a ledger entry without first disabling the trigger -- an operation that would be visible in database audit logs.

### Why Immutability Matters

**Audit integrity.** If ledger records can be modified, an attacker (or a bug) can alter the financial history of the platform. Immutable records mean the history is permanent and tamper-evident.

**Regulatory compliance.** Financial record-keeping regulations universally require that transaction records be preserved. Deleting or altering them is a compliance violation.

**Debugging.** When something goes wrong -- and in a system processing real money, things will go wrong -- immutable records provide a complete, unaltered timeline of events. You can replay history exactly as it happened.

### Corrections and Reversals

If a ledger entry needs to be "corrected" (for example, a refund), you do not update the original entry. You create a new entry with the opposite direction:

```sql
-- Original order payment
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($user_id, 'ORDER_PAYMENT', 1820000, 'DEBIT', $order_id);

INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($provider_settlement, 'ORDER_PAYMENT', 1820000, 'CREDIT', $order_id);

-- Order failed, issue refund (new entries, originals remain)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($user_id, 'ORDER_REFUND', 1820000, 'CREDIT', $order_id);

INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($provider_settlement, 'ORDER_REFUND', 1820000, 'DEBIT', $order_id);
```

After the refund, the original payment entries still exist. The user's calculated balance reflects both the payment and the refund: net zero. The audit trail shows exactly what happened and when.

## Debit and Credit Accounts

MERX uses several account types, each with its own role in the double-entry system:

### User Accounts

Every MERX user has an account. Their balance is calculated from ledger entries:

```sql
SELECT
  COALESCE(SUM(CASE WHEN direction = 'CREDIT' THEN amount_sun ELSE 0 END), 0) -
  COALESCE(SUM(CASE WHEN direction = 'DEBIT' THEN amount_sun ELSE 0 END), 0)
  AS balance_sun
FROM ledger
WHERE account_id = $user_id;
```

Credits increase the balance (deposits, refunds). Debits decrease it (order payments, withdrawals).

### Provider Settlement Account

When a user buys energy, the payment needs to reach the provider. The provider settlement account tracks what MERX owes to each provider:

```
User pays for order:
  User account:               DEBIT  1,820,000 SUN
  Provider settlement (Feee): CREDIT 1,820,000 SUN

MERX settles with provider:
  Provider settlement (Feee): DEBIT  1,820,000 SUN
  Treasury:                   CREDIT 1,820,000 SUN
```

At any point, the provider settlement account balance shows the total amount MERX owes to that provider and has not yet settled.

### Treasury Account

The treasury account represents MERX's on-chain TRX holdings. Deposits credit the treasury (TRX received). Withdrawals and provider settlements debit the treasury (TRX sent out).

### The Fundamental Equation

At all times:

```
Sum of all CREDITS = Sum of all DEBITS
```

If this equation fails, the system has a bug. MERX runs a reconciliation check periodically:

```sql
SELECT
  SUM(CASE WHEN direction = 'CREDIT' THEN amount_sun ELSE 0 END) AS total_credits,
  SUM(CASE WHEN direction = 'DEBIT' THEN amount_sun ELSE 0 END) AS total_debits
FROM ledger;

-- total_credits MUST equal total_debits
-- If not, alert immediately
```

## The Complete Transaction Flow

Here is how a typical energy purchase flows through the ledger:

### 1. User Deposits TRX

The deposit monitor detects an incoming TRX transfer to the MERX deposit address:

```sql
BEGIN;

-- Credit the user's account (their balance increases)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type, idempotency_key)
VALUES ($user_id, 'DEPOSIT', 100000000, 'CREDIT', $deposit_id, 'DEPOSIT', $tx_hash);

-- Debit the treasury (TRX received, treasury acknowledges the liability)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type, idempotency_key)
VALUES ($treasury_id, 'DEPOSIT', 100000000, 'DEBIT', $deposit_id, 'DEPOSIT', $tx_hash || '_treasury');

COMMIT;
```

### 2. User Buys Energy

The user places an order for 65,000 energy at 28 SUN/unit:

```sql
BEGIN;

-- Check balance with row lock
SELECT balance_sun FROM account_balances
WHERE account_id = $user_id
FOR UPDATE;

-- Verify sufficient balance
-- 65,000 * 28 = 1,820,000 SUN = 1.82 TRX

-- Debit user account (balance decreases)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type, idempotency_key)
VALUES ($user_id, 'ORDER_PAYMENT', 1820000, 'DEBIT', $order_id, 'ORDER', $idempotency_key);

-- Credit provider settlement (MERX now owes the provider)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type, idempotency_key)
VALUES ($provider_settlement_id, 'ORDER_PAYMENT', 1820000, 'CREDIT', $order_id, 'ORDER', $idempotency_key || '_settlement');

COMMIT;
```

### 3. Order Fails (Refund)

If the provider fails to delegate the energy, the order is refunded:

```sql
BEGIN;

-- Credit user account (balance restored)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type)
VALUES ($user_id, 'ORDER_REFUND', 1820000, 'CREDIT', $order_id, 'ORDER');

-- Debit provider settlement (MERX no longer owes the provider)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type)
VALUES ($provider_settlement_id, 'ORDER_REFUND', 1820000, 'DEBIT', $order_id, 'ORDER');

COMMIT;
```

The original payment entries remain. The refund entries are new records. The user's net balance change for this order is zero: 1,820,000 SUN debited, then 1,820,000 SUN credited.

## Balance Reconciliation with SELECT FOR UPDATE

The most dangerous moment in any financial system is the balance check before a deduction. Without proper locking, concurrent requests can both pass the balance check and both deduct, resulting in a negative balance.

### The Race Condition

```
Thread A: SELECT balance WHERE user_id = 1  -> 10 TRX
Thread B: SELECT balance WHERE user_id = 1  -> 10 TRX
Thread A: Deduct 8 TRX, new balance = 2 TRX
Thread B: Deduct 8 TRX, new balance = 2 TRX
Result: User had 10 TRX, spent 16 TRX, balance shows 2 TRX
```

The platform just lost 6 TRX.

### The Fix: SELECT FOR UPDATE

```sql
BEGIN;

-- Lock the row. Any other transaction trying to read this row
-- with FOR UPDATE will block until this transaction completes.
SELECT balance_sun FROM account_balances
WHERE account_id = $user_id
FOR UPDATE;

-- Now we have an exclusive lock. Check balance safely.
-- If insufficient, ROLLBACK.
-- If sufficient, proceed with ledger entries.

INSERT INTO ledger ...;

COMMIT;
-- Lock released. Next waiting transaction can proceed.
```

With `FOR UPDATE`, the scenario becomes:

```
Thread A: SELECT ... FOR UPDATE  -> 10 TRX (row locked)
Thread B: SELECT ... FOR UPDATE  -> BLOCKED (waiting for A's lock)
Thread A: Deduct 8 TRX, COMMIT  -> balance = 2 TRX (lock released)
Thread B: SELECT ... FOR UPDATE  -> 2 TRX (lock acquired)
Thread B: Deduct 8 TRX?         -> INSUFFICIENT BALANCE, ROLLBACK
```

No overspend. No lost funds. The serialization guarantee of `SELECT FOR UPDATE` ensures that balance checks and deductions are atomic.

### Performance Implications

`SELECT FOR UPDATE` serializes transactions per account. Two users can transact simultaneously without blocking each other (they lock different rows). But two concurrent orders for the same user must wait in line.

In practice, this is not a bottleneck. Individual users rarely submit truly concurrent requests. When they do (e.g., a misconfigured bot), serialization is the correct behavior -- you want those requests processed sequentially, not in parallel.

## Periodic Reconciliation

Beyond per-transaction integrity, MERX runs a periodic reconciliation that verifies the entire ledger:

```sql
-- 1. Verify global balance equation
SELECT
  SUM(CASE WHEN direction = 'CREDIT' THEN amount_sun ELSE 0 END) AS credits,
  SUM(CASE WHEN direction = 'DEBIT' THEN amount_sun ELSE 0 END) AS debits
FROM ledger;
-- credits MUST equal debits

-- 2. Verify per-account balances against on-chain state
-- Sum of all user balances should equal treasury holdings
-- minus pending provider settlements

-- 3. Check for orphaned references
-- Every reference_id should point to a valid order, deposit, or withdrawal
SELECT l.reference_id, l.reference_type
FROM ledger l
LEFT JOIN orders o ON l.reference_id = o.id AND l.reference_type = 'ORDER'
LEFT JOIN deposits d ON l.reference_id = d.id AND l.reference_type = 'DEPOSIT'
WHERE o.id IS NULL AND d.id IS NULL AND l.reference_type IS NOT NULL;
```

If any check fails, the system alerts immediately. The response is investigation and correction (via new ledger entries), never modification of existing records.

## Итоги

The MERX double-entry ledger provides:

1. **Integrity**: Every transaction is a balanced pair. Imbalances are detected immediately.
2. **Immutability**: Records cannot be modified or deleted. History is permanent.
3. **Concurrency safety**: SELECT FOR UPDATE prevents race conditions on balance checks.
4. **Auditability**: Complete financial history with bidirectional references.
5. **Reconciliation**: Periodic checks verify the entire system state.

This is not novel. It is a 700-year-old accounting technique applied to a blockchain platform. The novelty is that most crypto platforms skip it -- and pay the price in lost funds, unexplainable discrepancies, and auditor nightmares.

Platform: [https://merx.exchange](https://merx.exchange)
Documentation: [https://merx.exchange/docs](https://merx.exchange/docs)

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
