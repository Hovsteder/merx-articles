# MERX API认证: 密钥、权限和速率限制

每个API集成都从认证开始。做对了,您的自动化能量交易可以全天候平稳运行。做错了,您将面对泄露的凭证、莫名其妙的403错误,或在最糟糕的时刻让生产系统停止运行的速率限制封禁。

MERX提供两种认证方式,各自针对不同的使用场景设计。本文详细介绍这两种方式 - 工作原理、适用场景、如何精细管理权限,以及API各接口的速率限制。

## 两种认证方式

MERX支持两种认证API请求的方式: API密钥和JWT令牌。它们服务于不同目的,不可互换。

### API密钥认证

API密钥是为服务器到服务器通信设计的长期凭证。您通过API或管理面板创建它们,分配特定权限,并在每个请求中通过 `X-API-Key` 头包含它们。

```bash
curl https://merx.exchange/api/v1/prices \
  -H "X-API-Key: merx_live_k7x9m2p4..."
```

以下场景适合使用API密钥:

- 您的后端服务代表用户调用MERX。
- 您运行创建订单或检查余额的定时任务。
- 您需要精细的权限控制(一个只能创建订单而不能提现的密钥)。
- 您需要无需交互式登录流程即可工作的凭证。

API密钥不会自动过期。它们在您显式撤销之前一直有效。这使它们对长期运行的服务很方便,但需要谨慎管理。

### JWT令牌认证

JWT令牌是登录后签发的短期凭证。您使用电子邮件和密码(或通过OAuth)认证,收到JWT,并在请求中通过 `Authorization` 头以 `Bearer` 前缀包含它。

```bash
curl https://merx.exchange/api/v1/balance \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

以下场景适合使用JWT:

- 人类用户通过前端与API交互。
- 您希望访问权限在一段时间后自动过期。
- 您正在构建带有登录流程的Web或移动应用。

MERX签发的JWT令牌在24小时后过期。过期后,客户端必须重新认证以获取新令牌。刷新令牌可以延长会话,无需用户重新登录。

### 如何选择

对于编程集成 - 支付机器人、自动化交易、后端服务 - 使用API密钥。它们更容易管理,支持精细权限,且不需要登录流程。

对于人类用户登录的面向用户的应用,使用JWT令牌。它们提供带自动过期的基于会话的访问,降低了客户端代码泄露凭证的风险。

您可以在同一个系统中同时使用两者。一个常见模式: 管理面板使用JWT认证(人工操作员登录管理设置),后端服务使用API密钥认证(自动化订单创建、余额监控)。

## 创建和管理API密钥

### 创建密钥

通过API本身或MERX Web控制面板创建API密钥。API接口为 `POST /api/v1/keys`:

```bash
curl -X POST https://merx.exchange/api/v1/keys \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": "production-order-bot",
    "permissions": ["create_orders", "view_orders", "view_balance"]
  }'
```

响应包含完整的API密钥。这是唯一一次返回完整密钥的机会。请立即将其存储到您的密钥管理器中。

```json
{
  "id": "key_8f3k2m9x",
  "name": "production-order-bot",
  "key": "merx_live_k7x9m2p4q8r1s5t3u7v2w6x0y4z...",
  "permissions": ["create_orders", "view_orders", "view_balance"],
  "created_at": "2026-03-30T10:00:00Z"
}
```

注意: 密钥创建接口需要JWT认证。您必须先登录才能创建API密钥。这是一个刻意的安全决策 - API密钥不能创建其他API密钥。

### 使用JavaScript SDK

```javascript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
  baseUrl: 'https://merx.exchange/api/v1',
});

// SDK自动包含X-API-Key头
const prices = await merx.prices.list();
const balance = await merx.account.getBalance();
```

### 使用Python SDK

```python
from merx_sdk import MerxClient

client = MerxClient(
    api_key="merx_live_k7x9m2p4...",
    base_url="https://merx.exchange/api/v1",
)

prices = client.get_prices()
balance = client.get_balance()
```

### 列出和撤销密钥

列出账户下所有活跃的密钥:

```bash
curl https://merx.exchange/api/v1/keys \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

立即撤销密钥:

```bash
curl -X DELETE https://merx.exchange/api/v1/keys/key_8f3k2m9x \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

撤销即时生效。使用被撤销密钥的任何请求立即返回401。没有宽限期。

## 权限类型

MERX API密钥支持精细权限。创建密钥时,您精确指定它可以执行哪些操作。缺少所需权限的密钥会收到403 Forbidden响应。

### 可用权限

| 权限 | 描述 | 典型使用场景 |
|------|------|-------------|
| `view_balance` | 读取账户余额和交易历史 | 监控面板、告警 |
| `view_orders` | 读取订单状态和订单历史 | 订单跟踪、报表 |
| `create_orders` | 创建新的能量和带宽订单 | 自动化交易机器人 |
| `broadcast` | 提交签名交易进行广播 | 自定义交易工作流 |

### 权限设计原则

**最小权限原则。** 只给每个密钥它所需的权限。监控面板不需要 `create_orders`。价格展示组件不需要 `view_balance`。

**不同职责使用不同密钥。** 为订单机器人使用一个密钥(`create_orders`、`view_orders`、`view_balance`),为监控系统使用另一个密钥(`view_balance`、`view_orders`)。如果监控密钥泄露,攻击者无法创建订单。

**API密钥没有提现权限。** 提现需要JWT认证。这是有意为之的。即使拥有全部权限的API密钥也无法从您的账户提现。这为最高风险操作增加了一层保护。

### 示例: 价格组件的最小权限密钥

面向公众的价格组件只需要获取价格。由于价格接口是公开的,它根本不需要认证,但如果您想跟踪使用量:

```bash
curl -X POST https://merx.exchange/api/v1/keys \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "website-price-widget",
    "permissions": []
  }'
