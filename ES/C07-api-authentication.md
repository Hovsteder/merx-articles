# Autenticación de API de MERX: Claves, Permisos y Límites de Velocidad

Toda integración de API comienza con autenticación. Si lo haces bien, tu comercio automatizado de energy funciona sin problemas las 24 horas. Si lo haces mal, te enfrentas a credenciales filtradas, errores 403 inexplicables o prohibiciones por límite de velocidad que detienen tus sistemas de producción en el peor momento posible.

MERX proporciona dos métodos de autenticación, cada uno diseñado para un caso de uso diferente. Este artículo cubre ambos en detalle: cómo funcionan, cuándo usar cada uno, cómo gestionar permisos de manera granular y qué límites de velocidad se aplican en toda la superficie de la API.

## Dos Métodos de Autenticación

MERX admite dos formas de autenticar solicitudes de API: claves de API y tokens JWT. Sirven propósitos diferentes y no son intercambiables.

### Autenticación con Clave de API

Las claves de API son credenciales de larga duración diseñadas para comunicación servidor a servidor. Las creas a través de la API o del panel de administración, asignas permisos específicos e incluyes en cada solicitud mediante el encabezado `X-API-Key`.

```bash
curl https://merx.exchange/api/v1/prices \
  -H "X-API-Key: merx_live_k7x9m2p4..."
```

Las claves de API son la opción correcta cuando:

- Tu servicio de backend llama a MERX en nombre de tus usuarios.
- Ejecutas trabajos programados que crean órdenes o verifican saldos.
- Deseas control de permisos granular (una clave de creación de órdenes que no puede retirar fondos).
- Necesitas credenciales que funcionen sin un flujo de inicio de sesión interactivo.

Las claves de API nunca caducan por sí solas. Siguen siendo válidas hasta que las revokes explícitamente. Esto las hace convenientes para servicios de larga duración, pero requiere una gestión cuidadosa.

### Autenticación con Token JWT

Los tokens JWT son credenciales de corta duración emitidas después de un inicio de sesión. Te autenticas con tu correo electrónico y contraseña (o a través de OAuth), recibes un JWT e incluyes en solicitudes mediante el encabezado `Authorization` con prefijo `Bearer`.

```bash
curl https://merx.exchange/api/v1/balance \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

Los JWT son la opción correcta cuando:

- Un usuario humano interactúa con una interfaz que llama a la API.
- Deseas acceso con límite de tiempo que caduque automáticamente.
- Estás construyendo una aplicación web o móvil con un flujo de inicio de sesión.

Los tokens JWT emitidos por MERX caducan después de 24 horas. Después de la caducidad, el cliente debe volver a autenticarse para obtener un nuevo token. Los tokens de actualización extienden las sesiones sin requerir que el usuario inicie sesión nuevamente.

### Cuál Usar

Para integraciones programáticas (bots de pago, comercio automatizado, servicios de backend), usa claves de API. Son más simples de gestionar, admiten permisos granulares y no requieren un flujo de inicio de sesión.

Para aplicaciones orientadas al usuario donde un humano inicia sesión, usa tokens JWT. Proporcionan acceso basado en sesión con caducidad automática, reduciendo el riesgo de filtración de credenciales desde código del lado del cliente.

Puedes usar ambos en el mismo sistema. Un patrón común: autenticación JWT para tu panel de administración (operadores humanos inician sesión para gestionar configuración), autenticación con clave de API para tus servicios de backend (creación automática de órdenes, monitoreo de saldo).

## Creación y Gestión de Claves de API

### Crear una Clave

Crea claves de API a través de la API misma o a través del panel web de MERX. El endpoint de API es `POST /api/v1/keys`:

```bash
curl -X POST https://merx.exchange/api/v1/keys \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": "production-order-bot",
    "permissions": ["create_orders", "view_orders", "view_balance"]
  }'
```

La respuesta incluye la clave de API completa. Esta es la única vez que se devuelve la clave completa. Almacénala inmediatamente en tu gestor de secretos.

```json
{
  "id": "key_8f3k2m9x",
  "name": "production-order-bot",
  "key": "merx_live_k7x9m2p4q8r1s5t3u7v2w6x0y4z...",
  "permissions": ["create_orders", "view_orders", "view_balance"],
  "created_at": "2026-03-30T10:00:00Z"
}
```

Nota: el endpoint de creación de clave requiere autenticación JWT. Debes iniciar sesión primero para crear claves de API. Esta es una decisión de seguridad deliberada: las claves de API no pueden crear otras claves de API.

### Usando el SDK de JavaScript

```javascript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
  baseUrl: 'https://merx.exchange/api/v1',
});

