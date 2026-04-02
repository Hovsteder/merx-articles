# 资源感知交易:MERX 如何自动优化每笔交易

## 忽视 TRON 资源的隐性成本

TRON 网络上的每笔交易都消耗两种资源:能量和带宽。能量驱动智能合约执行 - 每个操作码、每次存储写入、每次代币转账。带宽则覆盖交易本身的原始字节。当你没有委托或质押这些资源时,网络将从你的 TRX 中燃烧来支付费用。

对于简单的 USDT 转账,这种燃烧可能达到 27 TRX - 按当前价格约为 7 美元。对于 DEX 交换,可能超过 50 TRX。大多数钱包和 SDK 都默默接受这种情况。它们广播交易,网络燃烧你的 TRX,而你在完全不知道有更便宜选项的情况下支付了最高费用。

MERX 采取了根本不同的方法。通过 MERX MCP 服务器的每笔交易都会经过一个资源感知管道,该管道估算成本、检查现有资源、仅购买差额、等待委托,然后才签署和广播。结果是每笔交易稳定节省 80-90%。

本文详细解释了该管道的工作原理。

## 资源感知交易管道

该管道有六个阶段。每个阶段必须在下一个阶段开始之前完成。跳过或重新排序阶段会产生竞态条件,可能导致交易失败或资源购买浪费。

### 阶段 1:估算能量和带宽

在执行任何操作之前,MERX 需要确切知道这笔特定交易将消耗多少资源。这不是查询表或硬编码常量。MERX 使用 `triggerConstantContract` 对照区块链的当前状态,用确切参数模拟精确的交易。

对于从地址 A 向地址 B 转账 100 USDT:

```
Tool: estimate_transaction_cost
Input: {
  "from": "TAddressA...",
  "to": "TAddressB...",
  "contract": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "function": "transfer(address,uint256)",
  "parameters": ["TAddressB...", 100000000],
  "value": 0
}

Response:
{
  "energy_required": 64895,
  "bandwidth_required": 345,
  "cost_without_resources": 27.12,
  "cost_with_resources": 3.42
}
```

模拟基于实时区块链状态运行。如果收款方从未持有过 USDT,能量成本会更高(因为合约需要创建新的存储槽位)。如果发送方的授权额度需要更新,也会增加能量。每个变量都被考虑在内。

对于 SunSwap 交易,模拟捕获精确的路由、滑点计算和流动性池状态:

```
SunSwap V2 模拟结果:
- 所需能量: 223,354
- 所需带宽: 420
- 路由路径: TRX -> USDT 通过流动性池 0x...
```

这种精确性至关重要。高估会在未使用的能量上浪费金钱。低估会导致交易失败,你将同时损失能量购买成本和失败交易的带宽。

### 阶段 2:检查当前资源

智能体的地址可能已经从之前的委托、质押或每日免费带宽配额中拥有一些资源。MERX 检查已有可用资源:

```
Tool: check_address_resources
Input: { "address": "TAddressA..." }

Response:
{
  "energy": {
    "available": 12000,
    "total": 12000,
    "used": 0
  },
  "bandwidth": {
    "available": 1400,
    "total": 1500,
    "used": 100,
    "free_available": 1400,
    "free_total": 1500
  }
}
```

在这个示例中,地址有 12,000 可用能量和来自每日免费配额的 1,400 带宽。

### 阶段 3:计算差额

MERX 用所需资源减去可用资源,精确确定需要购买的数量:

```
所需能量:     64,895
可用能量:     12,000
能量差额:     52,895
-> 向上取整为: 65,000(最小订单单位)

所需带宽:     345
可用带宽:     1,400
带宽差额:     0(充足)
```

这里适用两条重要规则:

**能量最低限额:65,000 单位。** TRON 能量委托市场的最小交易单位约为 65,000 能量。如果差额少于 65,000,MERX 会向上取整到 65,000。如果差额为 0(地址已有足够能量),则不进行购买。

**带宽阈值:1,500 单位。** 如果带宽差额少于 1,500,MERX 会完全跳过带宽购买。每个 TRON 地址每天获得 1,500 免费带宽,持续恢复。对于大多数单笔交易,免费配额就足够了。只有在高频操作耗尽每日配额时,购买带宽才有意义。

### 阶段 4:购买差额

计算出精确差额后,MERX 向所有可用能量供应商查询最佳价格:

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
  "all_prices": [
    { "provider": "sohu", "price": 3.42 },
    { "provider": "catfee", "price": 3.51 },
    { "provider": "netts", "price": 3.65 },
    { "provider": "tronsave", "price": 3.78 }
  ]
}
```

MERX 向最便宜的供应商下单:

```
Tool: create_order
Input: {
  "energy_amount": 65000,
  "duration_hours": 1,
  "target_address": "TAddressA..."
}
```

订单已下达,供应商开始委托流程。

### 阶段 5:轮询直到委托到达

这是大多数实现出错的阶段。TRON 上的能量委托不是即时的。在供应商广播委托交易后,它必须经过网络确认。通常需要 3-6 秒,但在网络拥堵时可能更长。

MERX 以固定间隔轮询目标地址的资源余额:

```
轮询周期:
  t+0s:  check_address_resources -> 能量: 12,000 (尚未到达)
  t+3s:  check_address_resources -> 能量: 12,000 (尚未到达)
  t+6s:  check_address_resources -> 能量: 77,000 (委托已到达)
  -> 进入阶段 6
