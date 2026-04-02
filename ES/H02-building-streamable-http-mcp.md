# Construyendo un servidor MCP HTTP con streaming para produccion

The Model Context Protocol originally supported two transports: stdio (for local processes) and SSE (for hosted servers). In 2025, the protocol added a third transport -- Streamable HTTP -- that combines the simplicity of HTTP with the tiempo real capabilities of server-sent events and proper session management.

This article is a technical guide to building a production-ready Streamable HTTP MCP server. We cover Express.js setup, SSE streaming, session management with the Mcp-Session-Id header, OAuth discovery via well-known endpoints, connection lifecycle, and deployment with Docker. The examples are drawn from the MERX MCP server implementation, which serves as a working reference.

## Why Streamable HTTP

The stdio transport works well for local development but does not scale to hosted deployments. A hosted MCP server needs to serve multiple clients simultaneously, maintain state across requests, and handle disconnections gracefully.

The original SSE transport addressed hosting but had limitations: it used a single long-lived connection for all communication, making it difficult to implement proper request-response patterns. Error handling was awkward. Session state was implicit rather than explicit.

Streamable HTTP solves these problems:

- **Standard HTTP endpoints** for request-response operations (tool calls, resource reads)
- **SSE channels** for server-initiated events (notifications, progress updates)
- **Explicit session management** via the Mcp-Session-Id header
- **OAuth integration** for authentication and authorization
- **Stateless request handling** with optional stateful sessions

The result is an MCP transport that behaves like a well-designed REST API with an optional tiempo real channel.

## Server Architecture

A Streamable HTTP MCP server exposes three endpoint types:

```
POST   /mcp         - Main RPC endpoint (tool calls, resource reads)
GET    /mcp         - SSE event stream (server-to-client notifications)
DELETE /mcp         - Session termination
```

All three use the same path. The HTTP method determines the operation type. This is a deliberate design choice in the MCP specification -- it simplifies configuration and routing.

### Express.js Setup

```typescript
import express from 'express';
import { randomUUID } from 'crypto';

const app = express();
app.use(express.json());

// Session store
const sessions = new Map<string, SessionState>();

interface SessionState {
  id: string;
  createdAt: number;
  lastActivity: number;
  sseResponse: express.Response | null;
  tools: Map<string, ToolDefinition>;
  context: Record<string, unknown>;
}

// Main MCP endpoint
app.post('/mcp', handleMcpPost);
app.get('/mcp', handleMcpSse);
app.delete('/mcp', handleMcpDelete);

app.listen(3100, () => {
  console.log('MCP server listening on port 3100');
});
```

## Session Management

Sessions are the core state management mechanism. Every client interaction is associated with a session, identified by the `Mcp-Session-Id` header.

### Session Creation

When a client sends its first request without a session ID, the server creates a new session:

```typescript
async function handleMcpPost(
  req: express.Request,
  res: express.Response
): Promise<void> {
  let sessionId = req.headers['mcp-session-id'] as string;
  let session: SessionState;

  if (!sessionId || !sessions.has(sessionId)) {
    // New session
    sessionId = randomUUID();
    session = {
      id: sessionId,
      createdAt: Date.now(),
      lastActivity: Date.now(),
      sseResponse: null,
      tools: loadToolDefinitions(),
      context: {}
    };
    sessions.set(sessionId, session);
  } else {
    session = sessions.get(sessionId)!;
    session.lastActivity = Date.now();
  }

  // Include session ID in response
  res.setHeader('Mcp-Session-Id', sessionId);

  // Process the MCP request
  const result = await processRequest(req.body, session);
  res.json(result);
}
```

### The Mcp-Session-Id Header

The `Mcp-Session-Id` header serves multiple purposes:

1. **Client identification**: The server knows which client is making each request
2. **State continuity**: Tool results, context, and preferences persist across requests
3. **SSE association**: The server knows which SSE channel to push notifications to
4. **Security**: Requests with invalid session IDs are rejected

