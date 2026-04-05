# Создание Streamable HTTP MCP сервера для production

Model Context Protocol изначально поддерживал два транспорта: stdio (для локальных процессов) и SSE (для размещённых серверов). В 2025 году протокол получил третий транспорт — Streamable HTTP — который объединяет простоту HTTP с возможностями real-time сервер-инициированных событий и правильным управлением сессиями.

Эта статья — техническое руководство по созданию production-ready Streamable HTTP MCP сервера. Мы охватываем настройку Express.js, потоковую передачу SSE, управление сессиями с помощью заголовка Mcp-Session-Id, обнаружение OAuth через well-known endpoints, жизненный цикл соединения и развёртывание с Docker. Примеры взяты из реализации MERX MCP сервера, который служит рабочей ссылкой.

## Почему Streamable HTTP

Транспорт stdio хорошо работает для локальной разработки, но не масштабируется для размещённых развёртываний. Размещённый MCP сервер должен одновременно обслуживать нескольких клиентов, сохранять состояние между запросами и корректно обрабатывать отключения.

Исходный транспорт SSE решил проблему размещения, но имел ограничения: он использовал одно долгоживущее соединение для всех коммуникаций, что затрудняло реализацию правильных паттернов request-response. Обработка ошибок была неудобной. Состояние сессии было неявным, а не явным.

Streamable HTTP решает эти проблемы:

- **Стандартные HTTP endpoints** для операций request-response (вызовы инструментов, чтение ресурсов)
- **SSE каналы** для сервер-инициированных событий (уведомления, обновления прогресса)
- **Явное управление сессиями** через заголовок Mcp-Session-Id
- **Интеграция OAuth** для аутентификации и авторизации
- **Stateless обработка запросов** с опциональными stateful сессиями

Результат — MCP транспорт, который ведёт себя как хорошо спроектированное REST API с опциональным real-time каналом.

## Архитектура сервера

Streamable HTTP MCP сервер предоставляет три типа endpoints:

```
POST   /mcp         - Главный RPC endpoint (вызовы инструментов, чтение ресурсов)
GET    /mcp         - Поток событий SSE (сервер-клиент уведомления)
DELETE /mcp         - Завершение сессии
```

Все три используют один и тот же путь. HTTP метод определяет тип операции. Это целенаправленный выбор дизайна в спецификации MCP — он упрощает конфигурацию и маршрутизацию.

### Настройка Express.js

```typescript
import express from 'express';
import { randomUUID } from 'crypto';

const app = express();
app.use(express.json());

// Хранилище сессий
const sessions = new Map<string, SessionState>();

interface SessionState {
  id: string;
  createdAt: number;
  lastActivity: number;
  sseResponse: express.Response | null;
  tools: Map<string, ToolDefinition>;
  context: Record<string, unknown>;
}

// Главный MCP endpoint
app.post('/mcp', handleMcpPost);
app.get('/mcp', handleMcpSse);
app.delete('/mcp', handleMcpDelete);

app.listen(3100, () => {
  console.log('MCP server listening on port 3100');
});
```

## Управление сессиями

Сессии — это основной механизм управления состоянием. Каждое взаимодействие клиента связано с сессией, идентифицируемой заголовком `Mcp-Session-Id`.

### Создание сессии

Когда клиент отправляет свой первый запрос без ID сессии, сервер создаёт новую сессию:

```typescript
async function handleMcpPost(
  req: express.Request,
  res: express.Response
): Promise<void> {
  let sessionId = req.headers['mcp-session-id'] as string;
  let session: SessionState;

  if (!sessionId || !sessions.has(sessionId)) {
    // Новая сессия
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

  // Включить ID сессии в ответ
  res.setHeader('Mcp-Session-Id', sessionId);

  // Обработать MCP запрос
  const result = await processRequest(req.body, session);
  res.json(result);
}
```

### Заголовок Mcp-Session-Id

Заголовок `Mcp-Session-Id` служит нескольким целям:

1. **Идентификация клиента**: Сервер знает, какой клиент делает каждый запрос
2. **Непрерывность состояния**: Результаты инструментов, контекст и предпочтения сохраняются между запросами
3. **Связь SSE**: Сервер знает, какой SSE канал использовать для push уведомлений
4. **Безопасность**: Запросы с неверными ID сессий отклоняются

