# 通过 MCP 使用 SunSwap:AI 智能体在 DEX 上交易

## 缺失的桥梁

去中心化交易所多年来一直可以通过 Web 界面、移动钱包和编程 SDK 访问。但它们一直没有通过自然语言来访问。AI 智能体无法点击交换按钮。它无法导航 Web UI。虽然智能体在技术上可以通过原始交易构建来调用智能合约,但这要求智能体理解 ABI 编码、路由合约地址、流动性池机制以及 TRON 资源模型。

MERX 弥合了这一差距。通过 MCP 服务器,AI 智能体可以请求交换报价、了解预期输出和成本,并执行交易 - 全部通过结构化的工具调用,在抽象协议层复杂性的同时保持链上操作的完全透明性。

本文涵盖完整的 SunSwap 集成:获取报价、执行交换、处理授权、模拟能量成本,以及理解主网交易的真实数据。

## 获取交换报价

任何交易的第一步是了解你将得到什么。`get_swap_quote` 工具查询 SunSwap V2 的路由合约来计算给定输入的预期输出:

```
Tool: get_swap_quote
Input: {
  "from_token": "TRX",
  "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "amount": "100000",
  "from_address": "TYourAddress..."
}

Response:
{
  "input_token": "TRX",
  "input_amount": "0.1",
  "input_amount_sun": 100000,
  "output_token": "USDT",
  "output_amount": "0.032847",
  "output_amount_raw": "32847",
  "minimum_received": "0.032519",
  "price_impact": "0.001%",
  "route": ["WTRX", "USDT"],
  "router": "TKzxdSv2FZKQrEkFPRQi4qeCokAFkrntVM",
  "energy_required": 223354,
  "energy_cost_estimate_trx": 11.76,
  "liquidity": {
    "pool_address": "TPool...",
    "trx_reserve": "45,234,891.23",
    "usdt_reserve": "14,832,445.67"
  }
}
```

几个重要细节:

### 价格影响

`price_impact` 字段显示你的交易对池价格的影响程度。上述 0.1 TRX 交易的影响为 0.001% - 可忽略不计。对于更大的交易,这个数字会增长:

```
0.1 TRX 交换:      0.001% 影响
1,000 TRX 交换:    0.012% 影响
100,000 TRX 交换:  1.2% 影响
1,000,000 TRX 交换: 11.8% 影响
```

价格影响不是手续费 - 它是恒定乘积 AMM 机制的结构性后果。你的交易相对于池的流动性越大,执行价格越差。

### 最低接收量

`minimum_received` 字段考虑了滑点保护。默认情况下,MERX 计算 1% 的滑点容忍度,意味着如果输出低于报价量的 1% 以上,交换将回退。这防止了在报价和执行之间的抢跑和快速价格变动。

### 路由

`route` 字段显示代币路径。对于 TRX 到 USDT,路由通过 WTRX(Wrapped TRX),因为 SunSwap V2 的交易对是在 TRC20 代币之间,原生 TRX 必须先被包装。对于没有直接交易对的代币间交换,路由可能包括中间代币:

```
代币 A -> WTRX -> 代币 B (两跳路由)
```

多跳路由由于额外的合约交互而消耗更多能量。

### 所需能量

这是来自 `triggerConstantContract` 模拟的精确能量估算。不是硬编码常量,不是范围 - 而是在当前区块链状态下,使用这些特定参数的这笔特定交换将消耗的精确能量单位。

## 执行交换

一旦智能体审查了报价并决定继续,`execute_swap` 工具处理整个执行管道:

```
Tool: execute_swap
Input: {
  "from_token": "TRX",
  "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "amount": "100000",
  "slippage": 1.0,
  "deadline_minutes": 5
}

Response:
{
  "status": "completed",
  "tx_hash": "9c7b3a2f...",
  "input": {
    "token": "TRX",
    "amount": "0.1"
  },
  "output": {
    "token": "USDT",
    "amount": "0.032847"
  },
  "energy_used": 223354,
  "energy_purchased": 225000,
  "energy_cost_trx": 11.84,
  "block": 58234892,
  "gas_saved_trx": 41.16
}
```

### execute_swap 内部流程

`execute_swap` 工具编排完整的资源感知交易管道:

1. **模拟交换** - 使用精确的交换参数执行 `triggerConstantContract` 获取精确的能量需求(本例中为 223,354)
2. **检查当前资源** - 查询发送方地址的可用能量和带宽
3. **购买能量差额** - 查找最佳供应商价格,下单,等待委托确认
4. **构建交换交易** - 使用正确的函数选择器、参数和调用值构建 SunSwap V2 路由调用
5. **本地签名** - 私钥在智能体的机器上签署交易
6. **广播** - 签名的交易发送到 TRON 网络
7. **验证** - 轮询交易确认并解析结果

智能体调用一个工具。七个步骤在幕后执行。智能体收到一个包含完整结果的单一响应。

## 代币授权

当交换 TRC20 代币(而非原生 TRX)时,SunSwap 路由需要授权才能代替你消费代币。这是标准的 TRC20 `approve` 机制。

