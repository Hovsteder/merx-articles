# MERX 订单路由：你的订单如何获得最优价格

当你向 MERX 提交能量订单时，一系列决策在毫秒内发生：哪个供应商价格最优、他们能否完成此订单、如果失败会怎样、如何验证委托已上链。这就是订单路由引擎——将一个简单的 API 调用转化为优化的、容错的、经过验证的能量交付的组件。

本文完整介绍了 MERX 订单的生命周期，从你调用 `createOrder` 的那一刻到委托的能量出现在你的 TRON 地址的那一刻。

---

## 订单生命周期

每笔订单经过六个阶段：

```
1. Validation    -> Is the order well-formed?
2. Pricing       -> What is the best price right now?
3. Routing       -> Which provider(s) will fill it?
4. Execution     -> Submit to provider(s)
5. Verification  -> Confirm on-chain delegation
6. Settlement    -> Debit balance, record in ledger
```

让我们逐一详解。

---

## 阶段 1：验证

在任何路由逻辑运行之前，订单先被验证：

```typescript
// Input validation with Zod
const OrderSchema = z.object({
  energy: z.number().int().min(10000).max(100000000),
  targetAddress: z.string().refine(isValidTronAddress),
  duration: z.enum(['1h', '1d', '3d', '7d', '14d', '30d']),
  maxPrice: z.number().optional(),  // SUN per energy unit
  idempotencyKey: z.string().optional()
});
```

关键验证项：

- **能量数量**：必须是支持范围内的正整数。
- **目标地址**：必须是有效的、已激活的 TRON 地址。系统验证地址格式，并可选地检查链上账户是否存在。
- **时长**：必须是支持的委托期限之一。
- **最高价格**：可选上限。如果设置，只有当最优可用价格等于或低于此阈值时订单才会执行。
- **幂等键**：如果提供，使用相同键的重复提交将返回原始订单而非创建新订单。

### 幂等性

幂等键对于生产集成至关重要。网络问题可能导致客户端重试请求，可能创建重复订单。有了幂等键，第二次请求返回第一次的结果：

```typescript
const order = await client.createOrder({
  energy: 65000,
  targetAddress: 'TBuyerAddress...',
  duration: '1h',
  idempotencyKey: 'payment-123-energy'
});

// If called again with the same key, returns the same order
// No duplicate delegation, no double charge
```

---

## 阶段 2：定价

订单执行器从 Redis 缓存读取当前价格。价格监控每 30 秒更新一次，因此数据最多 30 秒前的。

```typescript
async function getBestPrices(
  energyAmount: number,
  duration: string
): Promise<ProviderPrice[]> {

  const allPrices = await redis.keys('prices:*');
  const validPrices = [];

  for (const key of allPrices) {
    const price = JSON.parse(await redis.get(key));

    // Filter: must support requested duration
    if (!price.durations.includes(duration)) continue;

    // Filter: must have sufficient availability
    if (price.availableEnergy < energyAmount) continue;

    // Filter: must pass health threshold
    const health = await getProviderHealth(price.provider);
    if (health.fillRate < 0.90) continue;

    validPrices.push(price);
  }

  // Sort by effective price (accounting for reliability)
  return validPrices.sort((a, b) => {
    const effectiveA = a.energyPricePerUnit / a.fillRate;
    const effectiveB = b.energyPricePerUnit / b.fillRate;
    return effectiveA - effectiveB;
  });
}
```

### 最高价格执行

如果买家指定了 `maxPrice`，当没有供应商能满足时订单将被拒绝：

```typescript
if (maxPrice && bestPrice.energyPricePerUnit > maxPrice) {
  throw new OrderError({
    code: 'PRICE_EXCEEDED',
    message: `Best available price (${bestPrice.energyPricePerUnit} SUN) exceeds your maximum (${maxPrice} SUN)`,
    details: { bestAvailable: bestPrice.energyPricePerUnit, maxPrice }
  });
}
```

这防止了价格飙升期间的意外扣费。

---

## 阶段 3：路由

路由引擎决定哪个或哪些供应商将完成订单。这是系统的核心智能。

### 简单情况：单供应商完成

对于在单一供应商容量范围内的订单：

```
Order: 65,000 energy
Best provider: itrx at 85 SUN/unit, 500,000 available

Route: 100% to itrx
```

### 拆分情况：多供应商完成

