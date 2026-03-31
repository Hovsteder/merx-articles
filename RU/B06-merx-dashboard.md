# Панель управления MERX: торговля energy без написания кода

Not every energy buyer is a developer. Not every developer wants to write code for a one-off purchase. And not every organization has the engineering bandwidth to integrate an API before they can start saving on TRON transaction costs.

The MERX dashboard at [merx.exchange](https://merx.exchange) provides a complete web interface for buying energy, comparing provider prices, managing your balance, tracking orders, and generating API keys -- all without writing a single line of code. This article walks through every feature of the dashboard and shows how to use it effectively.

## Начало работы

Navigate to [merx.exchange](https://merx.exchange) and create an account. The registration requires an email address and password. No KYC. No identity verification. No waiting period. Your account is active immediately.

Once logged in, you land on the main dashboard view. The interface follows a dark theme with high-contrast typography -- no visual noise, no unnecessary decoration, just the information you need to make purchase decisions.

## The Price Panel

The first thing you see on the dashboard is the live price panel. This displays current energy prices from all seven integrated providers, updated every 30 seconds.

```
Provider      1h Price    1d Price    Available
-------------------------------------------------
Feee          28 SUN      22 SUN      Yes
Netts         31 SUN      24 SUN      Yes
itrx          32 SUN      23 SUN      Yes
CatFee        30 SUN      25 SUN      Yes
PowerSun      33 SUN      26 SUN      Yes
TronSave      35 SUN      28 SUN      Yes
SoHu          34 SUN      27 SUN      Yes
```

The panel highlights the cheapest provider for each duration tier. Prices are displayed in SUN per energy unit -- the standard denomination that allows direct comparison across providers regardless of how they internally quote their rates.

### What the Prices Mean

Each price represents the cost per unit of energy for the specified rental duration. To calculate the total cost for your purchase:

```
Total cost = energy_amount * price_per_unit

Example:
  65,000 energy * 28 SUN/unit = 1,820,000 SUN = 1.82 TRX
```

The dashboard performs this calculation automatically when you enter an order. You see the total cost in both SUN and TRX before confirming.

### Price History

Below the live prices, a historical chart shows how prices have moved over the past 24 hours, 7 days, or 30 days. This helps you identify patterns -- prices tend to be lower during off-peak hours (00:00-08:00 UTC) and higher during peak transaction periods.

The chart is not just informational. If you are placing a large order and timing flexibility exists, waiting a few hours for a price dip can save a meaningful percentage.

## Creating an Order

The order creation form is the core function of the dashboard. Here is the process:

### Step 1: Enter Order Parameters

Fill in three fields:

- **Energy amount**: The number of energy units you need. If you are unsure, the dashboard provides quick-select buttons for common amounts:
  - 32,000 (minimal USDT transfer)
  - 65,000 (standard USDT transfer)
  - 200,000 (DEX swap)
  - 500,000 (complex contract interaction)
  - Custom amount

- **Target address**: The TRON address that will receive the energy delegation. This is the address that will execute the transaction needing energy. The dashboard validates the address format before proceeding.

- **Duration**: How long you need the energy. Options typically include 1 hour, 1 day, 3 days, 7 days, 14 days, and 30 days.

### Step 2: Review the Quote

After entering your parameters, the dashboard displays a detailed quote:

```
Order Summary
-------------------------------------------------
Energy:           65,000 units
Duration:         1 hour
Target:           TYourAddress...
Best provider:    Feee
Price per unit:   28 SUN
Total cost:       1,820,000 SUN (1.82 TRX)
Account balance:  50.00 TRX
Balance after:    48.18 TRX
```

The quote shows exactly which provider will fulfill the order, the per-unit price, the total cost, and the impact on your account balance. There are no hidden fees or markups.

### Step 3: Confirm and Execute

Click the confirm button. The order is submitted to the MERX backend, which routes it to the selected provider for execution. The delegation typically completes within seconds.

Once the order is filled, the dashboard updates to show the order status, the on-chain delegation transaction hash, and the time remaining on the rental.

## Managing Your Balance

### Depositing Funds

Before you can buy energy, you need a TRX balance on MERX. The deposit process:

1. Navigate to the Balance section
2. Click "Deposit"
3. The dashboard displays your unique deposit address
4. Send TRX to that address from any TRON wallet
5. The deposit is credited automatically after on-chain confirmation

The deposit monitor service watches for incoming transactions continuously. Credits typically appear within 1-2 minutes of the transaction being confirmed on-chain.

### Viewing Balance History

The balance section shows a complete history of all balance changes:

```
Date/Time            Type          Amount       Balance
-----------------------------------------------------------
2026-03-28 14:30     Deposit       +100.00 TRX  100.00 TRX
2026-03-28 14:35     Order #1247   -1.82 TRX    98.18 TRX
2026-03-28 16:20     Order #1248   -1.95 TRX    96.23 TRX
2026-03-29 09:00     Deposit       +50.00 TRX   146.23 TRX
2026-03-29 09:15     Order #1249   -5.40 TRX    140.83 TRX
```

Every entry corresponds to a double-entry ledger record in the MERX accounting system. The sum of all entries always reconciles to your current balance -- this is verifiable and auditable.

### Withdrawals

If you need to withdraw TRX from your MERX account:

1. Navigate to the Balance section
2. Click "Withdraw"
3. Enter the destination TRON address
4. Enter the amount to withdraw
5. Confirm the withdrawal

Withdrawals are processed by the treasury-signer service and broadcast to the TRON network. Processing time depends on network conditions but typically completes within a few minutes.

## Order History

The Order History view provides a searchable, filterable list of all your past orders. Each entry shows:

- **Order ID**: Unique identifier for reference
- **Date/Time**: When the order was placed
- **Energy Amount**: How many energy units were purchased
- **Duration**: Rental period
- **Provider**: Which provider fulfilled the order
- **Price**: Per-unit price in SUN
- **Total Cost**: Total amount charged in TRX
- **Status**: Pending, Active, Completed, or Failed
- **Target Address**: Where the energy was delegated

### Filtering and Search

You can filter orders by:

- Date range
- Status (active, completed, all)
- Provider
- Target address

This is particularly useful for organizations that purchase energy for multiple addresses and need to track costs per wallet or per application.

### Order Details

Clicking on any order opens a detail view with full information:

```
Order #1247
-------------------------------------------------
Status:           Completed
Created:          2026-03-28 14:35:22 UTC
Energy:           65,000 units
Duration:         1 hour (expired)
Provider:         Feee
Price:            28 SUN/unit
Total:            1.82 TRX
Target:           TYourAddress...
TX Hash:          7f3a2b...
Delegation Start: 2026-03-28 14:35:30 UTC
Delegation End:   2026-03-28 15:35:30 UTC
```

The transaction hash is a link to the on-chain delegation record, verifiable through any TRON block explorer.

## Generating API Keys

When you are ready to integrate MERX into your application programmatically, the dashboard lets you generate API keys without needing to contact support.

1. Navigate to the Settings or API section
2. Click "Generate API Key"
3. Provide a label for the key (e.g., "Production server", "Staging environment")
4. The key is displayed once -- copy it immediately
5. The key is stored hashed on the server; it cannot be retrieved again

You can manage multiple API keys, each with its own label. If a key is compromised, revoke it from the dashboard and generate a new one. Revoking a key is immediate -- all requests using the revoked key will fail with an authentication error.

### API Key Security

API keys authenticate your requests to the MERX REST API, WebSocket feeds, and SDKs. Treat them like passwords:

- Never commit API keys to version control
- Store them in environment variables or secret managers
- Use separate keys for development, staging, and production
- Rotate keys periodically

The dashboard shows the last-used timestamp for each key, making it easy to identify and revoke unused keys.

## Monitoring Active Delegations

The dashboard includes a real-time view of your active energy delegations:

```
Active Delegations
-------------------------------------------------
Target            Energy     Expires           Remaining
TAddr1...         65,000     2026-03-30 15:00  2h 30m
TAddr2...         200,000    2026-03-31 09:00  20h 30m
TAddr3...         500,000    2026-04-02 12:00  3d 2h 30m
```

For each active delegation, you can:

- See the exact expiration time
- View the remaining time
- See how much energy is available
- Extend the delegation by placing a new order for the same address

The system does not currently support canceling an active delegation early (energy delegation on TRON is an on-chain operation that runs until its specified duration expires), but you can always extend an existing delegation by purchasing additional energy for the same target address.

## Energy Estimation Tool

The dashboard includes a built-in energy estimation tool. If you are not sure how much energy your transaction will need, you can simulate it directly from the dashboard:

1. Enter the contract address (e.g., the USDT contract)
2. Select the function (e.g., transfer)
3. Enter the parameters (recipient address, amount)
4. Click "Estimate"

The tool calls `triggerConstantContract` under the hood and returns the exact energy required for your specific transaction against the current contract state. This eliminates guesswork and prevents over-purchasing or under-purchasing energy.

## Who the Dashboard Is For

### Business Operators

If you run a business that sends USDT payments -- payroll, vendor payments, remittances -- you do not need a developer to integrate an API. Open the dashboard, deposit TRX, and start buying energy before each batch of transfers. The cost savings are immediate and significant.

### Developers Evaluating MERX

Before committing to an API integration, use the dashboard to test the service. Place a few orders, observe the pricing, verify that delegations arrive on-chain as expected. Once you are satisfied, generate an API key and move to programmatic access.

### Finance Teams

The order history and balance views provide the reporting that finance teams need: what was spent, when, on what, from which provider. Export this data for reconciliation with your internal accounting systems.

### Occasional Users

If you make TRON transactions occasionally -- a few per week or per month -- the dashboard is likely all you need. No integration, no code, no maintenance. Just log in, buy energy, and save 90% on transaction fees.

## From Dashboard to API

The dashboard and the API share the same backend. Every action you perform in the dashboard -- checking prices, placing orders, viewing history -- maps directly to an API endpoint. When you are ready to automate:

```
Dashboard action          API equivalent
-----------------------------------------------------------
View prices               GET /api/v1/prices
Create order              POST /api/v1/orders
View order                GET /api/v1/orders/:id
View balance              GET /api/v1/balance
Estimate energy           POST /api/v1/estimate
```

The transition from dashboard user to API user is seamless. Your account, balance, and order history carry over. The only addition is the API key you generate from the dashboard itself.

Full documentation: [https://merx.exchange/docs](https://merx.exchange/docs)
Platform: [https://merx.exchange](https://merx.exchange)
