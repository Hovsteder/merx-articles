# x402 Kullandikca Ode: Hesap Olusturmadan TRON Energy Satin Alin

## The Registration Problem

Every blockchain service follows the same pattern: create an account, verify your email, generate an API key, fund an internal balance, then start using the service. For a human user, this is a minor inconvenience. For an autonomous AI agent, it is a hard stop.

Agents do not have email addresses. They do not want internal balances managed by third parties. They do not want to trust a platform with custody of their funds. What agents want is simple: pay for a service, receive the service, move on. No relationship. No state. No trust.

The x402 protocol makes this possible. Inspired by the HTTP 402 "Payment Required" status code (defined in 1997 but never widely implemented), x402 enables pay-per-use commerce where payment is verified on-chain rather than through accounts and API keys.

MERX implements x402 for energy purchases. Any entity with a TRON wallet - human, agent, or smart contract - can buy energy in a single transaction without creating an account, without depositing funds, and without any prior relationship with the MERX platform.

## How It Works

The x402 flow has five steps. Each step is verifiable on-chain, and at no point does the buyer need to trust MERX with custody of their funds.

### Step 1: Request an Invoice

The buyer calls `create_paid_order` with the desired energy parameters:

```
Tool: create_paid_order
Input: {
  "energy_amount": 65000,
  "duration_hours": 1,
  "target_address": "TBuyerAddress..."
}

Response:
{
  "invoice": {
    "order_id": "xpay_abc123",
    "amount_trx": 1.43,
    "amount_sun": 1430000,
    "pay_to": "TMerxTreasuryAddress...",
    "memo": "merx_xpay_abc123",
    "expires_at": "2026-03-30T12:05:00Z",
    "energy_amount": 65000,
    "duration_hours": 1,
    "target_address": "TBuyerAddress..."
  }
}
```

The invoice is a quote, not a commitment. No funds have moved. The buyer can inspect the price, compare it to alternatives, and decide whether to proceed. The invoice expires after 5 minutes - if not paid within that window, the quoted price is no longer guaranteed.

### Step 2: Sign the Payment Locally

The buyer constructs a TRX transfer transaction from their wallet to the MERX treasury address. The critical detail is the memo field, which must contain the exact string from the invoice.

```javascript
const tx = await tronWeb.transactionBuilder.sendTrx(
  invoice.pay_to,        // TMerxTreasuryAddress
  invoice.amount_sun,    // 1430000 (1.43 TRX in SUN)
  buyerAddress
);

// Add memo to transaction data
const txWithMemo = await tronWeb.transactionBuilder.addUpdateData(
  tx,
  invoice.memo,          // "merx_xpay_abc123"
  'utf8'
);

// Sign locally - private key never leaves the buyer's machine
const signedTx = await tronWeb.trx.sign(txWithMemo);
```

The payment is signed with the buyer's private key on the buyer's machine. The private key is never shared with MERX, never transmitted over the network, and never stored anywhere outside the buyer's control.

### Step 3: Broadcast the Payment

The signed transaction is broadcast to the TRON network:

```javascript
const result = await tronWeb.trx.sendRawTransaction(signedTx);
const txHash = result.txid;
```

At this point, the payment is on-chain. It is visible to anyone who queries the TRON blockchain. The TRX has moved from the buyer's address to the MERX treasury address, and the memo field contains the invoice identifier.

### Step 4: MERX Verifies On-Chain

MERX monitors its treasury address for incoming transactions. When a transaction arrives, MERX:

1. Reads the memo field
2. Matches it against outstanding invoices
3. Verifies the amount matches the invoice (exact amount required)
4. Verifies the invoice has not expired
5. Verifies the invoice has not already been paid (prevents double-claiming)

```
Verification:
  TX hash: 7f3a2b...
  From: TBuyerAddress...
  To: TMerxTreasuryAddress...
  Amount: 1,430,000 SUN (1.43 TRX) - MATCHES
  Memo: "merx_xpay_abc123" - MATCHES invoice
  Invoice status: UNPAID - OK
  Invoice expiry: 2026-03-30T12:05:00Z - NOT EXPIRED

  Result: VERIFIED
```

### Step 5: Energy Delegation

With payment verified, MERX places the energy order through its provider network:

```
Order placed:
  Energy: 65,000
  Duration: 1 hour
  Target: TBuyerAddress...
  Provider: sohu (best price at time of order)

Delegation confirmed after 4.1 seconds
Energy available at TBuyerAddress: 65,000
```

