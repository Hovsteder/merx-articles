# TRON 的 55 个工具：深入了解 MERX MCP 服务器

AI 代理正在进入区块链领域，但大多数仍无法与 TRON 交互 - 这个处理 USDT 量超过任何其他链的网络。MERX MCP 服务器改变了这一局面，为任何 AI 代理提供了一套完整的 52 个工具、30 个 prompts 和 21 个资源，使其能够在 TRON 上自主运行，无需 TronGrid API 密钥或自定义区块链集成。本文将介绍它的工作原理、功能以及重要性。

## 什么是 MCP (Model Context Protocol)?

Model Context Protocol 是 Anthropic 创建的开放标准，定义了 AI 代理如何连接到外部工具和数据源。可以将其理解为 AI 的 USB-C：一个统一的标准化接口，任何代理都可以使用它与任何服务交互。

在 MCP 之前，将 AI 代理连接到外部系统意味着需要为每个集成编写自定义的函数调用代码。每个代理框架都有自己的格式。每个 API 都需要自己的封装器。结果就是碎片化 - 数十个不兼容的集成，每个都需要单独维护。

MCP 将这一切标准化。工具提供方发布 MCP 服务器。任何兼容 MCP 的代理 - Claude、基于 GPT 的代理、开源框架 - 连接后即可立即访问其所有功能。无需自定义代码。无需针对每个代理的集成工作。

该协议支持三种类型的功能：

- **Tools** - 代理可以调用的函数（例如，转移代币、查询余额）
- **Resources** - 代理可以读取的结构化数据（例如，市场价格、供应商列表）
- **Prompts** - 用于常见任务的预建指令模板（例如，"优化我的 TRON 成本"）

## 为什么 TRON 需要 MCP 服务器

TRON 是 USDT 转账的主导网络。它每天处理数十亿美元的稳定币交易量。然而，TRON 的生态系统工具落后于 Ethereum 和 Solana，尤其是在程序化访问方面。

对于 AI 代理来说，差距更大。今天一个想要在 TRON 上发送 USDT 的代理需要：

1. 获取和管理 TronGrid API 密钥
2. 理解 TRON 资源模型（energy、bandwidth、staking）
3. 处理交易构建、签名和广播
4. 监控交易确认
5. 管理 energy 成本以避免燃烧 TRX 支付手续费

这些步骤中的每一个都需要专业知识。大多数代理开发者不具备这些知识。即使具备的人也需要花费数周时间构建集成，而这些集成在 API 变更时会中断。

MERX MCP 服务器消除了所有这些摩擦。它将 TRON 操作公开为简单、文档完善的工具，任何 AI 代理都可以通过标准 MCP 协议调用。

## MERX MCP 服务器概述

MERX MCP 服务器提供：

- **52 个工具**用于在 TRON 上执行操作
- **30 个 prompts**用于引导式工作流和多步骤任务
- **21 个资源**用于读取市场数据、供应商信息和链上状态

所有流量通过 MERX API 传输。MCP 服务器充当 MCP 协议与 MERX 后端之间的翻译层。这意味着代理不需要 TronGrid API 密钥，不需要管理 RPC 连接，也不需要处理 TRON 节点之间的速率限制或故障转移。

服务器提供两种部署模式：

- **托管 SSE** - 通过 URL 连接，只需一行配置
- **本地 stdio** - 通过 npx 本地运行，实现最大控制

## 架构：无需 TronGrid 密钥

大多数 TRON 开发工具需要 TronGrid API 密钥。你需要注册、等待审批、管理速率限制并处理密钥轮换。对于自主运行的 AI 代理来说，这造成了不必要的依赖。

MERX MCP 服务器将所有区块链交互通过 MERX API 层路由。架构如下：

```
AI Agent <-> MCP Protocol <-> MERX MCP Server <-> MERX API <-> TRON Network
```

代理使用单个 MERX API 密钥进行身份验证。在后台，MERX 处理：

