# 区块链的复式记账:MERX 会计架构

复式记账法发明于 13 世纪的意大利。它经历了此后的每一次金融创新 - 纸币、证券交易所、中央银行、电子资金转账,现在是区块链。它能持续存在是有原因的:它有效。每笔交易产生一对平衡的分录,任何不平衡都立即发出错误信号。

当我们为 MERX 构建会计系统时 - 一个处理 TRX 存款、能量购买、供应商结算和提款的平台 - 我们毫不犹豫地选择了复式记账。替代方案,即使用运行余额更新的单式追踪,是大多数加密平台处理会计的方式。也是大多数加密平台最终出现无法解释的余额差异的原因。

## 为什么加密需要复式记账

单式系统的问题在压力下显现:

**并发修改。** 两个订单同时执行,都读取余额为 100 TRX,都扣除 5 TRX,都写入 95 TRX。用户被收了一次而非两次。

**缺失审计轨迹。** 余额是 47.3 TRX。怎么到达的?需要从单独的交易记录重建。

**对账失败。** 所有用户余额之和应等于平台的 TRX 持有量。单式系统中,如果数字不匹配,没有系统性方法找到差异。

## 账本表

```sql
CREATE TABLE ledger (
  id            BIGSERIAL PRIMARY KEY,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  account_id    UUID NOT NULL REFERENCES accounts(id),
  entry_type    TEXT NOT NULL,
  amount_sun    BIGINT NOT NULL,
  direction     TEXT NOT NULL CHECK (direction IN ('DEBIT', 'CREDIT')),
  reference_id  UUID,
  reference_type TEXT,
  description   TEXT,
  idempotency_key TEXT UNIQUE
);
```

**amount_sun**: 所有金额以 SUN 存储(1 TRX = 1,000,000 SUN)。使用最小单位完全消除浮点运算。没有小数金额,没有舍入误差。

**direction**: DEBIT 或 CREDIT。用户账户中 CREDIT 增加余额,DEBIT 减少。

**idempotency_key**: 防止重复分录。

## 不可变规则

账本记录永不更新,永不删除。在数据库层面强制:

```sql
CREATE TRIGGER no_ledger_update
  BEFORE UPDATE ON ledger
  FOR EACH ROW
  EXECUTE FUNCTION prevent_ledger_update();

CREATE TRIGGER no_ledger_delete
  BEFORE DELETE ON ledger
  FOR EACH ROW
  EXECUTE FUNCTION prevent_ledger_delete();
```

### 更正和冲销

如果需要"更正"账本分录(例如退款),不更新原始分录。创建方向相反的新分录:

```sql
-- 原始订单付款
INSERT INTO ledger (...) VALUES ($user_id, 'ORDER_PAYMENT', 1820000, 'DEBIT', $order_id);
INSERT INTO ledger (...) VALUES ($provider_settlement, 'ORDER_PAYMENT', 1820000, 'CREDIT', $order_id);

-- 订单失败,发起退款(新分录,原始保留)
INSERT INTO ledger (...) VALUES ($user_id, 'ORDER_REFUND', 1820000, 'CREDIT', $order_id);
INSERT INTO ledger (...) VALUES ($provider_settlement, 'ORDER_REFUND', 1820000, 'DEBIT', $order_id);
```

## 使用 SELECT FOR UPDATE 进行余额对账

金融系统中最危险的时刻是扣款前的余额检查。没有适当的锁定,并发请求都可以通过余额检查并都扣款,导致负余额。

```sql
BEGIN;
-- 锁定行。其他尝试用 FOR UPDATE 读取此行的交易将阻塞
SELECT balance_sun FROM account_balances WHERE account_id = $user_id FOR UPDATE;
-- 检查余额,如果不足则 ROLLBACK
INSERT INTO ledger ...;
COMMIT;
```

## 基本等式

在任何时候:

```
所有 CREDIT 之和 = 所有 DEBIT 之和
```

如果此等式不成立,系统有 bug。MERX 定期运行对账检查。

## 总结

MERX 复式账本提供:

1. **完整性** - 每笔交易是平衡的分录对
2. **不可变性** - 记录不能修改或删除
3. **并发安全** - SELECT FOR UPDATE 防止余额检查的竞态条件
4. **可审计性** - 完整的财务历史和双向引用
5. **对账** - 定期检查验证整个系统状态

这不是新事物。这是 700 年历史的会计技术应用于区块链平台。新颖的是大多数加密平台跳过了它 - 然后在资金损失、无法解释的差异和审计噩梦中付出代价。

平台: [https://merx.exchange](https://merx.exchange)
文档: [https://merx.exchange/docs](https://merx.exchange/docs)
