# x402 按使用付费:无需账户购买 TRON 能量

## 注册问题

每个区块链服务都遵循相同的模式:创建账户、验证邮箱、生成 API 密钥、充值内部余额,然后开始使用服务。对于人类用户,这是小小的不便。对于自主 AI 智能体,这是一个硬性障碍。

智能体没有邮箱地址。它们不想要由第三方管理的内部余额。它们不想信任平台托管其资金。智能体想要的很简单:为服务付费,接受服务,继续前进。没有关系。没有状态。没有信任。

x402 协议使这成为可能。受 HTTP 402 "Payment Required" 状态码(1997 年定义但从未广泛实施)的启发,x402 实现了按使用付费的商务模式,付款通过链上验证而非账户和 API 密钥。

MERX 为能量购买实现了 x402。任何拥有 TRON 钱包的实体 - 人类、智能体或智能合约 - 都可以在一笔交易中购买能量,无需创建账户,无需存入资金,与 MERX 平台无任何先前关系。

## 工作原理

x402 流程有五个步骤。每一步都可以在链上验证,买方在任何时候都不需要信任 MERX 托管其资金。

### 步骤 1:请求发票

买方使用所需的能量参数调用 `create_paid_order`:

```
Tool: create_paid_order
Input: {
  "energy_amount": 65000,
  "duration_hours": 1,
  "target_address": "TBuyerAddress..."
}

Response:
{
  "invoice": {
    "order_id": "xpay_abc123",
    "amount_trx": 1.43,
    "amount_sun": 1430000,
    "pay_to": "TMerxTreasuryAddress...",
    "memo": "merx_xpay_abc123",
    "expires_at": "2026-03-30T12:05:00Z",
    "energy_amount": 65000,
    "duration_hours": 1,
    "target_address": "TBuyerAddress..."
  }
}
```

发票是报价,而非承诺。没有资金移动。买方可以检查价格、与替代方案比较,然后决定是否继续。发票在 5 分钟后过期 - 如果在该窗口内未付款,报价的价格不再保证。

### 步骤 2:本地签名付款

买方构建一笔从其钱包到 MERX 资金库地址的 TRX 转账交易。关键细节是备注字段,必须包含发票中的确切字符串。

```javascript
const tx = await tronWeb.transactionBuilder.sendTrx(
  invoice.pay_to,        // TMerxTreasuryAddress
  invoice.amount_sun,    // 1430000 (1.43 TRX 以 SUN 计)
  buyerAddress
);

// 向交易数据添加备注
const txWithMemo = await tronWeb.transactionBuilder.addUpdateData(
  tx,
  invoice.memo,          // "merx_xpay_abc123"
  'utf8'
);

// 本地签名 - 私钥不离开买方的机器
const signedTx = await tronWeb.trx.sign(txWithMemo);
```

付款使用买方的私钥在买方的机器上签名。私钥永不与 MERX 共享,永不通过网络传输,永不存储在买方控制之外的任何地方。

### 步骤 3:广播付款

签名的交易广播到 TRON 网络:

```javascript
const result = await tronWeb.trx.sendRawTransaction(signedTx);
const txHash = result.txid;
```

此时,付款在链上。任何查询 TRON 区块链的人都可以看到。TRX 已从买方地址移至 MERX 资金库地址,备注字段包含发票标识符。

### 步骤 4:MERX 链上验证

MERX 监控其资金库地址的入账交易。当交易到达时,MERX:

1. 读取备注字段
2. 与未完成的发票进行匹配
3. 验证金额与发票匹配(要求精确金额)
4. 验证发票未过期
5. 验证发票未被已支付(防止重复认领)

```
验证:
  交易哈希: 7f3a2b...
  发送方: TBuyerAddress...
  接收方: TMerxTreasuryAddress...
  金额: 1,430,000 SUN (1.43 TRX) - 匹配
  备注: "merx_xpay_abc123" - 匹配发票
  发票状态: UNPAID - 正常
  发票过期时间: 2026-03-30T12:05:00Z - 未过期

  结果: 已验证
```

### 步骤 5:能量委托

付款验证后,MERX 通过其供应商网络下达能量订单:

```
订单已下达:
  能量: 65,000
  持续时间: 1 小时
  目标: TBuyerAddress...
  供应商: sohu (下单时最佳价格)

委托在 4.1 秒后确认
TBuyerAddress 可用能量: 65,000
```

