# Üretim İçin Akışlı HTTP MCP Sunucusu Oluşturma

Model Context Protocol aslen iki taşıyıcıyı destekliyordu: stdio (yerel süreçler için) ve SSE (barındırılan sunucular için). 2025'te protokol üçüncü bir taşıyıcı ekledi -- Akışlı HTTP -- ki bu HTTP'nin basitliğini sunucu tarafından gönderilen olayların ve uygun oturum yönetiminin gerçek zamanlı yetenekleriyle birleştiriyor.

Bu makale, üretim ortamına hazır bir Akışlı HTTP MCP sunucusu oluşturmak için teknik bir kılavuzdur. Express.js kurulumunu, SSE akışını, Mcp-Session-Id başlığıyla oturum yönetimini, OAuth keşfini iyi bilinen uç noktalar aracılığıyla, bağlantı yaşam döngüsünü ve Docker ile dağıtımı ele alıyoruz. Örnekler, çalışan bir referans olarak hizmet eden MERX MCP sunucusu uygulamasından alınmıştır.

## Neden Akışlı HTTP

Stdio taşıyıcısı yerel geliştirme için iyi çalışır ancak barındırılan dağıtımlara ölçeklenmez. Barındırılan bir MCP sunucusu aynı anda birden fazla istemciyi sunabilmeli, istekler arasında durumu korumalı ve bağlantı kopmasını zarafetle işlemelidir.

Orijinal SSE taşıyıcısı barındırmayı ele aldı ancak sınırlamaları vardı: tüm iletişim için tek bir uzun süreli bağlantı kullanıyordu, bu da uygun istek-yanıt desenlerini uygulamayı zorlaştırdı. Hata işleme garip idi. Oturum durumu örtülü değil açık olmalıydı.

Akışlı HTTP bu sorunları çözer:

- **Standart HTTP uç noktaları** istek-yanıt işlemleri için (araç çağrıları, kaynak okuması)
- **SSE kanalları** sunucu tarafından başlatılan olaylar için (bildirimler, ilerleme güncellemeleri)
- **Açık oturum yönetimi** Mcp-Session-Id başlığı aracılığıyla
- **OAuth entegrasyonu** kimlik doğrulama ve yetkilendirme için
- **Durumsuz istek işleme** isteğe bağlı durum sahibi oturumlarla

Sonuç, isteğe bağlı gerçek zamanlı kanalı olan iyi tasarlanmış bir REST API gibi davranan bir MCP taşıyıcısıdır.

## Sunucu Mimarisi

Akışlı HTTP MCP sunucusu üç uç nokta türünü ortaya koymaktadır:

```
POST   /mcp         - Ana RPC uç noktası (araç çağrıları, kaynak okuması)
GET    /mcp         - SSE olay akışı (sunucudan istemciye bildirimler)
DELETE /mcp         - Oturum sonlandırma
```

Üçünün tümü aynı yolu kullanır. HTTP yöntemi işlem türünü belirler. Bu MCP belirtiminde kasıtlı bir tasarım seçimidir -- yapılandırma ve yönlendirmeyi basitleştirir.

### Express.js Kurulumu

```typescript
import express from 'express';
import { randomUUID } from 'crypto';

const app = express();
app.use(express.json());

// Oturum deposu
const sessions = new Map<string, SessionState>();

interface SessionState {
  id: string;
  createdAt: number;
  lastActivity: number;
  sseResponse: express.Response | null;
  tools: Map<string, ToolDefinition>;
  context: Record<string, unknown>;
}

// Ana MCP uç noktası
app.post('/mcp', handleMcpPost);
app.get('/mcp', handleMcpSse);
app.delete('/mcp', handleMcpDelete);

app.listen(3100, () => {
  console.log('MCP server listening on port 3100');
});
```

## Oturum Yönetimi

Oturumlar temel durumu yönetme mekanizmasıdır. Her istemci etkileşimi, `Mcp-Session-Id` başlığı tarafından tanımlanan bir oturumla ilişkilidir.

### Oturum Oluşturma

İstemci oturum kimliği olmadan ilk isteğini gönderdiğinde sunucu yeni bir oturum oluşturur:

```typescript
async function handleMcpPost(
  req: express.Request,
  res: express.Response
): Promise<void> {
  let sessionId = req.headers['mcp-session-id'] as string;
  let session: SessionState;

  if (!sessionId || !sessions.has(sessionId)) {
    // Yeni oturum
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

  // Yanıta oturum kimliğini dahil et
  res.setHeader('Mcp-Session-Id', sessionId);

  // MCP isteğini işle
  const result = await processRequest(req.body, session);
  res.json(result);
}
```

### Mcp-Session-Id Başlığı

`Mcp-Session-Id` başlığı birden fazla amaca hizmet eder:

