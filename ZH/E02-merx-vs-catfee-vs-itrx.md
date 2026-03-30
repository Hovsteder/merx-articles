# MERX vs CatFee vs ITRX vs TronSave：哪个 energy 供应商最便宜？

TRON energy 不是免费的，但你支付多少完全取决于你在哪里购买。同等数量 energy 在最便宜和最贵供应商之间的差价可达 3-4 倍。本文比较了主要的 TRON energy 供应商 - CatFee、ITRX、TronSave、TEM、Netts 和 PowerSun - 在定价、时长选项和可靠性方面的表现，并解释 MERX 聚合器如何消除逐个手动检查的需要。

## 2026 年的 TRON Energy 市场

TRON 上的每次智能合约交互都需要 energy。发送 USDT 大约需要 65,000 energy。DEX 兑换可消耗 200,000-500,000 energy。如果你没有 energy，TRON 网络会燃烧你的 TRX 来支付成本 - 费率大约是从第三方供应商租用 energy 的 4 倍。

这催生了一个市场。多个供应商现在出售 energy，每个都有不同的定价模型、时长等级和最低订单量。一些提供固定价格租赁。其他运营点对点市场，价格随供需波动。一些专门提供短期租赁用于单笔交易。其他则面向拥有持续交易量的企业提供长期合约。

问题在于没有哪个单一供应商总是最便宜的。价格全天根据网络需求、供应商库存和竞争动态而变化。一小时前最便宜的现在可能不再是最便宜的。

## 供应商简介

### CatFee

CatFee 提供固定价格的 energy 租赁，注重简洁性。其主要产品是 1 小时 energy，涵盖了绝大多数使用场景 - 大多数用户只需要 energy 用于单笔交易，1 小时提供了充足的执行时间。

**主要特征：**
- 固定定价，定期更新
- 主要时长：1 小时
- 适合短期、单笔交易使用场景
- 简洁的 API，集成直接
- 在个人用户和小型应用中广受欢迎

CatFee 在短期租赁领域建立了可靠的声誉。其 1 小时产品非常适合需要 energy 进行单笔 USDT 转账或合约交互且不想考虑时长选择的用户。

### ITRX

ITRX 提供四个不同时长等级的 energy，使其成为最灵活的固定价格供应商之一。其分级模型让用户可以选择与实际使用模式匹配的租赁期。

**时长等级：**
- 1 小时 - 用于单笔交易
- 1 天（24 小时） - 用于批量操作
- 3 天 - 用于中期项目
- 30 天 - 用于持续运营需求

**主要特征：**
- 每个等级固定定价，定期更新
- 较长时长价格尤其有竞争力
- 文档完善的 API
- 标准订单量的可靠执行率

ITRX 是那些提前了解 energy 需求并希望获得可预测定价的用户的不错选择。30 天等级对于运行持续 TRON 操作的企业特别有用。

### TronSave

TronSave 运营点对点市场模型，而非以固定价格出售 energy。委托人将其质押的 TRX 列出供租用，买方竞价购买可用供应。这创造了一个动态定价环境，成本随市场状况波动。

**主要特征：**
- P2P 市场，动态定价
- 灵活的时长选项
- 低需求期间价格可能低于固定价格供应商
- 高需求期间价格可能飙升
- 部分挂单的最低订单量较大
- 成熟的平台，流动性充足

TronSave 在网络需求低、委托人供应高时通常提供最佳价格。然而，在高峰期间，价格可能超过固定价格供应商。它奖励能够把握购买时机的用户。

### TEM (Tron Energy Market)

TEM 是另一个专注于大额订单的点对点 energy 市场。它主要服务于高交易量用户 - 交易所、支付处理商和大型 dApp - 这些用户需要大量 energy。

**主要特征：**
- P2P 市场模型
- 针对大额订单优化
- 大额交易量定价有竞争力
- 不太适合小额单笔交易
- 大额细分市场流动性不断增长

TEM 不适合只发送一笔 USDT 转账的用户。但对于每天执行数千笔交易的支付处理商来说，TEM 的批量定价可以带来显著的节省。

### Netts

Netts 提供固定价格 energy，注重短时长和激进定价。它经常位列 1 小时 energy 最便宜的供应商之中，使其成为注重成本的用户的理想选择。

**主要特征：**
- 固定定价，通常是市场最低
- 专注短时长（1 小时、1 天）
- 快速委托（下单后 energy 快速交付）
- 较低的最低订单量
- 良好的 API 可靠性

Netts 主要在价格上竞争。对于唯一优先考虑单笔交易每单位 energy 支付最少的用户，Netts 通常是答案 - 尽管随着供应商调整费率，这会有所变化。

### PowerSun

PowerSun 采用不同的方式，提供 10 个不同的时长等级，是市场上租赁期范围最广的。这种粒度让用户可以精确匹配 energy 租赁与其运营时间线。

**时长等级：**
- 10 分钟、30 分钟、1 小时、3 小时、6 小时
- 12 小时、1 天、3 天、7 天、30 天