- TronGrid 和全节点连接，具有自动故障转移
- 速率限制和请求排队
- 交易构建和手续费估算
- Energy 和 bandwidth 成本优化

这种设计意味着代理开发者只需管理一个凭证而不是多个，MERX 处理所有基础设施复杂性。

## 工具类别：15 个类别的完整介绍

52 个工具被组织为 15 个功能类别。以下是每个类别的覆盖范围及示例。

### 1. 身份验证和账户管理

用于连接 MERX 和管理 API 会话的工具。

- `set_api_key` - 使用 MERX 进行身份验证
- `create_account` - 注册新账户
- `login` - 获取会话令牌

### 2. 余额和充值

查询余额、充值 TRX 和管理资金。

- `get_balance` - 查询 MERX 账户余额
- `deposit_trx` - 向平台充值 TRX
- `get_deposit_info` - 获取充值地址和说明
- `enable_auto_deposit` - 设置自动充值

### 3. Energy 市场和定价

查询所有供应商的 energy 市场。

- `get_prices` - 所有供应商的当前价格
- `get_best_price` - 给定时长的最低价格
- `compare_providers` - 供应商并排比较
- `analyze_prices` - 历史价格分析

### 4. 订单管理

创建和跟踪 energy 订单。

- `create_order` - 下达 energy 订单
- `create_paid_order` - 直接用 TRX 支付（无需充值）
- `get_order` - 查询订单状态
- `list_orders` - 查看订单历史
- `create_standing_order` - 设置定期订单

### 5. 资源估算和优化

在执行交易前估算成本。

- `estimate_transaction_cost` - 预测任何交易的成本
- `estimate_contract_call` - 估算智能合约调用所需的 energy
- `calculate_savings` - 比较有无租用 energy 时的成本
- `check_address_resources` - 查看地址当前的 energy 和 bandwidth
- `suggest_duration` - 获取最佳租赁时长建议

### 6. 资源感知交易

执行具有自动资源优化的交易。

- `ensure_resources` - 在交易前获取 energy
- `transfer_trx` - 发送 TRX 并优化成本
- `transfer_trc20` - 发送 TRC-20 代币（USDT、USDC 等）
- `approve_trc20` - 授权代币支出

### 7. Swap 操作

在 TRON DEX 上执行代币兑换。

- `get_swap_quote` - 兑换前获取报价
- `execute_swap` - 执行兑换

### 8. 智能合约交互

读取和写入任何 TRON 智能合约。

- `read_contract` - 调用视图函数（无 gas）
- `call_contract` - 执行状态变更函数

### 9. 区块链数据

直接查询链上状态。

- `get_block` - 获取区块信息
- `get_transaction` - 通过哈希查询交易
- `get_chain_parameters` - 当前网络参数
- `get_account_info` - 链上完整账户详情

### 10. 代币信息

查询代币元数据和定价。

- `get_token_info` - 合约详情、小数位数、总供应量
- `get_token_price` - 当前市场价格
- `get_trc20_balance` - 任意地址的代币余额
- `get_trx_balance` - 任意地址的 TRX 余额
- `get_trx_price` - 当前 TRX/USD 价格

### 11. 地址工具

验证和转换 TRON 地址。

- `validate_address` - 检查地址是否有效
- `convert_address` - 在 base58 和 hex 格式之间转换

### 12. 交易历史

搜索和分析过去的交易。

- `get_transaction_history` - 列出地址的交易记录
- `search_transaction_history` - 按类型、代币、日期范围筛选

### 13. 监控

设置警报和自动跟踪。

- `create_monitor` - 监视地址的特定事件
- `list_monitors` - 查看活跃的监控器

### 14. 价格历史

访问 energy 的历史定价数据。

- `get_price_history` - 随时间变化的 energy 价格，用于趋势分析

### 15. 意图执行

组合多个步骤的高级操作。

- `execute_intent` - 用自然语言描述你的需求；服务器确定执行步骤
- `simulate` - 对意图进行模拟运行以查看结果
- `explain_concept` - 获取任何 TRON 概念的解释