The header is returned in every response. The client must include it in all subsequent requests:

```
Request:
  POST /mcp HTTP/1.1
  Content-Type: application/json
  Mcp-Session-Id: a1b2c3d4-e5f6-7890-abcd-ef1234567890

Response:
  HTTP/1.1 200 OK
  Content-Type: application/json
  Mcp-Session-Id: a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

### Session Expiration

Sessions should expire after a period of inactivity. Without expiration, the server accumulates dead sessions indefinitely:

```typescript
const SESSION_TTL_MS = 30 * 60 * 1000; // 30 minutes

// Run every 5 minutes
setInterval(() => {
  const now = Date.now();
  for (const [id, session] of sessions) {
    if (now - session.lastActivity > SESSION_TTL_MS) {
      // Close SSE connection if open
      if (session.sseResponse) {
        session.sseResponse.end();
      }
      sessions.delete(id);
    }
  }
}, 5 * 60 * 1000);
```

## SSE Event Stream

The GET endpoint establishes a server-sent events channel. The client opens this connection to receive tiempo real notifications from the server.

```typescript
async function handleMcpSse(
  req: express.Request,
  res: express.Response
): Promise<void> {
  const sessionId = req.headers['mcp-session-id'] as string;
  const session = sessions.get(sessionId);

  if (!session) {
    res.status(404).json({
      error: { code: 'SESSION_NOT_FOUND', message: 'Invalid session' }
    });
    return;
  }

  // Set SSE headers
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'Mcp-Session-Id': sessionId
  });

  // Store the response object for sending events later
  session.sseResponse = res;

  // Send initial keepalive
  res.write('event: ping\ndata: {}\n\n');

  // Keepalive every 30 seconds
  const keepalive = setInterval(() => {
    res.write('event: ping\ndata: {}\n\n');
  }, 30000);

  // Handle client disconnect
  req.on('close', () => {
    clearInterval(keepalive);
    session.sseResponse = null;
  });
}
```

### Sending Notifications

When the server needs to push data to the client -- por ejemplo, a actualizacion de precios or an estado de la orden change -- it writes to the stored SSE response:

```typescript
function sendNotification(
  session: SessionState,
  event: string,
  data: unknown
): void {
  if (session.sseResponse) {
    session.sseResponse.write(
      `event: ${event}\ndata: ${JSON.stringify(data)}\n\n`
    );
  }
}

// Example: notify client of a price change
sendNotification(session, 'price_update', {
  provider: 'feee',
  price_sun: 28,
  timestamp: Date.now()
});
```

### SSE Reconnection

Clients will lose SSE connections due to network issues, server restarts, or load balancer timeouts. The EventSource API handles reconnection automatically, but you should design your server to handle re-establishment gracefully:

```typescript
app.get('/mcp', async (req, res) => {
  const sessionId = req.headers['mcp-session-id'] as string;
  const session = sessions.get(sessionId);

  if (!session) {
    // Session expired during disconnect -- client must re-initialize
    res.status(404).json({
      error: {
        code: 'SESSION_EXPIRED',
        message: 'Session expired. Send initialize request to create new session.'
      }
    });
    return;
  }

  // Reconnection: close old SSE if still open
  if (session.sseResponse) {
    session.sseResponse.end();
  }

  // Establish new SSE connection for existing session
  // ... (same SSE setup as above)
});
```

## OAuth Well-Known Endpoints

For production deployments, MCP servers should support OAuth 2.0 for authentication. The MCP specification defines well-known endpoints for OAuth discovery:

```typescript
// OAuth authorization server metadata
app.get('/.well-known/oauth-authorization-server', (req, res) => {
  res.json({
    issuer: 'https://mcp.merx.exchange',
    authorization_endpoint: 'https://mcp.merx.exchange/oauth/authorize',
    token_endpoint: 'https://mcp.merx.exchange/oauth/token',
    registration_endpoint: 'https://mcp.merx.exchange/oauth/register',
    response_types_supported: ['code'],
    grant_types_supported: ['authorization_code', 'refresh_token'],
    code_challenge_methods_supported: ['S256'],
    token_endpoint_auth_methods_supported: ['client_secret_post']
  });
});