**主要特征：**
- 10 个时长等级 - 市场最多
- 每个等级固定定价
- 在 1 小时以下的租赁中（10 分钟和 30 分钟）特别有优势
- 适合需要极短时间窗口 energy 的用户
- 经过验证的技术栈

PowerSun 的 10 分钟和 30 分钟等级填补了大多数其他供应商未涉及的空白。如果你只需要 energy 用于一笔交易并希望最小化租赁窗口，这些超短时长可能比支付一整小时更具成本效益。

## 价格比较

Energy 价格以 SUN 每单位 energy 每天报价。越低越好。下表反映了代表性市场定价。实际价格持续变化。

| 供应商 | 1 小时 | 1 天 | 3 天 | 7 天 | 30 天 |
|---|---|---|---|---|---|
| CatFee | 28-35 SUN | - | - | - | - |
| ITRX | 30-38 SUN | 26-32 SUN | 24-30 SUN | - | 22-28 SUN |
| TronSave | 25-45 SUN | 24-40 SUN | 23-38 SUN | 22-35 SUN | 20-32 SUN |
| TEM | 30-42 SUN | 26-36 SUN | 24-34 SUN | 22-32 SUN | 20-30 SUN |
| Netts | 22-32 SUN | 24-30 SUN | - | - | - |
| PowerSun | 26-34 SUN | 24-32 SUN | 22-30 SUN | 22-28 SUN | 20-26 SUN |

**关键观察：**

- Netts 经常提供最低的 1 小时价格，但可用性可能有所不同
- TronSave 的范围最广，因为 P2P 定价随供需波动
- 较长时长通常在所有供应商中每天成本更低
- 没有单一供应商在所有时长中占据主导地位
- 同一时长下最便宜和最贵选项之间的差异可达 60-80%

## 时长比较：哪个供应商适合哪种使用场景

不同的使用模式需要不同的供应商。

### 单笔 USDT 转账（需要 1 笔交易的 energy）

**最佳选择：** Netts（通常最便宜）、CatFee（最可靠）、PowerSun 10 分钟等级（可用时总成本最低）

对于一次性转账，你需要几分钟的 energy，而不是几小时。PowerSun 的 10 分钟或 30 分钟等级在绝对价格上可能比其他供应商的 1 小时租赁更便宜，即使每天的费率更高。你支付的是更短的时间。

### 批量处理（几小时内 50-200 笔交易）

**最佳选择：** ITRX 1 天等级、PowerSun 3 小时或 6 小时等级

批量操作需要在特定时间窗口内持续的 energy。1 天的租赁提供了舒适的余量，而不会为多天承诺多付费用。

### 持续运营（交易所、支付处理商）

**最佳选择：** PowerSun 30 天、ITRX 30 天、TronSave（如果你能在低需求期间把握购买时机）

长期租赁提供最低的每日费率。对于持续运营，任何供应商的 30 天等级都将比反复购买 1 小时 energy 便宜得多。

### 不可预测的突发流量

**最佳选择：** MERX 聚合器，带自动路由

如果你的交易量不可预测 - 有时每小时 10 笔交易，有时 1,000 笔 - 没有单一供应商或时长等级是最优的。这正是聚合器最能发挥价值的地方。

## 聚合器优势

只检查一个供应商就认为你得到了最好的价格是一个代价高昂的错误。原因如下。

假设一个用户需要 65,000 energy 用于 USDT 转账。在请求时：

- Netts 报价 24 SUN/energy/天
- CatFee 报价 32 SUN/energy/天
- ITRX 报价 30 SUN/energy/天
- TronSave 有 26-40 SUN/energy/天的挂单

如果用户只查看 CatFee，他们支付 32 SUN。如果查看 Netts，他们支付 24 SUN。这对于完全相同网络上的完全相同 energy 有 33% 的差异。

现在将此乘以数千笔交易。一个每天执行 1,000 笔 USDT 转账的支付处理商，如果始终选择最便宜的供应商将节省大量费用 - 但在每笔交易前手动比较六个供应商是不现实的。

这就是聚合器解决的根本问题。它每次都实时检查所有供应商，并自动路由到最便宜的选项。

## MERX 如何解决比较问题

MERX 是一个资源交易所，聚合来自多个供应商的 energy（和 bandwidth）。当你通过 MERX 下单时，平台会：

1. 同时查询所有活跃供应商
2. 比较所请求 energy 数量和时长的价格
3. 将订单路由到提供最低价格的供应商
4. 处理委托和确认
5. 如果主要供应商不可用，则转向下一个最便宜的供应商

用户看到的是单一界面和单一价格 - 始终是当时可用的最佳价格。管理多个供应商账户、比较 API 和处理故障转移的复杂性被完全抽象化。

```bash
# Using the MERX JavaScript SDK
npm install merx-sdk
```

```typescript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({ apiKey: 'your-api-key' });

// Get the best price across all providers
const best = await merx.getBestPrice({
  energy: 65000,
  duration: '1h'
});

console.log(best.provider);   // e.g., "netts"
console.log(best.price);      // e.g., 24 SUN
console.log(best.total_trx);  // total cost in TRX

// Place the order - MERX routes to cheapest provider
const order = await merx.createOrder({
  energy: 65000,
  duration: '1h',
  target: 'TJnVmb5rFLHPqfDMRMGwMH2iofhzN3KXLG'
});
```

