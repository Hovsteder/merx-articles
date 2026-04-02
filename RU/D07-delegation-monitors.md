# Мониторы делегирования: не позвольте вашей energy TRON истечь

## The Expiration Problem

TRON energy delegations have a fixed duration. When you buy energy for 1 hour, you get exactly 1 hour. When that hour ends, the delegation is revoked, and your address returns to zero delegated energy. If your application sends a USDT transfer 61 minutes after the delegation started, that transaction burns TRX at full price.

For applications running 24/7, this creates a management burden. You need to track when each delegation expires, purchase a replacement before the current one lapses, and handle the gap between expiration and the new delegation arriving. Miss a renewal by even a few seconds, and transactions executed during that window incur the full TRX burn.

MERX delegation monitors eliminate this problem. They watch your delegations, track expiration times, and automatically renew before your energy runs out. The monitor runs on the MERX server 24/7 - it does not depend on your application being online, your MCP client being connected, or your AI agent being active.

## Типы мониторов

The MERX monitoring system supports three types of monitors, each designed for a different operational concern.

### delegation_expiry

Watches active energy delegations and takes action before they expire.

```
Tool: create_monitor
Input: {
  "type": "delegation_expiry",
  "config": {
    "address": "TYourAddress...",
    "renew_before_minutes": 10,
    "auto_renew": true,
    "max_price_per_unit": 0.00006,
    "renewal_amount": 500000,
    "renewal_duration_hours": 24
  },
  "notifications": {
    "channels": ["webhook"],
    "webhook_url": "https://your-app.com/hooks/energy-monitor"
  }
}
```

This monitor tracks all active delegations to the specified address. When any delegation is within 10 minutes of expiring, the monitor takes action:

1. If `auto_renew` is true and the current best price is at or below `max_price_per_unit`, it automatically places a new order for `renewal_amount` energy with `renewal_duration_hours` duration.
2. If the price exceeds `max_price_per_unit`, the monitor sends a notification instead of purchasing, alerting you that prices are too high for automatic renewal.
3. If `auto_renew` is false, the monitor only sends notifications, giving you time to act manually.

The `renew_before_minutes` parameter is critical. Set it high enough that the new delegation can be placed, confirmed, and activated before the old one expires. A 10-minute window provides ample time for order placement (seconds), provider processing (1-2 minutes), and delegation confirmation (3-6 seconds), with margin for network congestion.

### balance_threshold

Monitors the on-chain energy balance and triggers when it falls below a specified level.

```
Tool: create_monitor
Input: {
  "type": "balance_threshold",
  "config": {
    "address": "TYourAddress...",
    "energy_threshold": 200000,
    "bandwidth_threshold": 2000,
    "action": "ensure_resources",
    "energy_target": 500000,
    "bandwidth_target": 5000
  },
  "notifications": {
    "channels": ["telegram"],
    "telegram_chat_id": "-1001234567890"
  }
}
```

Unlike `delegation_expiry`, which is time-based, `balance_threshold` is consumption-based. It fires when your address's available energy drops below 200,000, regardless of why. This covers scenarios that expiry monitoring cannot:

- Multiple transactions consuming energy faster than expected
- A delegation being revoked early by the provider
- Staked energy decreasing due to a change in staking parameters

When triggered, the `ensure_resources` action checks the current balance, calculates the deficit to reach the target, and purchases only what is needed.

### price_alert

Monitors energy market prices and notifies when conditions are met.

```
Tool: create_monitor
Input: {
  "type": "price_alert",
  "config": {
    "condition": "below",
    "threshold": 0.000040,
    "resource_type": "energy",
    "duration_hours": 24,
    "cooldown_minutes": 360
  },
  "notifications": {
    "channels": ["webhook", "telegram"],
    "webhook_url": "https://your-app.com/hooks/price-alert",
    "telegram_chat_id": "-1001234567890"
  }
}
```

Price alerts are informational by default - they notify without purchasing. This is useful for teams that want to buy energy at historically low prices but prefer a human in the loop for purchase decisions.

The `cooldown_minutes` parameter prevents alert fatigue. If the price stays below the threshold for hours, you receive one alert every 6 hours instead of one every 30 seconds.

## Автоматическое продление: подробности

