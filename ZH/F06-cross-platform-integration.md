# 跨平台集成:在你现有的技术栈中使用 MERX

每个技术栈都不同。你的后端可能是 Node.js、Go、Python、Ruby 或 Java。MERX 提供五种集成方法:REST API、WebSocket、Webhook、语言 SDK 和用于 AI 智能体的 MCP 服务器。

## 集成方法概览

| 方法 | 最适用于 | 方向 | 延迟 | 语言 |
|---|---|---|---|---|
| REST API | 请求-响应操作 | 客户端 -> MERX | ~200ms | 任何 |
| WebSocket | 实时价格信息 | MERX -> 客户端 | 实时 | 任何 |
| Webhook | 异步通知 | MERX -> 客户端 | 事件驱动 | 任何 |
| JS SDK | Node.js/浏览器应用 | 双向 | ~200ms | JavaScript/TypeScript |
| Python SDK | Python 后端、脚本 | 双向 | ~200ms | Python |
| MCP 服务器 | AI 智能体集成 | 双向 | ~500ms | 任何(通过 MCP 协议) |

## REST API:通用集成

任何有 HTTP 支持的语言都可以使用:

```bash
curl -X POST https://merx.exchange/api/v1/prices \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"energy_amount": 65000, "duration": "1h"}'
```

### Go 示例

```go
func getBestPrice(amount int, duration string) (*PriceResponse, error) {
    body, _ := json.Marshal(PriceRequest{EnergyAmount: amount, Duration: duration})
    req, _ := http.NewRequest("POST", "https://merx.exchange/api/v1/prices", bytes.NewBuffer(body))
    req.Header.Set("Authorization", "Bearer "+apiKey)
    req.Header.Set("Content-Type", "application/json")
    resp, err := http.DefaultClient.Do(req)
    // ...
}
```

## WebSocket:实时价格流

```typescript
const ws = new WebSocket('wss://merx.exchange/ws',
  { headers: { 'Authorization': `Bearer ${API_KEY}` } });

ws.on('message', (data) => {
  const event = JSON.parse(data.toString());
  if (event.type === 'price_update') {
    console.log(`${event.provider}: ${event.price_sun} SUN`);
  }
});
```

## Webhook:异步事件通知

```typescript
app.post('/webhooks/merx', (req, res) => {
  const event = req.body;
  switch (event.type) {
    case 'order.filled': handleOrderFilled(event.data); break;
    case 'order.failed': handleOrderFailed(event.data); break;
  }
  res.status(200).json({ received: true });
});
```

## JavaScript SDK

```typescript
import { MerxClient } from 'merx-sdk';
const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY! });
const prices = await merx.getPrices({ energy_amount: 65000, duration: '1h' });
const order = await merx.createOrder({
  energy_amount: 65000, duration: '1h', target_address: 'TAddress...'
});
```

## Python SDK

```python
from merx import MerxClient
merx = MerxClient(api_key="your-key")
prices = merx.get_prices(energy_amount=65000, duration="1h")
order = merx.create_order(energy_amount=65000, duration="1h", target_address="TAddress...")
```

## 选择指南

| 你的情况 | 推荐集成 |
|---|---|
| 快速原型,任何语言 | REST API |
| Node.js 后端 | JS SDK |
| Python 后端 | Python SDK |
| 实时价格显示 | WebSocket |
| 事件驱动架构 | Webhook |
| 无服务器(Lambda) | REST API + Webhook |
| AI 智能体构建 | MCP 服务器 |
| Go/Java/Ruby 后端 | REST API |

## 迁移路径

从单一供应商迁移到 MERX 很简单:

1. 添加 MERX SDK
2. 替换价格检查为 `merx.getPrices()`
3. 替换下单为 `merx.createOrder()`
4. 添加 webhook 用于订单通知
5. 验证后移除旧供应商代码

迁移可以增量进行。在测试期间并行运行两个系统。

API 文档: [https://merx.exchange/docs](https://merx.exchange/docs)。MCP 服务器: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)。平台: [https://merx.exchange](https://merx.exchange)。

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
