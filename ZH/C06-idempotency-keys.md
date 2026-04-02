# 幂等性密钥: TRON能量订单的安全重试机制

网络请求会失败。连接会超时。负载均衡器会丢弃数据包。移动端客户端在请求中途可能失去信号。在任何分布式系统中,问题不在于故障是否会发生,而在于发生时如何处理。

对于大多数API调用,答案很简单: 重试请求。但当请求涉及资金时 - 创建一个会从余额中扣款的订单,或发起一笔链上资金转移的提现 - 简单的重试可能造成灾难性后果。您发送请求,服务器处理了它,但响应从未到达。您重试。服务器再次处理。您刚为同一笔订单支付了两次,更糟糕的是,触发了两笔链上提现。

这就是幂等性密钥要解决的问题。MERX在每个涉及资金的接口上实现了幂等性,使任何请求的重试都不会产生重复执行的风险。

## 幂等性在实践中意味着什么

如果执行多次与执行一次产生相同的结果,则该操作是幂等的。HTTP GET天然是幂等的 - 获取同一URL十次返回相同的数据。HTTP DELETE按照惯例也是幂等的 - 删除已经被删除的资源是空操作。

HTTP POST不是幂等的。每次POST到 `/api/v1/orders` 都会创建一个新订单。发送三次,得到三个订单,支付三次。

幂等性密钥使POST请求具有幂等行为。客户端生成一个唯一标识并随请求发送。服务器使用此密钥检测重复请求。如果相同的密钥再次到达,服务器返回第一次执行的结果,而非重新处理请求。

需要注意的是: 服务器不是简单地拒绝重复请求,而是返回原始响应。从客户端的角度来看,重试的行为与成功的首次调用完全相同。这对自动化至关重要 - 您的代码不需要区分"首次成功调用"和"成功重试"。

## 为什么这对TRON能量交易很重要

TRON能量订单涉及真实资金在真实系统中流转。当您通过MERX创建订单时,以下步骤依次发生:

1. 您的账户余额被扣除。
2. 订单被路由到最便宜的可用供应商。
3. 供应商在链上向您的目标地址委托能量。
4. 订单状态更新并触发Webhook。

如果连接在步骤1之后但在您收到确认之前断开,您无法知道订单是否已创建。没有幂等性,您的选择都很糟糕:

- **不重试。** 您可能有一个不知道的待处理订单,或者可能丢失了一个从未到达服务器的请求。您必须轮询订单列表来查明,增加了复杂性。
- **盲目重试。** 如果第一个请求确实处理了,您现在有两个订单和两笔余额扣款。对于每天处理数百个订单的自动化系统,这会快速积累。
- **带幂等性密钥重试。** 服务器识别出重复请求,返回现有订单,您的代码正常继续。没有双重扣费。无需人工对账。

相同的逻辑适用于提现。重复的提现请求可能在链上转移两次资金。使用幂等性密钥,重试将返回现有的提现记录。

## MERX如何实现幂等性

MERX在两个接口上支持 `Idempotency-Key` 头:

- `POST /api/v1/orders` - 能量和带宽订单创建
- `POST /api/v1/withdraw` - 余额提现

实现遵循简单的生命周期。

### 密钥格式和生成

幂等性密钥是客户端生成的字符串,最多64个字符。MERX不强制特定格式,但最佳实践是使用UUID(v4)或编码了上下文信息的结构化标识:

```
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

或带有业务上下文:

```
Idempotency-Key: order-user42-2026-03-30-batch7-item3
```

密钥必须对每个不同的操作是唯一的。使用相同密钥但不同请求参数(不同金额、不同目标地址)会返回错误,而不是静默忽略新参数。

### 服务端行为

当MERX收到带有 `Idempotency-Key` 头的请求时,执行以下逻辑:

1. **此密钥的首次请求。** 服务器正常处理请求 - 验证参数、扣除余额、创建订单。它存储密钥、请求哈希和响应。返回HTTP 201。

2. **相同密钥和相同参数的重复请求。** 服务器跳过所有处理,返回首次执行时存储的响应。响应完全相同,包括相同的订单ID、状态和时间戳。返回HTTP 200(非201),以便客户端在需要时可以区分。

3. **相同密钥但不同参数的重复请求。** 服务器返回HTTP 409 Conflict。这防止了密钥冲突导致返回不相关订单的微妙bug。

4. **第一个请求仍在处理时收到请求。** 服务器返回HTTP 409,并附带消息表明原始请求仍在进行中。这处理了重试在第一个请求完成之前到达的竞态条件。

### 密钥过期

幂等性密钥存储24小时。之后,相同的密钥可以用于新请求。在实践中,重试发生在几秒或几分钟内,而非几天后。24小时的窗口足以覆盖任何现实的重试场景,同时防止无限制的存储增长。

## 代码示例

### JavaScript SDK - 安全重试的订单创建

MERX JavaScript SDK在您传递 `idempotencyKey` 选项时自动处理幂等性密钥。以下是带重试逻辑的完整示例:

```javascript
import { MerxClient } from 'merx-sdk';
import { randomUUID } from 'crypto';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
  baseUrl: 'https://merx.exchange/api/v1',
});

async function createOrderSafe(params, maxRetries = 3) {
  const idempotencyKey = randomUUID();

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const order = await merx.orders.create(
        {
          energy_amount: params.energyAmount,
          target_address: params.targetAddress,
          duration_hours: params.durationHours,
        },
        {
          idempotencyKey,
        }
      );

      console.log(`Order created: ${order.id}, status: ${order.status}`);
      return order;
    } catch (err) {
      if (err.status === 409 && err.code === 'IDEMPOTENCY_CONFLICT') {
        // 相同密钥但不同参数 - 这是代码中的bug
        throw new Error('Idempotency key conflict - parameters mismatch');
      }

      if (attempt === maxRetries) {
        throw err;
      }

      const delay = Math.min(1000 * Math.pow(2, attempt - 1), 10000);
      console.log(`Attempt ${attempt} failed, retrying in ${delay}ms...`);
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }
}

// 使用
const order = await createOrderSafe({
  energyAmount: 65000,
  targetAddress: 'TTargetAddressHere',
  durationHours: 1,
});
```

此实现的关键要点:

- 幂等性密钥在重试循环之前生成一次。每次重试发送相同的密钥。
- 指数退避防止在瞬态故障期间对服务器造成压力。
- 参数不匹配导致的409冲突被视为不可重试的错误,因为它表明的是逻辑bug而非网络问题。

### Python SDK - 安全重试的提现

Python SDK遵循相同的模式。以下是提现的示例:

```python
import uuid
import time
from merx_sdk import MerxClient, MerxAPIError

client = MerxClient(
    api_key="your_api_key",
    base_url="https://merx.exchange/api/v1",
)


def withdraw_safe(address: str, amount_sun: int, max_retries: int = 3):
    idempotency_key = str(uuid.uuid4())

    for attempt in range(1, max_retries + 1):
        try:
            result = client.withdraw(
                address=address,
                amount=amount_sun,
                idempotency_key=idempotency_key,
            )
            print(f"Withdrawal initiated: {result['id']}")
            return result

        except MerxAPIError as e:
            if e.status_code == 409:
                raise ValueError(
                    "Idempotency conflict - check parameters"
                ) from e

            if attempt == max_retries:
                raise

            delay = min(2 ** (attempt - 1), 10)
            print(f"Attempt {attempt} failed: {e}. Retrying in {delay}s...")
            time.sleep(delay)

    raise RuntimeError("All retry attempts exhausted")


# 使用
result = withdraw_safe(
    address="TWithdrawalAddressHere",
    amount_sun=100_000_000,  # 100 TRX
)
```

### 原始HTTP - 使用curl直接调用API

如果您不使用SDK,`Idempotency-Key` 头是标准的HTTP头:

```bash
IDEMPOTENCY_KEY=$(uuidgen)

curl -X POST https://merx.exchange/api/v1/orders \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $IDEMPOTENCY_KEY" \
  -d '{
    "energy_amount": 65000,
    "target_address": "TTargetAddressHere",
    "duration_hours": 1
  }'
