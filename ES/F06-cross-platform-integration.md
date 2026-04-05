# Integración Multiplataforma: MERX en Tu Stack Existente

Cada stack de tecnología es diferente. Tu backend podría ser Node.js, Go, Python, Ruby o Java. Tu arquitectura podría ser monolítica o microservicios. Tus patrones de comunicación podrían ser síncronos, dirigidos por eventos, o una combinación. La pregunta no es si MERX puede encajar en tu stack, sino qué punto de integración te proporciona el máximo valor con la menor fricción.

MERX ofrece cinco métodos de integración: REST API, WebSocket, webhooks, SDKs de lenguaje, y un servidor MCP para agentes de IA. Este artículo explica cuándo usar cada uno, cómo interactúan, y cómo elegir el enfoque correcto para tu arquitectura específica.

## Descripción General de Métodos de Integración

| Método | Mejor Para | Dirección | Latencia | Lenguaje |
|---|---|---|---|---|
| REST API | Operaciones solicitud-respuesta | Cliente -> MERX | ~200ms | Cualquiera |
| WebSocket | Feeds de precios en tiempo real | MERX -> Cliente | Tiempo real | Cualquiera |
| Webhooks | Notificaciones asincrónicas | MERX -> Cliente | Dirigido por eventos | Cualquiera |
| JS SDK | Aplicaciones Node.js / navegador | Bidireccional | ~200ms | JavaScript/TypeScript |
| Python SDK | Backends Python, scripts | Bidireccional | ~200ms | Python |
| Servidor MCP | Integración de agentes de IA | Bidireccional | ~500ms | Cualquiera (vía protocolo MCP) |

## REST API: La Integración Universal

La REST API funciona desde cualquier lenguaje de programación con soporte HTTP. Es la integración del mínimo común denominador: si tu lenguaje puede hacer solicitudes HTTP, puede comunicarse con MERX.

### Cuándo Usar REST

- Tu backend está en un lenguaje sin SDK (Go, Java, Ruby, Rust, PHP)
- Necesitas compras ocasionales de energy, no interacción continua
- Tu arquitectura prefiere llamadas HTTP explícitas sobre conexiones persistentes
- Estás prototipando y quieres la integración más simple posible

### Implementación

La API sigue convenciones REST estándar con payloads JSON:

```bash
# Verificar precios
curl -X POST https://merx.exchange/api/v1/prices \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"energy_amount": 65000, "duration": "1h"}'

# Colocar una orden
curl -X POST https://merx.exchange/api/v1/orders \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: unique-request-id-123" \
  -d '{
    "energy_amount": 65000,
    "duration": "1h",
    "target_address": "TYourAddress..."
  }'

# Verificar el estado de la orden
curl https://merx.exchange/api/v1/orders/ORDER_ID \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Ejemplo en Go

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
)

type PriceRequest struct {
    EnergyAmount int    `json:"energy_amount"`
    Duration     string `json:"duration"`
}

type PriceResponse struct {
    Best struct {
        Provider string `json:"provider"`
        PriceSun int    `json:"price_sun"`
    } `json:"best"`
    Providers []struct {
        Provider string `json:"provider"`
        PriceSun int    `json:"price_sun"`
    } `json:"providers"`
}

func getBestPrice(amount int, duration string) (*PriceResponse, error) {
    body, _ := json.Marshal(PriceRequest{
        EnergyAmount: amount,
        Duration:     duration,
    })

    req, _ := http.NewRequest(
        "POST",
        "https://merx.exchange/api/v1/prices",
        bytes.NewBuffer(body),
    )
    req.Header.Set("Authorization", "Bearer "+apiKey)
    req.Header.Set("Content-Type", "application/json")

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var result PriceResponse
    json.NewDecoder(resp.Body).Decode(&result)
    return &result, nil
}
```

### Manejo de Errores

Todos los errores de API siguen un formato consistente:

```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Account balance is insufficient for this order",
    "details": {
      "required": 1820000,
      "available": 500000
    }
  }
}
```

