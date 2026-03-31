# 零佣金交易：MERX 商业模式

MERX 对能量交易收取零佣金。不在供应商价格上加价。没有隐藏费用。没有点差。你支付的正是供应商收取的价格，MERX 在此之上不添加任何费用。

这自然会引发疑问。一个没有收入的企业如何维持运营？这是一种最终以定价欺诈告终的亏本策略吗？有什么陷阱？

没有陷阱。但有策略。本文解释了 MERX 的零佣金模式如何运作、为什么存在，以及平台计划如何长期维持和变现。

---

## 零佣金的确切含义

当你通过 MERX 购买能量时，你支付的价格等于供应商的批发价格。如果最便宜的供应商以每单位 85 SUN 提供能量，你支付每单位 85 SUN。MERX 不添加利润空间。

举例说明，以下是典型的能量购买情况：

```
Energy ordered:         65,000 units
Best provider price:    85 SUN/unit
Total cost:             5,525,000 SUN = 5.525 TRX

MERX markup:            0 SUN
MERX fee:               0 SUN
Total charged to buyer: 5,525,000 SUN = 5.525 TRX
```

这是可验证的。每个订单响应都包含供应商名称和价格。你可以检查供应商自己的 API 来确认 MERX 没有虚报价格。这种透明是刻意的——它建立信任并使零佣金声明可审计。

---

## 为什么是零佣金

### 市场获取阶段

MERX 处于市场获取阶段。TRON 上的能量聚合市场还处于萌芽期——大多数开发者和企业仍然直接与单个供应商集成，或者更糟糕的是，为每笔交易销毁 TRX。当前的首要目标不是收入，而是用户增长。

零佣金消除了使用聚合器的主要反对意见："我为什么要付费给中间商，而不是直接去找供应商？"有了零佣金，中间商增加了价值（最优价格路由、故障转移、单一 API）而不增加成本。

### 飞轮效应

MERX 上的每个新用户都产生数据：订单量、价格敏感度、供应商偏好、使用模式。这些数据改善平台：

- **更多订单量**给 MERX 与供应商谈判更好费率的筹码。
- **更好的费率**吸引更多用户。
- **更多用户**产生更多数据用于路由优化。
- **更好的路由**带来更优的价格和更高的可靠性。

零佣金加速了飞轮。这是对网络效应的投资，会随时间复利增长。

### 与供应商直连定价的对比

各供应商自行设定价格，其中已包含利润。当你以 88 SUN/单位从 TronSave 购买时，这 88 已包括 TronSave 的运营成本和利润。MERX 原价传递此价格，不在其上添加任何费用。

一些供应商向大额买家提供批量折扣。MERX 通过聚合多个买家的交易量，有可能获得这些折扣并将节省传递给单个无法独自获得资格的用户。这是一个随平台交易量增长的未来收益。

---

## MERX 当前如何维持运营

运行 MERX 不是免费的。平台需要服务器、开发、监控和运营支持。在零佣金阶段，这些成本由创始团队作为商业投资承担。

### 当前成本结构

```
Infrastructure:
  - Dedicated server (Hetzner):       ~$150/month
  - Domain and SSL:                   ~$20/month
  - Monitoring and alerting:          ~$50/month

Development:
  - Founding team time:               Not externally funded
  - No venture capital (yet)
  - No token sale

Operational:
  - Provider API access:              Free (providers want volume)
  - TRON node access:                 Public nodes + own node
  - Redis, PostgreSQL:                Self-hosted on dedicated server
```

按任何标准来看，基础设施成本都很低。这是设计上的精益运营——在基础设施上每省一美元，就少一美元需要从用户那里回收。

---

## 未来收入模式

零佣金不是永久状态。这是 MERX 打算发展的市场的入场价。以下是收入模式如何演进：

### 阶段 1：零佣金（当前）

- 所有交易 0% 佣金。
- 目标：用户获取、供应商关系、平台成熟。
- 持续时间：直到建立有意义的订单量。

### 阶段 2：增值功能

第一笔收入来自超越基本价格路由的增值服务：

**常备订单和自动化**

```
Basic (free):     Manual orders, best-price routing
Premium:          Standing orders, auto-renewal, price alerts
                  Scheduled energy procurement
                  Webhook notifications for delegation events
```

**分析与洞察**

```
Basic (free):     Current prices, basic order history
Premium:          Price prediction models
                  Provider reliability scoring
                  Cost optimization recommendations
                  Custom reporting and export
```

**优先支持**

```
Basic (free):     Documentation, community support
Premium:          Direct support channel
                  SLA guarantees on order execution
                  Dedicated account management
```

基础服务——将订单路由到最优价格——保持免费。增值功能服务于从平台中获得足够价值以证明付费合理的高级用户。

### 阶段 3：基于交易量的定价

对于超高交易量用户（每天数百万能量单位），MERX 可能引入一个小额佣金来反映处理大额订单的运营成本：

```
Volume tier:       Commission:
0 - 1M energy/mo:  0% (forever free)
1M - 10M:          0.5%
10M - 100M:        0.3%
100M+:             Negotiated
```

