# 常备订单:全天候自动化 TRON 能量购买

## 手动方式无法扩展

如果你手动购买 TRON 能量,你的做法是错误的。不是因为过程困难 - 下单只需几秒钟 - 而是因为能量市场是动态的。价格全天波动。委托按固定时间表到期。你的应用程序的能量需求根据交易量而变化。手动管理所有这些意味着要么持续保持警惕,要么浪费金钱。

常备订单通过让你定义自动执行的规则来解决这个问题,全天候、全年无休。当条件满足时 - 价格降至阈值以下、计划触发,或能量余额降得过低 - MERX 会代你使用预定义的参数下单。

本文涵盖完整的常备订单系统:触发类型、操作类型、预算控制,以及最能节省成本的实用模式。

## 常备订单的工作原理

常备订单是存储在 MERX PostgreSQL 数据库中的持久规则。它由三个组件构成:

1. **触发器** - 激活订单的条件
2. **操作** - 触发时执行的动作
3. **约束** - 预算限制、执行频率和过期时间

MERX 后端持续评估触发器。价格触发器在每次从任何供应商收到新价格报价时检查(通常每 30 秒)。计划触发器使用由服务器评估的 cron 表达式。余额触发器通过定期轮询链上状态进行检查。

当触发器触发且所有约束都满足时,操作立即执行,无需任何手动干预。

## 创建常备订单

MCP 服务器通过 `create_standing_order` 工具暴露常备订单:

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

此订单将在市场价格降至每能量单位 0.00005 TRX 以下时购买 500,000 能量(24 小时),每天最多 3 次,每天花费不超过 100 TRX。

## 触发类型

### price_below

当最佳可用能量价格降至指定阈值以下时触发。

```json
{
  "type": "price_below",
  "threshold": 0.00005,
  "resource_type": "energy"
}
```

这是成本优化最常用的触发器。TRON 上的能量价格根据供需波动。在非高峰时段(通常为 UTC 00:00-06:00),价格通常比峰值低 15-25%。`price_below` 触发器让你自动在这些窗口期购买能量,无需亲自监控价格。

**实用建议:** 将阈值设置在过去 7 天价格的第 25 百分位。这意味着你在价格处于近期区间的最低四分之一时购买 - 足够便宜以节省资金,又足够频繁以保持地址的供应。

### schedule

按 cron 计划触发,不考虑市场条件。

```json
{
  "type": "schedule",
  "cron": "0 */6 * * *"
}
```

此触发器每 6 小时触发一次。适用于能量需求可预测的应用程序 - 如果你知道应用程序在工作时间处理大部分交易,可以安排能量购买在高峰之前到达。

常见 cron 模式:

```
"0 */6 * * *"    - 每 6 小时
"0 8 * * 1-5"   - 每个工作日 8:00 UTC
"*/30 * * * *"   - 每 30 分钟
"0 0 * * *"      - 每天午夜 UTC
```

### balance_below

当地址的可用能量降至阈值以下时触发。

```json
{
  "type": "balance_below",
  "energy_threshold": 100000,
  "check_address": "TYourAddress..."
}
```

此触发器是反应式的 - 它通过在余额变低时自动补充来确保地址永远不会耗尽能量。结合 `buy_resource` 操作,它创建了一个自维护的能量供应。

**工作原理:** MERX 定期(默认每 60 秒)轮询指定地址的链上资源余额。当可用能量降至阈值以下时,触发器触发。

**重要注意事项:** 在余额下降和新能量到达之间存在固有的延迟(轮询间隔 + 下单 + 委托确认)。将阈值设置得足够高,以便你的地址在此窗口期间不会耗尽。如果你的应用程序每分钟消耗 50,000 能量,100,000 的阈值给你大约 2 分钟的缓冲 - 足以让 MERX 补充。

## 操作类型

### buy_resource

从市场购买能量或带宽。

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

这是能量采购的标准操作。MERX 自动在执行时选择最便宜的可用供应商。`target_address` 接收委托的能量。

### ensure_resources

在购买前检查当前资源的高级操作。

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

与 `buy_resource` 始终购买指定数量不同,`ensure_resources` 首先检查地址已有的资源,只购买差额。如果地址已有 300,000 能量且目标为 500,000,则只购买 200,000。

这是 `balance_below` 触发器更安全的操作,因为它防止了多个触发器快速连续触发时的过度购买。

### notify_only

发送通知而不采取任何链上操作。

```json
{
  "type": "notify_only",
  "params": {
    "channels": ["webhook", "telegram"],
    "message": "能量价格降至阈值以下"
  }
}
```

当你想要了解情况但不想自动化时使用。常备订单监控条件并提醒你,但你手动做出购买决策。这是对全自动购买还不放心的团队的良好起点。

## 预算限制和约束

没有约束的常备订单是危险的。在持续价格下跌期间,没有消费限制的 `price_below` 触发器可能会耗尽你的全部余额。MERX 提供了几种约束机制:

### max_daily_spend_trx

常备订单在滚动 24 小时期间可以花费的最大总 TRX。

```json
{
  "max_daily_spend_trx": 200
}
```