The auto-renewal flow for `delegation_expiry` monitors works as follows:

### Step 1: Track Active Delegations

MERX maintains a record of all energy orders placed through the platform. Each order includes the delegation start time, duration, and expiration time. The monitor checks these records against the current time.

```
Active delegations for TYourAddress:
  Order #1234: 500,000 energy, expires 2026-03-30T14:00:00Z (47 min remaining)
  Order #1235: 200,000 energy, expires 2026-03-30T15:30:00Z (137 min remaining)
```

### Step 2: Evaluate Renewal Window

When any delegation enters the renewal window (e.g., 10 minutes before expiry):

```
Order #1234: 500,000 energy, expires in 9 minutes 42 seconds
-> Within renewal window (10 minutes)
-> Initiating renewal check
```

### Step 3: Price Check

The monitor queries current market prices:

```
Best price for 500,000 energy / 24 hours:
  Provider: sohu
  Price: 26.30 TRX
  Price per unit: 0.0000526

Max allowed: 0.00006 per unit
  0.0000526 <= 0.00006: PASS
```

### Step 4: Place Renewal Order

```
Order placed:
  Amount: 500,000 energy
  Duration: 24 hours
  Provider: sohu
  Cost: 26.30 TRX
  Target: TYourAddress...
  Status: CONFIRMED
```

### Step 5: Verify Delegation

The monitor polls the address to confirm the new delegation has been received:

```
Delegation confirmed:
  Previous energy: 487,231 (remaining from expiring delegation)
  New energy: 987,231 (previous + new delegation)
```

### Step 6: Notify

```
Webhook sent:
{
  "event": "delegation_renewed",
  "address": "TYourAddress...",
  "old_order": "1234",
  "new_order": "1236",
  "energy_amount": 500000,
  "cost_trx": 26.30,
  "provider": "sohu",
  "new_expiry": "2026-03-31T13:50:18Z"
}
```

The entire flow completes in under 30 seconds. The address never experienced a gap in energy coverage.

## Защита от ценовых скачков

The `max_price_per_unit` parameter on auto-renewal monitors is a critical safety mechanism. Energy prices can spike during periods of high demand. Without price protection, an auto-renewal during a price spike could cost 2-3x the normal rate.

When the market price exceeds the maximum:

```
Best price for 500,000 energy / 24 hours:
  Provider: catfee
  Price: 42.50 TRX
  Price per unit: 0.0000850

Max allowed: 0.00006 per unit
  0.0000850 > 0.00006: FAIL - Price exceeds maximum

Action: Notification sent instead of purchase
  "Energy delegation expiring in 8 minutes. Auto-renewal skipped:
   market price 0.0000850 exceeds maximum 0.00006. Manual action required."
```

The notification gives you the option to:
- Accept the higher price and place a manual order
- Wait for prices to normalize and accept a brief gap in coverage
- Adjust the max_price_per_unit on the monitor

### Setting the Right Maximum Price

To set an effective maximum price:

1. Check the `get_price_history` resource for the last 30 days
2. Identify the 95th percentile price (the price that 95% of quotes were at or below)
3. Set your maximum at or slightly above this level

This approach catches normal fluctuations while rejecting genuine price spikes.

## Running 24/7 Without an Agent

This is the key differentiator of MERX monitors compared to agent-side logic. An AI agent runs during a conversation session. When the session ends, the agent stops. If you implemented delegation tracking in your agent's code, it would only work while the agent is active.

MERX monitors run on the MERX server infrastructure:

- **PostgreSQL persistence** - Monitor configurations are stored in the database and survive server restarts
- **Server-side evaluation** - Triggers are evaluated by the MERX backend process, not by any client
- **Independent of MCP connections** - No client needs to be connected for monitors to function
- **Crash recovery** - If the MERX service restarts, monitors resume automatically from their last known state

An agent creates a monitor once. That monitor runs indefinitely (or until its expiration date) regardless of whether the agent ever connects again.

```
Day 1: Agent creates delegation_expiry monitor
Day 2: Agent is offline. Monitor renews delegation at 14:00.
Day 3: Agent is offline. Monitor renews delegation at 13:55.
Day 7: Agent reconnects. Checks monitor history:
  - 6 successful auto-renewals
  - 0 gaps in energy coverage
  - Total spent: 157.80 TRX
  - Average price: 0.0000526 per energy unit
```