The buyer now has 65,000 energy delegated to their address for 1 hour. They can use it for USDT transfers, DEX swaps, or any other smart contract interaction.

## The Complete Agent Flow

Here is how an AI agent executes the entire x402 flow through the MERX MCP sunucusu:

```
Agent: "I need to send 100 USDT to TRecipient. I don't have a MERX account."

Step 1: Agent calls create_paid_order
  -> Receives invoice: 1.43 TRX, memo "merx_xpay_abc123"

Step 2: Agent calls transfer_trx
  -> Sends 1.43 TRX to TMerxTreasury with memo "merx_xpay_abc123"
  -> TX hash: 7f3a2b...

Step 3: Agent waits for verification (automatic)
  -> MERX detects payment, verifies on-chain
  -> Energy delegated to agent's address

Step 4: Agent calls transfer_trc20
  -> Sends 100 USDT to TRecipient
  -> Uses delegated energy instead of burning TRX
  -> Cost: 1.43 TRX instead of ~27 TRX

Total interaction with MERX: 2 tool calls
Account created: No
API key used: No
Funds deposited: No
Trust required: Minimal (payment verified on-chain)
```

## Security: Why the Memo Matters

The memo field is the linchpin of x402 security. Without it, any TRX transfer to the MERX treasury could be claimed as payment for any invoice. This creates two attack vectors:

### Cross-Payment Attack

Without memo verification, an attacker could:
1. Request an invoice for 65,000 energy (1.43 TRX)
2. Wait for someone else to send 1.43 TRX to the MERX treasury for an unrelated reason
3. Claim that unrelated payment as their invoice payment
4. Receive free energy

The memo prevents this. Every invoice has a unique memo string. A payment is only matched to an invoice if the memo field contains the exact string. Random TRX transfers to the treasury address - which happen on any active address - are ignored because they lack a valid memo.

### Replay Attack

Without expiration and single-use enforcement, an attacker could:
1. Pay an invoice legitimately
2. Reference the same payment transaction hash for a second invoice
3. Receive energy twice for one payment

MERX prevents this with two mechanisms:
- Each invoice is marked as PAID after the first successful verification. A second payment claiming the same memo is rejected.
- Each payment transaction can only be matched to one invoice. The transaction hash is recorded and cannot be reused.

### Amount Verification

The payment amount must exactly match the invoice amount. Sending 1.42 TRX instead of 1.43 TRX results in a failed verification. This prevents attacks where an attacker sends a minimal amount (e.g., 0.000001 TRX) with a valid memo to claim an invoice at a fraction of the price.

## Real Mainnet Transaction

Here is a verified x402 transaction from TRON mainnet:

```
Invoice:
  Order ID: xpay_m7k2p9
  Energy: 65,000
  Duration: 1 hour
  Price: 1.43 TRX
  Memo: merx_xpay_m7k2p9

Payment Transaction:
  Hash: 8d4f1a7b3c2e...
  Block: 58,234,891
  From: TWallet...
  To: TMerxTreasury...
  Amount: 1,430,000 SUN
  Memo: merx_xpay_m7k2p9
  Status: CONFIRMED

Verification:
  Timestamp: 2026-03-28T14:22:17Z
  Match: Amount OK, Memo OK, Not expired, Not previously paid
  Result: VERIFIED

Energy Delegation:
  Delegated: 65,000 energy
  Provider: catfee
  Delegation TX: 3a8b2c...
  Confirmed at: 2026-03-28T14:22:23Z
  Expires at: 2026-03-28T15:22:23Z
```

From invoice request to energy delegation: 23 seconds. No account. No API key. No prior relationship.

## When to Use x402 vs Account-Based Access

x402 is ideal for:

- **AI agents** that operate autonomously and should not depend on managed accounts
- **One-time purchases** where the overhead of account creation exceeds the value of the transaction
- **Privacy-conscious users** who do not want to create accounts or provide identifying information
- **Testing and evaluation** where developers want to try the service before committing to an account
- **Cross-platform agents** that interact with many services and should not maintain separate accounts for each

Account-based access is better for:

- **High-volume operations** where the per-transaction overhead of x402 (invoice creation + payment broadcast + verification) matters
- **Standing orders and monitors** which require persistent server-side state linked to an account
- **Balance-based billing** where prepaid credit is more efficient than per-transaction payments
- **Team operations** where multiple users share an account with role-based access

