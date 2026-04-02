# MERX Energy Cost Estimation API: Know the Cost Before You Buy

Every TRON transaction consumes resources. A USDT transfer needs roughly 65,000 energy units. A smart contract approval burns around 15,000. A complex DeFi interaction might need 200,000 or more. The actual numbers depend on the contract, the operation, and current network parameters.

If you are building a product that handles TRON transactions on behalf of users - a wallet, a payment processor, a trading bot - you need to know the cost before you commit. How much energy is required? What will it cost to rent versus burn? How much does the user save?

MERX provides two endpoints that answer these questions precisely: `POST /api/v1/estimate` for general cost estimation and `GET /api/v1/orders/preview` for order-specific cost preview. Together, they let you show users exact costs and savings before a single SUN changes hands.

## The Estimation Endpoint

`POST /api/v1/estimate` calculates the energy and bandwidth requirements for a given transaction type and returns the cost comparison between renting energy through MERX and burning TRX at the protocol level.

### Basic Request

```bash
curl -X POST https://merx.exchange/api/v1/estimate \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "trc20_transfer",
    "target_address": "TTargetAddressHere"
  }'
```

### Response Format

```json
{
  "operation": "trc20_transfer",
  "target_address": "TTargetAddressHere",
  "energy_required": 64895,
  "bandwidth_required": 345,
  "costs": {
    "burn": {
      "trx_cost": 27370000,
      "trx_cost_readable": "27.37 TRX",
      "usd_equivalent": 2.19
    },
    "rental": {
      "best_provider": "sohu",
      "price_per_unit_sun": 22,
      "total_cost_sun": 1427690,
      "total_cost_trx": "1.43 TRX",
      "usd_equivalent": 0.11,
      "duration_hours": 1
    },
    "savings": {
      "trx_saved": "25.94 TRX",
      "percent": 94.8,
      "usd_saved": 2.08
    }
  },
  "address_resources": {
    "current_energy": 0,
    "current_bandwidth": 1200,
    "energy_deficit": 64895,
    "bandwidth_deficit": 0
  },
  "timestamp": "2026-03-30T10:00:00Z"
}
```

The response tells you everything you need to know for a transaction cost decision:

- **energy_required** - how much energy the operation needs. For a standard TRC-20 transfer, this is approximately 65,000 units, though the exact number depends on the contract and target address state.
- **bandwidth_required** - how much bandwidth the transaction uses. Most simple transfers need 300-400 bandwidth points. New accounts (activating an address for the first time) need more.
- **costs.burn** - what it costs if the user does nothing and lets the protocol burn TRX. This is the "default" cost.
- **costs.rental** - the cheapest available rental option through MERX. Includes the provider, per-unit price, total cost, and rental duration.
- **costs.savings** - the difference between burn and rental, expressed as absolute TRX saved, percentage saved, and USD equivalent.
- **address_resources** - current energy and bandwidth on the target address. If the address already has energy from staking or a previous delegation, the deficit is reduced accordingly.

### Supported Operations

The `operation` field accepts several predefined types that cover the most common TRON transactions:

#### trc20_transfer

The standard TRC-20 token transfer. This is the most common operation - sending USDT, USDC, or any other TRC-20 token from one address to another.

```json
{
  "operation": "trc20_transfer",
  "target_address": "TSenderAddressHere"
}
```

Energy required: approximately 64,895 units for USDT on a standard transfer (address already activated with a USDT balance). First-time transfers to an address that has never held the token cost more - up to 100,000 energy - because the contract must create a new storage slot.

#### trc20_approve

The TRC-20 approval transaction, used to allow a smart contract to spend tokens on your behalf. Required before interacting with DEX contracts, lending protocols, and most DeFi applications.

```json
{
  "operation": "trc20_approve",
  "target_address": "TApproverAddressHere"
}
```

Energy required: approximately 15,000-18,000 units, significantly less than a transfer.

#### trx_transfer

A simple TRX transfer. These primarily consume bandwidth, not energy, but the estimation endpoint handles them for completeness.

```json
{
  "operation": "trx_transfer",
  "target_address": "TSenderAddressHere"
}
```

Energy required: 0 (TRX transfers do not consume energy). Bandwidth required: approximately 270 bytes.

#### custom

For smart contract calls that do not fit the predefined types. Provide the contract address and function selector, and MERX estimates the energy consumption by simulating the call.

```json
{
  "operation": "custom",
  "target_address": "TCallerAddressHere",
  "contract_address": "TContractAddressHere",
  "function_selector": "stake(uint256)",
  "parameters": [
    {
      "type": "uint256",
      "value": "1000000"
    }
  ]
}
```

The custom operation runs a simulation against the TRON network to determine the actual energy and bandwidth consumption. This is the most accurate method for non-standard transactions but takes slightly longer (200-500ms versus 50ms for predefined types).

## The Order Preview Endpoint

While `POST /estimate` gives you resource requirements and cost comparisons, `GET /api/v1/orders/preview` shows you exactly what a MERX order would look like - including which provider would be selected, the exact debit from your balance, and any applicable fees.

