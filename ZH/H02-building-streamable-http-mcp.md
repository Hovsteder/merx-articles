# 构建生产级 Streamable HTTP MCP 服务器

Model Context Protocol 最初支持两种传输方式:stdio(本地进程)和 SSE(托管服务器)。2025 年,协议添加了第三种传输 - Streamable HTTP - 结合了 HTTP 的简单性和服务器推送事件的实时能力以及适当的会话管理。

本文是构建生产级 Streamable HTTP MCP 服务器的技术指南。涵盖 Express.js 设置、SSE 流、使用 Mcp-Session-Id 头的会话管理、OAuth 发现、连接生命周期和 Docker 部署。

## 服务器架构

```
POST   /mcp         - 主 RPC 端点(工具调用、资源读取)
GET    /mcp         - SSE 事件流(服务器到客户端通知)
DELETE /mcp         - 会话终止
```

### Express.js 设置

```typescript
import express from 'express';
import { randomUUID } from 'crypto';

const app = express();
app.use(express.json());

const sessions = new Map<string, SessionState>();

app.post('/mcp', handleMcpPost);
app.get('/mcp', handleMcpSse);
app.delete('/mcp', handleMcpDelete);

app.listen(3100, () => console.log('MCP server listening on port 3100'));
```

## 会话管理

```typescript
async function handleMcpPost(req: express.Request, res: express.Response) {
  let sessionId = req.headers['mcp-session-id'] as string;
  let session: SessionState;

  if (!sessionId || !sessions.has(sessionId)) {
    sessionId = randomUUID();
    session = {
      id: sessionId, createdAt: Date.now(), lastActivity: Date.now(),
      sseResponse: null, tools: loadToolDefinitions(), context: {}
    };
    sessions.set(sessionId, session);
  } else {
    session = sessions.get(sessionId)!;
    session.lastActivity = Date.now();
  }

  res.setHeader('Mcp-Session-Id', sessionId);
  const result = await processRequest(req.body, session);
  res.json(result);
}
```

### 会话过期

```typescript
const SESSION_TTL_MS = 30 * 60 * 1000; // 30 分钟
setInterval(() => {
  const now = Date.now();
  for (const [id, session] of sessions) {
    if (now - session.lastActivity > SESSION_TTL_MS) {
      if (session.sseResponse) session.sseResponse.end();
      sessions.delete(id);
    }
  }
}, 5 * 60 * 1000);
```

## SSE 事件流

```typescript
async function handleMcpSse(req: express.Request, res: express.Response) {
  const sessionId = req.headers['mcp-session-id'] as string;
  const session = sessions.get(sessionId);
  if (!session) { res.status(404).json({error: 'Invalid session'}); return; }

  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'Mcp-Session-Id': sessionId
  });

  session.sseResponse = res;
  const keepalive = setInterval(() => res.write('event: ping\ndata: {}\n\n'), 30000);
  req.on('close', () => { clearInterval(keepalive); session.sseResponse = null; });
}
```

## 请求处理

```typescript
async function processRequest(body: JsonRpcRequest, session: SessionState) {
  const { method, params, id } = body;
  switch (method) {
    case 'initialize': return { jsonrpc: '2.0', id, result: await handleInitialize(session) };
    case 'tools/list': return { jsonrpc: '2.0', id, result: { tools: listTools(session) } };
    case 'tools/call': return { jsonrpc: '2.0', id, result: await callTool(params, session) };
    case 'resources/list': return { jsonrpc: '2.0', id, result: { resources: listResources() } };
    case 'resources/read': return { jsonrpc: '2.0', id, result: await readResource(params) };
    case 'prompts/list': return { jsonrpc: '2.0', id, result: { prompts: listPrompts() } };
    case 'prompts/get': return { jsonrpc: '2.0', id, result: await getPrompt(params) };
    default: return { jsonrpc: '2.0', id, error: { code: -32601, message: `Unknown method: ${method}` } };
  }
}
```

## Docker 部署

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY dist/ ./dist/
EXPOSE 3100
HEALTHCHECK --interval=30s --timeout=5s CMD wget -qO- http://localhost:3100/health || exit 1
USER node
CMD ["node", "dist/server.js"]
```

### 负载均衡器配置

```nginx
location /mcp {
    proxy_pass http://mcp-server:3100;
    proxy_http_version 1.1;
    proxy_set_header Connection '';
    proxy_buffering off;
    proxy_cache off;
    proxy_read_timeout 3600s;
}
```

关键设置:
- `proxy_buffering off` - SSE 流正常工作所必需
- `proxy_read_timeout` - 防止代理断开空闲 SSE 连接

### Redis 会话存储

```typescript
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL);

async function getSession(id: string) {
  const data = await redis.get(`mcp:session:${id}`);
  return data ? JSON.parse(data) : null;
}

async function saveSession(session: SessionState) {
  await redis.setex(`mcp:session:${session.id}`, 1800, JSON.stringify(session));
}
```

Redis 支持的会话使水平扩展无需粘性会话。

## 总结

构建生产级 Streamable HTTP MCP 服务器的关键组件:

1. **POST /mcp** 用于请求-响应工具调用
2. **GET /mcp** 用于 SSE 事件流
3. **DELETE /mcp** 用于会话清理
4. **Mcp-Session-Id** 用于状态连续性
5. **OAuth well-known 端点**用于发现
6. **Docker 部署**配合适当的代理配置
7. **Redis 会话存储**用于水平扩展

MERX MCP 服务器在生产中实现了所有这些模式。完整源码: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)。

文档: [https://merx.exchange/docs](https://merx.exchange/docs)
平台: [https://merx.exchange](https://merx.exchange)

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