Заголовок возвращается в каждом ответе. Клиент должен включить его во все последующие запросы:

```
Запрос:
  POST /mcp HTTP/1.1
  Content-Type: application/json
  Mcp-Session-Id: a1b2c3d4-e5f6-7890-abcd-ef1234567890

Ответ:
  HTTP/1.1 200 OK
  Content-Type: application/json
  Mcp-Session-Id: a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

### Истечение сессии

Сессии должны истекать после периода неактивности. Без истечения сервер накапливает неиспользуемые сессии бесконечно:

```typescript
const SESSION_TTL_MS = 30 * 60 * 1000; // 30 минут

// Запуск каждые 5 минут
setInterval(() => {
  const now = Date.now();
  for (const [id, session] of sessions) {
    if (now - session.lastActivity > SESSION_TTL_MS) {
      // Закрыть SSE соединение, если оно открыто
      if (session.sseResponse) {
        session.sseResponse.end();
      }
      sessions.delete(id);
    }
  }
}, 5 * 60 * 1000);
```

## Поток событий SSE

GET endpoint устанавливает канал server-sent events. Клиент открывает это соединение для получения real-time уведомлений от сервера.

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

  // Установить SSE заголовки
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'Mcp-Session-Id': sessionId
  });

  // Сохранить объект ответа для отправки событий позже
  session.sseResponse = res;

  // Отправить начальный keepalive
  res.write('event: ping\ndata: {}\n\n');

  // Keepalive каждые 30 секунд
  const keepalive = setInterval(() => {
    res.write('event: ping\ndata: {}\n\n');
  }, 30000);

  // Обработать отключение клиента
  req.on('close', () => {
    clearInterval(keepalive);
    session.sseResponse = null;
  });
}
```

### Отправка уведомлений

Когда серверу нужно отправить данные клиенту — например, обновление цены или изменение статуса заказа — он пишет в сохранённый SSE ответ:

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

// Пример: уведомить клиента об изменении цены
sendNotification(session, 'price_update', {
  provider: 'feee',
  price_sun: 28,
  timestamp: Date.now()
});
```

### Переподключение SSE

Клиенты потеряют SSE соединения из-за проблем с сетью, перезагрузок сервера или таймаутов балансировщика нагрузки. API EventSource обрабатывает переподключение автоматически, но вы должны спроектировать сервер для правильного восстановления соединения:

```typescript
app.get('/mcp', async (req, res) => {
  const sessionId = req.headers['mcp-session-id'] as string;
  const session = sessions.get(sessionId);

  if (!session) {
    // Сессия истекла во время отключения — клиент должен переинициализироваться
    res.status(404).json({
      error: {
        code: 'SESSION_EXPIRED',
        message: 'Session expired. Send initialize request to create new session.'
      }
    });
    return;
  }

  // Переподключение: закрыть старое SSE, если оно ещё открыто
  if (session.sseResponse) {
    session.sseResponse.end();
  }

  // Установить новое SSE соединение для существующей сессии
  // ... (та же настройка SSE, что выше)
});
```

## Well-Known endpoints OAuth

Для production развёртываний MCP серверы должны поддерживать OAuth 2.0 для аутентификации. Спецификация MCP определяет well-known endpoints для обнаружения OAuth:

```typescript
// Метаданные сервера авторизации OAuth
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

// Метаданные MCP сервера (опционально, но рекомендуется)
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

### Аутентификация по API ключу (более простая альтернатива)

Для многих случаев production использования OAuth — это большая сложность, чем требуется. Более простой подход — аутентификация по API ключу через кастомный заголовок:

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

MERX MCP сервер поддерживает оба варианта: OAuth для автоматической регистрации агентов и аутентификацию по API ключу для прямой интеграции.

## Жизненный цикл соединения

Полный жизненный цикл Streamable HTTP MCP соединения:

```
1. Клиент отправляет POST /mcp с { method: "initialize" }
   Сервер создаёт сессию, возвращает Mcp-Session-Id
   Ответ включает возможности сервера (инструменты, подсказки, ресурсы)

2. Клиент отправляет GET /mcp с заголовком Mcp-Session-Id
   Сервер открывает SSE канал для этой сессии
   Keepalive ping отправляется каждые 30 секунд

3. Клиент отправляет POST /mcp с { method: "tools/list" }
   Сервер возвращает доступные инструменты для этой сессии

4. Клиент отправляет POST /mcp с { method: "tools/call", params: {...} }
   Сервер выполняет инструмент, возвращает результат
   Если долгоживущий, прогресс отправляется через SSE канал

5. Клиент отправляет POST /mcp с { method: "resources/read", params: {...} }
   Сервер возвращает запрошенные данные ресурса

6. (Повторить шаги 4-5 для продолжающегося взаимодействия)

7. Клиент отправляет DELETE /mcp с заголовком Mcp-Session-Id
   Сервер очищает состояние сессии и закрывает SSE
```

### Handshake инициализации

Первый запрос должен быть вызовом `initialize`. Это устанавливает совместимость версии протокола и обменивает возможности:

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

## Обработка запросов

POST endpoint обрабатывает JSON-RPC 2.0 форматированные запросы:

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

## Завершение сессии

Когда клиент отключается, очистите сессию:

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

  // Закрыть SSE, если оно открыто
  if (session.sseResponse) {
    session.sseResponse.end();
  }

  sessions.delete(sessionId);
  res.status(204).end();
}
```

## Развёртывание с Docker

Для production развёртывания упакуйте MCP сервер в контейнер:

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

### Интеграция Docker Compose

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

### Рассмотрения для балансировщика нагрузки

Если вы развёртываете за обратным прокси или балансировщиком нагрузки (nginx, Caddy, AWS ALB), настройте его для SSE:

```nginx
location /mcp {
    proxy_pass http://mcp-server:3100;
    proxy_http_version 1.1;
    proxy_set_header Connection '';
    proxy_set_header Host $host;
    proxy_buffering off;
    proxy_cache off;

    # SSE соединения могут быть долгоживущими
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
}
```

Ключевые параметры:
- `proxy_buffering off` — требуется для правильной потоковой передачи SSE
- `proxy_read_timeout` — предотвратить завершение idle SSE соединений прокси
- `Connection ''` — предотвратить добавление прокси `Connection: close`

### Affinity сессии

Если вы запускаете несколько экземпляров MCP сервера за балансировщиком нагрузки, вам нужна affinity сессии (sticky sessions). Заголовок `Mcp-Session-Id` должен маршрутизировать все запросы из одной сессии на один и тот же экземпляр сервера.

Или храните состояние сессии в Redis вместо памяти:

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
    1800, // 30 мин TTL
    JSON.stringify(session)
  );
}
```

Сессии, поддерживаемые Redis, обеспечивают горизонтальное масштабирование без sticky sessions.

## Здоровье и мониторинг

Production MCP серверы нуждаются в проверках здоровья и мониторинге:

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

Мониторьте:
- Количество активных сессий
- Задержку запроса по типу метода
- Количество SSE соединений
- Скорости успеха/неудачи вызова инструмента
- Использование памяти (следите за утечками сессии)

## Резюме

Создание production Streamable HTTP MCP сервера требует внимания к управлению сессиями, жизненному циклу SSE, аутентификации и инфраструктуре развёртывания. Ключевые компоненты:

1. **POST /mcp** для request-response вызовов инструментов
2. **GET /mcp** для потоковой передачи событий SSE
3. **DELETE /mcp** для очистки сессии
4. **Mcp-Session-Id** для непрерывности состояния
5. **Well-known endpoints OAuth** для обнаружения
6. **Развёртывание Docker** с правильной конфигурацией прокси
7. **Хранилище сессии Redis** для горизонтального масштабирования

MERX MCP сервер реализует все эти паттерны в production, обслуживая AI агентов, которым нужно взаимодействовать с блокчейном TRON. Полный исходный код доступен на [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp).

Документация: [https://merx.exchange/docs](https://merx.exchange/docs)
Платформа: [https://merx.exchange](https://merx.exchange)


## Попробуйте сейчас с AI

Добавьте MERX в Claude Desktop или любой MCP-совместимый клиент — без установки, без API ключа для read-only инструментов:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Спросите у вашего AI агента: "What is the cheapest TRON energy right now?" и получите live цены от всех подключённых провайдеров.

Полная документация MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)