Esta consistencia significa que tu lógica de manejo de errores funciona igual sin importar qué endpoint llames o qué proveedor esté involucrado detrás de escenas.

## WebSocket: Feeds de Precios en Tiempo Real

Las conexiones WebSocket proporcionan actualizaciones de precios en tiempo real sin polling. Los precios se transmiten a tu aplicación a medida que cambian en los siete proveedores.

### Cuándo Usar WebSocket

- Necesitas pantallas de precios en vivo (dashboards, interfaces de trading)
- Tu aplicación toma decisiones de compra basadas en movimientos de precios
- Quieres desencadenar acciones cuando los precios cruzan umbrales
- Estás construyendo herramientas de monitoreo en tiempo real

### Implementación

```typescript
import WebSocket from 'ws';

const ws = new WebSocket(
  'wss://merx.exchange/ws',
  { headers: { 'Authorization': `Bearer ${API_KEY}` } }
);

ws.on('open', () => {
  // Suscribirse a actualizaciones de precios para parámetros específicos
  ws.send(JSON.stringify({
    type: 'subscribe',
    channel: 'prices',
    params: {
      energy_amount: 65000,
      duration: '1h'
    }
  }));
});

ws.on('message', (data) => {
  const event = JSON.parse(data.toString());

  switch (event.type) {
    case 'price_update':
      console.log(
        `${event.provider}: ${event.price_sun} SUN`
      );
      break;

    case 'order_update':
      console.log(
        `Order ${event.order_id}: ${event.status}`
      );
      break;
  }
});
```

### Combinando WebSocket con REST

Un patrón común: usar WebSocket para monitoreo de precios y REST para colocación de órdenes:

```typescript
// WebSocket monitorea precios
ws.on('message', async (data) => {
  const event = JSON.parse(data.toString());

  if (event.type === 'price_update' &&
      event.price_sun <= targetPrice) {
    // REST API coloca la orden
    const response = await fetch(
      'https://merx.exchange/api/v1/orders',
      {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${API_KEY}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          energy_amount: 65000,
          duration: '1h',
          target_address: walletAddress
        })
      }
    );
  }
});
```

## Webhooks: Notificaciones de Eventos Asincrónicas

Los webhooks envían notificaciones a tu servidor cuando ocurren eventos. A diferencia de WebSocket (que requiere una conexión persistente), los webhooks funcionan con cualquier servidor que pueda recibir solicitudes HTTP POST.

### Cuándo Usar Webhooks

- Tu arquitectura es dirigida por eventos (colas de mensajes, funciones serverless)
- No puedes mantener conexiones WebSocket persistentes
- Necesitas entrega confiable con reintentos
- Quieres desacoplar la adquisición de energy de la ejecución de transacciones

### Implementación

```typescript
import express from 'express';

const app = express();
app.use(express.json());

app.post('/webhooks/merx', (req, res) => {
  const event = req.body;

  switch (event.type) {
    case 'order.filled':
      handleOrderFilled(event.data);
      break;

    case 'order.failed':
      handleOrderFailed(event.data);
      break;

    case 'standing_order.triggered':
      handleStandingOrderTriggered(event.data);
      break;

    case 'auto_energy.purchased':
      handleAutoEnergyPurchased(event.data);
      break;
  }

  // Siempre responder 200 para reconocer la recepción
  res.status(200).json({ received: true });
});

async function handleOrderFilled(
  data: OrderFilledEvent
): Promise<void> {
  // La energy está disponible -- proceder con la transacción
  const pendingTx = await db.getPendingTransaction(
    data.order_id
  );
  if (pendingTx) {
    await executeTransaction(pendingTx);
  }
}
```

### Confiabilidad de Webhooks

MERX reintenta las entregas de webhooks fallidas con backoff exponencial. Tu endpoint debe:

1. Responder con estado 200 rápidamente (dentro de 5 segundos)
2. Procesar el evento de forma asincrónica si el procesamiento toma tiempo
3. Manejar entregas duplicadas de forma idempotente (usa el ID del evento para deduplicación)

