# TRON 上能量委托的工作原理

能量委托是使整个 TRON 能量租赁市场成为可能的机制。没有它，能量将不可转让——质押者只能使用自己的能量，无法出售或共享。理解委托在协议层面的工作方式，对于在 TRON 上构建应用或评估能量供应商的任何人来说都至关重要。

本文详细解释了 Stake 2.0 的委托机制：供应商如何将能量委托给买家、委托期间和委托之后发生了什么，以及 MERX 如何跨多个供应商管理完整的委托生命周期。

---

## Stake 2.0 基础

TRON 引入 Stake 2.0（也称为 Stake v2）作为原始资源冻结机制的替代方案。Stake 2.0 的核心创新是将质押与资源使用分离。在原始模型（Stake 1.0）下，冻结的 TRX 产生的能量只能由冻结者自己的地址使用。Stake 2.0 引入了委托，允许一个地址将其质押资源定向到网络上的任何其他地址。

### 三个操作

Stake 2.0 委托涉及三个链上操作：

**1. 质押（freezeBalanceV2）**

供应商将 TRX 锁定在质押合约中。这将按照质押量和全网总质押量的比例产生能量。能量在 24 小时窗口内持续恢复。

```
Provider stakes 36,000 TRX
Network allocates ~65,000 energy/day to provider
```

**2. 委托（delegateResource）**

供应商将其一部分能量定向到目标地址。这是实际的委托操作。此交易确认后，目标地址可以像使用自己的能量一样使用委托的能量。

```json
{
  "owner_address": "TProviderAddress...",
  "receiver_address": "TBuyerAddress...",
  "resource": "ENERGY",
  "balance": 36000000000,
  "lock": true,
  "lock_period": 3
}
```

关键参数：
- `owner_address`：供应商地址（委托方）
- `receiver_address`：买家地址（接收方）
- `resource`：ENERGY 或 BANDWIDTH
- `balance`：支持此委托的 TRX 数量（以 SUN 为单位）
- `lock`：委托是否锁定最短期限
- `lock_period`：锁定时长（天数，如果 lock 为 true）

**3. 取消委托（undelegateResource）**

当委托期限到期时，供应商通过取消委托来回收资源。此交易确认后，能量停止流向买家地址并返回供应商的资源池。

```json
{
  "owner_address": "TProviderAddress...",
  "receiver_address": "TBuyerAddress...",
  "resource": "ENERGY",
  "balance": 36000000000
}
```

---

## 委托生命周期

一个完整的委托经历几个阶段。理解每个阶段对于围绕能量租赁构建可靠系统至关重要。

### 阶段 1：下单

买家为特定地址和时长请求能量。这发生在链下——买家向供应商（直接或通过 MERX 等聚合器）付款，供应商将委托排入队列。

### 阶段 2：链上委托

供应商广播 `delegateResource` 交易。这通常在 TRON 上 3-6 秒内确认（一个区块）。确认后，能量立即在买家地址可用。

```
Buyer's effective energy = own_staked_energy + all_delegated_energy
```

重要提示：买家可以立即使用委托的能量。没有预热期。一旦委托交易被确认，买家的下一次智能合约调用将消耗委托的能量。

### 阶段 3：活跃委托期

在委托期间，买家使用能量进行操作。此阶段的关键行为：

- **恢复**：委托的能量与自行质押的能量以相同的速率恢复——在 24 小时内持续恢复。
- **消耗优先级**：当买家同时拥有自行质押和委托的能量时，TRON 不区分它们。所有能量被汇总使用。
- **多重委托**：买家可以同时从多个供应商接收委托。能量数量是累加的。

### 阶段 4：到期与回收

当锁定期到期时，供应商可以执行 `undelegateResource` 交易来回收资源。这不是自动的——供应商必须主动调用此函数。

```
Timeline:
  T+0:   delegateResource confirmed (energy active)
  T+3d:  Lock period expires
  T+3d+: Provider calls undelegateResource (energy removed)
```

如果供应商不取消委托，能量将无限期地继续流向买家。供应商有动力及时回收，因为在取消委托之前，支持委托的质押 TRX 无法用于其他客户。

### 阶段 5：取消委托后

取消委托确认后，供应商的 TRX 返回其"可委托"池。然后他们可以将其委托给下一个客户。买家的可用能量将减少取消委托的数量。

---

## 锁定期机制

委托交易中的 `lock` 参数和 `lock_period` 值得特别注意。

### 锁定委托

当 `lock: true` 时，供应商承诺在至少 `lock_period` 天内维持委托。在此期间，供应商无法取消委托。这为买家提供了资源可用性的保证。

```
lock_period = 3 days

Day 0: Delegation created, lock starts
Day 1: Provider CANNOT undelegate
Day 2: Provider CANNOT undelegate
Day 3: Lock expires, provider CAN undelegate
```

### 非锁定委托

当 `lock: false` 时，供应商可以随时取消委托——甚至在委托后立即取消。这对买家来说是有风险的，因为供应商可能在买家使用之前就回收能量。

实际上，信誉良好的供应商始终使用锁定委托。MERX 验证所有委托都针对购买的时长正确锁定。

### 锁定期粒度

TRON 的锁定期以天为单位指定（技术上是区块，但协议映射到大约 24 小时的周期）。最小实际锁定为 1 天。市场上常见的时长：

| 锁定期 | 典型使用场景 |
|-------------|-----------------|
| 0（未锁定） | 不推荐 |
| 1 天 | 短期/测试 |
| 3 天 | 标准租赁 |
| 7 天 | 周运营 |
| 14 天 | 双周覆盖 |
| 30 天 | 月度合同 |

---

## 能量交付验证

你如何知道委托确实发生了？有几种验证方法。

### 链上验证

直接查询 TRON 网络：

