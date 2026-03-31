# Постоянные ордера: автоматизация закупок energy TRON 24/7

## The Manual Approach Does Not Scale

If you are buying TRON energy manually, you are doing it wrong. Not because the process is difficult - placing an order takes seconds - but because the energy market is dynamic. Prices fluctuate throughout the day. Delegations expire on fixed schedules. Your application's energy needs change based on transaction volume. Managing all of this manually means either constant vigilance or wasted money.

Standing orders solve this by letting you define rules that execute automatically, 24 hours a day, 7 days a week. When a condition is met - price drops below your threshold, a schedule fires, or your energy balance falls too low - MERX places the order on your behalf with the parameters you defined in advance.

This article covers the full standing order system: trigger types, action types, budget controls, and the practical patterns that save the most money.

## Как работают постоянные ордера

A standing order is a persistent rule stored in MERX's PostgreSQL database. It consists of three components:

1. **Trigger** - the condition that activates the order
2. **Action** - what happens when the trigger fires
3. **Constraints** - budget limits, execution frequency, and expiration

The MERX backend evaluates triggers continuously. Price-based triggers are checked every time a new price quote arrives from any provider (typically every 30 seconds). Schedule-based triggers use cron expressions evaluated by the server. Balance-based triggers are checked periodically by polling on-chain state.

When a trigger fires and all constraints are satisfied, the action executes immediately without any manual intervention.

## Создание постоянного ордера

The MCP server exposes standing orders through the `create_standing_order` tool:

```
Tool: create_standing_order
Input: {
  "trigger": {
    "type": "price_below",
    "threshold": 0.00005,
    "resource_type": "energy"
  },
  "action": {
    "type": "buy_resource",
    "params": {
      "energy_amount": 500000,
      "duration_hours": 24,
      "target_address": "TYourAddress..."
    }
  },
  "constraints": {
    "max_daily_spend_trx": 100,
    "max_executions_per_day": 3,
    "expires_at": "2026-06-30T00:00:00Z"
  }
}
```

This order will purchase 500,000 energy for 24 hours whenever the market price drops below 0.00005 TRX per energy unit, up to 3 times per day, spending no more than 100 TRX daily.

## Типы триггеров

### price_below

Fires when the best available energy price drops below your specified threshold.

```json
{
  "type": "price_below",
  "threshold": 0.00005,
  "resource_type": "energy"
}
```

This is the most common trigger for cost optimization. Energy prices on TRON fluctuate based on supply and demand. During off-peak hours (typically 00:00-06:00 UTC), prices often drop 15-25% below peak levels. A `price_below` trigger lets you automatically buy energy during these windows without monitoring prices yourself.

**Practical tip:** Set your threshold at the 25th percentile of the last 7 days' prices. This means you buy when prices are in the bottom quarter of their recent range - cheap enough to save money, frequent enough to keep your address supplied.

### schedule

Fires on a cron schedule, regardless of market conditions.

```json
{
  "type": "schedule",
  "cron": "0 */6 * * *"
}
```

This trigger fires every 6 hours. Useful for applications with predictable energy needs - if you know your application processes most transactions during business hours, schedule energy purchases to arrive just before the rush.

Common cron patterns:

```
"0 */6 * * *"    - Every 6 hours
"0 8 * * 1-5"   - Every weekday at 8:00 UTC
"*/30 * * * *"   - Every 30 minutes
"0 0 * * *"      - Daily at midnight UTC
```

### balance_below

Fires when your address's available energy drops below a threshold.

```json
{
  "type": "balance_below",
  "energy_threshold": 100000,
  "check_address": "TYourAddress..."
}
```

This trigger is reactive - it ensures your address never runs out of energy by automatically replenishing when the balance gets low. Combined with a `buy_resource` action, it creates a self-maintaining energy supply.

**How it works:** MERX polls the on-chain resource balance of the specified address at regular intervals (every 60 seconds by default). When the available energy drops below the threshold, the trigger fires.

