# Implementacion del protocolo x402: facturar, pagar, verificar

HTTP status code 402 -- "Payment Required" -- was defined in the original HTTP/1.1 specification in 1997. The spec marked it as "reserved for future use." Twenty-nine years later, the future has arrived.

The x402 protocol turns that reserved status code into a real payment mechanism. It enables pay-per-use commerce where payment is verified en cadena rather than through accounts, clave de APIs, or credit cards. Any entity with a blockchain wallet -- human, bot, or autonomous agent -- can pay for a service in a single transaction without creating an account or establishing any prior relationship with the service provider.

MERX implements x402 for TRON energy purchases. This article is a full technical guide to the implementation: invoice creation, payment with memo, TronGrid verification (including the hex vs base58 address matching problem), the x402 system user, balance crediting, and ejecucion de ordenes. We cover every step, every edge case, and every security consideration.

## The x402 Flow Overview

The protocol has five steps:

```
1. INVOICE   - Buyer requests a quote, server returns payment instructions
2. PAY       - Buyer sends TRX with a memo containing the invoice ID
3. VERIFY    - Server detects the on-chain payment and validates it
4. CREDIT    - Server credits the x402 system account and creates ledger entries
5. EXECUTE   - Server executes the energy order and delegates to the buyer
```

Each step is designed to be trustless. The buyer never sends funds to an unverified address. The server never executes an order without confirmed payment. The memo field ties the payment to a specific invoice, preventing cross-payment attacks.

## Step 1: Invoice Creation

The buyer calls the `create_paid_order` endpoint with the desired energy parameters:

```
POST /api/v1/orders/paid
Content-Type: application/json

{
  "energy_amount": 65000,
  "duration_hours": 1,
  "target_address": "TBuyerAddress..."
}
```

The server calculates the cost based on current mejor precios across all providers and returns an invoice:

```json
{
  "invoice": {
    "order_id": "xpay_a7f3c2d1",
    "amount_trx": 1.43,
    "amount_sun": 1430000,
    "pay_to": "TMerxTreasuryAddress...",
    "memo": "merx_xpay_a7f3c2d1",
    "expires_at": "2026-03-30T12:05:00Z",
    "energy_amount": 65000,
    "duration_hours": 1,
    "target_address": "TBuyerAddress..."
  }
}
```

### Invoice Design Decisions

**5-minute expiration.** The invoice expires 5 minutes after creation. This window is long enough for the buyer to review, sign, and broadcast the payment. It is short enough to prevent stale-price exploitation -- if precios de energia change significantly, the buyer should request a new invoice at the current price.

**Exact amount required.** The payment must match the invoice amount exactly. Not more, not less. This prevents ambiguity in matching payments to invoices. If a buyer sends 2 TRX for a 1.43 TRX invoice, the payment is rejected (the excess would create accounting complexity with no upside).

**Unique memo.** The memo field contains a unique identifier that ties the payment to this specific invoice. This is the critical security mechanism -- more on this below.

### Server-Side Invoice Storage

```sql
INSERT INTO x402_invoices (
  order_id,
  amount_sun,
  pay_to,
  memo,
  expires_at,
  energy_amount,
  duration_hours,
  target_address,
  status
) VALUES (
  'xpay_a7f3c2d1',
  1430000,
  'TMerxTreasuryAddress...',
  'merx_xpay_a7f3c2d1',
  '2026-03-30T12:05:00Z',
  65000,
  1,
  'TBuyerAddress...',
  'PENDING'
);
```

The invoice is stored with status PENDING. It will transition to PAID when verified, or EXPIRED when the TTL passes without payment.

## Step 2: Payment with Memo

The buyer constructs and signs a TRX transfer transaction. The critical implementation detail is the memo field.

```javascript
const tronWeb = new TronWeb({
  fullHost: 'https://api.trongrid.io'
});

// Build the base transaction
const tx = await tronWeb.transactionBuilder.sendTrx(
  invoice.pay_to,       // TMerxTreasuryAddress
  invoice.amount_sun,   // 1430000
  buyerAddress           // TBuyerAddress
);

// Add the memo
const txWithMemo = await tronWeb.transactionBuilder.addUpdateData(
  tx,
  invoice.memo,         // "merx_xpay_a7f3c2d1"
  'utf8'
);

// Sign locally
const signedTx = await tronWeb.trx.sign(txWithMemo, privateKey);

// Broadcast
const result = await tronWeb.trx.sendRawTransaction(signedTx);
console.log('TX hash:', result.txid);
```