### Request

```bash
curl "https://merx.exchange/api/v1/orders/preview?energy_amount=65000&target_address=TTargetAddressHere&duration_hours=1" \
  -H "X-API-Key: your_api_key"
```

### Response

```json
{
  "preview": {
    "energy_amount": 65000,
    "target_address": "TTargetAddressHere",
    "duration_hours": 1,
    "provider": "sohu",
    "price_per_unit_sun": 22,
    "subtotal_sun": 1430000,
    "fee_sun": 14300,
    "total_sun": 1444300,
    "total_trx": "1.44 TRX",
    "your_balance_sun": 50000000,
    "balance_after_sun": 48555700,
    "estimated_fill_time_seconds": 5
  },
  "alternatives": [
    {
      "provider": "catfee",
      "price_per_unit_sun": 25,
      "total_sun": 1641250,
      "total_trx": "1.64 TRX"
    },
    {
      "provider": "netts",
      "price_per_unit_sun": 28,
      "total_sun": 1838200,
      "total_trx": "1.84 TRX"
    }
  ]
}
```

The preview includes:

- **provider** - which provider MERX would route the order to at current prices.
- **subtotal_sun** - the raw cost of energy at the provider's rate.
- **fee_sun** - the MERX platform fee.
- **total_sun** - the total amount that would be debited from your balance.
- **balance_after_sun** - your balance after the order, so you can verify affordability.
- **estimated_fill_time_seconds** - how long the delegation typically takes with this provider.
- **alternatives** - other providers that could fill the order, sorted by price. Useful for showing users their options.

### Difference Between Estimate and Preview

| Feature | POST /estimate | GET /orders/preview |
|---------|---------------|-------------------|
| Purpose | General cost analysis | Exact order planning |
| Auth required | Yes | Yes |
| Shows savings vs burn | Yes | No |
| Shows your balance | No | Yes |
| Shows platform fees | No | Yes |
| Shows alternatives | No | Yes |
| Shows address resources | Yes | No |
| Supports custom contracts | Yes | No |

Use `estimate` when you need to tell a user "this transaction will cost X if you burn TRX, or Y if you rent energy - saving you Z percent." Use `preview` when the user has decided to buy energy and you want to show the exact order before they confirm.

## Real-World Examples with Numbers

### Example 1: USDT Payment Processor

You run a payment processor that sends USDT to merchants. Before each payout, you estimate the cost:

```javascript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
});

async function estimatePayoutCost(senderAddress) {
  const estimate = await merx.estimate({
    operation: 'trc20_transfer',
    target_address: senderAddress,
  });

  console.log(`Energy needed: ${estimate.energy_required}`);
  console.log(`Burn cost: ${estimate.costs.burn.trx_cost_readable}`);
  console.log(`Rental cost: ${estimate.costs.rental.total_cost_trx}`);
  console.log(`Savings: ${estimate.costs.savings.percent}%`);

  // Decide whether to rent or burn based on savings threshold
  if (estimate.costs.savings.percent > 50) {
    return { method: 'rent', cost: estimate.costs.rental };
  } else {
    return { method: 'burn', cost: estimate.costs.burn };
  }
}
```

Typical output:

```
Energy needed: 64895
Burn cost: 27.37 TRX
Rental cost: 1.43 TRX
Savings: 94.8%
```

At current prices, renting energy almost always saves more than 90 percent compared to burning.

### Example 2: Wallet Cost Display

You build a TRON wallet and want to show the user the transaction cost before they confirm a send:

```python
from merx_sdk import MerxClient

client = MerxClient(api_key="your_api_key")


def get_transfer_cost(sender_address: str, token: str = "usdt") -> dict:
    """Get the cost to send a TRC-20 token, accounting for existing resources."""

    estimate = client.estimate(
        operation="trc20_transfer",
        target_address=sender_address,
    )

    resources = estimate["address_resources"]
    costs = estimate["costs"]

    # If the address already has enough energy, the transfer is free
    if resources["energy_deficit"] == 0:
        return {
            "cost": "0 TRX",
            "note": "Address has sufficient energy",
        }

    return {
        "without_energy": costs["burn"]["trx_cost_readable"],
        "with_energy": costs["rental"]["total_cost_trx"],
        "savings": f'{costs["savings"]["percent"]}%',
        "energy_deficit": resources["energy_deficit"],
    }
```

The wallet UI can display something like:

```
Send 100 USDT to TRecipient...

Transaction cost:
  Without energy rental:  27.37 TRX (burned)
  With MERX energy:       1.43 TRX (rented)
  You save:               94.8%

[ Rent Energy and Send ]    [ Send Without Energy ]
```

### Example 3: Batch Transfer Cost Projection

You need to send USDT to 500 addresses and want to know the total cost upfront:

```javascript
async function estimateBatchCost(addresses) {
  let totalBurnCost = 0;
  let totalRentalCost = 0;
  let totalEnergy = 0;

  // Estimate for a representative sample (first 10 addresses)
  // Most TRC-20 transfers cost the same energy
  const sampleSize = Math.min(10, addresses.length);
  const sample = addresses.slice(0, sampleSize);

  for (const address of sample) {
    const estimate = await merx.estimate({
      operation: 'trc20_transfer',
      target_address: address,
    });

    totalBurnCost += estimate.costs.burn.trx_cost;
    totalRentalCost += estimate.costs.rental.total_cost_sun;
    totalEnergy += estimate.energy_required;
  }

  // Extrapolate to full batch
  const avgBurnCost = totalBurnCost / sampleSize;
  const avgRentalCost = totalRentalCost / sampleSize;
  const avgEnergy = totalEnergy / sampleSize;

  const projectedBurnTRX = (avgBurnCost * addresses.length) / 1_000_000;
  const projectedRentalTRX = (avgRentalCost * addresses.length) / 1_000_000;
  const projectedEnergy = avgEnergy * addresses.length;

  console.log(`Batch size: ${addresses.length} transfers`);
  console.log(`Total energy needed: ${projectedEnergy.toLocaleString()}`);
  console.log(`Cost without MERX: ${projectedBurnTRX.toFixed(2)} TRX`);
  console.log(`Cost with MERX: ${projectedRentalTRX.toFixed(2)} TRX`);
  console.log(
    `Total savings: ${(projectedBurnTRX - projectedRentalTRX).toFixed(2)} TRX`
  );
}
```

For 500 transfers at current market rates:

```
Batch size: 500 transfers
Total energy needed: 32,447,500
Cost without MERX: 13,685.00 TRX
Cost with MERX: 715.00 TRX
Total savings: 12,970.00 TRX
```

At approximately 0.08 USD per TRX, that is a savings of over 1,000 USD on a single batch.

### Example 4: Custom Contract Call Estimation

You interact with a custom staking contract and need to estimate the cost of a `stake()` call:

```bash
curl -X POST https://merx.exchange/api/v1/estimate \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "custom",
    "target_address": "TCallerAddressHere",
    "contract_address": "TStakingContractHere",
    "function_selector": "stake(uint256)",
    "parameters": [
      {
        "type": "uint256",
        "value": "50000000"
      }
    ]
  }'
```

The response includes the simulated energy consumption, which for a complex staking contract might be 120,000 to 200,000 units - significantly more than a simple transfer.

## Integrating Estimates into Order Flow

The estimation and preview endpoints fit naturally into a user-facing order flow:

1. **User initiates a transaction** (e.g., send USDT).
2. **Call POST /estimate** to get energy requirements and cost comparison.
3. **Display burn cost versus rental cost** with savings percentage.
4. **User chooses to rent energy.**
5. **Call GET /orders/preview** to show exact order details with fees.
6. **User confirms.**
7. **Call POST /orders** with an idempotency key to create the order.
8. **Poll or wait for webhook** confirming energy delegation.
9. **Execute the original transaction** with the delegated energy.

Steps 2-3 are informational. No money moves. The user sees transparent pricing before committing. This builds trust and reduces support requests from users surprised by costs.

## Error Handling

Both endpoints return standard MERX error responses:

```json
{
  "error": {
    "code": "INVALID_ADDRESS",
    "message": "Target address is not a valid TRON address",
    "details": {
      "address": "invalid_address_here"
    }
  }
}
```

Common errors:

| Code | Cause | Resolution |
|------|-------|-----------|
| `INVALID_ADDRESS` | Target address fails TRON address validation | Verify the address format (T-prefix, base58) |
| `INVALID_OPERATION` | Unrecognized operation type | Use one of: trc20_transfer, trc20_approve, trx_transfer, custom |
| `SIMULATION_FAILED` | Custom contract call simulation failed | Check contract address and function selector |
| `NO_PROVIDERS` | No providers available for the required energy amount | Try again later or reduce the energy amount |
| `INSUFFICIENT_BALANCE` | Balance too low for the previewed order (preview only) | Deposit more TRX to your MERX account |

## Caching Considerations

Estimation results are valid for a short window. Energy prices change as providers adjust rates, and network parameters can shift the burn cost. For most use cases:

- **Cache estimates for 30-60 seconds** if you are displaying costs in a UI. Prices do not change faster than MERX polls providers (every 30 seconds).
- **Always fetch a fresh preview** immediately before order creation. The preview reflects the exact cost at that moment.
- **Do not cache custom contract simulations** if the contract state changes frequently. The simulation result depends on on-chain state at the time of execution.

## Conclusion

The estimation and preview endpoints remove the guesswork from TRON energy purchasing. Instead of renting a fixed amount of energy and hoping it is enough, you know exactly how much you need. Instead of accepting whatever price is available, you see every option ranked by cost.

For developers building products on TRON, these endpoints transform energy from an unpredictable cost into a known, optimizable line item. Check the cost, show the savings, let the user decide, then execute with confidence.

- MERX platform: [merx.exchange](https://merx.exchange)
- Full API documentation: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- Python SDK: [github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)


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