Python SDK 提供相同的功能：

```bash
pip install merx-sdk
```

```python
from merx_sdk import MerxClient

merx = MerxClient(api_key="your-api-key")

best = merx.get_best_price(energy=65000, duration="1h")
print(f"Best provider: {best['provider']} at {best['price']} SUN")

order = merx.create_order(
    energy=65000,
    duration="1h",
    target="TJnVmb5rFLHPqfDMRMGwMH2iofhzN3KXLG"
)
```

SDK 可在 [npm](https://www.npmjs.com/package/merx-sdk) 和 [PyPI](https://pypi.org/project/merx-sdk/) 获取。源代码：[JavaScript SDK](https://github.com/Hovsteder/merx-sdk-js)、[Python SDK](https://github.com/Hovsteder/merx-sdk-python)。

## 成本示例：1,000 笔 USDT 转账

让我们计算使用不同方法执行 1,000 笔 USDT 转账的成本。每笔转账大约需要 65,000 energy。

### 不租用 energy（燃烧 TRX）

当你没有 energy 时，TRON 会燃烧你余额中的 TRX 来支付成本。按照当前网络参数，标准 USDT 转账大约燃烧 13.5 TRX。

- 每笔转账成本：约 13.5 TRX
- 1,000 笔转账成本：约 13,500 TRX
- 按 TRX = $0.24 计算：约 $3,240

### 使用 CatFee（1 小时，固定价格 32 SUN）

- 每笔转账的 energy 成本：约 2.08 TRX
- Bandwidth 和手续费：约 0.5 TRX
- 每笔转账成本：约 2.58 TRX
- 1,000 笔转账成本：约 2,580 TRX
- 按 TRX = $0.24 计算：约 $619

### 使用 Netts（1 小时，固定价格 24 SUN）

- 每笔转账的 energy 成本：约 1.56 TRX
- Bandwidth 和手续费：约 0.5 TRX
- 每笔转账成本：约 2.06 TRX
- 1,000 笔转账成本：约 2,060 TRX
- 按 TRX = $0.24 计算：约 $494

### 使用 ITRX（30 天 22 SUN，如果你能承诺）

- 每笔转账的 energy 成本：约 1.43 TRX（摊销）
- Bandwidth 和手续费：约 0.5 TRX
- 每笔转账成本：约 1.93 TRX
- 1,000 笔转账成本：约 1,930 TRX
- 按 TRX = $0.24 计算：约 $463

### 使用 MERX（自动路由到最便宜的供应商）

MERX 在每次下单时检查所有供应商，并路由到最便宜的可用选项。假设 MERX 在每笔交易中获取最佳可用价格（全天在不同供应商之间变化）：

- 每笔转账的平均 energy 成本：约 1.50 TRX（当时最便宜的加权平均值）
- Bandwidth 和手续费：约 0.5 TRX
- 每笔转账成本：约 2.00 TRX
- 1,000 笔转账成本：约 2,000 TRX
- 按 TRX = $0.24 计算：约 $480

**1,000 笔转账的节省摘要：**

| 方法 | 总成本 (TRX) | 总成本 (USD) | 相比燃烧的节省 |
|---|---|---|---|
| 无 energy（燃烧 TRX） | 13,500 | $3,240 | - |
| CatFee (32 SUN) | 2,580 | $619 | 81% |
| Netts (24 SUN) | 2,060 | $494 | 85% |
| ITRX 30 天 (22 SUN) | 1,930 | $463 | 86% |
| MERX（自动路由） | 2,000 | $480 | 85% |

关键在于 MERX 并非在每笔交易中都比每个供应商便宜 - 有时 Netts 最便宜，有时 ITRX 最便宜。其价值在于 MERX 始终接近最低价，因为它总是选择当前最佳选项，而无需你在六个供应商处维护账户并手动比较价格。

## 何时使用单一供应商 vs 聚合器

**使用单一供应商的情况：**
- 你已与该供应商协商了定制费率
- 你有锁定价格的长期合约
- 你每月只需要 energy 用于少量交易
- 你已测试并信任某个特定供应商

**使用 MERX 等聚合器的情况：**
- 你每天执行超过 100 笔交易
- 你的交易量不可预测
- 你希望最小化持续的运营开销
- 你需要供应商宕机时的自动故障转移
- 你希望使用单一 API 和单一计费关系

## 开始使用

要开始跨所有供应商比较价格，请访问 [merx.exchange](https://merx.exchange)。该平台展示所有集成供应商的实时定价。

对于程序化访问，[MERX 文档](https://merx.exchange/docs)涵盖了 API 身份验证、订单创建和价格查询。SDK 可用于 JavaScript 和 Python。对于 AI 代理集成，[MERX MCP 服务器](https://github.com/Hovsteder/merx-mcp)提供 52 个可通过 Model Context Protocol 访问的工具。

TRON energy 市场竞争激烈，这种竞争有利于买方。但要获取这些优势，需要看到完整的市场 - 而不仅仅是一个供应商的价格表。这就是聚合器的意义所在。
