# Cómo MERX Agrega Todos los Proveedores de Energía en una Sola API

El mercado de energía TRON en 2026 tiene un problema de fragmentación. Al menos siete proveedores principales ofrecen servicios de delegación de energía, cada uno con su propia API, modelo de precios y patrones de disponibilidad. Si quieres el mejor precio, necesitas integrarte con todos ellos, monitorear sus precios continuamente, manejar sus particularidades individuales y construir lógica de failover para cuando uno se cae.

O puedes hacer una sola llamada API a MERX.

Este artículo explica cómo MERX agrega todos los proveedores principales de energía en una sola API - la arquitectura detrás del monitoreo de precios, enrutamiento al mejor precio, failover automático, y la simplificación operacional que viene de reemplazar siete integraciones con una.

---

## El Panorama de Proveedores

El mercado de energía TRON incluye múltiples proveedores, cada uno operando de forma independiente. A principios de 2026, los proveedores principales incluyen:

- **TronSave** - uno de los primeros servicios de alquiler de energía
- **Feee** - precios competitivos con acceso API
- **itrx** - enfoque en órdenes de energía al por mayor
- **CatFee** - posicionamiento en el mercado medio
- **Netts** - participante más nuevo con precios agresivos
- **SoHu** - proveedor con enfoque en el mercado chino
- **PowerSun** - staking directo y delegación

Cada proveedor tiene su propio:

- Formato de API y método de autenticación
- Estructura de precios (algunos cobran por unidad de energía, otros por TRX)
- Tamaños mínimos y máximos de órdenes
- Duraciones disponibles (1h, 1d, 3d, 7d, etc.)
- Métodos de pago soportados
- Páginas de estado y características de disponibilidad

### El Costo de Integración

Integrarse con un solo proveedor es directo. Integrarse con todos ellos para obtener el mejor precio es una tarea de ingeniería significativa:

```
Por proveedor:
  - Implementación del cliente API:     2-3 días
  - Normalización de precios:           1 día
  - Manejo de errores:                  1 día
  - Pruebas:                            1-2 días
  - Mantenimiento continuo:             2-4 horas/mes

7 proveedores x 5-7 días = 35-49 días de integración inicial
7 proveedores x 3 horas/mes = 21 horas/mes de mantenimiento continuo
```

Este es el costo de integración que MERX elimina. En lugar de mantener siete integraciones de proveedores, mantienes una integración de MERX. MERX maneja el resto.

---

## La Arquitectura de MERX

MERX se sitúa entre tu aplicación y el ecosistema de proveedores. La arquitectura tiene tres componentes principales:

### 1. Monitor de Precios

El monitor de precios es un servicio dedicado que continuamente consulta a cada proveedor integrado por precios actuales. Cada 30 segundos, consulta la API de cada proveedor, normaliza la respuesta a un formato estándar, y publica el resultado en un canal pub/sub de Redis.

```
Cada 30 segundos:
  Para cada proveedor:
    1. Consultar API del proveedor por precios actuales
    2. Normalizar a formato estándar (SUN por unidad de energía)
    3. Validar respuesta (rechazar valores atípicos, datos antiguos)
    4. Publicar en Redis: canal "prices:{provider}"
    5. Almacenar en historial de precios (PostgreSQL)
```

El intervalo de 30 segundos es elegido deliberadamente. El polling más rápido estresaría las APIs de proveedores y añadiría valor mínimo (los precios rara vez cambian segundo a segundo). El polling más lento arriesgaría servir precios antiguos.

### 2. Caché de Precios en Redis

Redis sirve como el caché de precios en tiempo real. Cada actualización de precio del monitor de precios se almacena en Redis con un TTL (tiempo de vida) de 60 segundos - el doble del intervalo de polling. Si los datos de precio de un proveedor son más antiguos que 60 segundos, se expiran automáticamente y se excluyen de las decisiones de enrutamiento.

```
Estructura de clave Redis:
  prices:tronsave     -> { energy: 88, bandwidth: 2, updated: 1711756800 }
  prices:feee         -> { energy: 92, bandwidth: 3, updated: 1711756800 }
  prices:itrx         -> { energy: 85, bandwidth: 2, updated: 1711756800 }
  prices:catfee       -> { energy: 95, bandwidth: 3, updated: 1711756800 }
  ...

  prices:best         -> { provider: "itrx", energy: 85, updated: 1711756800 }
```

La clave `prices:best` se recalcula en cada actualización de precio, dando a la API acceso instantáneo al mejor precio actual sin escanear todos los proveedores.

### 3. Ejecutor de Órdenes

Cuando colocas una orden a través de la API de MERX, el ejecutor de órdenes la recibe y determina el enrutamiento óptimo:

```
Orden recibida: 65,000 energía para TBuyerAddress

1. Leer prices:best de Redis -> itrx a 85 SUN/unidad
2. Verificar disponibilidad de itrx para 65,000 energía -> disponible
3. Enviar orden a itrx
4. Monitorear confirmación de delegación en cadena
5. Verificar que energía llegó a TBuyerAddress
6. Notificar al comprador (webhook + WebSocket)
```

Si el proveedor más barato no puede completar la orden (stock insuficiente, error de API, timeout), el ejecutor automáticamente pasa al siguiente proveedor más barato.

---

## Normalización de Precios

Diferentes proveedores cotizan precios en formatos diferentes. Algunos cotizan en SUN por unidad de energía. Algunos cotizan TRX total para una cantidad dada de energía. Algunos incluyen bandwidth en el precio; otros cobran por separado.

MERX normaliza todo a un formato único:

```typescript
interface NormalizedPrice {
  provider: string;
  energyPricePerUnit: number;    // SUN por unidad de energía
  bandwidthPricePerUnit: number; // SUN por unidad de bandwidth
  minOrder: number;              // Mínimo de unidades de energía
  maxOrder: number;              // Máximo de unidades de energía
  availableEnergy: number;       // Disponible actualmente
  durations: string[];           // Duraciones soportadas
  lastUpdated: number;           // Timestamp Unix
}
```

Esta normalización es crítica. Sin ella, comparar precios entre proveedores requeriría que el consumidor entienda el modelo de precios de cada proveedor. Con ella, la comparación de precios es un simple ordenamiento numérico.

---

## Enrutamiento al Mejor Precio en Detalle

El algoritmo de enrutamiento no es simplemente "elegir el más barato". Varios factores influyen en la decisión de enrutamiento:

### Factor 1: Precio

El factor principal. Si todo lo demás es igual, el proveedor más barato gana.

### Factor 2: Disponibilidad

Un proveedor que cotiza 80 SUN pero con solo 10,000 energía disponible no puede completar una orden de 65,000 energía. El enrutador debe verificar el inventario disponible.

### Factor 3: Confiabilidad

MERX rastrea la tasa histórica de completación, tiempo de respuesta y tasa de fallos de cada proveedor. Un proveedor con tasa de completación del 95% es penalizado relativamente a uno con 99%, incluso si el proveedor del 95% es ligeramente más barato.

```
Precio efectivo = precio_cotizado / tasa_completacion

Proveedor A: 85 SUN, tasa completación 99% -> 85.86 efectivo
Proveedor B: 82 SUN, tasa completación 94% -> 87.23 efectivo
Ganador: Proveedor A a pesar del precio cotizado más alto
```

### Factor 4: Soporte de Duración

No todos los proveedores soportan todas las duraciones. Si necesitas una delegación de 1 hora, los proveedores que solo ofrecen mínimos diarios se excluyen.

### División de Órdenes

Para órdenes grandes que exceden la capacidad de cualquier proveedor único, el enrutador divide la orden entre múltiples proveedores:

```
Orden: 500,000 energía

Proveedor A: 200,000 disponibles a 85 SUN -> completar 200,000
Proveedor B: 180,000 disponibles a 87 SUN -> completar 180,000
Proveedor C: 300,000 disponibles a 92 SUN -> completar 120,000

Total completado: 500,000 energía
Tasa mezclada: 87.28 SUN/unidad
```

El comprador ve una sola orden con una tasa mezclada. La complejidad de la ejecución multi-proveedor está completamente oculta.

---

## Failover Automático

El failover es donde la agregación realmente justifica su valor. Cuando te integras directamente con un proveedor y se cae, tu aplicación deja de funcionar. Con MERX, los fallos de proveedores se manejan de forma transparente.

### Cadena de Failover

```
Fallo del proveedor principal
  |
  v
Marcar proveedor como no saludable (excluir del enrutamiento por 5 minutos)
  |
  v
Reintentar con siguiente proveedor más barato
  |
  v
Si segundo proveedor falla, intentar tercero
  |
  v
Si todos los proveedores fallan, retornar error al comprador con orientación de reintento
```

### Rastreo de Salud

El monitor de precios mantiene un puntaje de salud para cada proveedor:

```
Componentes del puntaje de salud:
  - Última extracción de precios exitosa: debe estar dentro de 60s
  - Tiempo de respuesta de API: penalizar > 2 segundos
  - Tasa de completación de órdenes reciente: penalizar < 95%
  - Tasa de errores reciente: penalizar > 5%
```

Los proveedores no saludables se excluyen del enrutamiento hasta que se recuperen. La recuperación se detecta automáticamente cuando el monitor de precios obtiene exitosamente precios de ellos de nuevo.

### Cero Tiempo de Inactividad para Compradores