**Important caveat:** There is inherent latency between the balance dropping and the new energy arriving (polling interval + order placement + delegation confirmation). Set the threshold high enough that your address does not run out during this window. If your application consumes 50,000 energy per minute, a threshold of 100,000 gives you approximately 2 minutes of runway - sufficient for MERX to replenish.

## Типы действий

### buy_resource

Purchase energy or bandwidth from the market.

```json
{
  "type": "buy_resource",
  "params": {
    "energy_amount": 500000,
    "duration_hours": 24,
    "target_address": "TYourAddress..."
  }
}
```

This is the standard action for energy procurement. MERX automatically selects the cheapest available provider at the time of execution. The `target_address` receives the delegated energy.

### ensure_resources

A higher-level action that checks current resources before purchasing.

```json
{
  "type": "ensure_resources",
  "params": {
    "target_address": "TYourAddress...",
    "energy_target": 500000,
    "bandwidth_target": 5000
  }
}
```

Unlike `buy_resource`, which always buys the specified amount, `ensure_resources` first checks what the address already has and only purchases the deficit. If the address already has 300,000 energy and the target is 500,000, it buys 200,000.

This is the safer action for `balance_below` triggers, as it prevents over-purchasing when multiple triggers fire in rapid succession.

### notify_only

Send a notification without taking any on-chain action.

```json
{
  "type": "notify_only",
  "params": {
    "channels": ["webhook", "telegram"],
    "message": "Energy price dropped below threshold"
  }
}
```

Use this when you want awareness without automation. The standing order monitors the condition and alerts you, but you make the purchase decision manually. This is a good starting point for teams that are not yet comfortable with fully automated purchases.

## Бюджетные лимиты и ограничения

Standing orders without constraints are dangerous. A `price_below` trigger with no spending limit could drain your entire balance during a sustained price dip. MERX provides several constraint mechanisms:

### max_daily_spend_trx

The maximum total TRX the standing order can spend in a rolling 24-hour period.

```json
{
  "max_daily_spend_trx": 200
}
```

Once this limit is reached, the trigger continues to fire but the action is suppressed until the 24-hour window rolls forward.

### max_executions_per_day

The maximum number of times the action can execute in a rolling 24-hour period.

```json
{
  "max_executions_per_day": 5
}
```

This prevents rapid-fire execution during volatile periods. Even if the price bounces above and below the threshold 20 times in an hour, the action executes at most 5 times per day.

### min_interval_minutes

The minimum time between consecutive executions.

```json
{
  "min_interval_minutes": 60
}
```

This enforces a cooldown period. After the action executes, the trigger is suppressed for 60 minutes regardless of conditions.

### expires_at

The standing order automatically deactivates after this timestamp.

```json
{
  "expires_at": "2026-06-30T00:00:00Z"
}
```

Always set an expiration on standing orders. An orphaned standing order that runs indefinitely after you've forgotten about it is a liability.

### Total Budget

A hard cap on total spending across the lifetime of the standing order.

```json
{
  "max_total_spend_trx": 5000
}
```

Once the lifetime spend reaches this limit, the standing order is permanently deactivated.

## Каналы уведомлений

Standing orders can send notifications on trigger activation, successful execution, execution failure, and budget limit reached.

### Webhook

```json
{
  "notification_channels": [
    {
      "type": "webhook",
      "url": "https://your-app.com/hooks/merx",
      "headers": { "Authorization": "Bearer your-secret" }
    }
  ]
}
```

The webhook receives a JSON payload with the trigger details, action taken, and execution result.

### Telegram

```json
{
  "notification_channels": [
    {
      "type": "telegram",
      "chat_id": "-1001234567890"
    }
  ]
}
```

MERX sends a formatted message to the specified Telegram chat. Useful for teams that monitor operations through Telegram groups.

## Управление постоянными ордерами

### List Active Orders

