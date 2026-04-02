# MERX能量成本估算API: 购买前明确成本

每笔TRON交易都消耗资源。一笔USDT转账大约需要65,000单位能量。一笔智能合约授权大约消耗15,000单位。一次复杂的DeFi交互可能需要200,000单位甚至更多。实际数字取决于合约、操作类型和当前网络参数。

如果您正在构建代表用户处理TRON交易的产品 - 钱包、支付处理器、交易机器人 - 您需要在执行前了解成本。需要多少能量?租赁与燃烧的成本分别是多少?用户能节省多少?

MERX提供两个接口精确回答这些问题: `POST /api/v1/estimate` 用于通用成本估算,`GET /api/v1/orders/preview` 用于特定订单的成本预览。两者结合,让您在任何SUN资金变动之前向用户展示确切的成本和节省。

## 估算接口

`POST /api/v1/estimate` 计算给定交易类型所需的能量和带宽,并返回通过MERX租赁能量与在协议层面燃烧TRX之间的成本对比。

### 基础请求

```bash
curl -X POST https://merx.exchange/api/v1/estimate \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "trc20_transfer",
    "target_address": "TTargetAddressHere"
  }'
```

### 响应格式

```json
{
  "operation": "trc20_transfer",
  "target_address": "TTargetAddressHere",
  "energy_required": 64895,
  "bandwidth_required": 345,
  "costs": {
    "burn": {
      "trx_cost": 27370000,
      "trx_cost_readable": "27.37 TRX",
      "usd_equivalent": 2.19
    },
    "rental": {
      "best_provider": "sohu",
      "price_per_unit_sun": 22,
      "total_cost_sun": 1427690,
      "total_cost_trx": "1.43 TRX",
      "usd_equivalent": 0.11,
      "duration_hours": 1
    },
    "savings": {
      "trx_saved": "25.94 TRX",
      "percent": 94.8,
      "usd_saved": 2.08
    }
  },
  "address_resources": {
    "current_energy": 0,
    "current_bandwidth": 1200,
    "energy_deficit": 64895,
    "bandwidth_deficit": 0
  },
  "timestamp": "2026-03-30T10:00:00Z"
}
```

响应告诉您做出交易成本决策所需的一切信息:

- **energy_required** - 操作所需的能量。标准TRC-20转账大约需要65,000单位,具体数字取决于合约和目标地址状态。
- **bandwidth_required** - 交易使用的带宽。大多数简单转账需要300-400个带宽点。新账户(首次激活地址)需要更多。
- **costs.burn** - 如果用户不采取任何措施,让协议燃烧TRX的成本。这是"默认"成本。
- **costs.rental** - 通过MERX可用的最便宜租赁选项。包括供应商、每单位价格、总成本和租赁时长。
- **costs.savings** - 燃烧与租赁之间的差异,以节省的TRX绝对值、百分比和美元等值表示。
- **address_resources** - 目标地址当前的能量和带宽。如果地址已经通过质押或之前的委托拥有能量,缺口会相应减少。

### 支持的操作类型

`operation` 字段接受几种预定义类型,涵盖最常见的TRON交易:

#### trc20_transfer

标准的TRC-20代币转账。这是最常见的操作 - 将USDT、USDC或任何其他TRC-20代币从一个地址发送到另一个地址。

```json
{
  "operation": "trc20_transfer",
  "target_address": "TSenderAddressHere"
}
```

所需能量: 标准转账(地址已激活且有USDT余额)大约64,895单位。首次向从未持有过该代币的地址转账成本更高 - 最多100,000能量 - 因为合约必须创建新的存储槽位。

#### trc20_approve

TRC-20授权交易,用于允许智能合约代表您花费代币。在与DEX合约、借贷协议和大多数DeFi应用交互前必需。

```json
{
  "operation": "trc20_approve",
  "target_address": "TApproverAddressHere"
}
```

所需能量: 大约15,000-18,000单位,显著低于转账。

#### trx_transfer

简单的TRX转账。主要消耗带宽而非能量,但估算接口出于完整性目的也处理它们。

```json
{
  "operation": "trx_transfer",
  "target_address": "TSenderAddressHere"
}
```

所需能量: 0(TRX转账不消耗能量)。所需带宽: 大约270字节。

#### custom

对于不属于预定义类型的智能合约调用。提供合约地址和函数选择器,MERX通过模拟调用来估算能量消耗。

```json
{
  "operation": "custom",
  "target_address": "TCallerAddressHere",
  "contract_address": "TContractAddressHere",
  "function_selector": "stake(uint256)",
  "parameters": [
    {
      "type": "uint256",
      "value": "1000000"
    }
  ]
}
```

自定义操作对TRON网络运行模拟以确定实际的能量和带宽消耗。这是非标准交易最准确的方法,但耗时略长(200-500ms,而预定义类型为50ms)。

## 订单预览接口

`POST /estimate` 给出资源需求和成本对比,而 `GET /api/v1/orders/preview` 展示MERX订单的确切样子 - 包括将选择哪个供应商、从余额中扣除的确切金额,以及任何适用的费用。

### 请求