// El SDK incluye automáticamente el encabezado X-API-Key
const prices = await merx.prices.list();
const balance = await merx.account.getBalance();
```

### Usando el SDK de Python

```python
from merx_sdk import MerxClient

client = MerxClient(
    api_key="merx_live_k7x9m2p4...",
    base_url="https://merx.exchange/api/v1",
)

prices = client.get_prices()
balance = client.get_balance()
```

### Listar y Revocar Claves

Lista todas las claves activas para tu cuenta:

```bash
curl https://merx.exchange/api/v1/keys \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

Revoca una clave inmediatamente:

```bash
curl -X DELETE https://merx.exchange/api/v1/keys/key_8f3k2m9x \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

La revocación es instantánea. Cualquier solicitud que use la clave revocada devuelve 401 inmediatamente. No hay período de gracia.

## Tipos de Permisos

Las claves de API de MERX admiten permisos granulares. Al crear una clave, especificas exactamente qué operaciones puede realizar. Una clave sin un permiso requerido recibe una respuesta 403 Forbidden.

### Permisos Disponibles

| Permiso | Descripción | Caso de Uso Típico |
|---------|-------------|-------------------|
| `view_balance` | Leer saldo de cuenta e historial de transacciones | Paneles de monitoreo, alertas |
| `view_orders` | Leer estado de órdenes e historial de órdenes | Seguimiento de órdenes, reportes |
| `create_orders` | Crear nuevas órdenes de energy y bandwidth | Bots de comercio automatizado |
| `broadcast` | Enviar transacciones firmadas para transmisión | Flujos de transacciones personalizados |

### Principios de Diseño de Permisos

**Mínimo privilegio.** Dale a cada clave solo los permisos que necesita. Un panel de monitoreo no necesita `create_orders`. Un widget de visualización de precios no necesita `view_balance`.

**Claves separadas para preocupaciones separadas.** Usa una clave para tu bot de órdenes (`create_orders`, `view_orders`, `view_balance`) y una clave diferente para tu sistema de monitoreo (`view_balance`, `view_orders`). Si la clave de monitoreo se filtra, el atacante no puede crear órdenes.

**Sin permiso de retiro en claves de API.** Los retiros requieren autenticación JWT. Esto es intencional. Una clave de API, incluso con permisos completos, no puede retirar fondos de tu cuenta. Esto añade una capa de protección para la operación de mayor riesgo.

### Ejemplo: Clave Mínima para un Widget de Precios

Un widget de precios de acceso público solo necesita obtener precios. No necesita autenticación en absoluto ya que el endpoint de precios es público, pero si deseas rastrear el uso:

```bash
curl -X POST https://merx.exchange/api/v1/keys \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "website-price-widget",
    "permissions": []
  }'
```

Una clave con un array de permisos vacío puede acceder a endpoints públicos (precios, lista de proveedores) mientras te permite rastrear volumen de solicitudes y aplicar límites de velocidad por clave.

### Ejemplo: Clave de Bot de Comercio Completo

```bash
curl -X POST https://merx.exchange/api/v1/keys \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "trading-bot-prod",
    "permissions": ["create_orders", "view_orders", "view_balance"]
  }'
