# 30 个提示和 21 个资源:为什么 MERX 是唯一的全协议 MCP 服务器

## MCP 有三个原语,而非一个

Model Context Protocol 定义了三种服务器可以向 AI 客户端暴露的能力类型:

1. **工具 (Tools)** - 可执行的操作(发送交易、购买能量、执行交换)
2. **提示 (Prompts)** - 引导模型完成复杂工作流的预构建模板
3. **资源 (Resources)** - 模型可以读取的结构化数据(价格信息、网络参数、文档)

大多数 MCP 服务器只实现了工具。它们暴露少量可调用函数,就此收工,然后自称"兼容 MCP"。这在技术上是正确的,就像一辆有引擎但没有方向盘的车在技术上也是交通工具一样。

仅有工具让模型有了行动能力,但没有何时或如何使用它们的指导。没有提示,模型必须每次从零开始弄清楚复杂的多步工作流。没有资源,模型没有结构化数据可供参考 - 它必须执行工具调用才能读取本应被动可用的信息。

MERX 实现了全部三种原语:21 个工具、30 个提示和 21 个资源。本文解释每种原语的作用、为什么三者都很重要,以及 MERX 如何使用它们创建一个在质量上不同于仅工具实现的 MCP 服务器。

## 原语 1:工具(操作)

工具是最直观的原语。工具是模型可以调用来执行操作的函数。它有名称、描述、类型化参数,并返回结构化响应。

MERX 暴露 21 个工具,覆盖 TRON 区块链操作的完整范围:

### 钱包操作
- `set_private_key` - 使用私钥配置钱包(自动推导地址)
- `get_trx_balance` - 检查任意地址的 TRX 余额
- `get_trc20_balance` - 检查 TRC20 代币余额
- `transfer_trx` - 向地址发送 TRX
- `transfer_trc20` - 向地址发送 TRC20 代币
- `check_address_resources` - 查看能量和带宽分配

### 能量市场
- `get_prices` - 所有供应商的当前价格
- `get_best_price` - 查找最便宜的能量报价
- `create_order` - 从市场购买能量
- `create_paid_order` - 通过 x402 购买能量(无需账户)
- `ensure_resources` - 自动购买所需资源
- `list_orders` - 查看订单历史
- `get_order` - 获取特定订单的详情

### DEX 交易
- `get_swap_quote` - 获取带精确模拟的 SunSwap V2 报价
- `execute_swap` - 执行带自动资源管理的交换
- `approve_trc20` - 设置代币消费授权

### 自动化
- `create_standing_order` - 设置自动能量购买
- `list_standing_orders` - 查看活跃的常备订单
- `create_monitor` - 设置委托或余额监控器
- `list_monitors` - 查看活跃的监控器

### 高级
- `execute_intent` - 执行多步交易计划
- `estimate_contract_call` - 模拟任意合约调用以进行能量估算

每个工具都有 JSON Schema 定义的参数、自然语言描述和结构化返回类型。模型确切知道每个工具的功能、需要什么参数以及将返回什么。

但仅有工具是不够的。

## 原语 2:提示(模板)

提示是模型可以用来处理特定类型请求的预构建模板。可以将其视为配方:"当用户要求 X 时,这是分步方法,包括要调用哪些工具、以什么顺序,以及如何解读结果。"

MERX 提供 30 个提示,分布在 10 个类别中。

### 入门 (3 个提示)

**setup_wallet** - 引导用户完成钱包配置
**first_energy_purchase** - 引导首次能量购买者
**platform_overview** - 解释 MERX 的全部功能

### 能量管理 (4 个提示)

**buy_cheapest_energy** - 以最佳价格查找和购买能量
**compare_providers** - 详细的供应商比较
**optimize_costs** - 分析支出并建议优化
**ensure_resources_for_tx** - 为特定交易准备资源

### 交易执行 (4 个提示)

**send_usdt** - 带资源优化的完整 USDT 转账
**send_trx** - TRX 转账(更简单 - 不需要能量)
**swap_tokens** - 带完整管道的 DEX 交换
**execute_complex_intent** - 多步交易计划

### 自动化 (4 个提示)

**setup_standing_order** - 配置自动能量购买
**setup_delegation_monitor** - 配置能量委托的自动续期
**setup_balance_monitor** - 配置基于余额的警报和操作
**review_automation** - 审计和优化现有自动化

### 分析 (4 个提示)

**analyze_prices** - 市场价格分析
**calculate_savings** - 计算使用委托能量的节省
**estimate_transaction_cost** - 任何交易类型的成本估算
**portfolio_overview** - 完整的账户概览

### 账户管理 (3 个提示)

**account_setup**、**deposit_guide**、**withdrawal_guide** - 账户生命周期操作的分步模板。

### x402 (2 个提示)

**x402_purchase** - 引导零注册购买流程。
**x402_explain** - 解释 x402 的工作原理和使用时机。

### 网络信息 (2 个提示)

**explain_energy**、**explain_bandwidth** - 使用资源数据解释 TRON 资源模型的教育模板。

### 故障排除 (2 个提示)

**transaction_failed**、**delegation_not_arrived** - 帮助识别和解决常见问题的诊断模板。

### 集成 (2 个提示)

**sdk_quickstart**、**api_integration** - 面向将 MERX 集成到应用程序中的开发者的模板。

### 为什么提示重要