### Why Memo, Not Amount

An earlier design considered using unique amounts (e.g., 1,430,017 SUN instead of 1,430,000 SUN) to identify payments. This approach is fragile:

- Amount collisions are possible (two invoices might have the same price)
- It requires the buyer to pay a non-round amount
- It does not work when multiple invoices have identical parameters

The memo field provides an unambiguous identifier with no collision risk.

### Private Key Safety

The buyer's clave privada never leaves their device. The transaction is constructed, signed, and broadcast entirely on the buyer's machine. MERX never sees, requests, or has access to the buyer's clave privada. This is a fundamental security property of the x402 protocol.

## Step 3: TronGrid Verification

After the payment is broadcast, MERX must verify it en cadena. This is where the implementation gets interesting -- and where a significant technical challenge emerges.

### The Monitoring Loop

The MERX deposit monitor continuously watches the treasury address for incoming transactions:

```typescript
async function monitorTreasuryForX402Payments(): Promise<void> {
  const treasuryAddress = process.env.TREASURY_ADDRESS;

  while (true) {
    const transactions = await tronWeb.trx.getTransactionsRelated(
      treasuryAddress,
      'to',
      { limit: 50, only_confirmed: true }
    );

    for (const tx of transactions) {
      await processIncomingTransaction(tx);
    }

    await sleep(3000); // Check every 3 seconds (one block)
  }
}
```

### The Hex vs Base58 Address Problem

Aqui esta the technical challenge that consumed more debugging time than any other part of the x402 implementation.

TRON addresses exist in two formats:

- **Base58**: `TJRabPrwbZy45sbavfcjinPJC18kjpRTv8` (human-readable, starts with T)
- **Hex**: `415a523b449890854c8fc460ab602df9f31fe4293f` (41-prefixed hex, used internally)

When you query TronGrid for transaction details, the response uses hex addresses. When your invoice stores the buyer's address and the treasury address, they are in base58. If you compare them directly, they will never match.

```typescript
// Transaction from TronGrid API
const txData = {
  owner_address: '415a523b449890854c8fc460ab602df9f31fe4293f',  // hex
  to_address: '41e552f6487585c2b58bc2c9bb4492bc1f17132cd0',    // hex
  amount: 1430000
};

// Invoice from database
const invoice = {
  pay_to: 'TJRabPrwbZy45sbavfcjinPJC18kjpRTv8',  // base58
  target_address: 'TBuyerAddressBase58...',         // base58
  amount_sun: 1430000
};

// Direct comparison FAILS
txData.to_address === invoice.pay_to  // false (hex vs base58)
```

### The Fix: Convert Before Comparing

Every address comparison must convert both sides to the same format:

```typescript
function normalizeAddress(address: string): string {
  if (address.startsWith('41') && address.length === 42) {
    // Hex format -- convert to base58
    return tronWeb.address.fromHex(address);
  }
  if (address.startsWith('T') && address.length === 34) {
    // Already base58
    return address;
  }
  throw new Error(`Invalid TRON address format: ${address}`);
}

function addressesMatch(a: string, b: string): boolean {
  return normalizeAddress(a) === normalizeAddress(b);
}
```

This normalization function is used in every address comparison throughout the x402 verification pipeline. Missing a single comparison point would create a vulnerability.

### Full Verification Logic

