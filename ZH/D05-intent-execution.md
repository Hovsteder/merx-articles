# 意图执行:TRON 上 AI 智能体的多步计划

## 逐步执行的问题

AI 智能体在 TRON 上交互时面临一个结构性问题,尤其是当任务涉及多个链上操作时。考虑一个简单的场景:智能体需要向 Alice 发送 100 USDT,然后将 50 TRX 交换为 USDT。每个操作都需要自己的能量购买、自己的委托等待和自己的广播周期。

使用逐步方法,智能体至少需要 8 次工具调用:

1. 估算 USDT 转账的能量
2. 为 USDT 转账购买能量
3. 等待委托
4. 执行 USDT 转账
5. 估算交换的能量
6. 为交换购买能量
7. 等待委托
8. 执行交换

每次能量购买都是一次独立的市场交互,有自己的交易开销。每次委托等待增加 3-6 秒的延迟。总耗时可能超过 30 秒,而这本应是一个简单的两步任务。

MERX 通过意图执行解决了这个问题 - 一个系统,接受 AI 智能体的多步计划,模拟每一步,优化所有步骤的资源购买,并按顺序执行整个计划。

## 什么是意图

在 MERX 系统中,意图是对智能体想要完成的事情的声明性描述,表达为一个有序的操作列表。智能体指定期望的结果,MERX 处理执行机制。

意图与一系列工具调用有三个重要区别:

1. **资源优化** - MERX 可以跨所有步骤批量购买能量,在单次订单中购买所需的总能量,而不是逐步购买。

2. **预验证** - 在任何步骤执行之前,每一步都会被模拟。如果 5 步计划的第 3 步会失败,智能体在第 1 步广播之前就知道了。

3. **原子化规划** - 智能体一次提交整个计划,让 MERX 了解完整的工作范围。这使得在逐步提交时不可能实现的优化成为可能。

## execute_intent 工具

MCP 服务器通过 `execute_intent` 工具暴露意图执行:

```
Tool: execute_intent
Input: {
  "steps": [
    {
      "action": "transfer_trc20",
      "params": {
        "to": "TAliceAddress...",
        "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "100000000"
      }
    },
    {
      "action": "swap",
      "params": {
        "from_token": "TRX",
        "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "50000000",
        "slippage": 0.5
      }
    }
  ],
  "resource_strategy": "batch_cheapest"
}
```

响应包括每一步的模拟结果、总资源成本,以及完成后每一步的执行状态。

## 支持的操作

意图系统支持以下操作类型:

### transfer_trx

向地址发送 TRX。这是一个原生转账,消耗带宽但不消耗能量。

```json
{
  "action": "transfer_trx",
  "params": {
    "to": "TRecipient...",
    "amount_sun": 1000000
  }
}
```

### transfer_trc20

向地址发送 TRC20 代币(USDT、USDC 等)。消耗智能合约调用的能量。

```json
{
  "action": "transfer_trc20",
  "params": {
    "to": "TRecipient...",
    "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "amount": "100000000"
  }
}
```

### swap

在 SunSwap V2 上执行代币交换。包括对特定交换参数的精确能量模拟。

```json
{
  "action": "swap",
  "params": {
    "from_token": "TRX",
    "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "amount": "100000000",
    "slippage": 0.5
  }
}
```

### approve

为 TRC20 代币设置消费授权。在交换代币前必需(交换 TRX 时不需要)。

```json
{
  "action": "approve",
  "params": {
    "token_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "spender": "TRouterAddress...",
    "amount": "unlimited"
  }
}
```

### call_contract

执行任意智能合约调用。这是对特定操作类型未覆盖的操作的后备方案。

```json
{
  "action": "call_contract",
  "params": {
    "contract_address": "TContractAddress...",
    "function_selector": "stake(uint256)",
    "parameters": [{ "type": "uint256", "value": "1000000" }],
    "call_value": 0
  }
}
```

### buy_resource

作为计划中的一步购买能量或带宽。当智能体想要显式控制资源时序时使用。

