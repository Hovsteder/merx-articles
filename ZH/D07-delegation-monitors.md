# 委托监控器:永不让你的 TRON 能量过期

## 过期问题

TRON 能量委托有固定期限。当你购买 1 小时的能量时,你恰好得到 1 小时。当那一小时结束时,委托被撤销,你的地址回到零委托能量。如果你的应用程序在委托开始后第 61 分钟发送 USDT 转账,该交易将以全价燃烧 TRX。

对于全天候运行的应用程序,这造成了管理负担。你需要追踪每个委托何时到期,在当前委托失效之前购买替换,并处理到期和新委托到达之间的间隙。即使续期延迟几秒钟,在该窗口内执行的交易也会产生完整的 TRX 燃烧费用。

MERX 委托监控器消除了这个问题。它们监视你的委托,追踪到期时间,并在能量耗尽之前自动续期。监控器在 MERX 服务器上全天候运行 - 不依赖于你的应用程序是否在线、MCP 客户端是否连接或 AI 智能体是否活跃。

## 监控类型

MERX 监控系统支持三种类型的监控器,每种针对不同的运营关注点。

### delegation_expiry

监视活跃的能量委托并在它们到期前采取行动。

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

此监控器追踪到指定地址的所有活跃委托。当任何委托距到期不到 10 分钟时,监控器采取行动:

1. 如果 `auto_renew` 为 true 且当前最佳价格等于或低于 `max_price_per_unit`,它会自动下一个新订单,购买 `renewal_amount` 能量,持续时间为 `renewal_duration_hours`。
2. 如果价格超过 `max_price_per_unit`,监控器发送通知而不购买,提醒你价格过高无法自动续期。
3. 如果 `auto_renew` 为 false,监控器只发送通知,让你有时间手动操作。

`renew_before_minutes` 参数至关重要。设置得足够高,以便在旧委托到期之前完成新委托的下单、确认和激活。10 分钟的窗口为下单(秒级)、供应商处理(1-2 分钟)和委托确认(3-6 秒)提供了充足时间,还有应对网络拥堵的余量。

### balance_threshold

监控链上能量余额,当降至指定水平以下时触发。

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

与基于时间的 `delegation_expiry` 不同,`balance_threshold` 基于消耗。当地址的可用能量降至 200,000 以下时触发,不管原因。这覆盖了到期监控无法处理的场景:

- 多笔交易消耗能量的速度超出预期
- 供应商提前撤销委托
- 质押参数变更导致质押能量减少

触发时,`ensure_resources` 操作检查当前余额,计算达到目标的差额,只购买所需的部分。

### price_alert

监控能量市场价格,在满足条件时通知。

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

价格提醒默认是信息性的 - 通知但不购买。适用于想在历史低价买入但倾向于让人参与购买决策的团队。

`cooldown_minutes` 参数防止通知疲劳。如果价格在阈值以下持续数小时,你每 6 小时收到一次提醒,而不是每 30 秒一次。

## 自动续期详情

`delegation_expiry` 监控器的自动续期流程如下:

### 步骤 1:追踪活跃委托

MERX 维护通过平台下的所有能量订单的记录。每个订单包括委托开始时间、持续时间和到期时间。监控器将这些记录与当前时间进行对照检查。

```
TYourAddress 的活跃委托:
  订单 #1234: 500,000 能量,到期 2026-03-30T14:00:00Z (剩余 47 分钟)
  订单 #1235: 200,000 能量,到期 2026-03-30T15:30:00Z (剩余 137 分钟)
```

### 步骤 2:评估续期窗口

当任何委托进入续期窗口(例如到期前 10 分钟):

```
订单 #1234: 500,000 能量,9 分 42 秒后到期
-> 在续期窗口内 (10 分钟)
-> 启动续期检查
```

### 步骤 3:价格检查

监控器查询当前市场价格:

```
500,000 能量 / 24 小时的最佳价格:
  供应商: sohu
  价格: 26.30 TRX
  单价: 0.0000526

最大允许: 0.00006/单位
  0.0000526 <= 0.00006: 通过
```

### 步骤 4:下续期订单

```
订单已下达:
  数量: 500,000 能量
  持续时间: 24 小时
  供应商: sohu
  成本: 26.30 TRX
  目标: TYourAddress...
  状态: 已确认
```

### 步骤 5:验证委托

监控器轮询地址以确认新委托已收到:

```
委托已确认:
  之前的能量: 487,231 (即将到期的委托剩余)
  新的能量: 987,231 (之前 + 新委托)
```

### 步骤 6:通知

```
Webhook 已发送:
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

整个流程在 30 秒内完成。地址从未经历过能量覆盖的间隙。

## 价格保护

自动续期监控器上的 `max_price_per_unit` 参数是一个关键的安全机制。能量价格在高需求期可能飙升。没有价格保护,价格飙升期间的自动续期可能花费正常费率的 2-3 倍。

当市场价格超过最大值时:

```
500,000 能量 / 24 小时的最佳价格:
  供应商: catfee
  价格: 42.50 TRX
  单价: 0.0000850

最大允许: 0.00006/单位
  0.0000850 > 0.00006: 失败 - 价格超过最大值

操作: 发送通知而非购买
  "能量委托将在 8 分钟内到期。自动续期已跳过:
   市场价格 0.0000850 超过最大值 0.00006。需要手动操作。"
