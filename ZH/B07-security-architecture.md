# MERX 安全架构：我们如何保护你的资金

当一个平台处理金融交易时，安全不是一个功能——它是一切的基础。单一漏洞可以永久性地摧毁用户信任。在 MERX，安全考量从第一天起就影响了每一个架构决策，而不是事后补丁式的添加。

本文详述了 MERX 的安全架构：资金如何被保护、数据完整性如何维护、系统如何防御常见攻击向量，以及使平台具有韧性的设计原则。

---

## 原则 1：不托管私钥

MERX 从不持有、存储或接触你的 TRON 私钥。这是一个基本的设计决策，消除了整个类别的攻击。

### 工作原理

当你使用 MERX 购买能量时，委托从供应商的地址发生到你的地址。MERX 协调此交易但永远不需要访问你的钱包。你的私钥留在你的设备上、硬件钱包中或你管理它的任何地方。

流程：

```
1. You tell MERX: "Delegate 65,000 energy to TMyAddress"
2. MERX tells the provider: "Delegate 65,000 energy to TMyAddress"
3. The provider delegates from TProviderAddress to TMyAddress
4. MERX verifies the delegation on-chain
5. Your private key was never involved
```

### 这为什么重要

如果 MERX 被攻破，攻击者无法窃取你的 TRX 或代币，因为 MERX 没有你的密钥。与那些要求你将代币存入平台控制地址的平台相比——那些平台持有你的密钥（或你资金的密钥），形成单点故障。

### 资金库例外

MERX 管理自己的资金库地址用于接收充值和处理提现。资金库私钥作为 Docker secret 存储，仅 `treasury-signer` 服务可以访问。它永远不会暴露给 API 服务、Web 前端或任何其他组件。下文详述此隔离机制。

---

## 原则 2：复式记账账本

MERX 上的每笔金融操作都创建配对的账本记录。这是每家银行和金融机构过去 700 年来使用的同一会计原则。它是有效的。

### 工作原理

每笔交易创建两条记录：一笔借方和一笔贷方。所有借方之和始终等于所有贷方之和。如果不等，说明出了问题，系统会立即检测到。

```sql
-- Order payment example
INSERT INTO ledger (account_id, type, amount_sun, direction)
VALUES
  ($user_id, 'ORDER_PAYMENT', 5525000, 'DEBIT'),
  ($provider_settlement, 'ORDER_PAYMENT', 5525000, 'CREDIT');
```

### 不可变性

账本记录永远不会被更新或删除。如果交易需要撤销（例如退款），会创建方向相反的新账本记录：

```sql
-- Refund: new entries, original entries remain
INSERT INTO ledger (account_id, type, amount_sun, direction, reference_id)
VALUES
  ($user_id, 'REFUND', 5525000, 'CREDIT', $original_order_id),
  ($provider_settlement, 'REFUND', 5525000, 'DEBIT', $original_order_id);
```

原始借方记录永远不会被修改。退款贷方记录明确引用原始记录，创建完整的审计轨迹。

### 这为什么重要

如果攻击者攻破应用层并试图虚增用户余额，账本记录将不平衡。定期对账检查会立即检测到：

```sql
-- Reconciliation query: should always return 0
SELECT SUM(CASE direction
  WHEN 'DEBIT' THEN amount_sun
  WHEN 'CREDIT' THEN -amount_sun
END) as imbalance
FROM ledger;
```

任何非零结果都触发即时告警和调查。

---

## 原则 3：原子余额操作

每次余额变更都使用 `SELECT FOR UPDATE` 来防止竞态条件。这不是可选的——它在数据库层面强制执行。

### 竞态条件问题

没有适当的锁定，一个余额为 10 TRX 的用户可以同时提交两笔各 8 TRX 的订单：

```
Thread 1: SELECT balance WHERE user_id = 1    -> 10 TRX
Thread 2: SELECT balance WHERE user_id = 1    -> 10 TRX
Thread 1: balance (10) >= order (8)? YES       -> proceed
Thread 2: balance (10) >= order (8)? YES       -> proceed
Thread 1: UPDATE balance = 10 - 8 = 2 TRX
Thread 2: UPDATE balance = 10 - 8 = 2 TRX

Result: User spent 16 TRX with only 10 TRX balance
```