MERX 自动处理这一点。在执行 TRC20 到 TRC20 或 TRC20 到 TRX 的交换之前,execute_swap 工具会检查路由合约是否对输入代币有足够的授权额度:

```
授权检查:
  代币: USDT (TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t)
  消费方: SunSwap V2 Router (TKzxdSv2FZKQrEkFPRQi4qeCokAFkrntVM)
  当前授权额度: 0
  需要: 100,000,000 (100 USDT)

  -> 需要授权
```

如果需要授权,MERX:

1. 模拟授权交易以获取其能量成本
2. 将授权能量加到交换能量中进行合并购买
3. 首先执行授权交易
4. 等待确认
5. 然后执行交换

```
合并能量需求:
  授权: 46,312 能量
  交换: 223,354 能量
  总计: 269,666 能量

  已购买: 270,000 能量,14.20 TRX
```

智能体不需要了解授权。如果需要,它就会发生。如果代币已经被授权,该步骤就被跳过。

## 真实交换:0.1 TRX 兑换 USDT

以下是通过 MERX MCP 服务器在主网执行的真实交换的完整细节:

### 交换前状态

```
地址: TWallet...
TRX 余额: 523.456789 TRX
USDT 余额: 0.00 USDT
可用能量: 0
可用带宽: 1,420 (免费)
```

### 报价

```
get_swap_quote:
  输入: 0.1 TRX (100,000 SUN)
  输出: 0.032847 USDT
  价格影响: 0.001%
  所需能量: 223,354
  路由: WTRX -> USDT
```

### 执行

```
execute_swap:
  已购能量: 225,000,11.84 TRX (供应商: sohu)
  委托确认: 4.1 秒
  授权: 不需要 (TRX -> USDT,原生 TRX 不需要授权)
  交换交易广播: 9c7b3a2f...
  交换交易确认: 区块 58,234,892
```

### 交换后状态

```
地址: TWallet...
TRX 余额: 511.516789 TRX (523.456789 - 0.1 交换 - 11.84 能量)
USDT 余额: 0.032847 USDT
可用能量: 1,646 (225,000 购买 - 223,354 使用)
可用带宽: 1,000 (免费,被交易部分消耗)
```

### 成本明细

```
能量购买:      11.84 TRX
交换输入:       0.10 TRX
总花费:        11.94 TRX

不购买能量(TRX 燃烧):
  交换能量燃烧:  约 53.00 TRX
  交换输入:       0.10 TRX
  总计:          约 53.10 TRX

节省:           41.16 TRX (77.5%)
```

## 对比:原始 TronWeb vs MERX MCP

### 原始 TronWeb(不使用 MERX)

```javascript
// 1. 估算能量(手动)
const estimation = await tronWeb.transactionBuilder.triggerConstantContract(
  routerAddress, 'swapExactETHForTokens(uint256,address[],address,uint256)',
  { callValue: 100000 },
  [{ type: 'uint256', value: 0 },
   { type: 'address[]', value: [wtrxAddr, usdtAddr] },
   { type: 'address', value: myAddr },
   { type: 'uint256', value: deadline }],
  myAddr
);
// 2. 在某处购买能量(单独的集成)
// 3. 等待委托(手动轮询)
// 4. 构建交换交易
// 5. 签名
// 6. 广播
// 7. 验证
// 约 50 行代码,多个 API 集成
```

### MERX MCP(一次工具调用)

```
Tool: execute_swap
Input: {
  "from_token": "TRX",
  "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "amount": "100000",
  "slippage": 1.0
}
// 一次调用。一切内部处理。
```

智能体不需要了解路由地址、WTRX 包装、ABI 编码、能量市场、委托机制或交易构建。它表达意图("将 TRX 交换为 USDT")并接收结果。

## 风险和限制

### 滑点

即使有滑点保护,快速的价格变动也可能导致交换回退。如果交换回退,用于回退交易的能量会损失(但代币不会)。智能体可以用更高的滑点容忍度或更小的金额重试。

### MEV 和抢跑

TRON 的区块生产模型与以太坊不同,MEV 生态系统不太发达。但是,SunSwap 上的大额交换仍然可能被监控内存池的高级参与者抢跑。对于大额交易,考虑分成多笔较小的交换,分散在不同区块中。

### 流动性

SunSwap V2 的流动性在不同代币对之间差异显著。主要交易对(TRX/USDT、TRX/USDC)有深厚的流动性。较小的代币可能有薄弱的池,即使中等规模的交易也会造成显著的价格影响。始终在执行前检查报价。

## 结论

通过 AI 智能体进行 DEX 交易不再是理论上的能力。MERX 用两次工具调用使其可以实际操作:一次报价,一次执行。能量模拟精确。资源购买自动化。代币授权透明处理。

对于在 TRON 上运行的 AI 智能体,通过 MCP 使用 SunSwap 是链上交易的最快途径。没有 Web UI。没有手动交易构建。没有资源管理开销。

报价。执行。完成。

---

**链接:**
- MERX 平台: [https://merx.exchange](https://merx.exchange)
- MCP 服务器 (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP 服务器 (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)