买方现在有 65,000 能量委托到其地址,持续 1 小时。可以用于 USDT 转账、DEX 交换或任何其他智能合约交互。

## 完整的智能体流程

以下是 AI 智能体通过 MERX MCP 服务器执行完整 x402 流程的过程:

```
智能体: "我需要向 TRecipient 发送 100 USDT。我没有 MERX 账户。"

步骤 1: 智能体调用 create_paid_order
  -> 收到发票: 1.43 TRX,备注 "merx_xpay_abc123"

步骤 2: 智能体调用 transfer_trx
  -> 发送 1.43 TRX 到 TMerxTreasury,备注 "merx_xpay_abc123"
  -> 交易哈希: 7f3a2b...

步骤 3: 智能体等待验证(自动)
  -> MERX 检测到付款,链上验证
  -> 能量委托到智能体地址

步骤 4: 智能体调用 transfer_trc20
  -> 发送 100 USDT 到 TRecipient
  -> 使用委托能量而非燃烧 TRX
  -> 成本: 1.43 TRX 而非约 27 TRX

与 MERX 的总交互: 2 次工具调用
创建账户: 否
使用 API 密钥: 否
存入资金: 否
所需信任: 最小(付款在链上验证)
```

## 安全性:为什么备注很重要

备注字段是 x402 安全性的关键。没有它,任何发送到 MERX 资金库的 TRX 转账都可能被声称为任何发票的付款。这创造了两个攻击向量:

### 跨支付攻击

没有备注验证的话,攻击者可以:
1. 请求 65,000 能量的发票(1.43 TRX)
2. 等待其他人因不相关的原因向 MERX 资金库发送 1.43 TRX
3. 声称该不相关的付款为其发票付款
4. 获得免费能量

备注防止了这种情况。每张发票都有唯一的备注字符串。只有当备注字段包含精确字符串时,付款才会与发票匹配。缺少有效备注的随机 TRX 转账会被忽略。

### 重放攻击

没有过期和一次性使用强制的话,攻击者可以:
1. 合法支付一张发票
2. 用同一笔付款交易哈希引用第二张发票
3. 用一次付款获得两次能量

MERX 通过两种机制防止这种情况:
- 每张发票在首次成功验证后标记为 PAID。声称相同备注的第二次付款会被拒绝。
- 每笔付款交易只能匹配一张发票。交易哈希被记录且不能重复使用。

### 金额验证

付款金额必须与发票金额精确匹配。发送 1.42 TRX 而非 1.43 TRX 会导致验证失败。这防止了攻击者使用有效备注发送极小金额(例如 0.000001 TRX)来以极低价格认领发票的攻击。

## 真实主网交易

以下是来自 TRON 主网的已验证 x402 交易:

```
发票:
  订单 ID: xpay_m7k2p9
  能量: 65,000
  持续时间: 1 小时
  价格: 1.43 TRX
  备注: merx_xpay_m7k2p9

付款交易:
  哈希: 8d4f1a7b3c2e...
  区块: 58,234,891
  发送方: TWallet...
  接收方: TMerxTreasury...
  金额: 1,430,000 SUN
  备注: merx_xpay_m7k2p9
  状态: CONFIRMED

验证:
  时间戳: 2026-03-28T14:22:17Z
  匹配: 金额正确,备注正确,未过期,未被支付过
  结果: 已验证

能量委托:
  已委托: 65,000 能量
  供应商: catfee
  委托交易: 3a8b2c...
  确认时间: 2026-03-28T14:22:23Z
  到期时间: 2026-03-28T15:22:23Z
```

从发票请求到能量委托:23 秒。无账户。无 API 密钥。无先前关系。

## 何时使用 x402 vs 基于账户的访问

x402 适合:

- 自主运行且不应依赖托管账户的 **AI 智能体**
- **一次性购买**,账户创建的开销超过交易价值
- 不想创建账户或提供身份信息的**注重隐私的用户**
- 开发者想在承诺创建账户之前**试用和评估**服务
- 与许多服务交互且不应为每个服务维护单独账户的**跨平台智能体**

基于账户的访问更适合:

- **高频操作**,x402 的每笔交易开销(发票创建 + 付款广播 + 验证)很重要
- **常备订单和监控器**,需要与账户关联的持久服务端状态
- **余额计费**,预付积分比每笔交易付款更高效
- **团队操作**,多个用户共享具有基于角色访问权限的账户

