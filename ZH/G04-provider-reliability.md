# 供应商可靠性:正常运行时间、速度和成交率对比

价格是选择 TRON 能量供应商时最显眼的指标。但仅凭价格不能说明全部。报价 22 SUN 的供应商如果订单需要 10 分钟才成交、15% 的失败率或委托在规定时间之前失效,那就毫无价值。

## 可靠性维度

### 正常运行时间
衡量供应商 API 响应和接受订单的时间比例。单个供应商可能维持 98-99% 的正常运行时间,1% 的停机意味着每年 87 小时。

### 成交速度
从下单到能量委托出现在目标地址的时间:
- **快速供应商**: 10-30 秒
- **中等供应商**: 30-120 秒
- **慢速供应商**: 2-10 分钟

### 成交率
成功完成的订单比例。95% 听起来很高,但对于每天 500 单的支付处理器,意味着 25 笔失败。

### 委托一致性
能量委托是否在整个声明的时长内持续。

## 聚合与可靠性

### 单一供应商
99% 正常运行时间 x 97% 成交率 = 96.03% 有效成功率

### 聚合模型(7 个供应商)
所有 7 个同时宕机的概率:0.01^7 = 几乎为零。如果主供应商无法完成订单,自动路由到下一个可用供应商:

```typescript
const order = await merx.createOrder({
  energy_amount: 65000, duration: '1h', target_address: wallet
});
// order.provider 告诉你谁完成了订单
// 你的代码无需处理供应商故障
```

## MERX 追踪的指标

- **API 响应时间** - 供应商 API 响应速度
- **订单成交时间** - 下单到确认委托的时间
- **成交率** - 成功完成订单的比例
- **价格准确性** - 成交价是否匹配报价
- **时长合规性** - 委托是否持续到规定时间
- **错误模式** - 错误类型和频率

价格相同时,更可靠的供应商获得订单。

## 构建可靠性感知系统

```typescript
async function reliableEnergyPurchase(amount: number, wallet: string): Promise<Order> {
  for (let attempt = 1; attempt <= 3; attempt++) {
    try {
      const order = await merx.createOrder({
        energy_amount: amount, duration: '5m', target_address: wallet
      });
      const filled = await waitForFill(order.id, { timeout: 60000 });
      if (filled) return order;
    } catch (error) {
      if (attempt === 3) throw error;
      await delay(2000 * attempt);
    }
  }
  throw new Error('Energy purchase failed');
}
```

## 结论

供应商可靠性远不止 API 是否响应。成交速度、成交率、委托一致性和故障转移能力共同决定你的能量采购是否真正支持你的运营。

聚合模型通过消除单一供应商依赖来实现接近完美的实际可靠性。对于任何交易吞吐量重要的自动化系统,聚合带来的可靠性提升与价格优化一样有价值。

在 [https://merx.exchange/docs](https://merx.exchange/docs) 探索供应商比较工具或在 [https://merx.exchange](https://merx.exchange) 测试平台。