```json
{
  "action": "buy_resource",
  "params": {
    "resource_type": "energy",
    "amount": 130000,
    "duration_hours": 1
  }
}
```

## 资源策略

`resource_strategy` 参数控制 MERX 如何处理意图各步骤的能量购买。

### batch_cheapest

这是默认和推荐的策略。MERX 模拟所有步骤,合计所需的总能量,减去可用资源,并为整个意图进行一次能量购买。

```
步骤 1 (transfer_trc20): 64,895 能量
步骤 2 (swap):           223,354 能量
总需求:                  288,249 能量
当前可用:                0 能量
购买:                    290,000 能量 (取整到订单单位)
```

一次购买。一次委托等待。然后所有步骤使用汇集的能量按顺序执行。

优势:
- 单次市场交互(更低的开销)
- 单次委托等待(更低的延迟)
- 大额订单可能获得批量折扣
- 更简单的故障处理

### per_step

每一步独立购买自己的能量。当步骤有条件性或需要最小化风险时使用(如果步骤 1 失败,你还没有为步骤 2 购买能量)。

```
步骤 1: 购买 65,000 能量 -> 等待 -> 执行转账
步骤 2: 购买 225,000 能量 -> 等待 -> 执行交换
```

这种策略更慢(两次委托等待),但如果执行在计划中途停止,则浪费的能量更少。

## 有状态模拟

意图系统的模拟引擎在各步骤之间维护状态。这对于后续步骤依赖前面步骤结果的计划至关重要。

考虑这个意图:"将 50 TRX 交换为 USDT,然后将收到的 USDT 发送给 Alice。"

模拟引擎:

1. 模拟步骤 1(交换)。结果:智能体收到 16.42 USDT。
2. 更新模拟状态以反映新的 USDT 余额。
3. 对照更新后的状态模拟步骤 2(转账 16.42 USDT 给 Alice)。
4. 确认步骤 2 使用步骤 1 的余额可以成功。

没有有状态模拟,步骤 2 会对照智能体的当前余额(可能不包括交换得到的 USDT)进行模拟。模拟会错误地报告步骤 2 将因余额不足而失败。

```
Tool: execute_intent
Input: {
  "steps": [
    {
      "action": "swap",
      "params": {
        "from_token": "TRX",
        "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "50000000",
        "slippage": 0.5
      }
    },
    {
      "action": "transfer_trc20",
      "params": {
        "to": "TAliceAddress...",
        "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "use_previous_output"
      }
    }
  ],
  "resource_strategy": "batch_cheapest"
}
```

`use_previous_output` 参数告诉意图系统使用前一步的输出金额作为此步骤的输入金额。

## 模拟响应

在执行开始之前,意图系统返回一个模拟摘要:

```json
{
  "simulation": {
    "steps": [
      {
        "action": "transfer_trc20",
        "energy_required": 64895,
        "bandwidth_required": 345,
        "simulated_success": true,
        "estimated_cost_trx": 3.42
      },
      {
        "action": "swap",
        "energy_required": 223354,
        "bandwidth_required": 420,
        "simulated_success": true,
        "estimated_output": "16.42 USDT",
        "estimated_cost_trx": 11.76
      }
    ],
    "total_energy": 288249,
    "total_bandwidth": 765,
    "total_cost_trx": 15.18,
    "resource_purchase": {
      "energy": 290000,
      "price": 15.24,
      "provider": "sohu"
    }
  },
  "status": "ready_to_execute"
}
```

智能体在任何链上操作之前就能看到带有成本的完整计划。如果成本不可接受或某一步会失败,智能体可以在不花费任何费用的情况下修改计划。

## 执行和错误处理

一旦智能体确认计划(或启用自动执行),意图逐步执行:

```json
{
  "execution": {
    "steps": [
      {
        "action": "transfer_trc20",
        "status": "completed",
        "tx_hash": "abc123...",
        "energy_used": 64895,
        "block": 58234567
      },
      {
        "action": "swap",
        "status": "completed",
        "tx_hash": "def456...",
        "energy_used": 223354,
        "output_amount": "16.42",
        "block": 58234568
      }
    ],
    "total_energy_used": 288249,
    "total_energy_purchased": 290000,
    "energy_wasted": 1751,
    "status": "all_steps_completed"
  }
}
```