即使在 0.5% 的佣金下，总成本仍然低于大多数用户通过直接供应商集成所支付的费用（因为他们无法获得最优价格路由）。

### 阶段 4：供应商服务

随着 MERX 发展为主导的聚合层，供应商从订单流中受益。未来的收入来源可能包括：

- **优先展示**：供应商付费获取路由优先权（保持透明——买家始终看到实际价格）。
- **供应商分析**：市场份额数据、竞争性定价情报。
- **结算服务**：MERX 处理收款和汇款，向供应商收取少量处理费。

---

## 与供应商加价的对比

MERX 的零佣金与供应商已经收取的费用相比如何？

供应商不是慈善机构。他们的报价中包含了运营成本和利润率。典型的供应商加价结构如下：

```
Provider cost structure:
  - TRX staking cost (opportunity cost):     Base cost
  - Infrastructure (servers, nodes):          5-10% of base
  - Development and maintenance:              5-10% of base
  - Profit margin:                            10-30% of base

Estimated markup over raw staking cost:       20-50%
```

当你以 85 SUN/单位从供应商购买时，大约 55-70 SUN 覆盖质押的原始成本，15-30 SUN 是供应商的管理费用和利润。这是正常且可持续的。

MERX 不添加另一层利润。供应商收取的 85 SUN 就是你支付的 85 SUN。与其他聚合模式对比：

```
Traditional aggregator:
  Provider price:     85 SUN/unit
  Aggregator markup:  5-15%
  You pay:            89-98 SUN/unit

MERX:
  Provider price:     85 SUN/unit
  MERX markup:        0%
  You pay:            85 SUN/unit
```

---

## 信任问题

"如果你不收费，你就是产品。"这在任何免费服务中都是合理的关切。让我们直接回应。

### MERX 不出售你的数据

订单数据用于优化路由和改善平台。它不会出售给第三方。没有广告模式。你的交易量和模式是你的隐私。

### MERX 不托管你的资金

MERX 持有用于订单执行的存款余额。这些是运营余额，不是托管资产。你可以随时提取余额。平台不会投资、借贷或以其他方式使用你存入的资金。

### MERX 不抢跑订单

订单执行器在执行时路由到最优可用价格。它不会延迟订单等待更好的价格（如果 MERX 持有头寸这样做会受益），也不会在有更便宜供应商可用时路由到更贵的供应商。

### 代码可审查

MERX SDK 是开源的：

- JavaScript SDK：[https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- Python SDK：[https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)
- MCP 服务器：[https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)

每笔订单都包含供应商名称和链上交易哈希。你可以独立验证委托确实以 MERX 报价的价格发生。

---

## 为什么不直接使用供应商？

如果 MERX 收取零佣金，为什么不直接与供应商集成，跳过聚合层？

你可以。但以下是你放弃的：

### 最优价格路由

供应商价格全天变化。上午 9 点最便宜的供应商可能在下午 3 点最贵。没有持续监控（MERX 每 30 秒执行一次），你在任何给定时刻可能都没有获得最优价格。

### 自动故障转移

如果你的单一供应商宕机，你的应用停止运行。有了 MERX，供应商故障对你不可见——订单自动路由到下一个最便宜的选项。

### 运营简便性

七个供应商集成意味着七套 API 文档、七种认证方式、七种错误格式、七套需要跟踪的 API 变更。一个 MERX 集成替代了所有这些。

### 面向未来

新供应商进入市场。现有供应商更改 API 或关闭。有了 MERX，新供应商自动对你可用，已关闭的供应商被移除——无需代码变更。

零佣金模式意味着你在零额外成本的情况下获得所有这些好处，与直接使用供应商没有差别。直接使用供应商的唯一合理原因是如果你有 MERX 不支持的特定需求——如果是这样，团队希望了解详情。

---

## 致早期用户

早期用户的零佣金费率没有时间限制。在此阶段加入的用户将在平台发展过程中享有优惠条款。这不是先低价后涨价的策略；这是对早期用户承担更多风险（更新的平台、更少的跟踪记录）的认可，他们理应获得相应的回报。

如果收入模式最终包含佣金，早期用户将获得以下之一：
- 永久保持零佣金，或
- 相对于后来用户获得显著折扣

任何定价变更都将提前充分告知。

---

## 快速上手

创建账户并开始零佣金交易：

1. 访问 [https://merx.exchange](https://merx.exchange)
2. 创建账户
3. 生成 API 密钥
4. 下第一笔订单

```typescript
import { MerxClient } from 'merx-sdk';

const client = new MerxClient({ apiKey: 'your-key' });

// See current best prices (no commission added)
const prices = await client.getPrices({ energy: 65000 });
console.log(`Best price: ${prices.bestPrice.perUnit} SUN/unit`);
console.log(`Provider: ${prices.bestPrice.provider}`);
console.log(`MERX fee: 0`);
```

文档：[https://merx.exchange/docs](https://merx.exchange/docs)

---

*本文是 MERX 知识系列的一部分。MERX 是首个区块链资源交易所，在所有主要 TRON 能量供应商之间提供零佣金能量交易和最优价格路由。*