对于大额订单或最便宜供应商库存有限时：

```
Order: 500,000 energy

Provider A: 200,000 available at 85 SUN
Provider B: 180,000 available at 87 SUN
Provider C: 300,000 available at 92 SUN

Routing plan:
  Leg 1: Provider A -> 200,000 energy at 85 SUN
  Leg 2: Provider B -> 180,000 energy at 87 SUN
  Leg 3: Provider C -> 120,000 energy at 92 SUN

Blended rate: (200K*85 + 180K*87 + 120K*92) / 500K = 87.28 SUN
```

路由器从最便宜的供应商开始，从每个供应商尽可能多地获取，然后再转向下一个。

### 故障转移链

每个路由计划都包含一个故障转移链——在主要供应商失败时的备选供应商有序列表：

```
Primary:   Provider A (85 SUN)
Failover 1: Provider B (87 SUN)
Failover 2: Provider C (92 SUN)
Failover 3: Provider D (95 SUN)
```

如果供应商 A 执行失败（API 错误、超时、资金不足），执行器自动切换到供应商 B，无需买家的任何操作。

---

## 阶段 4：执行

执行器将订单提交给选定的供应商并监控完成情况。

### 执行流程

```typescript
async function executeOrder(
  order: Order,
  routingPlan: RoutingPlan
): Promise<ExecutionResult> {

  const results: LegResult[] = [];

  for (const leg of routingPlan.legs) {
    try {
      const result = await executeLeg(leg, order);
      results.push(result);
    } catch (error) {
      // Primary provider failed - try failover
      const failoverResult = await executeWithFailover(
        leg,
        order,
        routingPlan.failoverChain
      );
      results.push(failoverResult);
    }
  }

  return {
    orderId: order.id,
    legs: results,
    totalEnergy: results.reduce((sum, r) => sum + r.energy, 0),
    totalCostSun: results.reduce((sum, r) => sum + r.costSun, 0)
  };
}

async function executeWithFailover(
  failedLeg: RoutingLeg,
  order: Order,
  failoverChain: Provider[]
): Promise<LegResult> {

  for (const provider of failoverChain) {
    try {
      const result = await executeLeg(
        { ...failedLeg, provider: provider.name },
        order
      );
      return result;
    } catch (error) {
      // Log and continue to next failover
      continue;
    }
  }

  throw new OrderError({
    code: 'ALL_PROVIDERS_FAILED',
    message: 'Order could not be filled by any available provider'
  });
}
```

### 超时处理

每个供应商执行都有严格的超时。如果供应商在超时窗口内（通常 30 秒）未确认订单，执行器转向故障转移链：

```
T+0:    Submit order to Provider A
T+30s:  No response -> timeout, failover to Provider B
T+31s:  Submit order to Provider B
T+35s:  Provider B acknowledges -> proceed to verification
```

---

## 阶段 5：验证

供应商的确认还不够。MERX 验证能量委托确实出现在了 TRON 区块链上。

### 链上验证流程

```typescript
async function verifyDelegation(
  targetAddress: string,
  expectedEnergy: number,
  delegationTxHash: string
): Promise<VerificationResult> {

  // Step 1: Verify the delegation transaction exists
  const tx = await tronWeb.trx.getTransaction(delegationTxHash);
  if (!tx || tx.ret[0].contractRet !== 'SUCCESS') {
    return { verified: false, reason: 'Transaction not found or failed' };
  }

  // Step 2: Check target address resources
  const resources = await tronWeb.trx.getAccountResources(targetAddress);
  const currentEnergy = resources.EnergyLimit || 0;

  // Step 3: Verify energy increased by expected amount (with tolerance)
  const tolerance = expectedEnergy * 0.02; // 2% tolerance
  if (currentEnergy < expectedEnergy - tolerance) {
    return { verified: false, reason: 'Energy amount below expected' };
  }

  return {
    verified: true,
    txHash: delegationTxHash,
    energyDelivered: currentEnergy,
    verifiedAt: new Date()
  };
}
```

### 为什么 2% 容差

能量委托数量基于 TRX 到能量的转换比率，该比率在下单和委托确认之间可能略有变化。2% 的容差考虑了这一点，同时不接受严重不正确的数量。

### 验证时序

```
T+0:    Provider acknowledges order
T+3-6s: Delegation transaction confirms on TRON (1 block)
T+10s:  MERX queries target address resources
T+10s:  Verification complete, buyer notified
```