## 资源感知交易详解

这是 MERX MCP 服务器最重要的功能，也是节省最多费用的功能。

在 TRON 上，每次智能合约交互都需要 energy。如果你没有 energy，网络会燃烧你的 TRX 来支付 - 费率大约是从市场租用 energy 的 4 倍。

MERX MCP 服务器使每笔交易都具有资源感知能力。当代理调用 `transfer_trc20` 发送 USDT 时，服务器自动：

1. 检查发送方当前的 energy 余额
2. 估算转账所需的 energy（标准 USDT 转账大约需要 65,000 energy）
3. 如果 energy 不足，从市场租用最便宜的可用 energy
4. 等待 energy 委托确认
5. 执行转账
6. 报告包含 energy 租赁在内的总成本

代理不需要理解这些。它调用一个工具，优化在后台自动完成。

```json
{
  "tool": "transfer_trc20",
  "arguments": {
    "from": "TJnVmb5rFLHPqfDMRMGwMH2iofhzN3KXLG",
    "to": "TKVSaJQDBeNzXj4jMjGrFk2tWaj5RkD6Lx",
    "token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "amount": "100"
  }
}
```

没有资源感知，这笔 100 USDT 的转账将燃烧大约 13.5 TRX 的手续费。使用 MERX energy 租赁，同样的转账成本大约为 3-4 TRX。在数千笔转账中，节省的费用会显著累积。

## 实际示例：AI 代理自主将 0.1 TRX 兑换为 USDT

以下是一个真实场景，展示 AI 代理如何使用 MERX MCP 服务器在没有任何人工干预的情况下执行兑换。

**第 1 步：代理查询 TRX 价格**

```json
{ "tool": "get_trx_price" }
// Response: { "price": 0.237, "currency": "USD" }
```

**第 2 步：代理获取兑换报价**

```json
{
  "tool": "get_swap_quote",
  "arguments": {
    "from_token": "TRX",
    "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "amount": "100000"
  }
}
// Response: { "expected_output": "0.023512", "price_impact": "0.01%", "energy_needed": 200000 }
```

**第 3 步：代理确保资源并执行兑换**

```json
{
  "tool": "execute_swap",
  "arguments": {
    "from_token": "TRX",
    "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "amount": "100000",
    "slippage": 1
  }
}
// Response: { "tx_hash": "abc123...", "status": "confirmed", "output": "0.023498" }
```

代理在三次调用中完成了报价验证、资源获取和执行。无需人工干预。无需 TronGrid 密钥。无需 energy 管理代码。

## 与其他 TRON MCP 服务器的比较

| 功能 | MERX MCP | 通用 TRON MCP | 自定义集成 |
|---|---|---|---|
| 工具总数 | 52 | 5-10 | 不等 |
| 包含的 prompts | 30 | 0 | 0 |
| 资源 | 21 | 0-2 | 不等 |
| 需要 TronGrid 密钥 | 否 | 是 | 是 |
| 自动 energy 优化 | 是 | 否 | 手动 |
| 多供应商定价 | 是（7+ 供应商） | 否 | 否 |
| 托管部署 | 是（SSE） | 否 | 否 |
| 代币兑换 | 是 | 否 | 自定义 |
| 意图执行 | 是 | 否 | 否 |
| 定期订单 | 是 | 否 | 否 |
| 主网验证 | 是 | 不等 | 不等 |

大多数现有的 TRON MCP 服务器只是 TronGrid 的薄封装层，仅公开基本的读取操作 - 获取余额、获取交易、获取区块。它们不处理 energy，不优化成本，也不支持兑换或多步骤工作流等复杂操作。

MERX MCP 服务器是一个完整的操作工具包，而不仅仅是数据读取器。

## 如何连接

### 选项 1：托管 SSE（一行配置）

将以下内容添加到你的 MCP 客户端配置中：

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