```

轮询使用从 3 秒开始的指数退避,最大等待 60 秒后超时。如果委托在超时窗口内未到达,管道会报告错误而不是广播一个会燃烧 TRX 的交易。

这个轮询阶段正是防止困扰简单实现的竞态条件的关键。没有它,顺序将是:购买能量,立即广播交易,交易在委托确认前执行,TRX 仍然被燃烧,你同时为能量和燃烧付了费。

### 阶段 6:本地签名并广播

只有在确认委托的能量在链上可用之后,MERX 才会签署和广播实际交易:

```
1. 使用 TronWeb 构建交易对象
2. 使用本地私钥签名(密钥不离开本机)
3. 广播到 TRON 网络
4. 返回交易哈希
```

交易现在使用委托的能量执行,消耗约 65,000 能量单位而不是燃烧 27 TRX。

## ensure_resources 工具:一次调用完成整个管道

对于希望使用管道但不想单独管理每个阶段的智能体,MERX 提供了 `ensure_resources`:

```
Tool: ensure_resources
Input: {
  "address": "TAddressA...",
  "energy_needed": 65000,
  "bandwidth_needed": 345
}
```

这个单一工具调用在内部执行阶段 2 到阶段 5。它检查当前资源,计算差额,找到最佳价格,下单,并轮询直到委托到达。智能体只有在地址完全配置好并准备好进行交易时才会收到响应。

## 真实示例:SunSwap 交易

以下是一笔真实 SunSwap V2 交易的完整管道 - 将 0.1 TRX 交换为 USDT。

**阶段 1 - 模拟:**

```
triggerConstantContract(
  contract: SunSwapV2Router,
  function: swapExactETHForTokens,
  parameters: [0, [WTRX, USDT], address, deadline],
  call_value: 100000  // 0.1 TRX (以 SUN 计)
)

结果: energy_estimate = 223,354
```

**阶段 2 - 检查资源:**

```
地址资源:
  能量: 0
  带宽: 1,420 (免费)
```

**阶段 3 - 计算差额:**

```
能量差额: 223,354
-> 取整到最近的订单单位: 225,000
带宽差额: 0 (免费配额可覆盖所需的 345)
```

**阶段 4 - 购买:**

```
225,000 能量 / 1 小时的最佳价格:
  供应商: catfee
  价格: 11.82 TRX
```

**阶段 5 - 轮询:**

```
委托在 4.2 秒后确认
地址现在有 225,000 可用能量
```

**阶段 6 - 执行:**

```
交换交易已广播
交易哈希: abc123...
消耗能量: 223,354
剩余能量: 1,646 (将随委托到期)
净成本: 11.82 TRX 而非约 53 TRX 燃烧
节省: 78%
```

整个管道自主执行。智能体请求将 TRX 交换为 USDT,MERX 在幕后处理了每个资源计算、购买和时序问题。

## 为什么操作顺序很重要

管道的严格排序防止了三类故障:

### 竞态条件:购买后立即广播

如果你在同一个区块中购买能量并广播交易,委托可能尚未处理完毕。交易在没有委托能量的情况下执行,燃烧 TRX,你付了两次费用 - 一次是能量费(未使用),一次是 TRX 燃烧。

MERX 通过在链上确认委托后才继续操作来防止这种情况。

### 高估:硬编码能量值

许多工具使用硬编码的能量估算(例如,"USDT 转账始终消耗 65,000 能量")。但实际成本取决于涉及的特定地址、它们的代币持有历史、合约的内部状态,甚至区块编号。向新地址转账的成本高于向已持有该代币的地址转账。

MERX 通过使用真实参数对照实时区块链状态模拟精确的交易来防止这种情况。

### 低估:资源不足

如果你低估了能量需求,交易在执行过程中能量耗尽,它将失败。你会损失交易尝试的带宽,购买的能量也浪费在了失败的交易上。

MERX 通过使用 `triggerConstantContract` 进行精确模拟并在订购时添加小额缓冲来防止这种情况。

## 实际效果差异

不使用资源感知交易(标准钱包行为):

```
USDT 转账: 27 TRX 燃烧 (约 $7.00)
SunSwap 交易: 53 TRX 燃烧 (约 $13.75)
授权 + 交换: 68 TRX 燃烧 (约 $17.65)
```

使用 MERX 资源感知管道:

```
USDT 转账: 3.42 TRX (能量购买)
SunSwap 交易: 11.82 TRX (能量购买)
授权 + 交换: 14.93 TRX (能量购买)
```

对于每天执行 100 笔 USDT 转账的智能体,这意味着每天 $700 与 $342 的差异 - 年度节省超过 $130,000。

## 开发者集成

如果你正在构建与 TRON 交互的应用程序,集成 MERX 资源感知管道不需要架构变更。MCP 服务器在内部处理整个管道。

直接 API 集成:

```bash
# 第 1 步:获取估算
curl -X POST https://merx.exchange/api/v1/estimate \
  -H "Content-Type: application/json" \
  -d '{"from": "T...", "to": "T...", "amount": 100000000}'

# 第 2 步:确保资源
curl -X POST https://merx.exchange/api/v1/ensure-resources \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{"address": "T...", "energy_needed": 65000}'

# 第 3 步:广播你的交易(客户端签名)
```

`ensure-resources` 端点处理价格比较、订单下达和委托轮询。你的应用程序只有在地址准备好时才会收到响应。

## 结论

资源感知交易不是一种优化。它是与 TRON 网络交互的正确方式。在不首先确保有充足资源的情况下广播交易,就像在不检查服务器是否可达的情况下发送 HTTP 请求一样 - 它可能有效,但一旦失败,你将付出代价。

MERX 让正确的方法成为默认方法。每笔交易都通过管道。每个资源差额都被精确计算。每次购买都以最佳可用价格进行。每次委托都在交易广播前得到确认。

结果是在与 TRON 区块链的每次交互中实现可预测的最低成本交易。

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