// MCP server metadata (optional but recommended)
app.get('/.well-known/mcp-configuration', (req, res) => {
  res.json({
    mcp_endpoint: 'https://mcp.merx.exchange/mcp',
    capabilities: {
      tools: true,
      prompts: true,
      resources: true
    },
    authentication: {
      type: 'oauth2',
      discovery_url: 'https://mcp.merx.exchange/.well-known/oauth-authorization-server'
    }
  });
});
```

### API Key Authentication (Simpler Alternative)

For many production caso de usos, OAuth is more complexity than needed. A simpler approach is clave de API authentication via a custom header:

```typescript
function authenticateRequest(
  req: express.Request,
  res: express.Response
): string | null {
  const apiKey = req.headers['x-api-key'] as string;

  if (!apiKey) {
    res.status(401).json({
      error: { code: 'UNAUTHORIZED', message: 'Missing x-api-key header' }
    });
    return null;
  }

  const userId = validateApiKey(apiKey);
  if (!userId) {
    res.status(401).json({
      error: { code: 'INVALID_KEY', message: 'Invalid API key' }
    });
    return null;
  }

  return userId;
}
```

The MERX MCP server supports both: OAuth for automated agent registration and clave de API authentication for direct integration.

## Connection Lifecycle

The full lifecycle of a Streamable HTTP MCP connection:

```
1. Client sends POST /mcp with { method: "initialize" }
   Server creates session, returns Mcp-Session-Id
   Response includes server capabilities (tools, prompts, resources)

2. Client sends GET /mcp with Mcp-Session-Id header
   Server opens SSE channel for this session
   Keepalive pings sent every 30 seconds

3. Client sends POST /mcp with { method: "tools/list" }
   Server returns available tools for this session

4. Client sends POST /mcp with { method: "tools/call", params: {...} }
   Server executes tool, returns result
   If long-running, progress sent via SSE channel

5. Client sends POST /mcp with { method: "resources/read", params: {...} }
   Server returns requested resource data

6. (Repeat steps 4-5 for ongoing interaction)

7. Client sends DELETE /mcp with Mcp-Session-Id header
   Server cleans up session state and closes SSE
```

### The Initialize Handshake

The first request must be an `initialize` call. This establishes protocol version compatibility and exchanges capabilities:

```typescript
async function handleInitialize(
  session: SessionState
): Promise<InitializeResult> {
  return {
    protocolVersion: '2025-03-26',
    capabilities: {
      tools: { listChanged: true },
      prompts: { listChanged: true },
      resources: {
        subscribe: true,
        listChanged: true
      }
    },
    serverInfo: {
      name: 'merx-mcp',
      version: '1.4.0'
    }
  };
}
```

## Request Processing

The POST endpoint handles JSON-RPC 2.0 formatted requests:

```typescript
async function processRequest(
  body: JsonRpcRequest,
  session: SessionState
): Promise<JsonRpcResponse> {
  const { method, params, id } = body;

  try {
    switch (method) {
      case 'initialize':
        return { jsonrpc: '2.0', id, result: await handleInitialize(session) };

      case 'tools/list':
        return { jsonrpc: '2.0', id, result: { tools: listTools(session) } };

      case 'tools/call':
        return { jsonrpc: '2.0', id, result: await callTool(params, session) };

      case 'resources/list':
        return { jsonrpc: '2.0', id, result: { resources: listResources() } };

      case 'resources/read':
        return { jsonrpc: '2.0', id, result: await readResource(params) };

      case 'prompts/list':
        return { jsonrpc: '2.0', id, result: { prompts: listPrompts() } };

      case 'prompts/get':
        return { jsonrpc: '2.0', id, result: await getPrompt(params) };

      default:
        return {
          jsonrpc: '2.0', id,
          error: { code: -32601, message: `Unknown method: ${method}` }
        };
    }
  } catch (err) {
    return {
      jsonrpc: '2.0', id,
      error: { code: -32000, message: (err as Error).message }
    };
  }
}
```

## Session Termination

When a client disconnects, clean up the session:

```typescript
async function handleMcpDelete(
  req: express.Request,
  res: express.Response
): Promise<void> {
  const sessionId = req.headers['mcp-session-id'] as string;
  const session = sessions.get(sessionId);

  if (!session) {
    res.status(404).json({
      error: { code: 'SESSION_NOT_FOUND', message: 'Invalid session' }
    });
    return;
  }

  // Close SSE if open
  if (session.sseResponse) {
    session.sseResponse.end();
  }

  sessions.delete(sessionId);
  res.status(204).end();
}
```

## Deployment with Docker

For production deployment, containerize the Servidor MCP:

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --production

COPY dist/ ./dist/

EXPOSE 3100

HEALTHCHECK --interval=30s --timeout=5s \
  CMD wget -qO- http://localhost:3100/health || exit 1

USER node
CMD ["node", "dist/server.js"]
```