### 解决方案

```sql
BEGIN;

-- Lock the row - second transaction waits here
SELECT balance_sun FROM accounts
WHERE user_id = $1
FOR UPDATE;

-- Check balance
-- If insufficient: ROLLBACK
-- If sufficient: proceed

UPDATE accounts
SET balance_sun = balance_sun - $order_amount
WHERE user_id = $1
  AND balance_sun >= $order_amount;  -- Double-check in UPDATE

COMMIT;
```

`FOR UPDATE` 获取行级锁。第二个事务阻塞直到第一个提交或回滚。在第一个事务提交后（余额减少到 2 TRX），第二个事务读取更新后的余额（2 TRX）并正确拒绝余额不足的订单。

---

## 原则 4：输入验证

所有输入在处理前都使用 Zod 模式验证。包括 API 请求、webhook 载荷、供应商响应和内部服务消息。

### API 输入验证

```typescript
const CreateOrderSchema = z.object({
  energy: z.number()
    .int('Energy must be an integer')
    .min(10000, 'Minimum order is 10,000 energy')
    .max(100000000, 'Maximum order is 100,000,000 energy'),

  targetAddress: z.string()
    .regex(/^T[1-9A-HJ-NP-Za-km-z]{33}$/, 'Invalid TRON address format')
    .refine(isValidTronAddress, 'Invalid TRON address checksum'),

  duration: z.enum(['1h', '1d', '3d', '7d', '14d', '30d']),

  maxPrice: z.number()
    .positive()
    .optional(),

  idempotencyKey: z.string()
    .max(255)
    .optional()
});
```

每个字段都有类型、范围限制和验证。没有原始用户输入能到达业务逻辑或数据库查询。

### SQL 注入防护

所有数据库查询使用参数化语句。永远不使用字符串拼接来构建 SQL：

```typescript
// Never this:
const query = `SELECT * FROM users WHERE id = '${userId}'`;  // SQL injection

// Always this:
const query = 'SELECT * FROM users WHERE id = $1';
const result = await db.query(query, [userId]);
```

这通过代码审查和 lint 规则强制执行。代码库中没有原始 SQL 字符串插值的机制。

### TRON 地址验证

TRON 地址在多个层面被验证：

1. **格式检查**：必须匹配 TRON 地址正则表达式（以 T 开头，34 个字符，base58）。
2. **校验和验证**：地址包含检测拼写错误的校验和。
3. **链上验证**（可选）：确认地址存在且已激活。

向无效地址发送能量会浪费资源且无法在链上撤销。严格的验证防止了这种情况。

---

## 原则 5：服务隔离

MERX 作为一组隔离的 Docker 容器运行，每个容器权限最小化且没有不必要的访问。

### 容器架构

```
Docker network:
  |
  |-- api           (port 3000, public-facing)
  |-- price-monitor (no external ports)
  |-- order-executor (no external ports)
  |-- ledger        (no external ports)
  |-- treasury-signer (no external ports, Docker secret access)
  |-- deposit-monitor (no external ports)
  |-- withdrawal-executor (no external ports)
  |
  |-- postgresql    (port 5432, internal only)
  |-- redis         (port 6379, internal only)
```

### 关键隔离特性

- **API 服务无法访问资金库私钥。** 只有 `treasury-signer` 容器可以读取包含密钥的 Docker secret。
- **价格监控无法修改余额。** 它只有对供应商 API 的读取权限和对 Redis 价格频道的写入权限。
- **订单执行器无法直接修改账本。** 它将结算事件发布到 Redis，由账本服务消费。
- **PostgreSQL 和 Redis 不对外暴露。** 它们仅从 Docker 网络内部可访问。

### 这为什么重要

如果攻击者攻破了 API 服务（最暴露的组件），他们无法：
- 访问资金库私钥（不同的容器，Docker secret）。
- 直接修改账本记录（不同的服务，对账本表没有数据库写入权限）。
- 绕过余额检查（在数据库层面通过 FOR UPDATE 强制执行）。

任何单一服务被攻破的影响范围都被设计限制。

---

## 原则 6：限频与滥用防护

### API 限频

每个 API 端点都有适合其用途的限频设置：