```

通知给你以下选项:
- 接受更高价格并手动下单
- 等待价格恢复正常并接受短暂的覆盖间隙
- 调整监控器上的 max_price_per_unit

### 设置合理的最高价格

要设置有效的最高价格:

1. 查看 `get_price_history` 资源获取过去 30 天的数据
2. 确定第 95 百分位价格(95% 的报价等于或低于该价格)
3. 将最高价格设置在此水平或略高

这种方法捕捉正常波动同时拒绝真正的价格飙升。

## 无需智能体的全天候运行

这是 MERX 监控器与智能体端逻辑相比的关键差异化因素。AI 智能体在对话会话期间运行。会话结束时,智能体停止。如果你在智能体代码中实现了委托追踪,它只在智能体活跃时才有效。

MERX 监控器在 MERX 服务器基础设施上运行:

- **PostgreSQL 持久性** - 监控器配置存储在数据库中,可在服务器重启后存活
- **服务端评估** - 触发器由 MERX 后端进程评估,不由任何客户端评估
- **独立于 MCP 连接** - 无需客户端连接即可使监控器运行
- **崩溃恢复** - 如果 MERX 服务重启,监控器从其最后已知状态自动恢复

智能体只需创建一次监控器。该监控器无限期运行(或直到其到期日期),不管智能体是否再次连接。

```
第 1 天: 智能体创建 delegation_expiry 监控器
第 2 天: 智能体离线。监控器在 14:00 续期委托。
第 3 天: 智能体离线。监控器在 13:55 续期委托。
第 7 天: 智能体重新连接。检查监控器历史:
  - 6 次成功的自动续期
  - 0 次能量覆盖间隙
  - 总花费: 157.80 TRX
  - 平均价格: 0.0000526/能量单位
```

## 组合监控器实现稳健覆盖

单一监控类型无法覆盖所有故障模式。生产环境推荐的配置结合两到三种监控类型:

### 推荐设置

```
监控器 1: delegation_expiry
  目的: 到期前主动续期
  配置:
    renew_before_minutes: 10
    auto_renew: true
    max_price: 0.00006
    renewal_amount: 500,000
    renewal_duration: 24 小时

监控器 2: balance_threshold
  目的: 捕捉意外的能量耗尽
  配置:
    energy_threshold: 100,000
    action: ensure_resources
    energy_target: 500,000

监控器 3: price_alert
  目的: 低价时机会购买
  配置:
    condition: below
    threshold: 0.000035
    cooldown: 360 分钟
    action: notify_only
```

监控器 1 处理正常情况 - 以可接受的价格进行计划续期。监控器 2 处理异常情况 - 突然的能量消耗高峰、提前撤销委托,或监控器 1 被价格保护阻止。监控器 3 提醒你手动大量购买的特殊机会。

三个监控器共同提供:
- 正常条件下的零间隙能量覆盖
- 价格暂时飙升时的自动后备
- 成本优化机会的提醒

## 管理监控器

### 列出活跃监控器

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

### 监控器历史

每个监控器维护详细的执行日志:

```json
{
  "history": [
    {
      "timestamp": "2026-03-30T02:00:12Z",
      "trigger_reason": "委托将在 9 分 48 秒后到期",
      "action_taken": "auto_renew",
      "order_id": "ord_xyz789",
      "energy_purchased": 500000,
      "cost_trx": 26.30,
      "provider": "sohu",
      "status": "success"
    },
    {
      "timestamp": "2026-03-29T01:55:33Z",
      "trigger_reason": "委托将在 9 分 27 秒后到期",
      "action_taken": "notification_only",
      "reason": "价格 0.0000780 超过最大值 0.0000600",
      "status": "skipped"
    }
  ]
}
```

此历史提供完整的可审计性。你可以看到每次续期的确切时间、花费、使用的供应商,以及为何跳过了某些续期。

## 永不过期的经济学

考虑一个每天处理 500 笔 USDT 转账的应用程序。每笔转账大约需要 65,000 能量。

不使用监控器(手动管理,偶尔有间隙):

```
每周平均间隙: 3 次 (每次约 15 分钟)
间隙期间的交易: 约 15 笔
间隙期间燃烧的 TRX: 约 15 x 27 = 405 TRX/周
间隙导致的年度燃烧: 约 21,060 TRX (约 $5,475)
```

使用 MERX 委托监控器:

```
每周间隙: 0
间隙导致的 TRX 燃烧: 0
监控器成本(自动续期): 约 0 额外 (无论如何都会购买同样的能量)
年度节省: 约 $5,475
```

监控器不产生额外成本。你无论如何都在购买同样的能量 - 监控器只是确保购买之间没有间隙。节省完全来自消除未管理到期窗口期间的 TRX 燃烧。

## 结论

能量委托会到期。这是 TRON 网络无法避免的事实。但可以避免的是让它们在没有准备好替代的情况下到期所带来的成本。

MERX 委托监控器将手动且容易出错的过程转变为自动化、可靠的系统。它们在服务器上运行,独立于任何客户端连接。它们在委托到期前续期。它们尊重你的价格限制。它们通知你异常情况。

设置一次。再也不用考虑委托到期的问题。

---

**链接:**
- MERX 平台: [https://merx.exchange](https://merx.exchange)
- MCP 服务器 (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP 服务器 (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)
