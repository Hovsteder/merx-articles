# 能量委托中的竞态条件:我们如何解决它们

这是一个关于一个 bug 的故事,该 bug 导致每笔交易花费 19 TRX 而不是节省 1.43 TRX。这是一个在测试中看起来正确、在测试网上正常工作、只在主网条件下才暴露的竞态条件的故事。

## 这个 Bug

初始实现遵循顺序模式:

```typescript
async function executeWithEnergy(targetAddress, energyAmount, transactionFn) {
  const order = await merx.createOrder({
    energy_amount: energyAmount, target_address: targetAddress, duration: '1h'
  });
  await waitForOrderStatus(order.id, 'FILLED');
  const txHash = await transactionFn();
  return txHash;
}
```

问题在于"FILLED"的含义。在 MERX 系统中,订单在供应商确认已发起委托时标记为 FILLED。但"已提交"不等于"已确认"。委托是一笔需要被包含在区块中的 TRON 交易。在提交和确认之间,有一个窗口 - 通常 3 到 6 秒。

在该窗口期间,目标地址尚没有委托的能量。

### 主网上发生的情况

```
时间线:
  T+0.0s    订单创建,发送给供应商
  T+0.3s    供应商响应:委托交易已提交
  T+0.4s    订单状态 -> FILLED
  T+0.5s    用户交易广播(USDT 转账)
  T+0.6s    用户交易包含在区块 N
  T+3.2s    委托交易包含在区块 N+1

  结果:
    - 区块 N 中的用户交易:委托尚不存在
    - 消耗能量: 0
    - TRX 燃烧: 27.3 TRX
    - 能量租赁成本: 1.82 TRX
    - 总成本: 29.12 TRX (租赁 + 燃烧)
    - 预期成本: 1.82 TRX (仅租赁)
```

每次竞态条件发生多付 27.3 TRX。在糟糕的日子里,10-20% 的订单会触发此竞态条件。

### 为什么测试没有捕获

测试网上,网络拥堵最小,委托交易通常包含在下一个区块中。测试工具在步骤之间有内置的 2 秒延迟,掩盖了竞态条件。

## 修复

### 轮询直到链上确认

```typescript
async function executeWithEnergy(targetAddress, energyAmount, transactionFn) {
  const baseline = await checkAddressResources(targetAddress);
  const baselineEnergy = baseline.energy_limit - baseline.energy_used;

  const order = await merx.createOrder({
    energy_amount: energyAmount, target_address: targetAddress, duration: '1h'
  });
  await waitForOrderStatus(order.id, 'FILLED');

  // 轮询链上资源直到委托确认
  const confirmed = await pollUntilDelegationConfirmed(
    targetAddress, baselineEnergy, energyAmount,
    { maxAttempts: 15, intervalMs: 2000 }
  );

  if (!confirmed) {
    throw new Error('委托未在链上确认。不执行交易。');
  }

  const txHash = await transactionFn();
  return txHash;
}
```

### 真实主网数据

修复部署后一周的生产流量测量:

```
委托确认时序:
  首次轮询确认 (0-2s):     12%
  第二次轮询确认 (2-4s):    61%
  第三次轮询确认 (4-6s):    22%
  第四次轮询确认 (6-8s):    4%
  第五次及以上 (8s+):       1%
  超时 (30s 内未确认):      0.04%

从订单 FILLED 到链上确认的平均时间: 3.1 秒
```

### 成本影响

```
修复前 (30 天):
  受竞态条件影响的订单: ~180
  每次事件平均多付: ~19 TRX
  竞态条件总成本: ~3,420 TRX

修复后 (30 天):
  竞态条件事件: 0
  超时失败: 2 (均被超时捕获,交易未执行)
```

## 经验教训

1. **"已提交"不等于"已确认"** - 在任何与区块链交互的系统中,提交交易和确认之间总有间隔。
2. **查链,而非查服务** - 唯一权威来源是区块链本身。
3. **测试网隐藏时序 bug** - 时序敏感的 bug 在测试网上可能永远不会触发。
4. **大声失败** - 轮询超时时抛出错误比执行后燃烧 TRX 要好得多。

## MERX 实现

MERX MCP 服务器中的 `ensure_resources` 工具实现了此模式。智能体不需要自己实现轮询逻辑。竞态条件在平台层面处理。

```
Tool: ensure_resources
Input: { "address": "TYourAddress...", "energy_needed": 65000 }

Response: {
  "status": "confirmed",
  "energy_available": 65000,
  "confirmation_time_ms": 4200,
  "order_id": "ord_abc123"
}
```

平台: [https://merx.exchange](https://merx.exchange)
文档: [https://merx.exchange/docs](https://merx.exchange/docs)
MCP 服务器: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