```typescript
async function verifyX402Payment(tx: TronTransaction): Promise<void> {
  // 1. Extract memo from transaction data
  const memo = extractMemo(tx);
  if (!memo || !memo.startsWith('merx_xpay_')) {
    return; // Not an x402 payment, skip
  }

  // 2. Find matching invoice
  const invoice = await findInvoiceByMemo(memo);
  if (!invoice) {
    console.warn(`No invoice found for memo: ${memo}`);
    return;
  }

  // 3. Check invoice status
  if (invoice.status !== 'PENDING') {
    console.warn(`Invoice ${invoice.order_id} already ${invoice.status}`);
    return; // Prevents double-claiming
  }

  // 4. Check expiration
  if (new Date() > new Date(invoice.expires_at)) {
    await markInvoiceExpired(invoice.order_id);
    console.warn(`Invoice ${invoice.order_id} expired`);
    return;
  }

  // 5. Verify amount (exact match required)
  if (tx.amount !== invoice.amount_sun) {
    console.warn(
      `Amount mismatch: TX=${tx.amount}, invoice=${invoice.amount_sun}`
    );
    return;
  }

  // 6. Verify recipient (hex vs base58 safe comparison)
  if (!addressesMatch(tx.to_address, invoice.pay_to)) {
    console.warn('Recipient address mismatch');
    return;
  }

  // All checks passed -- payment is valid
  await processValidPayment(invoice, tx);
}
```

### Memo Extraction

The memo is stored in the transaction's `raw_data.data` field as a hex-encoded string:

```typescript
function extractMemo(tx: TronTransaction): string | null {
  try {
    const hexData = tx.raw_data?.data;
    if (!hexData) return null;

    // Decode hex to UTF-8
    const memo = Buffer.from(hexData, 'hex').toString('utf8');
    return memo;
  } catch {
    return null;
  }
}
```

## Step 4: The x402 System User

When an x402 payment is verified, MERX needs to credit the payment and execute the order. But x402 payments are account-less -- the buyer does not have a MERX account. How do you create entradas del libro mayor without an account?

The solution is the x402 system user. This is a special internal account that represents all x402 transactions:

```sql
INSERT INTO accounts (id, email, type)
VALUES (
  'x402-system-00000000-0000-0000-0000-000000000000',
  'x402@system.merx.exchange',
  'SYSTEM'
);
```

### Balance Crediting

When an x402 payment is verified, the system:

1. Credits the x402 system account (balance increases)
2. Debits the treasury account (TRX received)
3. Immediately debits the x402 system account (order payment)
4. Credits the provider settlement account (payment to provider)

```sql
BEGIN;

-- Credit x402 system account (payment received)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($x402_system_id, 'X402_PAYMENT', 1430000, 'CREDIT', $order_id);

-- Debit treasury (TRX received on-chain)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($treasury_id, 'X402_PAYMENT', 1430000, 'DEBIT', $order_id);

-- Debit x402 system account (order payment)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($x402_system_id, 'ORDER_PAYMENT', 1430000, 'DEBIT', $order_id);

-- Credit provider settlement (MERX owes provider)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($provider_settlement_id, 'ORDER_PAYMENT', 1430000, 'CREDIT', $order_id);

COMMIT;
```

The x402 system account balance should always be zero or near-zero: every credit (payment received) is immediately offset by a debit (order executed). If the balance grows, it means payments are being received but orders are not executing -- an alert condition.

## Step 5: Order Execution

After the payment is credited, the order is executed through the standard MERX order pipeline:

```typescript
async function processValidPayment(
  invoice: X402Invoice,
  tx: TronTransaction
): Promise<void> {
  // Mark invoice as PAID
  await updateInvoiceStatus(invoice.order_id, 'PAID', tx.txid);

  // Create ledger entries (as shown above)
  await createX402LedgerEntries(invoice, tx);

  // Execute the energy order
  const order = await executeOrder({
    energy_amount: invoice.energy_amount,
    duration_hours: invoice.duration_hours,
    target_address: invoice.target_address,
    source: 'x402',
    reference_tx: tx.txid
  });

  // Update invoice with order result
  await updateInvoiceWithOrder(invoice.order_id, order);
}
```

The ejecucion de ordenes follows the same path as any other MERX order: enrutamiento al mejor precio, provider selection, delegation, and en cadena verification (including the polling fix for race conditions described in our previous article).

## Security: Memo Verification Prevents Cross-Payment

The memo field is the linchpin of x402 security. Without it, a critical attack vector exists:

### The Cross-Payment Attack

Imagine x402 without memos. Two users request invoices simultaneously:

```
Alice requests 65,000 energy for TAliceAddress. Invoice: 1.43 TRX to TMerxTreasury.
Bob requests 65,000 energy for TBobAddress. Invoice: 1.43 TRX to TMerxTreasury.
```

Both invoices have the same amount and the same pay_to address. If Bob pays for his invoice, how does MERX know whether to delegate energy to TAliceAddress or TBobAddress? Without a memo, the payment is ambiguous.

Worse: Bob could pay once and claim both invoices. Or Alice could claim Bob's payment for her own invoice.

### How Memos Prevent This

```
Alice's invoice: memo = "merx_xpay_alice123"
Bob's invoice:   memo = "merx_xpay_bob456"

Alice's payment TX: 1.43 TRX to TMerxTreasury, memo = "merx_xpay_alice123"
Bob's payment TX:   1.43 TRX to TMerxTreasury, memo = "merx_xpay_bob456"

Verification:
  Alice's TX memo matches Alice's invoice -> delegate to TAliceAddress
  Bob's TX memo matches Bob's invoice -> delegate to TBobAddress
```

Each payment is unambiguously linked to its invoice. Hay no way to cross-claim.

### Additional Security Checks

Beyond memo matching, the verification pipeline includes:

**Double-payment prevention.** Once an invoice is marked PAID, subsequent payments with the same memo are rejected. The payer would need to contact support for a refund (or the system would return the funds automatically if the amount exceeds the invoice).

**Amount exactness.** The payment must match the invoice amount precisely. This prevents partial payments (which would require complex partial-fill logic) and overpayments (which would require refund logic).

**Expiration enforcement.** Payments received after the invoice expires are not processed. This prevents stale-price exploitation where a buyer requests an invoice during a low-price period, waits for prices to rise, and then pays the old invoice.

**Address verification.** The payment must go to the correct treasury address. If a user somehow pays a different address (copy-paste error, phishing), the payment will not be detected by the monitor.

## Error Handling

### Payment Without Invoice

If a TRX transfer arrives at the treasury address with a memo that does not match any invoice (typo, expired invoice, test transaction), the payment is logged but not processed. The funds remain in the treasury. In a production system, this would trigger a support alert for manual review and potential refund.

### Provider Failure After Payment

If the proveedor de energia fails to delegate after a verified payment:

```typescript
try {
  const order = await executeOrder(invoice);
} catch (error) {
  // Order failed -- refund the x402 system account
  await createRefundLedgerEntries(invoice);

  // Mark invoice as REFUND_REQUIRED
  await updateInvoiceStatus(invoice.order_id, 'REFUND_REQUIRED');

  // Alert ops team for manual TRX refund to the payer's address
  await alertOps({
    type: 'X402_REFUND_REQUIRED',
    invoice: invoice.order_id,
    payer_address: extractSenderFromTx(tx),
    amount_sun: invoice.amount_sun
  });
}
```

The refund creates new entradas del libro mayor (never modifies existing ones) and flags the invoice for manual refund processing.

### Network Congestion

During high network congestion, the gap between payment broadcast and payment confirmation can extend beyond the 5-minute invoice window. The system handles this by checking the transaction timestamp (when it was broadcast) rather than the confirmation timestamp (when it was included in a block). If the transaction was broadcast before the invoice expired, it is accepted even if confirmation comes after expiration.

## Resumen

The x402 implementation in MERX demonstrates that trustless, account-free payments are practical today. The key design decisions:

1. **Invoice with unique memo** -- unambiguous payment-to-order linking
2. **Exact amount matching** -- eliminates partial/over-payment complexity
3. **5-minute expiration** -- prevents stale-price exploitation
4. **Hex-to-base58 normalization** -- solves the TronGrid address format problem
5. **x402 system user** -- enables partida doble accounting without buyer accounts
6. **Immutable entradas del libro mayor** -- full audit trail for every x402 transaction

The protocol turns HTTP 402 from a 29-year-old placeholder into a working payment mechanism. For AI agents that cannot create accounts or manage clave de APIs, x402 makes TRON energy accessible through a single en cadena transaction.

Plataforma: [https://merx.exchange](https://merx.exchange)
Documentacion: [https://merx.exchange/docs](https://merx.exchange/docs)
Servidor MCP: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