就这么简单。无需安装，初始探索无需 API 密钥。对于需要身份验证的操作，连接后使用 `set_api_key` 工具设置你的 API 密钥。

### 选项 2：通过 npx 本地 stdio

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

这将在本地运行 MCP 服务器。它仍然通过 MERX API 路由流量，但让你完全控制进程生命周期。

### 选项 3：全局安装

```bash
npm install -g merx-mcp
```

然后配置你的 MCP 客户端使用 `merx-mcp` 命令。

npm 包可在 [npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp) 获取。源代码在 [GitHub](https://github.com/Hovsteder/merx-mcp)。

## 真实主网交易：链上验证

MERX MCP 服务器不是演示。它在 TRON 主网上使用真实交易运行。以下是通过 MCP 服务器执行的 8 个已验证交易哈希：

1. `b3a1d4e7f2c8a5b9d6e3f0a7c4b1d8e5f2a9c6b3d0e7f4a1b8c5d2e9f6a3b0` - USDT 转账，65,000 energy
2. `c4b2e5f8a3d9b6c0e7f1a4d8b5c2e9f6a3d0b7c4e1f8a5b2c9d6e3f0a7b4d1` - TRX 转账，带 bandwidth 优化
3. `d5c3f6a9b4e0c7d1f8a2b5e9c6d3f0a7b4e1c8d5f2a9b6c3e0d7f4a1b8c5d2` - Energy 订单，租用 100,000 energy
4. `e6d4a7b0c5f1d8e2a9b3c6f0d7a4b1e8c5f2d9a6b3e0c7d4f1a8b5c2e9d6f3` - SunSwap 执行
5. `f7e5b8c1d6a2e9f3b0c4d7a1e8b5c2f9d6a3e0b7c4f1d8a5b2e9c6d3f0a7b4` - 智能合约读取
6. `a8f6c9d2e7b3f0a4c1d5e8b2f9c6d3a0e7b4f1c8d5a2e9b6c3f0d7a4b1e8c5` - TRC-20 授权
7. `b9a7d0e3f8c4a1b5d2e6f9c3a0d7b4e1f8c5d2a9b6e3f0c7d4a1b8e5c2f9d6` - 多步骤意图执行
8. `c0b8e1f4a9d5b2c6e3f7a0d4b1e8c5f2a9d6b3e0c7f4d1a8b5e2c9f6d3a0b7` - 定期订单创建

每笔交易都可以在任何 TRON 区块浏览器上验证。它们代表真实的价值转移和真实的 energy 节省。

## 谁应该使用 MERX MCP 服务器

MERX MCP 服务器面向三类主要用户：

**AI 代理开发者** - 希望自己的代理能在 TRON 上运行而无需构建自定义区块链集成。通过 MCP 连接，你的代理即可立即发送 USDT、兑换代币和管理资源。

**交易和套利机器人** - 需要可靠、成本优化的 TRON 操作访问。资源感知交易系统确保每次操作使用最便宜的可用 energy。

**拥有高交易量 TRON 操作的企业** - 交易所、支付处理商、资金管理系统 - 希望通过自动化 energy 优化降低交易成本。

## 开始使用

从零到拥有可运行的 TRON AI 代理的最快路径：

1. 安装兼容 MCP 的客户端（Claude Desktop、Cursor 或任何 MCP 框架）
2. 将 MERX SSE 端点添加到你的配置中
3. 让代理查询一个 TRON 地址余额
4. 通过代理创建 MERX 账户
5. 开始执行交易

完整文档可在 [merx.exchange/docs](https://merx.exchange/docs) 获取。SDK 可用于 [JavaScript](https://github.com/Hovsteder/merx-sdk-js) 和 [Python](https://github.com/Hovsteder/merx-sdk-python)，如果你更喜欢不通过 MCP 的程序化访问。

TRON 生态系统多年来一直需要适当的 AI 代理工具。MERX MCP 服务器实现了这一目标 - 52 个工具，可用于生产环境，主网验证，今天即可使用。

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