```

## Límites de Velocidad

MERX aplica límites de velocidad por categoría de endpoint, no por clave. Todos los límites de velocidad se miden en solicitudes por minuto desde la misma identidad autenticada (clave de API o sesión JWT).

### Tabla de Límites de Velocidad

| Categoría de Endpoint | Límite de Velocidad | Ejemplos |
|----------------------|-------------------|---------|
| Datos de precios | 300/min | `GET /prices`, `GET /prices/history` |
| Creación de órdenes | 10/min | `POST /orders` |
| Consultas de órdenes | 60/min | `GET /orders`, `GET /orders/:id` |
| Retiros | 5/min | `POST /withdraw` |
| Datos de cuenta | 60/min | `GET /balance`, `GET /keys` |
| Gestión de claves | 10/min | `POST /keys`, `DELETE /keys/:id` |

### Encabezados de Límite de Velocidad

Cada respuesta incluye información de límite de velocidad en encabezados HTTP:

```
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 287
X-RateLimit-Reset: 1711785660
```

- `X-RateLimit-Limit` - máximo de solicitudes permitidas en la ventana actual.
- `X-RateLimit-Remaining` - solicitudes restantes antes de alcanzar el límite.
- `X-RateLimit-Reset` - marca de tiempo Unix cuando se reinicia la ventana.

### Cuando Alcanzas el Límite

Exceder el límite de velocidad devuelve HTTP 429 Too Many Requests con un encabezado `Retry-After` indicando cuántos segundos esperar:

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

### Manejo de Límites de Velocidad en Código

Los SDK manejan automáticamente los límites de velocidad con comportamiento de reintentos configurable:

```javascript
const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
  maxRetries: 3,
  retryOnRateLimit: true, // Automáticamente espera y reintenta en 429
});
```

Para clientes HTTP sin procesar, implementa backoff basado en el encabezado `Retry-After`:

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

## Mejores Prácticas de Seguridad

### Almacena Claves en Variables de Entorno

Nunca codifiques claves de API en el código fuente. Usa variables de entorno o un gestor de secretos:

```bash
# archivo .env (nunca confirmes esto)
MERX_API_KEY=merx_live_k7x9m2p4q8r1s5t3u7v2w6x0y4z...
```

```javascript
// Carga desde el entorno
const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
});
```

### Rota Claves Periódicamente

Crea una clave nueva, actualiza tus servicios para usarla, luego revoca la clave anterior. MERX admite múltiples claves activas simultáneamente, para que puedas rotar sin tiempo de inactividad:

1. Crea una clave nueva con los mismos permisos.
2. Desplega la clave nueva a tus servicios.
3. Verifica que la clave nueva funcione en producción.
4. Revoca la clave anterior.

### Monitorea el Uso de Claves

Revisa tu lista de claves de API periódicamente. Revoca las claves que ya no se usan. Cada clave tiene una marca de tiempo `last_used_at`: si una clave no se ha usado en meses, es candidata para revocación.

### Nunca Expongas Claves en Código del Lado del Cliente

Las claves de API nunca deben aparecer en JavaScript ejecutándose en un navegador, paquetes de aplicaciones móviles o cualquier código que los usuarios finales puedan inspeccionar. Si necesitas llamar a MERX desde una interfaz, envía las solicitudes a través de tu backend, que mantiene la clave de API del lado del servidor.

```
Navegador -> Tu Backend (mantiene clave de API) -> API de MERX
```

### Usa Claves Separadas para Entornos Separados

Mantén claves distintas para desarrollo, pruebas y producción. Si una clave de desarrollo se filtra, no puede afectar la producción. MERX actualmente no tiene claves con alcance de entorno, pero las convenciones de nombres ayudan:

```
dev-price-monitor
staging-order-bot
prod-order-bot
prod-balance-alerter
```

### Audita en Caso de Sospecha

Si sospechas que una clave ha sido comprometida, revócala inmediatamente y crea un reemplazo. Verifica tu historial reciente de órdenes y retiros para actividad no autorizada. MERX registra todo el uso de claves de API con direcciones IP, lo que puede ayudar a identificar la fuente del acceso no autorizado.

## Juntando Todo

Una configuración de autenticación completa para un sistema de producción típicamente se ve así:

1. Inicia sesión a través del panel web para obtener una sesión JWT.
2. Crea claves de API separadas para cada servicio: bot de órdenes, monitoreo, reportes.
3. Asigna permisos mínimos a cada clave.
4. Almacena las claves en tu gestor de secretos o variables de entorno.
5. Implementa manejo de límites de velocidad con reintentos automáticos.
6. Configura rotación de claves en un horario regular (trimestral es razonable).
7. Monitorea el uso de claves y revoca las claves no utilizadas.

La autenticación es el fundamento de cada integración de MERX. Dedicar una hora a hacerlo bien ahorra días de depuración e incidentes de seguridad más adelante.

- Plataforma y panel: [merx.exchange](https://merx.exchange)
- Referencia completa de API: [merx.exchange/docs](https://merx.exchange/docs)
- SDK de JavaScript: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- SDK de Python: [github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)


## Pruébalo Ahora con IA

Añade MERX a Claude Desktop o cualquier cliente compatible con MCP -- sin instalación, sin necesidad de clave de API para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregúntale a tu agente de IA: "¿Cuál es el energy de TRON más barato en este momento?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)