```typescript
const processedEvents = new Set<string>();

app.post('/webhooks/merx', async (req, res) => {
  const event = req.body;

  // Verificación de idempotencia
  if (processedEvents.has(event.id)) {
    res.status(200).json({ received: true });
    return;
  }

  processedEvents.add(event.id);

  // Reconocer inmediatamente
  res.status(200).json({ received: true });

  // Procesar de forma asincrónica
  processEvent(event).catch(console.error);
});
```

## SDK de JavaScript

El SDK de JavaScript/TypeScript envuelve la REST API y las conexiones WebSocket con interfaces tipadas y métodos de conveniencia.

### Cuándo Usar el SDK de JS

- Tu backend es Node.js o tu frontend es basado en navegador
- Quieres tipos de TypeScript y autocompletado en el IDE
- Prefieres llamadas de método sobre solicitudes HTTP sin procesar
- Necesitas tanto REST como WebSocket en un solo paquete

### Implementación

```typescript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY!
});

// Precios
const prices = await merx.getPrices({
  energy_amount: 65000,
  duration: '1h'
});

// Órdenes
const order = await merx.createOrder({
  energy_amount: 65000,
  duration: '1h',
  target_address: 'TAddress...'
});

// Órdenes permanentes
const standing = await merx.createStandingOrder({
  energy_amount: 65000,
  max_price_sun: 25,
  duration: '1h',
  repeat: true
});

// Estimación de energy
const estimate = await merx.estimateEnergy({
  contract_address: 'TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t',
  function_selector: 'transfer(address,uint256)',
  parameter: [recipient, amount],
  owner_address: sender
});

// WebSocket (incluido en el SDK)
const ws = merx.connectWebSocket();
ws.on('price_update', (data) => {
  console.log(`${data.provider}: ${data.price_sun} SUN`);
});
```

El SDK maneja autenticación, serialización de solicitudes, análisis de errores y validación de tipos automáticamente.

## SDK de Python

El SDK de Python proporciona la misma funcionalidad para backends Python, pipelines de análisis de datos y scripts de automatización.

### Cuándo Usar el SDK de Python

- Tu backend es Python (Django, Flask, FastAPI)
- Estás construyendo herramientas de análisis o reporte de datos
- Tus scripts de DevOps están en Python
- Estás integrando con bots de trading basados en Python

### Implementación

```python
from merx import MerxClient

merx = MerxClient(api_key="your-api-key")

# Obtener precios
prices = merx.get_prices(
    energy_amount=65000,
    duration="1h"
)
print(f"Mejor: {prices.best.price_sun} SUN "
      f"vía {prices.best.provider}")

# Colocar orden
order = merx.create_order(
    energy_amount=65000,
    duration="1h",
    target_address="TAddress..."
)

# Estimar energy para una llamada de contrato
estimate = merx.estimate_energy(
    contract_address="TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    function_selector="transfer(address,uint256)",
    parameter=[recipient, amount],
    owner_address=sender
)
print(f"Energy necesaria: {estimate.energy_required}")
```

### Ejemplo de Análisis de Datos

```python
import pandas as pd
from merx import MerxClient

merx = MerxClient(api_key="your-api-key")

# Extraer historial de precios para análisis
history = merx.get_price_history(
    energy_amount=65000,
    duration="1h",
    period="30d"
)

df = pd.DataFrame(history.prices)
print(f"Precio promedio: {df['price_sun'].mean():.1f} SUN")
print(f"Precio mínimo: {df['price_sun'].min()} SUN")
print(f"Precio máximo: {df['price_sun'].max()} SUN")
print(f"Desv. est: {df['price_sun'].std():.1f} SUN")
```

## Servidor MCP: Integración con Agentes de IA

El servidor MCP de MERX (Protocolo de Contexto de Modelo) permite que los agentes de IA interactúen directamente con el mercado de energy de TRON. Este es el punto de integración más nuevo y habilita un modelo de interacción fundamentalmente diferente.