没有提示,收到"帮我购买能量"请求的模型需要:
1. 弄清哪些工具相关
2. 确定正确的操作顺序
3. 决定先收集什么信息
4. 处理它可能不了解的边缘情况

有了 `buy_cheapest_energy` 提示,模型就有了一个经过测试、优化的工作流,可以处理边缘情况,以正确的格式呈现信息,并遵循正确的操作顺序。区别就像只给人一套工具和同时给人工具加手册。

## 原语 3:资源(数据)

资源是模型可以在不执行操作调用的情况下读取的结构化数据。与执行操作的工具不同,资源提供信息 - 它们使信息被动可用。

MERX 提供 21 个资源:14 个静态和 7 个动态模板。

### 静态资源 (14 个)

静态资源提供在会话之间不变的固定参考数据:

```
merx://docs/getting-started
merx://docs/energy-explained
merx://docs/bandwidth-explained
merx://docs/resource-model
merx://docs/providers-overview
merx://docs/x402-protocol
merx://docs/standing-orders-guide
merx://docs/monitors-guide
merx://docs/intent-execution-guide
merx://docs/api-reference
merx://docs/sdk-js
merx://docs/sdk-python
merx://docs/faq
merx://docs/troubleshooting
```

### 动态模板资源 (7 个)

模板资源接受参数并返回特定于请求的数据:

```
merx://prices/current
merx://prices/history/{period}
merx://providers/{provider_name}
merx://account/{address}/resources
merx://account/{address}/balances
merx://network/parameters
merx://network/chain-info
```

### 为什么资源重要

资源为模型提供上下文而无需工具调用的开销。当用户问"我的能量余额是多少?"时,模型可以直接查看资源 `merx://account/{address}/resources`。当模型在创建常备订单之前需要参考文档时,它读取 `merx://docs/standing-orders-guide`。

没有资源,每条信息都需要工具调用。区别在于一个必须主动请求每条数据的模型(仅工具)和一个拥有丰富信息环境可供参考的模型(工具 + 资源)。

## 全协议优势

当三种原语协同工作时,模型在根本不同的层面上运作:

### 场景:用户要求"帮我设置能量自动化"

**仅工具 MCP 服务器:**
1. 模型猜测哪些自动化工具存在
2. 以试错方式调用工具
3. 可能遗漏重要配置选项
4. 没有最佳实践指导
5. 没有背景参考资料

**全协议 MERX MCP 服务器:**
1. 模型读取 `merx://docs/standing-orders-guide` 获取参考资料
2. 模型激活 `setup_standing_order` 提示获取分步工作流
3. 模型读取 `merx://prices/history/7d` 推荐价格阈值
4. 模型使用优化参数调用 `create_standing_order` 工具
5. 模型读取 `merx://docs/monitors-guide` 并建议互补的监控器
6. 模型调用 `create_monitor` 工具进行委托到期保护
7. 结果:完整、配置良好的自动化,决策有文档支持

全协议方法不仅是略好 - 它产生质的不同的结果,因为模型同时拥有指导(提示)、知识(资源)和能力(工具)。

## 其他 MCP 服务器缺少什么

截至 2026 年初对区块链相关 MCP 服务器的调查显示了一致的模式:

| 服务器 | 工具 | 提示 | 资源 | 完整协议 |
|---|---|---|---|---|
| 通用 ETH MCP | 5-8 | 0 | 0 | 否 |
| Solana MCP | 10-12 | 0 | 0 | 否 |
| Bitcoin MCP | 3-4 | 0 | 0 | 否 |
| 多链 MCP | 15-20 | 0 | 0 | 否 |
| MERX | 21 | 30 | 21 | 是 |

没有其他区块链 MCP 服务器实现提示或资源。它们都是仅工具的,这意味着:

- 多步操作没有工作流指导
- 模型推理时没有文档可用
- 没有用于上下文和参考的被动数据访问
- 每次交互都从零开始,没有结构性知识

MERX 是唯一一个模型拥有完整运营环境的 MCP 服务器 - 行动能力(工具)、如何行动的知识(提示)和为决策提供信息的数据(资源)。

## 数据

MERX 的数据:

- **21 个工具**,跨越 5 个类别(钱包、能量市场、DEX、自动化、高级)
- **30 个提示**,跨越 10 个类别(入门、能量、交易、自动化、分析、账户、x402、网络、故障排除、集成)
- **21 个资源**(14 个静态文档、7 个动态数据模板)
- 通过单个 MCP 服务器暴露的 **72 项总能力**

对于连接到 MERX 的 AI 智能体,这意味着:
- 它可以执行任何 TRON 操作(工具)
- 它知道任何场景的最佳方法(提示)
- 它始终拥有实时数据和文档(资源)

## 结论

MCP 是一个有三种原语的协议,而非一种。仅实现工具就像构建一个有端点但没有文档和数据源的 API。它在技术上可以工作,但迫使消费者自己弄清一切。

MERX 实现了完整协议,因为完整协议是 MCP 被设计的使用方式。工具用于行动。提示用于指导。资源用于知识。三者共同工作,创建一个 AI 智能体可以像拥有多年经验的人类开发者一样在 TRON 区块链上运营的环境。

三十个提示。二十一个资源。二十一个工具。没有其他区块链 MCP 服务器能接近。

---

**链接:**
- MERX 平台: [https://merx.exchange](https://merx.exchange)
- MCP 服务器 (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP 服务器 (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)