```

使用相同的 `IDEMPOTENCY_KEY` 值重试完全相同的curl命令,您将得到原始订单而非新订单。

## 生产系统的最佳实践

### 在正确的层级生成密钥

幂等性密钥应代表业务意图,而非HTTP请求。如果用户点击一次"购买能量",这是一个业务操作对应一个密钥 - 无论需要多少次HTTP重试。

不要在每次重试时生成新密钥。这完全违背了目的。

### 发送前存储密钥

在队列处理订单的系统中,在发起API调用之前将幂等性密钥写入数据库。如果进程崩溃并重启,它从数据库中获取相同的密钥并安全地重试。

```javascript
// 先将意图写入数据库
const intent = await db.orderIntents.create({
  idempotencyKey: randomUUID(),
  energyAmount: 65000,
  targetAddress: 'TTargetAddressHere',
  status: 'pending',
});

// 然后使用存储的密钥发起API调用
const order = await merx.orders.create(
  { energy_amount: intent.energyAmount, target_address: intent.targetAddress },
  { idempotencyKey: intent.idempotencyKey }
);

// 用结果更新意图
await db.orderIntents.update(intent.id, {
  status: 'completed',
  orderId: order.id,
});
```

### 处理所有响应代码

您的重试逻辑应处理以下情况:

| HTTP状态码 | 含义 | 操作 |
|------------|------|------|
| 201 | 首次成功创建 | 存储结果,继续 |
| 200 | 重复密钥,相同参数 | 视为成功(相同结果) |
| 409 | 密钥冲突或仍在处理中 | 不要重试 - 需要调查 |
| 429 | 速率限制 | 延迟后重试 |
| 500+ | 服务器错误 | 带退避重试 |

### 使用结构化密钥便于调试

虽然UUID工作得很好,但结构化密钥使调试更容易:

```
order-{userId}-{date}-{sequenceNumber}
withdraw-{userId}-{timestamp}-{nonce}
```

在调查支持案例时,结构化密钥能立即告诉您谁发起了操作以及何时发起。

### 设置合理的重试上限

三到五次带指数退避的重试覆盖了绝大多数瞬态故障。如果服务器确实宕机了,重试50次不会有帮助,只会在日志中产生噪音。

合理的上限: 最多重试3次,延迟为1秒、2秒和4秒。如果三次都失败,将错误上报给监控系统,而非无限重试。

## 常见错误

**每次重试时生成新密钥。** 这是最常见的错误。如果每次重试使用新密钥,服务器将每个都视为唯一请求。结果是重复订单。

**跨不同操作共享密钥。** 每个不同的业务操作需要自己的密钥。如果您对两个不同的订单(不同金额、不同地址)使用相同的密钥,第二个将以409失败。

**不处理200与201的区别。** 虽然两者都表示成功,但状态码告诉您这是首次执行还是重放。这对日志和指标很有用 - 了解重试命中重复的频率能反映网络可靠性。

**忽略Webhook的幂等性。** MERX在订单状态变化时发送Webhook。您的Webhook处理器也应该是幂等的 - 如果两次收到相同的 `order.filled` 事件,处理两次不应造成问题。使用订单ID作为您自己处理逻辑的天然幂等性密钥。

## 总结

幂等性密钥是API调用中的一个小增加 - 一个头、一个UUID - 但它消除了一整类资金安全bug。对于任何以编程方式创建TRON能量订单或发起提现的系统,它们不是可选的。它们是一个能够优雅处理网络故障的系统与一个每次连接断开都产生工单的系统之间的区别。

MERX在所有涉及资金的接口上支持幂等性密钥。JavaScript SDK和Python SDK都提供了内置支持。从第一天就开始使用它们 - 在已经遭遇过重复订单的生产系统中补充幂等性,要比从一开始就内建困难得多。

- MERX平台: [merx.exchange](https://merx.exchange)
- API文档: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- Python SDK: [github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)

## Try It Now with AI

Add MERX to Claude Desktop or any MCP-compatible client -- zero install, no API key needed for read-only tools:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Ask your AI agent: "What is the cheapest TRON energy right now?" and get live prices from all connected providers.

Full MCP documentation: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)