```typescript
// Using TronWeb
const accountResources = await tronWeb.trx.getAccountResources(buyerAddress);
console.log('Total energy limit:', accountResources.EnergyLimit);
console.log('Energy used:', accountResources.EnergyUsed);
console.log('Available:', accountResources.EnergyLimit - accountResources.EnergyUsed);
```

`EnergyLimit` 字段反映来自所有来源的总能量：自行质押和委托。委托后 `EnergyLimit` 的增加确认了交付。

### 委托记录查询

TRON 提供 API 来查询某地址的委托记录：

```typescript
// Get all delegations received by an address
const delegations = await tronWeb.trx.getDelegatedResourceV2(
  buyerAddress,
  providerAddress
);
```

这将返回从给定供应商到买家的具体委托金额和锁定期。

### MERX 验证

MERX 在每笔订单后执行自动验证：

1. 下单并付款。
2. 供应商执行委托交易。
3. MERX 监控区块链上的委托交易确认。
4. MERX 查询买家的账户资源以验证能量增加。
5. 如果验证失败（能量未在超时时间内到账），订单被标记并退款给买家。

```typescript
import { MerxClient } from 'merx-sdk';

const client = new MerxClient({ apiKey: 'your-key' });

// Place order
const order = await client.createOrder({
  energy: 65000,
  targetAddress: 'TBuyerAddress...',
  duration: '1d'
});

// Check order status (includes delegation verification)
const status = await client.getOrder(order.id);
console.log(status.delegationStatus);
// 'pending' | 'confirmed' | 'verified' | 'failed'
```

---

## 多供应商委托

单个地址可以同时从多个供应商接收委托。这就是 MERX 等聚合器如何将大额订单拆分到多个供应商的原理。

### 工作原理

```
Buyer address: TBuyer123...

Delegation 1: Provider A -> 200,000 energy (65,000 x 3)
Delegation 2: Provider B -> 130,000 energy (65,000 x 2)
Delegation 3: Provider C ->  65,000 energy (65,000 x 1)

Total delegated energy: 395,000/day
Plus buyer's own staked energy: 0
Total available energy: 395,000/day
```

所有委托都是独立的。供应商 A 取消委托不会影响供应商 B 或 C 的委托。买家的总能量就是所有活跃委托加上自己质押能量的总和。

### 这对聚合为何重要

当买家通过 MERX 订购 500,000 能量时，可能没有单一供应商有这么多可用量。MERX 可以拆分订单：

```
Order: 500,000 energy for TBuyer123...

Routing:
  Provider A: 200,000 energy at 82 SUN/unit  = 16,400,000 SUN
  Provider B: 180,000 energy at 85 SUN/unit  = 15,300,000 SUN
  Provider C: 120,000 energy at 88 SUN/unit  = 10,560,000 SUN

Total cost: 42,260,000 SUN
Effective rate: 84.52 SUN/unit
```

买家看到一个订单和一个混合费率。在后台，三笔独立的委托交易在链上执行。

---

## MERX 如何管理委托生命周期

MERX 处理每一笔委托的完整生命周期，从下单到到期，涵盖所有集成的供应商。

### 订单执行

1. **价格检查**：轮询所有供应商获取当前最优价格。
2. **路由**：选择能够完成订单的最便宜的供应商。
3. **执行**：通过供应商的 API 提交订单。
4. **监控**：监视链上委托交易。
5. **验证**：确认能量到达目标地址。
6. **通知**：通过 webhook 或 WebSocket 通知买家。

### 活跃期间

- **健康监控**：定期验证委托仍然有效。
- **资源追踪**：监控买家的能量使用以检测问题。
- **告警**：如果能量即将耗尽或使用模式表明需要更多能量，通知买家。

### 到期时

- **倒计时通知**：在委托到期前提醒买家。
- **续约选项**：以当前最优价格提供自动续约。
- **到期后验证**：确认能量已被供应商正确回收。

### 错误处理

- **委托失败**：如果供应商未在预期时间内完成委托，MERX 自动路由到下一个最便宜的供应商。
- **部分完成**：如果供应商只能完成订单的一部分，MERX 从其他供应商补充剩余部分。
- **供应商宕机**：如果供应商不可达，其价格从订单簿中移除，订单路由到可用的供应商。

---

## 常见边界情况

### 委托到期前能量已用完

买家没有义务"归还"未使用的能量。当委托到期且供应商取消委托时，能量只是不再可用。无需归还。

### 供应商委托了超过购买量的能量

偶尔，供应商可能会委托超过买家支付的能量（由于内部核算差异）。买家在委托期间受益于额外的能量。MERX 追踪订购和交付的确切数量。

### 同一地址的多个订单

如果买家在不同时间为同一目标地址下了多个订单，它们会累积。每个委托都是独立的，按照各自的锁定期到期。

### 地址不存在

TRON 允许向任何有效地址委托，即使该地址从未被激活。当地址被激活时，能量将可用。但大多数供应商和 MERX 在处理订单前会验证目标地址是否存在且已激活。

---

## 结论

能量委托是 TRON 资源经济的基础。Stake 2.0 机制将能量从不可转让的资源转变为可交易的商品，使整个供应商生态系统成为可能。理解委托的工作原理——锁定机制、验证流程、多供应商组合——对于在 TRON 上构建生产系统的任何人来说都至关重要。

MERX 将跨多个供应商管理委托的复杂性抽象为单一 API 调用，但了解底层运作原理能帮助你在时长、数量和供应商选择上做出更好的决策。

在 [https://merx.exchange/docs](https://merx.exchange/docs) 探索 MERX API 并开始以编程方式管理委托。

---

*本文是 MERX TRON 基础设施知识系列的一部分。MERX 是首个区块链资源交易所。GitHub：[https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)。*
