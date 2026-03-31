# 为你的 AI 智能体配备 TRON 钱包

## 为什么 AI 智能体需要链上访问能力

关于 AI 智能体的讨论已经超越了聊天机器人和代码助手的范畴。下一个前沿领域是能够在区块链网络上自主行动的智能体 - 检查余额、发送交易、购买资源、管理投资组合,无需在每一步都进行人工干预。

TRON 是稳定币转账的基础设施。每天超过 500 亿 USDT 在 TRON 上流转,而该网络的资源模型 - 能量和带宽替代 gas 费用 - 使其特别适合高频、低成本的操作。但历来将 AI 智能体连接到 TRON 需要构建自定义集成、管理 RPC 端点、处理资源估算,以及应对 TronWeb 的复杂性。

MERX 通过单个 MCP 服务器解决了这个问题,为任何兼容的 AI 智能体 - Claude、Cursor 或任何 Model Context Protocol 客户端 - 提供了一个集钱包和资源交易所于一体的完整解决方案。

本文将引导你完成从零开始到一个能够持有密钥、检查余额并在 TRON 主网上购买能量的自主智能体的全过程。

## 什么是 MCP,为什么它很重要

Model Context Protocol (MCP) 是由 Anthropic 创建的开放标准,允许 AI 模型通过统一接口与外部工具、数据源和服务进行交互。可以将其视为 AI 的 USB 接口 - 任何兼容 MCP 的客户端都可以连接到任何 MCP 服务器,无需自定义集成代码。

MCP 服务器暴露三种原语:

- **工具 (Tools)** - 智能体可以执行的操作(发送 TRX、购买能量、执行交换)
- **提示 (Prompts)** - 引导智能体完成复杂工作流的预构建模板
- **资源 (Resources)** - 智能体可以读取的结构化数据(价格信息、网络参数、账户状态)

MERX 实现了全部三种原语,提供 21 个工具、30 个提示和 21 个资源。没有其他区块链 MCP 服务器能提供如此全面的覆盖。

## 安装

MERX MCP 服务器作为标准 npm 包发布。全局安装或使用 npx 直接运行:

```bash
npm install -g merx-mcp
```

或将其添加到你的 MCP 客户端配置中。对于 Claude Desktop,编辑 `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "merx": {
      "command": "npx",
      "args": ["-y", "merx-mcp"],
      "env": {
        "MERX_NETWORK": "mainnet"
      }
    }
  }
}
```

对于 Cursor,配置放在项目的 `.cursor/mcp.json` 中:

```json
{
  "mcpServers": {
    "merx": {
      "command": "npx",
      "args": ["-y", "merx-mcp"]
    }
  }
}
```

这就是全部设置。无需 API 密钥。无需注册。无需 OAuth 流程。

## 设置钱包

你的智能体首先需要一个 TRON 地址。MERX 提供了 `set_private_key` 工具,接受十六进制编码的私钥并自动推导出对应的 TRON 地址。

```
Tool: set_private_key
Input: { "private_key": "your_64_char_hex_private_key" }

Response:
{
  "address": "TYourDerivedTronAddress...",
  "network": "mainnet",
  "status": "ready"
}
```

私钥绝不会离开本地机器。它存储在 MCP 服务器的运行时内存中,仅在会话期间有效,专门用于在广播前在本地签署交易。MERX 服务器永远不会看到你的私钥 - 所有签名操作都在客户端完成。

### 生成新钱包

如果需要为智能体创建新钱包,可以使用任何兼容 TRON 的工具。智能体本身也可以使用标准密码学库创建:

```javascript
const TronWeb = require('tronweb');
const account = TronWeb.utils.accounts.generateAccount();
console.log('Address:', account.address.base58);
console.log('Private Key:', account.privateKey);
```

用少量 TRX 为地址充值以激活账户和进行基本操作,然后将私钥传递给 `set_private_key`。

## 核心钱包操作

钱包配置完成后,智能体可以使用完整的链上操作套件。

### 查询余额

```
Tool: get_trx_balance
Input: { "address": "TYourAddress..." }

Response:
{
  "balance": 1523.456789,
  "balance_sun": 1523456789
}
```

查询 TRC20 代币:

```
Tool: get_trc20_balance
Input: {
  "address": "TYourAddress...",
  "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t"
}

Response:
{
  "balance": "2500.00",
  "token": "USDT",
  "decimals": 6
}
```

### 发送 TRX

```
Tool: transfer_trx
Input: {
  "to": "TRecipientAddress...",
  "amount_trx": 100
}
```

智能体在本地签署交易并广播到 TRON 网络。工具返回交易哈希用于链上验证。

### 发送 TRC20 代币

```
Tool: transfer_trc20
Input: {
  "to": "TRecipientAddress...",
  "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "amount": "500"
}
```

TRC20 转账消耗能量。这正是 MERX 的优势所在 - 智能体可以在执行转账之前自动估算、购买和委托能量,将成本从大约 27 TRX(燃烧)降低到不到 4 TRX(委托能量)。

## 购买能量 - 核心价值主张

TRON 的资源模型意味着每次智能合约交互都需要消耗能量。一次简单的 USDT 转账大约需要 65,000 能量。SunSwap 交易可能消耗超过 200,000 能量。没有委托能量的话,这些成本将通过按照当前网络费率燃烧 TRX 来支付。

MERX 从多个供应商聚合能量,为智能体提供单一工具进行购买:

```
Tool: create_order
Input: {
  "energy_amount": 65000,
  "duration_hours": 1,
  "target_address": "TYourAddress..."
}
```

智能体还可以在购买前查看当前市场价格:

```
Tool: get_best_price
Input: {
  "energy_amount": 65000,
  "duration_hours": 1
}

Response:
{
  "best_price": 3.42,
  "provider": "sohu",
  "price_per_unit": 0.0000526
}
```

### ensure_resources 工具

对于希望专注于意图而非机制的智能体,`ensure_resources` 是处理一切的高级工具:

```
Tool: ensure_resources
Input: {
  "address": "TYourAddress...",
  "energy_needed": 65000,
  "bandwidth_needed": 350
}
```

该工具检查地址已有的资源,计算缺口,仅购买所需的部分,并在委托到达后才返回结果。智能体不需要了解能量市场、供应商 API 或委托机制。

## 通过 x402 实现零注册购买路径

MERX 提供了一条完全不需要创建账户的路径。x402 协议支持按使用付费的能量购买,付款在链上验证。

流程如下:

1. 智能体调用 `create_paid_order` 获取发票
2. 发票指定确切的 TRX 金额和备注字符串
3. 智能体使用本地钱包签署并广播付款交易
4. MERX 使用备注作为关联键在链上验证付款
5. 能量被委托到目标地址

```
Tool: create_paid_order
Input: {
  "energy_amount": 65000,
  "duration_hours": 1,
  "target_address": "TYourAddress..."
}

Response:
{
  "invoice": {
    "amount_trx": 1.43,
    "pay_to": "TMerxTreasuryAddress...",
    "memo": "merx_ord_abc123",
    "expires_at": "2026-03-30T12:05:00Z"
  }
}
```

智能体发送带有指定备注的 TRX,订单即被执行。无需 API 密钥。无需邮箱。无需注册表单。这是自主智能体与服务之间的纯链上商务。

## 真实的自主智能体工作流

以下是连接到 MERX 的自主智能体会话的完整示例:

**第 1 步:智能体配置钱包**

智能体从安全环境变量加载私钥并调用 `set_private_key`。MERX 推导出 TRON 地址并确认钱包已准备就绪。

**第 2 步:智能体检查财务状况**

智能体调用 `get_trx_balance` 和 `get_trc20_balance` 查询 USDT。现在它知道自己有 500 TRX 和 2,000 USDT。

**第 3 步:智能体收到任务 - "向此地址发送 100 USDT"**

智能体调用 `check_address_resources` 查看可用的能量和带宽。发现没有委托能量。

**第 4 步:智能体估算成本**

智能体调用 `estimate_transaction_cost` 进行 USDT 转账估算。响应显示需要 65,000 能量。没有能量的话,这将燃烧大约 27 TRX。购买能量后,成本约为 3.5 TRX。

**第 5 步:智能体购买能量**

智能体使用能量需求调用 `ensure_resources`。MERX 找到最便宜的供应商,下单,并等待委托到达。

**第 6 步:智能体执行转账**

能量委托完成后,智能体调用 `transfer_trc20` 发送 100 USDT。交易消耗委托的能量而不是燃烧 TRX。

**第 7 步:智能体验证结果**

智能体使用交易哈希调用 `get_transaction` 确认成功。

总成本:大约 3.5 TRX 而不是 27 TRX。智能体在没有任何人工干预的情况下节省了 87% 的费用。

## 安全考虑

### 私钥管理

私钥仅存在于 MCP 服务器的运行时内存中。它从不传输到 MERX 服务器,从不写入磁盘,也从不包含在 API 调用中。所有交易签名都在本地完成。

对于生产部署,将私钥存储在基础设施的密钥管理系统中(AWS Secrets Manager、HashiCorp Vault 或安全运行时中的环境变量),并在启动时传递给 MCP 服务器。

### 交易限额

对于自主智能体,建议实施安全防护措施:

- 在智能体逻辑中设置最大交易金额
- 使用带预算限制的常备订单进行周期性购买
- 监控智能体的钱包余额,当余额低于阈值时发出警报
- 使用资金有限的专用钱包而非主要资金库

### 网络选择

MERX 同时支持主网和 Shasta 测试网。始终先在 Shasta 上测试新的智能体工作流:

```json
{
  "env": {
    "MERX_NETWORK": "shasta"
  }
}
```

只有在验证完整流程后才切换到主网。

## 超越基本钱包操作

一旦智能体拥有钱包,MERX MCP 服务器就能解锁远超简单转账的功能:

- 通过 SunSwap 进行 **DEX 交易**,配合精确的能量模拟
- **常备订单**,在价格低于阈值时自动购买能量
- **委托监控器**,在能量到期前自动续费
- **多步意图**,批量处理多个操作并优化资源购买
- 跨所有能量供应商的**价格分析**,寻找最佳交易

每项功能都作为智能体可调用的工具暴露,配以引导智能体完成复杂工作流的提示和提供实时市场数据的资源。

## 立即开始

从零到可用智能体钱包的最快路径:

1. 安装 MCP 服务器:`npm install -g merx-mcp`
2. 配置你的 MCP 客户端(Claude Desktop、Cursor 或任何兼容 MCP 的工具)
3. 生成或导入 TRON 私钥
4. 调用 `set_private_key` 激活钱包
5. 调用 `get_trx_balance` 验证连接

整个设置过程不超过五分钟。无需注册,无需 API 密钥,无需审批流程。

MERX 是 AI 智能体与 TRON 区块链之间的桥梁。MCP 服务器开源,协议标准化,能量市场已上线运营。

你的智能体已经准备好拥有一个钱包了。给它一个。

---

**链接:**
- MERX 平台: [https://merx.exchange](https://merx.exchange)
- MCP 服务器 (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP 服务器 (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)