```

权限数组为空的密钥可以访问公开接口(价格、供应商列表),同时允许您跟踪请求量并按密钥应用速率限制。

### 示例: 完整交易机器人密钥

```bash
curl -X POST https://merx.exchange/api/v1/keys \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "trading-bot-prod",
    "permissions": ["create_orders", "view_orders", "view_balance"]
  }'
```

## 速率限制

MERX按接口类别应用速率限制,而非按密钥。所有速率限制以每分钟来自同一认证身份(API密钥或JWT会话)的请求数衡量。

### 速率限制表

| 接口类别 | 速率限制 | 示例 |
|----------|---------|------|
| 价格数据 | 300次/分钟 | `GET /prices`、`GET /prices/history` |
| 订单创建 | 10次/分钟 | `POST /orders` |
| 订单查询 | 60次/分钟 | `GET /orders`、`GET /orders/:id` |
| 提现 | 5次/分钟 | `POST /withdraw` |
| 账户数据 | 60次/分钟 | `GET /balance`、`GET /keys` |
| 密钥管理 | 10次/分钟 | `POST /keys`、`DELETE /keys/:id` |

### 速率限制头

每个响应都在HTTP头中包含速率限制信息:

```
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 287
X-RateLimit-Reset: 1711785660
```

- `X-RateLimit-Limit` - 当前窗口内允许的最大请求数。
- `X-RateLimit-Remaining` - 达到限制前的剩余请求数。
- `X-RateLimit-Reset` - 窗口重置的Unix时间戳。

### 触及限制时

超出速率限制返回HTTP 429 Too Many Requests,附带 `Retry-After` 头指示需要等待的秒数:

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Retry after 12 seconds.",
    "details": {
      "limit": 10,
      "window": "1m",
      "retry_after": 12
    }
  }
}
```

### 在代码中处理速率限制

SDK通过可配置的重试行为自动处理速率限制:

```javascript
const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
  maxRetries: 3,
  retryOnRateLimit: true, // 收到429时自动等待并重试
});
```

对于原始HTTP客户端,基于 `Retry-After` 头实现退避:

```python
import requests
import time

def merx_request(method, path, **kwargs):
    url = f"https://merx.exchange/api/v1{path}"
    headers = {"X-API-Key": API_KEY}

    for attempt in range(3):
        response = requests.request(method, url, headers=headers, **kwargs)

        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", 10))
            print(f"Rate limited. Waiting {retry_after}s...")
            time.sleep(retry_after)
            continue

        response.raise_for_status()
        return response.json()

    raise Exception("Rate limit retries exhausted")
```

## 安全最佳实践

### 将密钥存储在环境变量中

永远不要在源代码中硬编码API密钥。使用环境变量或密钥管理器:

```bash
# .env文件(永远不要提交)
MERX_API_KEY=merx_live_k7x9m2p4q8r1s5t3u7v2w6x0y4z...
```

```javascript
// 从环境变量加载
const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
});
```

### 定期轮换密钥

创建新密钥,更新您的服务使用新密钥,然后撤销旧密钥。MERX支持同时存在多个活跃密钥,因此您可以零停机时间轮换:

1. 创建具有相同权限的新密钥。
2. 将新密钥部署到您的服务。
3. 验证新密钥在生产环境中正常工作。
4. 撤销旧密钥。

### 监控密钥使用情况

定期检查您的API密钥列表。撤销不再使用的密钥。每个密钥都有 `last_used_at` 时间戳 - 如果一个密钥已经数月未使用,它就是撤销的候选。

### 永远不要在客户端代码中暴露密钥

API密钥永远不应出现在浏览器中运行的JavaScript、移动应用包或任何终端用户可以检查的代码中。如果您需要从前端调用MERX,请通过持有API密钥的后端代理请求。

```
浏览器 -> 您的后端(持有API密钥) -> MERX API
```

### 不同环境使用不同密钥

为开发、测试和生产环境维护不同的密钥。如果开发密钥泄露,不会影响生产环境。MERX目前没有环境范围的密钥,但命名约定有助于管理:

```
dev-price-monitor
staging-order-bot
prod-order-bot
prod-balance-alerter
```

### 疑似泄露时立即审计

如果您怀疑密钥已被泄露,立即撤销并创建替代密钥。检查您最近的订单和提现历史,查看是否有未授权的活动。MERX记录所有API密钥使用情况及IP地址,这有助于确定未授权访问的来源。

## 完整方案

生产系统的完整认证设置通常如下:

1. 通过Web控制面板登录获取JWT会话。
2. 为每个服务创建单独的API密钥: 订单机器人、监控、报表。
3. 为每个密钥分配最小权限。
4. 将密钥存储在密钥管理器或环境变量中。
5. 实现带自动重试的速率限制处理。
6. 设定定期密钥轮换计划(季度轮换是合理的)。
7. 监控密钥使用情况并撤销未使用的密钥。

认证是每个MERX集成的基础。花一个小时做好它,可以省去日后数天的调试和安全事件处理。

- 平台和控制面板: [merx.exchange](https://merx.exchange)
- 完整API参考: [merx.exchange/docs](https://merx.exchange/docs)
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