### Docker Compose Integration

```yaml
services:
  mcp-server:
    build:
      context: ./packages/mcp-server
      dockerfile: Dockerfile
    ports:
      - "3100:3100"
    environment:
      - MERX_API_URL=http://api:3000
      - NODE_ENV=production
      - SESSION_TTL_MS=1800000
    depends_on:
      - api
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 256M
```

### Load Balancer Considerations

If you deploy behind a reverse proxy or load balancer (nginx, Caddy, AWS ALB), configure it for SSE:

```nginx
location /mcp {
    proxy_pass http://mcp-server:3100;
    proxy_http_version 1.1;
    proxy_set_header Connection '';
    proxy_set_header Host $host;
    proxy_buffering off;
    proxy_cache off;

    # SSE connections can be long-lived
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
}
```

Key settings:
- `proxy_buffering off` -- required for SSE to stream properly
- `proxy_read_timeout` -- prevent the proxy from killing idle SSE connections
- `Connection ''` -- prevent the proxy from adding `Connection: close`

### Session Affinity

If you run multiple MCP server instances behind a load balancer, you need session affinity (sticky sessions). The `Mcp-Session-Id` header should route all requests from the same session to the same server instance.

Alternatively, store session state in Redis instead of in-memory:

```typescript
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

async function getSession(id: string): Promise<SessionState | null> {
  const data = await redis.get(`mcp:session:${id}`);
  return data ? JSON.parse(data) : null;
}

async function saveSession(session: SessionState): Promise<void> {
  await redis.setex(
    `mcp:session:${session.id}`,
    1800, // 30 min TTL
    JSON.stringify(session)
  );
}
```

Redis-backed sessions enable horizontal scaling without sticky sessions.

## Health and Monitoring

Production MCP servers need verificacion de saluds and monitoring:

```typescript
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    sessions: sessions.size,
    uptime: process.uptime(),
    memory: process.memoryUsage()
  });
});
```

Monitor:
- Active session count
- Request latency per method type
- SSE connection count
- Tool call success/failure rates
- Memory usage (watch for session leaks)

## Resumen

Building a Streamable HTTP MCP server for production requires attention to session management, SSE lifecycle, authentication, and deployment infrastructure. The key components:

1. **POST /mcp** for request-response tool calls
2. **GET /mcp** for SSE event streaming
3. **DELETE /mcp** for session cleanup
4. **Mcp-Session-Id** for state continuity
5. **OAuth well-known endpoints** for discovery
6. **Docker deployment** with proper proxy configuration
7. **Redis session store** for horizontal scaling

The MERX MCP server implements all of these patterns in production, serving AI agents that need to interact with the TRON blockchain. The full source is available at [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp).

Documentacion: [https://merx.exchange/docs](https://merx.exchange/docs)
Plataforma: [https://merx.exchange](https://merx.exchange)

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