从下单到验证交付的整个过程通常需要 15-45 秒。

---

## 阶段 6：结算

验证后，进行财务结算：

### 余额扣减

```sql
-- Atomic balance check and deduction
BEGIN;

SELECT balance_sun FROM accounts
WHERE user_id = $1
FOR UPDATE;

-- Verify sufficient balance
-- If insufficient, ROLLBACK and return error

UPDATE accounts
SET balance_sun = balance_sun - $2
WHERE user_id = $1;

COMMIT;
```

`SELECT FOR UPDATE` 确保余额检查和扣减之间没有竞态条件。如果两笔订单同时处理，第二笔将等待第一笔完成后再检查余额。

### 账本记录

每笔结算创建一条不可变的账本记录：

```sql
INSERT INTO ledger (
  user_id, type, amount_sun,
  reference_type, reference_id,
  balance_before, balance_after,
  created_at
) VALUES (
  $1, 'ORDER_PAYMENT', $2,
  'order', $3,
  $4, $5,
  NOW()
);
```

账本记录仅追加。永远不会被更新或删除。这创建了每笔财务操作的完整、可审计的历史记录。

---

## 追踪你的订单

MERX API 通过 REST 和 WebSocket 提供实时订单状态：

```typescript
import { MerxClient } from '@merx/sdk';

const client = new MerxClient({ apiKey: 'your-key' });

// REST: poll order status
const order = await client.getOrder('ord_abc123');
console.log(order.status);
// 'pending' | 'executing' | 'verifying' | 'completed' | 'failed'

// WebSocket: real-time updates
client.onOrderUpdate('ord_abc123', (update) => {
  console.log(`Order ${update.orderId}: ${update.status}`);
  if (update.status === 'completed') {
    console.log(`Delegation TX: ${update.delegationTxHash}`);
  }
});
```

### 订单状态流

```
pending -> executing -> verifying -> completed
                |                       |
                v                       v
              failed              partially_filled
```

- **pending**：订单已接收，等待执行。
- **executing**：已提交给供应商，等待确认。
- **verifying**：供应商已确认，等待链上验证。
- **completed**：能量已在目标地址验证。
- **failed**：所有供应商均失败。余额未扣费。
- **partially_filled**：部分段完成，其他段失败。仅对已完成部分扣费。

---

## 错误处理

### 供应商错误

每个供应商可能以独特的方式失败。订单执行器将所有供应商错误规范化为标准错误码：

```json
{
  "error": {
    "code": "PROVIDER_UNAVAILABLE",
    "message": "Primary provider could not fill the order. Failover to secondary provider.",
    "details": {
      "primaryProvider": "tronsave",
      "primaryError": "timeout",
      "filledBy": "feee",
      "priceImpact": "+2 SUN/unit"
    }
  }
}
```

### 部分完成

对于多段订单，部分段可能成功而其他段失败。MERX 的处理方式：

1. 正常完成成功的段。
2. 为失败的段尝试故障转移。
3. 如果故障转移也失败，返回部分完成结果。
4. 仅对实际交付的能量向买家扣费。

---

## 性能

典型的订单执行时间：

```
Validation:    < 5ms
Pricing:       < 10ms (Redis read)
Routing:       < 5ms
Execution:     5-30 seconds (provider API + blockchain)
Verification:  3-10 seconds (blockchain confirmation)

Total: 10-45 seconds from API call to verified delivery
```

瓶颈始终在区块链。MERX 的内部处理增加不到 50ms 的开销。其余时间在等待供应商和 TRON 网络。

---

## 结论

订单路由是 MERX 价值最直观的体现。一次 API 调用触发了一系列优化：最优价格选择、多供应商拆分、自动故障转移、链上验证和原子结算。每一步都旨在确保你获得最便宜的可用能量，以可靠的方式交付，具有完整的可审计性。

复杂性是真实的，但这是 MERX 要管理的复杂性，而不是你的。你的集成仍然是一次 API 调用。

在 [https://merx.exchange](https://merx.exchange) 开始路由订单。完整 API 参考请访问 [https://merx.exchange/docs](https://merx.exchange/docs)。

---

*本文是 MERX 技术系列的一部分。MERX 是首个区块链资源交易所。SDK：[https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) 和 [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)。MCP 服务器：[https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)。*