```bash
curl "https://merx.exchange/api/v1/orders/preview?energy_amount=65000&target_address=TTargetAddressHere&duration_hours=1" \
  -H "X-API-Key: your_api_key"
```

### 响应

```json
{
  "preview": {
    "energy_amount": 65000,
    "target_address": "TTargetAddressHere",
    "duration_hours": 1,
    "provider": "sohu",
    "price_per_unit_sun": 22,
    "subtotal_sun": 1430000,
    "fee_sun": 14300,
    "total_sun": 1444300,
    "total_trx": "1.44 TRX",
    "your_balance_sun": 50000000,
    "balance_after_sun": 48555700,
    "estimated_fill_time_seconds": 5
  },
  "alternatives": [
    {
      "provider": "catfee",
      "price_per_unit_sun": 25,
      "total_sun": 1641250,
      "total_trx": "1.64 TRX"
    },
    {
      "provider": "netts",
      "price_per_unit_sun": 28,
      "total_sun": 1838200,
      "total_trx": "1.84 TRX"
    }
  ]
}
```

预览包括:

- **provider** - MERX在当前价格下会将订单路由到哪个供应商。
- **subtotal_sun** - 按供应商费率计算的能量原始成本。
- **fee_sun** - MERX平台费用。
- **total_sun** - 将从您余额中扣除的总金额。
- **balance_after_sun** - 订单后您的余额,便于验证是否负担得起。
- **estimated_fill_time_seconds** - 使用该供应商完成委托通常需要多长时间。
- **alternatives** - 其他可以完成订单的供应商,按价格排序。用于向用户展示其选择。

### 估算与预览的区别

| 功能 | POST /estimate | GET /orders/preview |
|------|---------------|-------------------|
| 目的 | 通用成本分析 | 精确订单规划 |
| 需要认证 | 是 | 是 |
| 显示与燃烧的对比节省 | 是 | 否 |
| 显示您的余额 | 否 | 是 |
| 显示平台费用 | 否 | 是 |
| 显示备选方案 | 否 | 是 |
| 显示地址资源 | 是 | 否 |
| 支持自定义合约 | 是 | 否 |

当您需要告诉用户"这笔交易如果燃烧TRX将花费X,如果租赁能量将花费Y - 为您节省百分之Z"时,使用 `estimate`。当用户已决定购买能量,您想在确认前展示精确订单信息时,使用 `preview`。

## 带真实数字的实际示例

### 示例一: USDT支付处理器

您运营一个向商户发送USDT的支付处理器。每次付款前,您估算成本:

```javascript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
});

async function estimatePayoutCost(senderAddress) {
  const estimate = await merx.estimate({
    operation: 'trc20_transfer',
    target_address: senderAddress,
  });

  console.log(`Energy needed: ${estimate.energy_required}`);
  console.log(`Burn cost: ${estimate.costs.burn.trx_cost_readable}`);
  console.log(`Rental cost: ${estimate.costs.rental.total_cost_trx}`);
  console.log(`Savings: ${estimate.costs.savings.percent}%`);

  // 根据节省阈值决定租赁还是燃烧
  if (estimate.costs.savings.percent > 50) {
    return { method: 'rent', cost: estimate.costs.rental };
  } else {
    return { method: 'burn', cost: estimate.costs.burn };
  }
}
```

典型输出:

```
Energy needed: 64895
Burn cost: 27.37 TRX
Rental cost: 1.43 TRX
Savings: 94.8%
```

以当前价格,租赁能量几乎总是比燃烧节省超过90%。

### 示例二: 钱包成本展示

您构建了一个TRON钱包,想在用户确认发送前展示交易成本:

```python
from merx_sdk import MerxClient

client = MerxClient(api_key="your_api_key")


def get_transfer_cost(sender_address: str, token: str = "usdt") -> dict:
    """获取发送TRC-20代币的成本,考虑现有资源。"""

    estimate = client.estimate(
        operation="trc20_transfer",
        target_address=sender_address,
    )

    resources = estimate["address_resources"]
    costs = estimate["costs"]

    # 如果地址已有足够能量,转账免费
    if resources["energy_deficit"] == 0:
        return {
            "cost": "0 TRX",
            "note": "Address has sufficient energy",
        }

    return {
        "without_energy": costs["burn"]["trx_cost_readable"],
        "with_energy": costs["rental"]["total_cost_trx"],
        "savings": f'{costs["savings"]["percent"]}%',
        "energy_deficit": resources["energy_deficit"],
    }
```

钱包UI可以这样展示:

```
发送100 USDT至TRecipient...

交易费用:
  不租赁能量:  27.37 TRX (燃烧)
  使用MERX能量: 1.43 TRX (租赁)
  为您节省:     94.8%

[ 租赁能量并发送 ]    [ 不租赁直接发送 ]
```

### 示例三: 批量转账成本预测

您需要向500个地址发送USDT,想提前了解总成本:

```javascript
async function estimateBatchCost(addresses) {
  let totalBurnCost = 0;
  let totalRentalCost = 0;
  let totalEnergy = 0;

  // 对代表性样本进行估算(前10个地址)
  // 大多数TRC-20转账消耗相同的能量
  const sampleSize = Math.min(10, addresses.length);
  const sample = addresses.slice(0, sampleSize);

  for (const address of sample) {
    const estimate = await merx.estimate({
      operation: 'trc20_transfer',
      target_address: address,
    });

    totalBurnCost += estimate.costs.burn.trx_cost;
    totalRentalCost += estimate.costs.rental.total_cost_sun;
    totalEnergy += estimate.energy_required;
  }

  // 外推至完整批次
  const avgBurnCost = totalBurnCost / sampleSize;
  const avgRentalCost = totalRentalCost / sampleSize;
  const avgEnergy = totalEnergy / sampleSize;

  const projectedBurnTRX = (avgBurnCost * addresses.length) / 1_000_000;
  const projectedRentalTRX = (avgRentalCost * addresses.length) / 1_000_000;
  const projectedEnergy = avgEnergy * addresses.length;

  console.log(`Batch size: ${addresses.length} transfers`);
  console.log(`Total energy needed: ${projectedEnergy.toLocaleString()}`);
  console.log(`Cost without MERX: ${projectedBurnTRX.toFixed(2)} TRX`);
  console.log(`Cost with MERX: ${projectedRentalTRX.toFixed(2)} TRX`);
  console.log(
    `Total savings: ${(projectedBurnTRX - projectedRentalTRX).toFixed(2)} TRX`
  );
}
```

500笔转账在当前市场价格下:

```
Batch size: 500 transfers
Total energy needed: 32,447,500
Cost without MERX: 13,685.00 TRX
Cost with MERX: 715.00 TRX
Total savings: 12,970.00 TRX
```

按大约0.08美元/TRX计算,仅一个批次就节省了超过1,000美元。

### 示例四: 自定义合约调用估算

您与一个自定义质押合约交互,需要估算 `stake()` 调用的成本:

```bash
curl -X POST https://merx.exchange/api/v1/estimate \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "custom",
    "target_address": "TCallerAddressHere",
    "contract_address": "TStakingContractHere",
    "function_selector": "stake(uint256)",
    "parameters": [
      {
        "type": "uint256",
        "value": "50000000"
      }
    ]
  }'
```

响应包含模拟的能量消耗,对于复杂的质押合约可能是120,000到200,000单位 - 显著高于简单转账。

## 将估算集成到订单流程中

估算和预览接口自然地融入面向用户的订单流程:

1. **用户发起交易**(例如发送USDT)。
2. **调用POST /estimate** 获取能量需求和成本对比。
3. **展示燃烧成本与租赁成本对比**,附节省百分比。
4. **用户选择租赁能量。**
5. **调用GET /orders/preview** 展示包含费用的精确订单详情。
6. **用户确认。**
7. **调用POST /orders** 并附带幂等性密钥创建订单。
8. **轮询或等待Webhook** 确认能量委托。
9. **使用委托的能量执行原始交易。**

步骤2-3是信息展示。没有资金移动。用户在做出承诺前看到透明的定价。这建立信任并减少用户因意外成本而产生的支持请求。

## 错误处理

两个接口都返回标准的MERX错误响应:

```json
{
  "error": {
    "code": "INVALID_ADDRESS",
    "message": "Target address is not a valid TRON address",
    "details": {
      "address": "invalid_address_here"
    }
  }
}
```

常见错误:

| 代码 | 原因 | 解决方案 |
|------|------|---------|
| `INVALID_ADDRESS` | 目标地址未通过TRON地址验证 | 验证地址格式(T前缀,base58) |
| `INVALID_OPERATION` | 无法识别的操作类型 | 使用以下之一: trc20_transfer、trc20_approve、trx_transfer、custom |
| `SIMULATION_FAILED` | 自定义合约调用模拟失败 | 检查合约地址和函数选择器 |
| `NO_PROVIDERS` | 没有供应商可以提供所需的能量数量 | 稍后重试或减少能量数量 |
| `INSUFFICIENT_BALANCE` | 余额不足以支付预览的订单(仅预览) | 向您的MERX账户充值更多TRX |

## 缓存注意事项

估算结果在短时间窗口内有效。能量价格随供应商调整费率而变化,网络参数也可能改变燃烧成本。对于大多数使用场景:

- **将估算缓存30-60秒**(如果在UI中展示成本)。价格变化不会快于MERX轮询供应商的频率(每30秒)。
- **在创建订单前始终获取最新预览。** 预览反映该时刻的确切成本。
- **不要缓存自定义合约模拟**(如果合约状态频繁变化)。模拟结果取决于执行时的链上状态。

## 总结

估算和预览接口消除了TRON能量购买中的猜测。不用租赁固定数量的能量然后期望它足够,您可以准确知道需要多少。不用接受任何可用的价格,您可以看到按成本排序的每个选项。

对于在TRON上构建产品的开发者,这些接口将能量从一项不可预测的成本转变为一个已知的、可优化的支出项目。检查成本,展示节省,让用户决定,然后放心执行。

- MERX平台: [merx.exchange](https://merx.exchange)
- 完整API文档: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- Python SDK: [github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)

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