### 执行中途失败

如果某一步在执行期间(而非模拟期间)失败,意图系统停止并报告失败:

```json
{
  "execution": {
    "steps": [
      {
        "action": "transfer_trc20",
        "status": "completed",
        "tx_hash": "abc123..."
      },
      {
        "action": "swap",
        "status": "failed",
        "error": "SLIPPAGE_EXCEEDED",
        "message": "输出 15.89 USDT 低于最低值 16.34 USDT"
      }
    ],
    "status": "partial_execution",
    "completed_steps": 1,
    "failed_step": 2,
    "remaining_energy": 223354
  }
}
```

步骤 1 已在链上提交,无法撤销。智能体收到剩余的能量余额,可以决定如何继续 - 使用调整后的参数重试失败的步骤,执行其他操作,或让能量过期。

## 真实示例:资金库再平衡

以下是智能体可能为资金库管理执行的真实多步意图:

"将 1,000 TRX 交换为 USDT,向运营钱包发送 300 USDT,向营销钱包发送 200 USDT,保留其余部分。"

```
Tool: execute_intent
Input: {
  "steps": [
    {
      "action": "swap",
      "params": {
        "from_token": "TRX",
        "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "1000000000",
        "slippage": 1.0
      }
    },
    {
      "action": "transfer_trc20",
      "params": {
        "to": "TOpsWallet...",
        "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "300000000"
      }
    },
    {
      "action": "transfer_trc20",
      "params": {
        "to": "TMarketingWallet...",
        "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "200000000"
      }
    }
  ],
  "resource_strategy": "batch_cheapest"
}
```

模拟:

```
步骤 1 (swap):     223,354 能量
步骤 2 (transfer): 29,631 能量  (OpsWallet 已有 USDT)
步骤 3 (transfer): 64,895 能量  (MarketingWallet 是 USDT 新地址)
总计:              317,880 能量

批量购买: 320,000 能量,16.83 TRX,来自 catfee

不使用意图批量处理: 3 次单独购买 = 约 18.20 TRX
使用意图批量处理: 1 次购买 = 16.83 TRX
批量处理节省: 1.37 TRX + 降低的延迟 (1 次等待 vs 3 次)
```

## 何时使用意图 vs 单独工具

使用 `execute_intent` 的场景:

- 任务涉及两个或更多链上操作
- 步骤之间有依赖关系(步骤 2 使用步骤 1 的输出)
- 你想通过批量处理最小化总资源成本
- 你需要在提交前预验证整个计划

使用单独工具的场景:

- 任务是单个操作
- 智能体需要根据外部输入在步骤之间做出决策
- 步骤之间有较长的时间间隔
- 智能体想要对每个执行阶段有最大控制权

## 意图与智能体自主性

意图系统专为智能体自主性而设计。一个收到"再平衡资金库"这样高层指令的智能体可以将其分解为具体步骤、构建意图、模拟、审查成本并执行 - 整个过程无需任何人工干预。

模拟步骤充当智能体的安全检查。在投入任何资金之前,智能体可以验证每一步都会成功,总成本在预算范围内,预期输出与期望结果匹配。这相当于人类在点击"确认"之前审查交易,但由智能体本身以编程方式执行。

结合常备订单用于周期性资源购买和监控器用于余额警报,意图系统使完全自主的链上操作成为可能,全天候运行而无需人类监督。

## 结论

单步执行是区块链自动化的入门阶段。真正的智能体工作流涉及多个操作、步骤间的依赖关系,以及跨完整计划的资源优化。

MERX 意图执行让 AI 智能体能够以计划的方式思考,而不是单个动作。模拟一切。优化全范围的资源。以每一步都经过预验证的信心执行。

区块链不是单一操作环境。你的智能体也不应该是。

---

**链接:**
- MERX 平台: [https://merx.exchange](https://merx.exchange)
- MCP 服务器 (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP 服务器 (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)