许多用户从 x402 开始评估,随着使用量增长迁移到基于账户的访问。两种模式互补,而非竞争。

## x402 与智能体商务的未来

x402 协议代表了服务消费方式的更广泛转变。传统的 SaaS 计费 - 月度订阅、分级定价、使用限制 - 假设客户是一个会创建账户、评估定价层级并做出购买决策的人类。当客户是需要自主、实时、为特定任务按比例做出购买决策的 AI 智能体时,这个模型就失效了。

x402 与智能体经济契合:

- **无需注册** - 智能体没有传统意义上的身份。x402 只需要一个钱包地址。
- **按使用定价** - 智能体应该在使用时、按需使用的量付费。
- **链上验证** - 信任通过密码学证明建立,而非通过账户凭证或商业关系。
- **可组合性** - 智能体可以使用 x402 从 MERX 支付能量,然后使用该能量与任何 TRON 智能合约交互。付款和服务是解耦的。

MERX 是第一个实现 x402 的能量交易所。随着协议的成熟,我们预计其他区块链服务将采用类似的针对智能体消费优化的按使用付费模式。

## 开发者实现细节

### 构建 x402 客户端

如果你正在构建一个不使用 MCP 服务器而使用 MERX x402 的应用程序,以下是最小实现:

```javascript
const TronWeb = require('tronweb');

async function buyEnergyX402(tronWeb, energyAmount, durationHours, targetAddress) {
  // 步骤 1: 获取发票
  const invoiceRes = await fetch('https://merx.exchange/api/v1/x402/invoice', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      energy_amount: energyAmount,
      duration_hours: durationHours,
      target_address: targetAddress
    })
  });
  const invoice = await invoiceRes.json();

  // 步骤 2: 构建付款交易
  const tx = await tronWeb.transactionBuilder.sendTrx(
    invoice.pay_to,
    invoice.amount_sun,
    tronWeb.defaultAddress.base58
  );

  const txWithMemo = await tronWeb.transactionBuilder.addUpdateData(
    tx, invoice.memo, 'utf8'
  );

  // 步骤 3: 签名和广播
  const signedTx = await tronWeb.trx.sign(txWithMemo);
  const result = await tronWeb.trx.sendRawTransaction(signedTx);

  // 步骤 4: 轮询订单完成状态
  let order;
  for (let i = 0; i < 20; i++) {
    await new Promise(r => setTimeout(r, 3000));
    const orderRes = await fetch(
      `https://merx.exchange/api/v1/x402/order/${invoice.order_id}`
    );
    order = await orderRes.json();
    if (order.status === 'completed') break;
  }

  return order;
}
```

### 错误处理

常见故障模式及其解决方案:

| 错误 | 原因 | 解决方案 |
|---|---|---|
| `INVOICE_EXPIRED` | 未在 5 分钟内发送付款 | 请求新发票 |
| `AMOUNT_MISMATCH` | 付款金额与发票不同 | 发送精确金额;如果价格变化则请求新发票 |
| `MEMO_NOT_FOUND` | 付款缺少备注字段 | 使用正确备注重新发送;失败尝试的资金不会自动退款 |
| `ALREADY_PAID` | 发票已被履行 | 检查订单状态;如果你在轮询,这不是错误 |
| `PROVIDER_UNAVAILABLE` | 没有供应商能完成订单 | 几分钟后重试;发票付款将记入 MERX 余额以供手动提款 |

### 退款政策

如果 MERX 在付款验证后无法履行订单(例如所有供应商暂时不可用),付款金额将记入与付款方 TRON 地址关联的可认领余额。付款方可以通过单独的端点认领此余额而无需创建账户 - 认领通过使用发送原始付款的相同私钥签名消息来验证。

## 结论

HTTP 402 状态码在近 30 年前定义,带有原生互联网支付的愿景。那个愿景超前于时代。区块链终于提供了使其成为现实的基础设施。

MERX x402 将能量购买变成一个单一的原子流程:请求、付款、接收。无账户。无 API 密钥。除区块链提供的信任之外无其他信任假设。

对于 AI 智能体,这是自然的购买模式。对于开发者,这是最简单的集成路径。对于生态系统,这是区块链服务在智能体经济中将如何被消费的概念证明。

---

**链接:**
- MERX 平台: [https://merx.exchange](https://merx.exchange)
- MCP 服务器 (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- MCP 服务器 (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)