### Cuándo Usar MCP

- Estás construyendo agentes de IA que gestionen operaciones de TRON
- Quieres gestión conversacional de energy
- Estás usando Claude, ChatGPT u otros LLMs con capacidades de uso de herramientas
- Quieres prototipar estrategias de energy rápidamente a través del lenguaje natural

### Cómo Funciona

El servidor MCP en [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp) expone las capacidades de MERX como herramientas que los agentes de IA pueden llamar:

- `get_prices` -- Verificar precios actuales de energy
- `create_order` -- Comprar energy
- `analyze_prices` -- Obtener estadísticas de precios
- `estimate_energy` -- Simular necesidades de energy de transacciones
- `check_resources` -- Verificar saldo de energy de la billetera

Un agente de IA puede usar estas herramientas para responder preguntas como "¿Cuál es la energy más barata disponible ahora?" o ejecutar comandos como "Compra 65,000 de energy para mi billetera al mejor precio."

### Integración

```json
{
  "mcpServers": {
    "merx": {
      "command": "npx",
      "args": ["merx-mcp"],
      "env": {
        "MERX_API_KEY": "your-api-key"
      }
    }
  }
}
```

## Elegir la Integración Correcta

### Matriz de Decisión

| Tu Situación | Integración Recomendada |
|---|---|
| Prototipo rápido, cualquier lenguaje | REST API |
| Backend Node.js | SDK de JS |
| Backend Python | SDK de Python |
| Pantalla de precios en tiempo real | WebSocket |
| Arquitectura dirigida por eventos | Webhooks |
| Serverless (Lambda, Cloud Functions) | REST API + Webhooks |
| Construcción de agentes de IA | Servidor MCP |
| Backend Go / Java / Ruby | REST API |
| Aplicación completa | SDK de JS/Python + Webhooks |

### Combinando Puntos de Integración

La mayoría de los sistemas de producción usan múltiples métodos de integración:

- **SDK + Webhooks**: Usar el SDK para solicitudes salientes (precios, órdenes) y webhooks para notificaciones entrantes (orden completada, alertas de precios)
- **WebSocket + REST**: Usar WebSocket para monitoreo y REST para acciones
- **REST + Webhooks**: El stack completo agnóstico de lenguaje
- **MCP + SDK**: Agente de IA para estrategia, SDK para ejecución

## Ruta de Migración

Si actualmente estás usando la API de un único proveedor, migrar a MERX es sencillo:

1. **Agregar el SDK de MERX** a tu proyecto
2. **Reemplazar verificaciones de precios** con `merx.getPrices()` -- mismos datos, más proveedores
3. **Reemplazar colocación de órdenes** con `merx.createOrder()` -- mismo flujo, enrutamiento del mejor precio
4. **Agregar webhooks** para notificaciones de órdenes (si no es ya dirigido por eventos)
5. **Eliminar código del proveedor antiguo** una vez que la integración de MERX se verifique

La migración puede hacerse de forma incremental. Ejecutar ambos sistemas en paralelo durante las pruebas, comparando precios y resultados de órdenes antes de cambiar completamente.

## Conclusión

MERX está diseñado para encajar en cualquier stack de tecnología a través del método de integración que coincida con tu arquitectura. REST por universalidad, WebSocket por tiempo real, webhooks para eventos, SDKs por experiencia del desarrollador, y MCP para agentes de IA.

La elección no es exclusiva: combina puntos de integración para que coincidan con tus necesidades. Comienza con la opción más simple que resuelva tu problema inmediato, y expande a medida que tus requisitos crecen.

Documentación de API en [https://merx.exchange/docs](https://merx.exchange/docs). Servidor MCP en [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp). Plataforma en [https://merx.exchange](https://merx.exchange).

## Pruébalo Ahora con IA

Agrega MERX a Claude Desktop o cualquier cliente compatible con MCP -- sin instalación, sin necesidad de clave API para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregunta a tu agente de IA: "¿Cuál es la energy de TRON más barata en este momento?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)