Desde la perspectiva del comprador, los fallos de proveedores son invisibles. Su llamada API tiene éxito mientras al menos un proveedor esté operativo. En la práctica, tener siete o más proveedores significa que la interrupción total del mercado es esencialmente imposible - las probabilidades de que todos los proveedores estén caídos simultáneamente son negligibles.

---

## Una API Reemplaza Muchas

Aquí está lo que tu integración se ve con MERX versus integración directa con múltiples proveedores:

### Sin MERX

```typescript
// Pseudocódigo: integración directa con múltiples proveedores

// Inicializar 7 clientes de proveedores
const tronsave = new TronSaveClient(apiKey1);
const feee = new FeeeClient(apiKey2);
const itrx = new ItrxClient(apiKey3);
// ... 4 más

// Obtener precios de todos los proveedores
const prices = await Promise.allSettled([
  tronsave.getPrice(65000),
  feee.getPrice(65000),
  itrx.getPrice(65000),
  // ... 4 más
]);

// Normalizar formatos de respuesta diferentes
const normalized = prices
  .filter(p => p.status === 'fulfilled')
  .map(p => normalizePrice(p.value)); // lógica compleja por proveedor

// Ordenar por precio, verificar disponibilidad, manejar errores...
const best = normalized.sort((a, b) => a.price - b.price)[0];

// Colocar orden con mejor proveedor
try {
  const order = await getClient(best.provider).createOrder({
    energy: 65000,
    target: buyerAddress,
    // Parámetros específicos del proveedor...
  });
} catch (e) {
  // Failover al siguiente proveedor...
  // Más manejo de errores específico del proveedor...
}
```

### Con MERX

```typescript
import { MerxClient } from 'merx-sdk';

const client = new MerxClient({ apiKey: 'your-merx-key' });

// Obtener mejor precio entre todos los proveedores
const prices = await client.getPrices({ energy: 65000 });
console.log(`Mejor: ${prices.bestPrice.provider} a ${prices.bestPrice.perUnit} SUN`);

// Colocar orden - enrutada automáticamente al mejor proveedor
const order = await client.createOrder({
  energy: 65000,
  targetAddress: buyerAddress,
  duration: '1h'
});

// Listo. Failover, reintentos, verificación manejados automáticamente.
```

Siete integraciones se convierten en una. Cientos de líneas de enrutamiento y código de failover se convierten en cuatro líneas. El mantenimiento continuo de cambios de API de proveedores cae a cero.

---

## Feed de Precios en Tiempo Real

Para aplicaciones que quieren mostrar precios en vivo o tomar decisiones de enrutamiento en tiempo real, MERX proporciona un feed de precios por WebSocket:

```typescript
const client = new MerxClient({ apiKey: 'your-key' });

client.onPriceUpdate((update) => {
  console.log(`${update.provider}: ${update.energyPrice} SUN/unidad`);
  console.log(`Mejor precio: ${update.bestPrice} SUN/unidad`);
});
```

El feed de WebSocket publica cada actualización de precio del monitor de precios - aproximadamente cada 30 segundos por proveedor. Esto permite a las aplicaciones mostrar precios en vivo sin polling.

---

## Transparencia del Proveedor

MERX no oculta qué proveedor completó tu orden. Cada respuesta de orden incluye el nombre del proveedor, el precio pagado, y el hash de transacción de delegación en cadena:

```json
{
  "orderId": "ord_abc123",
  "status": "completed",
  "provider": "itrx",
  "energy": 65000,
  "pricePerUnit": 85,
  "totalCostSun": 5525000,
  "delegationTxHash": "abc123def456...",
  "verifiedAt": "2026-03-30T12:00:00Z"
}
```

Siempre sabes de dónde vino tu energía, qué pagaste, y puedes verificar la delegación en cadena de forma independiente.

---

## Primeros Pasos

La agregación de MERX está disponible a través de la API REST, el SDK de JavaScript, el SDK de Python, y el servidor MCP para agentes de IA:

- **Documentación de API**: [https://merx.exchange/docs](https://merx.exchange/docs)
- **SDK de JavaScript**: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- **SDK de Python**: [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)
- **Servidor MCP**: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)

Crea una cuenta en [https://merx.exchange](https://merx.exchange), obtén una clave API, y comienza a enrutar órdenes de energía al mejor precio disponible con una sola llamada API.

---

*Este artículo es parte de la serie técnica de MERX. MERX es el primer intercambio de recursos blockchain, agregando todos los principales proveedores de energía TRON en una sola API con enrutamiento al mejor precio y failover automático.*


## Pruébalo Ahora con IA

Añade MERX a Claude Desktop o cualquier cliente compatible con MCP -- sin instalación, sin clave API necesaria para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregunta a tu agente de IA: "¿Cuál es la energía TRON más barata en este momento?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)