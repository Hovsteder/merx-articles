# 购买 TRON 能量的最佳时机:数据驱动分析

每个 TRON 能量买家都面临同一个问题:现在买,还是一小时后价格会更好?答案取决于数据 - 历史价格模式、当前市场条件和你对时机风险的具体容忍度。

## 百分位购买策略

最有效的时机方法不是试图击中绝对最低点,而是瞄准一个平衡成本节省与成交可靠性的百分位。

| 目标百分位 | 成交频率 | vs 中位数节省 | 风险水平 |
|---|---|---|---|
| 第 5 百分位 | 每天约 1-2 小时 | 最大 | 高 |
| 第 10 百分位 | 每天约 2-3 小时 | 很高 | 中高 |
| 第 25 百分位 | 每天约 6 小时 | 高 | 中等 |
| 第 50 百分位(中位数) | 每天约 12 小时 | 中等 | 低 |

### 选择策略

**时间灵活的操作**(批处理、非紧急分发):目标 10-25 百分位。

**半紧急操作**(标准业务处理):目标 25-50 百分位。

**时间关键的操作**(实时支付):目标 50-75 百分位或按市场价购买。

## 分层常备订单

```typescript
// 第 1 层:激进 - 价格很低时大量购买
const tier1 = await merx.createStandingOrder({
  energy_amount: 200000, max_price_sun: 22, duration: '1h', repeat: true
});

// 第 2 层:温和 - 低于平均价购买
const tier2 = await merx.createStandingOrder({
  energy_amount: 100000, max_price_sun: 25, duration: '1h', repeat: true
});

// 第 3 层:保守 - 可靠成交
const tier3 = await merx.createStandingOrder({
  energy_amount: 65000, max_price_sun: 30, duration: '1h', repeat: true
});
```

价格很低时买更多,平均价时买适量,较高价时买最低需求。结果是混合平均成本始终低于市场费率。

## 缓冲策略

维护能量缓冲,让常备订单有时间成交而不中断操作:

```typescript
class TimingStrategy {
  async execute(): Promise<void> {
    const resources = await this.merx.checkResources(this.wallet);
    if (resources.energy.available < this.emergencyThreshold) {
      await this.buyAtMarket(); // 紧急:按市场价购买
    } else if (resources.energy.available < this.bufferEnergy) {
      await this.createModerateOrder(); // 低于缓冲:温和价格
    } else {
      await this.createAggressiveOrder(); // 缓冲充足:激进价格
    }
  }
}
```

## 结论

购买 TRON 能量的最佳时机不是特定的某个小时或某天 - 而是价格达到你的目标水平的时刻,由常备订单自动捕捉。

数据驱动的方法很简单:分析历史价格了解分布,设定目标在 25 百分位,创建常备订单,维护缓冲,追踪结果并调整。

在 [https://merx.exchange](https://merx.exchange) 开始分析价格或在 [https://merx.exchange/docs](https://merx.exchange/docs) 探索分析 API。