1. **İstemci kimliği**: Sunucu her isteği hangi istemcinin yaptığını bilir
2. **Durum sürekliliği**: Araç sonuçları, bağlam ve tercihler istekler arasında kalıcıdır
3. **SSE ilişkisi**: Sunucu hangi SSE kanalına bildirimler gönderecekse onu bilir
4. **Güvenlik**: Geçersiz oturum kimliğine sahip istekler reddedilir

Başlık her yanıtta döndürülür. İstemci tüm sonraki isteklere bunu dahil etmelidir:

```
İstek:
  POST /mcp HTTP/1.1
  Content-Type: application/json
  Mcp-Session-Id: a1b2c3d4-e5f6-7890-abcd-ef1234567890

Yanıt:
  HTTP/1.1 200 OK
  Content-Type: application/json
  Mcp-Session-Id: a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

### Oturum Süresi Dolma

Oturumlar bir işlem yapılmama süresinden sonra sona ermelidir. Süresi dolma olmadan sunucu ölü oturumları sonsuza kadar biriktir:

```typescript
const SESSION_TTL_MS = 30 * 60 * 1000; // 30 dakika

// Her 5 dakikada bir çalıştır
setInterval(() => {
  const now = Date.now();
  for (const [id, session] of sessions) {
    if (now - session.lastActivity > SESSION_TTL_MS) {
      // Açıksa SSE bağlantısını kapat
      if (session.sseResponse) {
        session.sseResponse.end();
      }
      sessions.delete(id);
    }
  }
}, 5 * 60 * 1000);
```

## SSE Olay Akışı

GET uç noktası sunucu tarafından gönderilen olaylar kanalı kurar. İstemci sunucudan gerçek zamanlı bildirimler almak için bu bağlantıyı açar.

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

  // SSE başlıklarını ayarla
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'Mcp-Session-Id': sessionId
  });

  // Yanıt nesnesini daha sonra olayları göndermek için sakla
  session.sseResponse = res;

  // İlk keepalive gönder
  res.write('event: ping\ndata: {}\n\n');

  // Her 30 saniyede keepalive
  const keepalive = setInterval(() => {
    res.write('event: ping\ndata: {}\n\n');
  }, 30000);

  // İstemci bağlantı kesilmesini işle
  req.on('close', () => {
    clearInterval(keepalive);
    session.sseResponse = null;
  });
}
```

### Bildirim Gönderme

Sunucunun istemciye veri göndermesi gerektiğinde -- örneğin fiyat güncellemesi veya sipariş durumu değişikliği -- saklanan SSE yanıtına yazar:

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

// Örnek: istemciyi fiyat değişikliği hakkında bilgilendirme
sendNotification(session, 'price_update', {
  provider: 'feee',
  price_sun: 28,
  timestamp: Date.now()
});
```

### SSE Yeniden Bağlantı

İstemciler ağ sorunları, sunucu yeniden başlatmaları veya yük dengeleyici zaman aşımları nedeniyle SSE bağlantılarını kaybedecektir. EventSource API'si yeniden bağlantıyı otomatik olarak işler, ancak sunucunuzu yeniden kurulumla zarafetle başa çıkmak için tasarlamalısınız:

```typescript
app.get('/mcp', async (req, res) => {
  const sessionId = req.headers['mcp-session-id'] as string;
  const session = sessions.get(sessionId);

  if (!session) {
    // Bağlantı süresi içinde oturum süresi doldu -- istemci yeniden başlatmalı
    res.status(404).json({
      error: {
        code: 'SESSION_EXPIRED',
        message: 'Session expired. Send initialize request to create new session.'
      }
    });
    return;
  }

  // Yeniden bağlantı: hala açıksa eski SSE'yi kapat
  if (session.sseResponse) {
    session.sseResponse.end();
  }

  // Mevcut oturum için yeni SSE bağlantısı kurunuz
  // ... (yukarıdakiyle aynı SSE kurulumu)
});
```

## OAuth İyi Bilinen Uç Noktaları

Üretim dağıtımları için MCP sunucuları kimlik doğrulama için OAuth 2.0'ı desteklemelidir. MCP belirtimi OAuth keşfi için iyi bilinen uç noktaları tanımlar:

```typescript
// OAuth yetkilendirme sunucusu meta verileri
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

// MCP sunucusu meta verileri (isteğe bağlı ama önerilen)
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

### API Anahtarı Kimlik Doğrulaması (Daha Basit Alternatif)

Birçok üretim kullanım durumu için OAuth gerekli olduğundan daha karmaşıktır. Daha basit bir yaklaşım, özel bir başlık aracılığıyla API anahtarı kimlik doğrulamasıdır:

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

MERX MCP sunucusu her ikisini de destekler: otomatik ajan kaydı için OAuth ve doğrudan entegrasyon için API anahtarı kimlik doğrulaması.

## Bağlantı Yaşam Döngüsü

Akışlı HTTP MCP bağlantısının tam yaşam döngüsü:

