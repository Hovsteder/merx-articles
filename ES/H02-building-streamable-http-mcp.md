# Construyendo un Servidor HTTP Transmisible de MCP para Producción

El Protocolo de Contexto del Modelo originalmente soportaba dos transportes: stdio (para procesos locales) y SSE (para servidores alojados). En 2025, el protocolo añadió un tercer transporte -- HTTP Transmisible -- que combina la simplicidad de HTTP con las capacidades en tiempo real de server-sent events y gestión adecuada de sesiones.

Este artículo es una guía técnica para construir un servidor MCP Transmisible HTTP listo para producción. Cubrimos configuración de Express.js, streaming SSE, gestión de sesiones con el encabezado Mcp-Session-Id, descubrimiento de OAuth mediante endpoints well-known, ciclo de vida de conexión e implementación con Docker. Los ejemplos se extraen de la implementación del servidor MERX MCP, que sirve como referencia funcional.

## Por qué HTTP Transmisible

El transporte stdio funciona bien para desarrollo local pero no escala a implementaciones alojadas. Un servidor MCP alojado necesita servir múltiples clientes simultáneamente, mantener estado entre solicitudes y manejar desconexiones con elegancia.

El transporte SSE original abordaba el alojamiento pero tenía limitaciones: utilizaba una única conexión de larga duración para toda la comunicación, lo que dificultaba la implementación de patrones adecuados de solicitud-respuesta. El manejo de errores era incómodo. El estado de la sesión era implícito en lugar de explícito.

HTTP Transmisible resuelve estos problemas:

- **Endpoints HTTP estándar** para operaciones de solicitud-respuesta (llamadas a herramientas, lecturas de recursos)
- **Canales SSE** para eventos iniciados por el servidor (notificaciones, actualizaciones de progreso)
- **Gestión de sesiones explícita** mediante el encabezado Mcp-Session-Id
- **Integración de OAuth** para autenticación y autorización
- **Manejo de solicitudes sin estado** con sesiones opcionales con estado

El resultado es un transporte MCP que se comporta como una API REST bien diseñada con un canal en tiempo real opcional.

## Arquitectura del Servidor

Un servidor MCP HTTP Transmisible expone tres tipos de endpoints:

```
POST   /mcp         - Endpoint RPC principal (llamadas a herramientas, lecturas de recursos)
GET    /mcp         - Flujo de eventos SSE (notificaciones de servidor a cliente)
DELETE /mcp         - Terminación de sesión
```

Los tres utilizan la misma ruta. El método HTTP determina el tipo de operación. Esta es una elección deliberada de diseño en la especificación de MCP -- simplifica la configuración y el enrutamiento.

### Configuración de Express.js

```typescript
import express from 'express';
import { randomUUID } from 'crypto';

const app = express();
app.use(express.json());

// Almacén de sesiones
const sessions = new Map<string, SessionState>();

interface SessionState {
  id: string;
  createdAt: number;
  lastActivity: number;
  sseResponse: express.Response | null;
  tools: Map<string, ToolDefinition>;
  context: Record<string, unknown>;
}

// Endpoint MCP principal
app.post('/mcp', handleMcpPost);
app.get('/mcp', handleMcpSse);
app.delete('/mcp', handleMcpDelete);

app.listen(3100, () => {
  console.log('MCP server listening on port 3100');
});
```

## Gestión de Sesiones

Las sesiones son el mecanismo central de gestión de estado. Cada interacción de cliente está asociada con una sesión, identificada por el encabezado `Mcp-Session-Id`.

### Creación de Sesión

Cuando un cliente envía su primera solicitud sin ID de sesión, el servidor crea una nueva sesión:

```typescript
async function handleMcpPost(
  req: express.Request,
  res: express.Response
): Promise<void> {
  let sessionId = req.headers['mcp-session-id'] as string;
  let session: SessionState;

  if (!sessionId || !sessions.has(sessionId)) {
    // Nueva sesión
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

  // Incluir ID de sesión en la respuesta
  res.setHeader('Mcp-Session-Id', sessionId);

  // Procesar la solicitud MCP
  const result = await processRequest(req.body, session);
  res.json(result);
}
```

### El Encabezado Mcp-Session-Id

El encabezado `Mcp-Session-Id` cumple múltiples propósitos:

1. **Identificación de cliente**: El servidor sabe qué cliente realiza cada solicitud
2. **Continuidad de estado**: Los resultados de herramientas, contexto y preferencias persisten entre solicitudes
3. **Asociación SSE**: El servidor sabe a qué canal SSE enviar notificaciones
4. **Seguridad**: Se rechazan las solicitudes con IDs de sesión inválidos

El encabezado se devuelve en cada respuesta. El cliente debe incluirlo en todas las solicitudes posteriores:

```
Solicitud:
  POST /mcp HTTP/1.1
  Content-Type: application/json
  Mcp-Session-Id: a1b2c3d4-e5f6-7890-abcd-ef1234567890

Respuesta:
  HTTP/1.1 200 OK
  Content-Type: application/json
  Mcp-Session-Id: a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

### Expiración de Sesión

Las sesiones deben expirar después de un período de inactividad. Sin expiración, el servidor acumula sesiones muertas indefinidamente:

```typescript
const SESSION_TTL_MS = 30 * 60 * 1000; // 30 minutos

// Ejecutar cada 5 minutos
setInterval(() => {
  const now = Date.now();
  for (const [id, session] of sessions) {
    if (now - session.lastActivity > SESSION_TTL_MS) {
      // Cerrar conexión SSE si está abierta
      if (session.sseResponse) {
        session.sseResponse.end();
      }
      sessions.delete(id);
    }
  }
}, 5 * 60 * 1000);
```

## Flujo de Eventos SSE

El endpoint GET establece un canal de server-sent events. El cliente abre esta conexión para recibir notificaciones en tiempo real del servidor.

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

  // Establecer encabezados SSE
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'Mcp-Session-Id': sessionId
  });

  // Almacenar el objeto de respuesta para enviar eventos después
  session.sseResponse = res;

  // Enviar keepalive inicial
  res.write('event: ping\ndata: {}\n\n');

  // Keepalive cada 30 segundos
  const keepalive = setInterval(() => {
    res.write('event: ping\ndata: {}\n\n');
  }, 30000);

  // Manejar desconexión de cliente
  req.on('close', () => {
    clearInterval(keepalive);
    session.sseResponse = null;
  });
}
```

### Envío de Notificaciones

Cuando el servidor necesita enviar datos al cliente -- por ejemplo, una actualización de precio o un cambio de estado de pedido -- escribe en la respuesta SSE almacenada:

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

// Ejemplo: notificar al cliente un cambio de precio
sendNotification(session, 'price_update', {
  provider: 'feee',
  price_sun: 28,
  timestamp: Date.now()
});
```

### Reconexión SSE

Los clientes perderán conexiones SSE debido a problemas de red, reinicios de servidor o timeouts del load balancer. La API EventSource maneja la reconexión automáticamente, pero debe diseñar su servidor para manejar el restablecimiento con elegancia:

```typescript
app.get('/mcp', async (req, res) => {
  const sessionId = req.headers['mcp-session-id'] as string;
  const session = sessions.get(sessionId);

  if (!session) {
    // Sesión expirada durante desconexión -- cliente debe reinicializar
    res.status(404).json({
      error: {
        code: 'SESSION_EXPIRED',
        message: 'Session expired. Send initialize request to create new session.'
      }
    });
    return;
  }

  // Reconexión: cerrar SSE anterior si aún está abierto
  if (session.sseResponse) {
    session.sseResponse.end();
  }

  // Establecer nueva conexión SSE para sesión existente
  // ... (misma configuración SSE que arriba)
});
```

## Endpoints Well-Known de OAuth

Para implementaciones en producción, los servidores MCP deben soportar OAuth 2.0 para autenticación. La especificación MCP define endpoints well-known para descubrimiento de OAuth:

```typescript
// Metadatos del servidor de autorización OAuth
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

// Metadatos del servidor MCP (opcional pero recomendado)
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

### Autenticación por Clave de API (Alternativa Más Simple)

Para muchos casos de uso en producción, OAuth es más complejidad de la necesaria. Un enfoque más simple es la autenticación por clave de API mediante un encabezado personalizado:

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

El servidor MERX MCP soporta ambos: OAuth para registro automático de agentes y autenticación por clave de API para integración directa.

## Ciclo de Vida de la Conexión

El ciclo de vida completo de una conexión MCP HTTP Transmisible:

```
1. Cliente envía POST /mcp con { method: "initialize" }
   Servidor crea sesión, devuelve Mcp-Session-Id
   La respuesta incluye capacidades del servidor (herramientas, prompts, recursos)

2. Cliente envía GET /mcp con encabezado Mcp-Session-Id
   Servidor abre canal SSE para esta sesión
   Pings de keepalive enviados cada 30 segundos

3. Cliente envía POST /mcp con { method: "tools/list" }
   Servidor devuelve herramientas disponibles para esta sesión

4. Cliente envía POST /mcp con { method: "tools/call", params: {...} }
   Servidor ejecuta herramienta, devuelve resultado
   Si es de larga duración, el progreso se envía mediante canal SSE

5. Cliente envía POST /mcp con { method: "resources/read", params: {...} }
   Servidor devuelve datos de recurso solicitados

6. (Repetir pasos 4-5 para interacción continua)

7. Cliente envía DELETE /mcp con encabezado Mcp-Session-Id
   Servidor limpia estado de sesión y cierra SSE
```

### El Apretón de Manos de Inicialización

La primera solicitud debe ser una llamada `initialize`. Esto establece compatibilidad de versión de protocolo e intercambia capacidades:

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

## Procesamiento de Solicitudes

El endpoint POST maneja solicitudes con formato JSON-RPC 2.0:

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

## Terminación de Sesión

Cuando un cliente se desconecta, limpiar la sesión:

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

  // Cerrar SSE si está abierto
  if (session.sseResponse) {
    session.sseResponse.end();
  }

  sessions.delete(sessionId);
  res.status(204).end();
}
```

## Implementación con Docker

Para implementación en producción, contenedorizar el servidor MCP:

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

### Integración con Docker Compose

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

### Consideraciones del Load Balancer

Si implementa detrás de un proxy inverso o load balancer (nginx, Caddy, AWS ALB), configúrelo para SSE:

```nginx
location /mcp {
    proxy_pass http://mcp-server:3100;
    proxy_http_version 1.1;
    proxy_set_header Connection '';
    proxy_set_header Host $host;
    proxy_buffering off;
    proxy_cache off;

    # Las conexiones SSE pueden ser de larga duración
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
}
```

Configuraciones clave:
- `proxy_buffering off` -- requerido para que SSE transmita correctamente
- `proxy_read_timeout` -- evitar que el proxy cierre conexiones SSE inactivas
- `Connection ''` -- evitar que el proxy añada `Connection: close`

### Afinidad de Sesión

Si ejecuta múltiples instancias del servidor MCP detrás de un load balancer, necesita afinidad de sesión (sticky sessions). El encabezado `Mcp-Session-Id` debe enrutar todas las solicitudes de la misma sesión a la misma instancia del servidor.

Alternativamente, almacenar estado de sesión en Redis en lugar de en memoria:

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
    1800, // TTL de 30 min
    JSON.stringify(session)
  );
}
```

Las sesiones respaldadas por Redis permiten escalado horizontal sin sticky sessions.

## Salud y Monitoreo

Los servidores MCP en producción necesitan verificaciones de salud y monitoreo:

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

Monitorear:
- Cantidad de sesiones activas
- Latencia de solicitud por tipo de método
- Cantidad de conexiones SSE
- Tasas de éxito/fallo de llamadas a herramientas
- Uso de memoria (vigilar fugas de sesiones)

## Resumen

Construir un servidor MCP HTTP Transmisible para producción requiere atención a la gestión de sesiones, ciclo de vida de SSE, autenticación e infraestructura de implementación. Los componentes clave:

1. **POST /mcp** para llamadas de herramientas de solicitud-respuesta
2. **GET /mcp** para transmisión de eventos SSE
3. **DELETE /mcp** para limpieza de sesión
4. **Mcp-Session-Id** para continuidad de estado
5. **Endpoints well-known de OAuth** para descubrimiento
6. **Implementación con Docker** con configuración adecuada del proxy
7. **Almacén de sesiones en Redis** para escalado horizontal

El servidor MERX MCP implementa todos estos patrones en producción, sirviendo a agentes de IA que necesitan interactuar con la blockchain de TRON. La fuente completa está disponible en [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp).

Documentación: [https://merx.exchange/docs](https://merx.exchange/docs)
Plataforma: [https://merx.exchange](https://merx.exchange)


## Pruébelo Ahora con IA

Añada MERX a Claude Desktop o cualquier cliente compatible con MCP -- sin instalación, sin necesidad de clave de API para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pídale a su agente de IA: "¿Cuál es la energía TRON más barata en este momento?" y obtenga precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)