达到此限制后,触发器继续触发,但操作被抑制,直到 24 小时窗口向前滚动。

### max_executions_per_day

操作在滚动 24 小时期间可以执行的最大次数。

```json
{
  "max_executions_per_day": 5
}
```

这防止了波动时期的快速连续执行。即使价格在一小时内上下波动 20 次,操作每天最多执行 5 次。

### min_interval_minutes

连续执行之间的最小时间间隔。

```json
{
  "min_interval_minutes": 60
}
```

这强制执行冷却期。操作执行后,触发器在 60 分钟内被抑制,不管条件如何。

### expires_at

常备订单在此时间戳后自动停用。

```json
{
  "expires_at": "2026-06-30T00:00:00Z"
}
```

始终为常备订单设置过期时间。一个被遗忘后无限期运行的孤立常备订单是一个隐患。

### 总预算

常备订单生命周期内总花费的硬上限。

```json
{
  "max_total_spend_trx": 5000
}
```

一旦生命周期花费达到此限制,常备订单将被永久停用。

## 通知渠道

常备订单可以在触发激活、成功执行、执行失败和达到预算限制时发送通知。

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

Webhook 接收包含触发详情、采取的操作和执行结果的 JSON 载荷。

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

MERX 向指定的 Telegram 群发送格式化消息。适合通过 Telegram 群组监控运营的团队。

## 管理常备订单

### 列出活跃订单

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

### 暂停和恢复

常备订单可以在不删除的情况下暂停。暂停的订单保留其配置和执行历史,但不会触发。

### 删除

永久删除常备订单。执行历史保留用于审计目的。

## 实用模式

### 模式 1:成本优化的全天候覆盖

适用于需要以最低成本持续能量覆盖的应用程序:

```
常备订单 1:
  触发: price_below 0.000045 TRX/能量
  操作: 购买 1,000,000 能量 (24 小时)
  约束: 每天最多 2 次执行

常备订单 2:
  触发: balance_below 200,000 能量
  操作: ensure_resources 到 500,000 能量 (1 小时)
  约束: 每天最多 100 TRX
```

订单 1 在价格下跌时购买大量便宜能量,提供 24 小时覆盖。订单 2 是安全网 - 如果价格从未降得足够低以触发订单 1,订单 2 通过在余额临界低时以市场价购买来确保地址永不断电。

### 模式 2:工作时间优化

适用于有可预测高峰时段的应用程序:

```
常备订单 1:
  触发: schedule "0 7 * * 1-5" (7 AM UTC,工作日)
  操作: 购买 2,000,000 能量 (12 小时)
  约束: 每天最多 200 TRX

常备订单 2:
  触发: schedule "0 19 * * 1-5" (7 PM UTC,工作日)
  操作: 购买 500,000 能量 (14 小时)
  约束: 每天最多 50 TRX
```

工作时间前大量能量,夜间少量能量。周末订单可以单独设置更低的数量。

### 模式 3:仅价格提醒

适合希望由人来做购买决策并配合自动化监控的团队:

```
常备订单:
  触发: price_below 0.000040 TRX/能量
  操作: notify_only
  渠道: webhook + telegram
  消息: "能量处于历史低位 - 考虑手动大量购买"
```

没有自动购买。团队在价格异常低时收到提醒,可以决定是否进行大额手动订单。

### 模式 4:可变负载的自动扩展

适用于交易量不可预测变化的应用程序:

```
常备订单 1:
  触发: balance_below 500,000 能量
  操作: ensure_resources 到 1,000,000 能量 (1 小时)
  约束: 最小间隔 15 分钟,每天最多 500 TRX

常备订单 2:
  触发: balance_below 100,000 能量
  操作: ensure_resources 到 500,000 能量 (1 小时)
  约束: 最小间隔 5 分钟,每天最多 1000 TRX
```

订单 1 处理正常负载,适度补充。订单 2 是应对流量高峰的高优先级后备,有更短的间隔和更高的预算。

## 常备订单与智能体集成

对于 AI 智能体,常备订单是持久行为的机制。智能体会话是临时的 - 当 MCP 连接关闭时,智能体无法再监控价格或购买能量。常备订单在 MERX 服务器上运行,独立于任何客户端连接。

智能体可以在一个会话中设置常备订单,并在所有未来的会话中受益:

```
会话 1 (设置):
  智能体创建基于价格的购买常备订单
  智能体创建带自动续期的余额监控器

会话 2 (几天后):
  智能体检查常备订单执行历史
  能量已自动购买 14 次
  所有交易都使用委托能量执行
  零手动干预
```

这是通往真正自主区块链操作的道路。智能体定义一次规则,规则持续执行。

## 结论

手动能量管理是一个已解决的问题。常备订单用自动化规则取代了人工监控,执行速度更快、更一致,成本更低。

定义你的触发器。设置你的约束。让 MERX 处理其余的事情。

你的应用程序全天候运行。你的能量采购也应该如此。

---

**链接:**
- MERX 平台: [https://merx.exchange](https://merx.exchange)
- MCP 服务器 (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP 服务器 (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)

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