```
Tool: list_standing_orders

Response:
{
  "standing_orders": [
    {
      "id": "so_abc123",
      "trigger": { "type": "price_below", "threshold": 0.00005 },
      "action": { "type": "buy_resource", "energy_amount": 500000 },
      "status": "active",
      "executions_today": 1,
      "total_spent_trx": 247.50,
      "created_at": "2026-03-15T10:00:00Z",
      "last_executed": "2026-03-30T02:15:00Z"
    }
  ]
}
```

### Pause and Resume

Standing orders can be paused without deleting them. A paused order retains its configuration and execution history but does not fire.

### Delete

Permanently removes the standing order. Execution history is retained for audit purposes.

## Практические паттерны

### Pattern 1: Cost-Optimized 24/7 Coverage

For applications that need continuous energy coverage at the lowest possible cost:

```
Standing Order 1:
  Trigger: price_below 0.000045 TRX/energy
  Action: buy 1,000,000 energy for 24 hours
  Constraint: max 2 executions/day

Standing Order 2:
  Trigger: balance_below 200,000 energy
  Action: ensure_resources to 500,000 energy (1 hour)
  Constraint: max 100 TRX/day
```

Order 1 buys large amounts of cheap energy when prices dip, providing 24-hour coverage. Order 2 is the safety net - if prices never dip low enough to trigger Order 1, Order 2 ensures the address never runs dry by buying at market price when the balance gets critically low.

### Pattern 2: Business Hours Optimization

For applications with predictable peak hours:

```
Standing Order 1:
  Trigger: schedule "0 7 * * 1-5" (7 AM UTC, weekdays)
  Action: buy 2,000,000 energy for 12 hours
  Constraint: max 200 TRX/day

Standing Order 2:
  Trigger: schedule "0 19 * * 1-5" (7 PM UTC, weekdays)
  Action: buy 500,000 energy for 14 hours
  Constraint: max 50 TRX/day
```

Heavy energy before business hours, light energy for overnight. Weekend orders can be separate with lower amounts.

### Pattern 3: Price Alert Only

For teams that want human decision-making with automated monitoring:

```
Standing Order:
  Trigger: price_below 0.000040 TRX/energy
  Action: notify_only
  Channels: webhook + telegram
  Message: "Energy at historic low - consider manual bulk purchase"
```

No automated purchasing. The team gets alerted when prices are exceptionally low and can decide whether to place a large manual order.

### Pattern 4: Auto-Scaling for Variable Load

For applications where transaction volume varies unpredictably:

```
Standing Order 1:
  Trigger: balance_below 500,000 energy
  Action: ensure_resources to 1,000,000 energy (1 hour)
  Constraint: min_interval 15 minutes, max 500 TRX/day

Standing Order 2:
  Trigger: balance_below 100,000 energy
  Action: ensure_resources to 500,000 energy (1 hour)
  Constraint: min_interval 5 minutes, max 1000 TRX/day
```

Order 1 handles normal load with moderate top-ups. Order 2 is the high-urgency backstop for traffic spikes, with a shorter interval and higher budget.

## Standing Orders and Agent Integration

For AI agents, standing orders are the mechanism for persistent behavior. An agent session is temporary - when the MCP connection closes, the agent can no longer monitor prices or buy energy. Standing orders run on the MERX server, independent of any client connection.

An agent can set up standing orders during one session and benefit from them across all future sessions:

```
Session 1 (setup):
  Agent creates standing orders for price-based purchasing
  Agent creates balance monitors with auto-renewal

Session 2 (days later):
  Agent checks standing order execution history
  Energy has been purchased automatically 14 times
  All transactions executed with delegated energy
  Zero manual intervention required
```

This is the path to truly autonomous blockchain operations. The agent defines the rules once, and the rules execute continuously.

## Заключение

Manual energy management is a solved problem. Standing orders replace human monitoring with automated rules that execute faster, more consistently, and at lower cost than any manual process.

Define your triggers. Set your constraints. Let MERX handle the rest.

Your application runs 24/7. Your energy procurement should too.

---

**Ссылки:**
- Платформа MERX: [https://merx.exchange](https://merx.exchange)
- MCP Server (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP Server (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)
