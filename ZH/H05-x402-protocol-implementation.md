# x402 协议实现:发票、付款、验证

HTTP 状态码 402 - "Payment Required" - 在 1997 年的原始 HTTP/1.1 规范中定义,标记为"保留供将来使用"。二十九年后,未来已来。

x402 协议将该保留状态码变为真正的支付机制。MERX 为 TRON 能量购买实现了 x402。本文是完整的技术指南:发票创建、带备注的付款、TronGrid 验证(包括 hex vs base58 地址匹配问题)、x402 系统用户、余额记账和订单执行。

## x402 流程概览

```
1. INVOICE   - 买方请求报价,服务器返回付款指示
2. PAY       - 买方发送带有发票 ID 备注的 TRX
3. VERIFY    - 服务器检测链上付款并验证
4. CREDIT    - 服务器记入 x402 系统账户并创建账本分录
5. EXECUTE   - 服务器执行能量订单并委托给买方
```

## 步骤 1:发票创建

```json
{
  "invoice": {
    "order_id": "xpay_a7f3c2d1",
    "amount_trx": 1.43,
    "amount_sun": 1430000,
    "pay_to": "TMerxTreasuryAddress...",
    "memo": "merx_xpay_a7f3c2d1",
    "expires_at": "2026-03-30T12:05:00Z"
  }
}
```

**5 分钟过期。** 防止过时价格利用。
**精确金额要求。** 防止模糊匹配。
**唯一备注。** 关键安全机制。

## 步骤 2:带备注的付款

```javascript
const tx = await tronWeb.transactionBuilder.sendTrx(
  invoice.pay_to, invoice.amount_sun, buyerAddress
);
const txWithMemo = await tronWeb.transactionBuilder.addUpdateData(
  tx, invoice.memo, 'utf8'
);
const signedTx = await tronWeb.trx.sign(txWithMemo, privateKey);
const result = await tronWeb.trx.sendRawTransaction(signedTx);
```

## 步骤 3:TronGrid 验证

### hex vs base58 地址问题

TRON 地址有两种格式:
- **Base58**: `TJRabPrwbZy45sbavfcjinPJC18kjpRTv8` (以 T 开头)
- **Hex**: `415a523b449890854c8fc460ab602df9f31fe4293f` (41 前缀)

TronGrid 查询返回 hex 地址,发票存储 base58 地址。直接比较永远不会匹配。

### 修复:比较前转换

```typescript
function normalizeAddress(address: string): string {
  if (address.startsWith('41') && address.length === 42) {
    return tronWeb.address.fromHex(address);
  }
  if (address.startsWith('T') && address.length === 34) {
    return address;
  }
  throw new Error(`Invalid TRON address format: ${address}`);
}

function addressesMatch(a: string, b: string): boolean {
  return normalizeAddress(a) === normalizeAddress(b);
}
```

### 完整验证逻辑

```typescript
async function verifyX402Payment(tx) {
  const memo = extractMemo(tx);
  if (!memo || !memo.startsWith('merx_xpay_')) return;

  const invoice = await findInvoiceByMemo(memo);
  if (!invoice) return;
  if (invoice.status !== 'PENDING') return; // 防止重复认领
  if (new Date() > new Date(invoice.expires_at)) return;
  if (tx.amount !== invoice.amount_sun) return; // 精确金额
  if (!addressesMatch(tx.to_address, invoice.pay_to)) return;

  await processValidPayment(invoice, tx);
}
```

## 步骤 4:x402 系统用户

x402 付款无账户。解决方案:特殊内部账户代表所有 x402 交易。

```sql
BEGIN;
-- 记入 x402 系统账户(收到付款)
INSERT INTO ledger (...) VALUES ($x402_system_id, 'X402_PAYMENT', 1430000, 'CREDIT', $order_id);
-- 借记资金库(链上收到 TRX)
INSERT INTO ledger (...) VALUES ($treasury_id, 'X402_PAYMENT', 1430000, 'DEBIT', $order_id);
-- 借记 x402 系统账户(订单付款)
INSERT INTO ledger (...) VALUES ($x402_system_id, 'ORDER_PAYMENT', 1430000, 'DEBIT', $order_id);
-- 记入供应商结算(MERX 欠供应商)
INSERT INTO ledger (...) VALUES ($provider_settlement_id, 'ORDER_PAYMENT', 1430000, 'CREDIT', $order_id);
COMMIT;
```

x402 系统账户余额应始终为零或接近零:每笔贷记立即被借记抵消。

## 安全:备注验证防止跨支付

没有备注,两个用户同时请求相同金额的发票将产生歧义 - 无法确定哪笔付款对应哪张发票。

```
Alice 的发票: memo = "merx_xpay_alice123"
Bob 的发票:   memo = "merx_xpay_bob456"

Alice 的付款交易: 1.43 TRX, memo = "merx_xpay_alice123"
Bob 的付款交易:   1.43 TRX, memo = "merx_xpay_bob456"

验证:
  Alice 的交易备注匹配 Alice 的发票 -> 委托到 TAliceAddress
  Bob 的交易备注匹配 Bob 的发票 -> 委托到 TBobAddress
```

每笔付款与其发票的链接是明确的。没有跨认领的可能。

## 错误处理

### 付款后供应商失败

```typescript
try {
  const order = await executeOrder(invoice);
} catch (error) {
  await createRefundLedgerEntries(invoice);
  await updateInvoiceStatus(invoice.order_id, 'REFUND_REQUIRED');
  await alertOps({
    type: 'X402_REFUND_REQUIRED',
    payer_address: extractSenderFromTx(tx),
    amount_sun: invoice.amount_sun
  });
}
```

退款创建新的账本分录(永不修改现有分录)并标记发票用于手动退款处理。

## 总结

MERX 中的 x402 实现的关键设计决策:

1. **带唯一备注的发票** - 明确的付款到订单链接
2. **精确金额匹配** - 消除部分/超额付款复杂性
3. **5 分钟过期** - 防止过时价格利用
4. **hex 到 base58 标准化** - 解决 TronGrid 地址格式问题
5. **x402 系统用户** - 在无买方账户的情况下实现复式记账
6. **不可变账本分录** - 每笔 x402 交易的完整审计轨迹

该协议将 HTTP 402 从 29 年的占位符变为可工作的支付机制。对于无法创建账户或管理 API 密钥的 AI 智能体,x402 通过单笔链上交易使 TRON 能量变得可访问。

平台: [https://merx.exchange](https://merx.exchange)
文档: [https://merx.exchange/docs](https://merx.exchange/docs)
MCP 服务器: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