## Комбинирование мониторов для надежного покрытия

A single monitor type cannot cover all failure modes. The recommended configuration for production use combines two or three monitor types:

### Recommended Setup

```
Monitor 1: delegation_expiry
  Purpose: Proactive renewal before expiry
  Config:
    renew_before_minutes: 10
    auto_renew: true
    max_price: 0.00006
    renewal_amount: 500,000
    renewal_duration: 24 hours

Monitor 2: balance_threshold
  Purpose: Catch unexpected energy depletion
  Config:
    energy_threshold: 100,000
    action: ensure_resources
    energy_target: 500,000

Monitor 3: price_alert
  Purpose: Opportunity buying at low prices
  Config:
    condition: below
    threshold: 0.000035
    cooldown: 360 minutes
    action: notify_only
```

Monitor 1 handles the normal case - scheduled renewals at acceptable prices. Monitor 2 handles abnormal cases - sudden energy consumption spikes, early delegation revocations, or Monitor 1 being blocked by price protection. Monitor 3 alerts you to exceptional opportunities for manual bulk purchasing.

Together, these three monitors provide:
- Zero-gap energy coverage under normal conditions
- Automatic fallback when prices spike temporarily
- Alerts for cost optimization opportunities

## Управление мониторами

### Listing Active Monitors

```
Tool: list_monitors

Response:
{
  "monitors": [
    {
      "id": "mon_abc123",
      "type": "delegation_expiry",
      "status": "active",
      "address": "TYourAddress...",
      "last_triggered": "2026-03-30T02:00:00Z",
      "total_renewals": 14,
      "total_spent_trx": 368.20,
      "next_expiry": "2026-03-31T02:00:00Z"
    },
    {
      "id": "mon_def456",
      "type": "balance_threshold",
      "status": "active",
      "address": "TYourAddress...",
      "last_triggered": "2026-03-28T15:42:00Z",
      "total_triggers": 2,
      "total_spent_trx": 52.60
    }
  ]
}
```

### Monitor History

Each monitor maintains a detailed execution log:

```json
{
  "history": [
    {
      "timestamp": "2026-03-30T02:00:12Z",
      "trigger_reason": "Delegation expiring in 9m48s",
      "action_taken": "auto_renew",
      "order_id": "ord_xyz789",
      "energy_purchased": 500000,
      "cost_trx": 26.30,
      "provider": "sohu",
      "status": "success"
    },
    {
      "timestamp": "2026-03-29T01:55:33Z",
      "trigger_reason": "Delegation expiring in 9m27s",
      "action_taken": "notification_only",
      "reason": "Price 0.0000780 exceeds max 0.0000600",
      "status": "skipped"
    }
  ]
}
```

This history provides full auditability. You can see exactly when each renewal happened, how much it cost, which provider was used, and why any renewals were skipped.

## Economics of Never Expiring

Consider an application that processes 500 USDT transfers per day. Each transfer requires approximately 65,000 energy.

Without monitors (manual management with occasional gaps):

```
Average gaps per week: 3 (each lasting ~15 minutes)
Transactions during gaps: ~15
TRX burned during gaps: ~15 x 27 = 405 TRX/week
Annual burn from gaps: ~21,060 TRX (~$5,475)
```

With MERX delegation monitors:

```
Gaps per week: 0
TRX burned from gaps: 0
Monitor cost (auto-renewal): ~0 additional (same energy would be purchased anyway)
Annual savings: ~$5,475
```

The monitors do not cost extra. You are purchasing the same energy either way - the monitors just ensure there are no gaps between purchases. The savings come entirely from eliminating the TRX burn during unmanaged expiration windows.

## Заключение

Energy delegations expire. This is a fact of the TRON network that cannot be avoided. What can be avoided is the cost of letting them expire without a replacement ready.

MERX delegation monitors turn a manual, error-prone process into an automated, reliable system. They run on the server, independent of any client connection. They renew delegations before they expire. They respect your price limits. They notify you of exceptions.

Set them up once. Never think about delegation expiry again.

---

**Ссылки:**
- Платформа MERX: [https://merx.exchange](https://merx.exchange)
- MCP Server (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP Server (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)

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