Many users start with x402 for evaluation and migrate to account-based access as their usage grows. The two models are complementary, not competing.

## x402 and the Future of Agent Commerce

The x402 protocol represents a broader shift in how services are consumed. Traditional SaaS billing - monthly subscriptions, tiered pricing, usage limits - assumes a human customer who creates an account, evaluates pricing tiers, and makes a purchasing decision. This model breaks down when the customer is an AI agent that needs to make purchasing decisions autonomously, in real-time, for specific tasks.

x402 aligns with the agent economy:

- **No registration** - Agents do not have identities in the traditional sense. x402 requires only a wallet address.
- **Per-use pricing** - Agents should pay for what they use, when they use it, in amounts proportional to the task at hand.
- **On-chain verification** - Trust is established through cryptographic proof, not through account credentials or business relationships.
- **Composability** - An agent can use x402 to pay for energy from MERX, then use that energy to interact with any TRON smart contract. The payment and the service are decoupled.

MERX is the first energy exchange to implement x402. As the protocol matures, we expect other blockchain services to adopt similar pay-per-use models optimized for agent consumption.

## Implementation Details for Developers

### Building an x402 Client

If you are building an application that uses MERX x402 without the MCP server, here is the minimal implementation:

```javascript
const TronWeb = require('tronweb');

async function buyEnergyX402(tronWeb, energyAmount, durationHours, targetAddress) {
  // Step 1: Get invoice
  const invoiceRes = await fetch('https://merx.exchange/api/v1/x402/invoice', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      energy_amount: energyAmount,
      duration_hours: durationHours,
      target_address: targetAddress
    })
  });
  const invoice = await invoiceRes.json();

  // Step 2: Build payment transaction
  const tx = await tronWeb.transactionBuilder.sendTrx(
    invoice.pay_to,
    invoice.amount_sun,
    tronWeb.defaultAddress.base58
  );

  const txWithMemo = await tronWeb.transactionBuilder.addUpdateData(
    tx, invoice.memo, 'utf8'
  );

  // Step 3: Sign and broadcast
  const signedTx = await tronWeb.trx.sign(txWithMemo);
  const result = await tronWeb.trx.sendRawTransaction(signedTx);

  // Step 4: Poll for order completion
  let order;
  for (let i = 0; i < 20; i++) {
    await new Promise(r => setTimeout(r, 3000));
    const orderRes = await fetch(
      `https://merx.exchange/api/v1/x402/order/${invoice.order_id}`
    );
    order = await orderRes.json();
    if (order.status === 'completed') break;
  }

  return order;
}
```

### Hata Yonetimi

Common failure modes and their resolutions:

| Error | Cause | Resolution |
|---|---|---|
| `INVOICE_EXPIRED` | Payment not sent within 5 minutes | Request a new invoice |
| `AMOUNT_MISMATCH` | Payment amount differs from invoice | Send exact amount; request new invoice if price changed |
| `MEMO_NOT_FOUND` | Payment missing memo field | Resend with correct memo; funds from failed attempt are not auto-refunded |
| `ALREADY_PAID` | Invoice was already fulfilled | Check order status; this is not an error if you are polling |
| `PROVIDER_UNAVAILABLE` | No provider can fill the order | Retry after a few minutes; the invoice payment will be credited to a MERX balance for manual withdrawal |

### Refund Policy

If MERX cannot fulfill the order after payment is verified (e.g., all providers are temporarily unavailable), the payment amount is credited to a claimable balance associated with the payer's TRON address. The payer can claim this balance through a separate endpoint without creating an account - the claim is verified by signing a message with the same private key that sent the original payment.

## Sonuc

The HTTP 402 status code was defined nearly 30 years ago with the vision of native internet payments. That vision was ahead of its time. Blockchains finally provide the infrastructure to make it real.

MERX x402 turns energy purchasing into a single atomic flow: request, pay, receive. No accounts. No API keys. No trust assumptions beyond what the blockchain provides.

For AI agents, this is the natural purchasing model. For developers, it is the simplest integration path. For the ecosystem, it is a proof of concept for how blockchain services will be consumed in the agent economy.

---

**Baglantilar:**
- MERX Platformu: [https://merx.exchange](https://merx.exchange)
- MCP Sunucusu (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP Sunucusu (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)