```
Public endpoints (prices, health):     100 requests/minute
Authenticated reads (orders, balance): 60 requests/minute
Authenticated writes (create order):   30 requests/minute
Withdrawals:                           5 requests/minute
```

限频按 API 密钥强制执行，在 Redis 中使用滑动窗口追踪。

### 提现安全保障

提现是风险最高的操作（将真实资产移出平台）。额外的安全保障包括：

- **限频**：每分钟最多 5 次提现请求。
- **金额限制**：每个账户的每日提现限额。
- **确认延迟**：大额提现触发冷却期。
- **余额验证**：`SELECT FOR UPDATE` 确保余额充足。
- **幂等性**：重复的提现请求（相同的幂等键）返回原始结果。

---

## 原则 7：Webhook 安全

MERX 发送 webhook 通知用于订单状态更新、充值和其他事件。Webhook 使用 HMAC-SHA256 签名以防止伪造。

### HMAC Webhook 工作原理

```
1. MERX computes: HMAC-SHA256(webhook_body, your_webhook_secret)
2. MERX includes the signature in the X-Merx-Signature header
3. Your server recomputes the HMAC with the same secret
4. If signatures match: genuine webhook. If not: forged, discard.
```

### 代码中的验证

```typescript
import crypto from 'crypto';

function verifyWebhook(body: string, signature: string, secret: string): boolean {
  const computed = crypto
    .createHmac('sha256', secret)
    .update(body)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(computed)
  );
}
```

注意使用了 `timingSafeEqual` 来防止时序攻击。简单的字符串比较（`===`）会通过响应时间差异泄露正确签名的信息。

---

## 原则 8：密钥管理

没有任何密钥被硬编码在源代码中。所有敏感值通过环境变量和 Docker secret 管理。

### 环境变量

```
# .env (never committed to git)
DATABASE_URL=postgresql://...
REDIS_URL=redis://...
API_JWT_SECRET=...
WEBHOOK_SIGNING_SECRET=...
TRON_API_KEY=...
```

### Docker Secret（高敏感值）

资金库私钥的敏感度过高，不适合放在环境变量中（可能被日志记录或通过进程检查泄露）。它作为 Docker secret 存储：

```yaml
# docker-compose.yml
services:
  treasury-signer:
    secrets:
      - treasury_private_key

secrets:
  treasury_private_key:
    file: /run/secrets/treasury_key
```

Docker secret 作为文件挂载在容器内部，仅服务进程可读。它们不会出现在环境变量列表、容器检查输出或日志中。

### Git 保护

`.gitignore` 文件排除所有敏感文件：

```
.env
*.key
*.pem
secrets/
```

这在第一次提交之前就设置好了，而不是之后。

---

## 监控与事件响应

### 自动告警

以下条件触发即时告警：

- 检测到账本不平衡（借贷不匹配）。
- 资金库余额低于阈值。
- 失败的认证尝试超过阈值（每个 IP 10 次/分钟）。
- 供应商 API 返回意外错误。
- 订单执行失败率超过 5%。

### 审计日志

每个安全相关操作都以结构化数据记录：

```json
{
  "event": "withdrawal_requested",
  "user_id": "usr_abc123",
  "amount_sun": 10000000,
  "destination": "TAddress...",
  "ip": "203.0.113.45",
  "timestamp": "2026-03-30T12:00:00Z"
}
```

日志保留用于取证分析和合规。它们仅追加，与应用数据分开存储。

---

## 结论

MERX 的安全不是单一功能，而是一组互锁的原则：不托管密钥、复式记账、原子余额操作、严格输入验证、服务隔离、限频、签名 webhook 和妥善的密钥管理。每个原则针对特定的威胁向量，它们共同创建了一个纵深防御架构，攻破任何单一组件都不会危及整个系统。

没有系统是无懈可击的。但通过从一开始就将安全设计到架构中——而非事后修补——MERX 最小化了攻击面，最大化了攻击者造成危害的成本。

查看开源组件：[https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)、[https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)、[https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)。

开始使用 MERX：[https://merx.exchange](https://merx.exchange)。

---

*本文是 MERX 技术系列的一部分。MERX 是首个区块链资源交易所，以安全为基础要求而非事后补救进行构建。*