```
1. İstemci POST /mcp gönderir { method: "initialize" } ile
   Sunucu oturum oluşturur, Mcp-Session-Id döndürür
   Yanıt sunucu yeteneklerini içerir (araçlar, istemler, kaynaklar)

2. İstemci GET /mcp gönderir Mcp-Session-Id başlığı ile
   Sunucu bu oturum için SSE kanalını açar
   Keepalive pingsleri her 30 saniyede gönderilir

3. İstemci POST /mcp gönderir { method: "tools/list" } ile
   Sunucu bu oturum için kullanılabilir araçları döndürür

4. İstemci POST /mcp gönderir { method: "tools/call", params: {...} } ile
   Sunucu aracı yürütür, sonucu döndürür
   Uzun süreli işlemlerse ilerleme SSE kanalı aracılığıyla gönderilir

5. İstemci POST /mcp gönderir { method: "resources/read", params: {...} } ile
   Sunucu istenen kaynak verilerini döndürür

6. (Devam eden etkileşim için 4-5. adımları tekrarla)

7. İstemci DELETE /mcp gönderir Mcp-Session-Id başlığı ile
   Sunucu oturum durumunu temizler ve SSE'yi kapatır
```

### Initialize El Sıkışması

İlk istek `initialize` çağrısı olmalıdır. Bu protokol sürümü uyumluluğunu kurar ve yetenekleri değiş tokuş eder:

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

## İstek İşleme

POST uç noktası JSON-RPC 2.0 biçimlendirilmiş istekleri işler:

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

## Oturum Sonlandırma

İstemci bağlantısını kestiğinde oturumu temizle:

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

  // Açıksa SSE'yi kapat
  if (session.sseResponse) {
    session.sseResponse.end();
  }

  sessions.delete(sessionId);
  res.status(204).end();
}
```

## Docker ile Dağıtım

Üretim dağıtımı için MCP sunucusunu containerleştir:

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

### Docker Compose Entegrasyonu

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

### Yük Dengeleyici Hususları

Tersine proxy veya yük dengeleyici arkasında dağıtırsanız (nginx, Caddy, AWS ALB), SSE için yapılandırın:

```nginx
location /mcp {
    proxy_pass http://mcp-server:3100;
    proxy_http_version 1.1;
    proxy_set_header Connection '';
    proxy_set_header Host $host;
    proxy_buffering off;
    proxy_cache off;

    # SSE bağlantıları uzun süreli olabilir
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
}
```

Anahtar ayarlar:
- `proxy_buffering off` -- SSE'nin düzgün akış yapması için gerekli
- `proxy_read_timeout` -- proxy'nin boşta kalan SSE bağlantılarını öldürmesini önle
- `Connection ''` -- proxy'nin `Connection: close` eklemesini önle

### Oturum Afinitesi

Yük dengeleyicinin arkasında birden fazla MCP sunucusu örneğini çalıştırırsanız, oturum afinitesine (yapışkan oturumlar) ihtiyacınız vardır. `Mcp-Session-Id` başlığı, aynı oturumdaki tüm istekleri aynı sunucu örneğine yönlendirmelidir.

Alternatif olarak, oturum durumunu bellek içi yerine Redis'te sakla:

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
    1800, // 30 dk TTL
    JSON.stringify(session)
  );
}
```

Redis tarafından desteklenen oturumlar, yapışkan oturumlar olmadan yatay ölçeklendirmeyi etkinleştirir.

## Sağlık ve İzleme

Üretim MCP sunucuları sağlık kontrolleri ve izleme gerektirir:

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

Izle:
- Etkin oturum sayısı
- Yöntem türü başına istek latansı
- SSE bağlantı sayısı
- Araç çağrısı başarı/başarısızlık oranları
- Bellek kullanımı (oturum sızıntılarını izle)

## Özet

Üretim için Akışlı HTTP MCP sunucusu oluşturmak, oturum yönetimi, SSE yaşam döngüsü, kimlik doğrulama ve dağıtım altyapısına dikkat gerektirir. Anahtar bileşenler:

1. **POST /mcp** istek-yanıt araç çağrıları için
2. **GET /mcp** SSE olay akışı için
3. **DELETE /mcp** oturum temizliği için
4. **Mcp-Session-Id** durum sürekliliği için
5. **OAuth iyi bilinen uç noktaları** keşif için
6. **Docker dağıtımı** uygun proxy yapılandırmasıyla
7. **Redis oturum deposu** yatay ölçeklendirme için

MERX MCP sunucusu, TRON blokzinciriyle etkileşime ihtiyaç duyan yapay zeka ajanlarına hizmet eden üretim ortamında tüm bu desenleri uygular. Tam kaynak [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp) adresinde mevcuttur.

Dokümantasyon: [https://merx.exchange/docs](https://merx.exchange/docs)
Platform: [https://merx.exchange](https://merx.exchange)


## Şimdi Yapay Zeka ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin -- sıfır kurulum, salt okunur araçlar için API anahtarı gerekmez:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Yapay zeka ajanınıza şunu sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatlar alın.

Tam MCP dokümantasyonu: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)