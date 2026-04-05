# Monitor de Precios MERX: Cómo Rastreamos Cada Proveedor Cada 30 Segundos

El monitor de precios es el corazón de MERX. Cada 30 segundos, se conecta con cada proveedor de energía integrado, obtiene sus precios actuales, normaliza los datos y los publica al resto del sistema. Sin él, el enrutamiento de mejor precio sería adivinanza. Con él, cada orden se enruta al proveedor más económico disponible basado en datos no más antiguos que 30 segundos.

Este artículo es un análisis técnico profundo de la arquitectura del monitor de precios: cómo sondea a los proveedores, cómo el patrón adapter mantiene el sistema extensible, cómo Redis pub/sub distribuye datos de precios en tiempo real, y cómo el historial de precios potencia análisis y toma de decisiones.

---

## Por Qué 30 Segundos

El intervalo de sondeo es una decisión de diseño deliberada. Los precios de energía en TRON no cambian cada segundo - no son como el forex al contado o libros de órdenes de criptomonedas. El precio de los proveedores típicamente se desplaza algunas veces por hora, a veces menos. Un intervalo de 30 segundos captura cada cambio de precio significativo mientras evita varios problemas:

- **Límites de velocidad de la API del proveedor**: la mayoría de los proveedores permiten 1-2 solicitudes por segundo. Con intervalos de 30 segundos, nos mantenemos bien dentro de los límites incluso con reintentos.
- **Sobrecarga de red**: sondear 7+ proveedores genera tráfico HTTP. En 30 segundos, esto es trivial. En 1 segundo, sería sustancial.
- **Frescura de datos vs ruido**: cambios de precio sub-30-segundos en mercados de energía son casi siempre ruido, no señal. 30 segundos filtra el ruido mientras captura movimientos reales.
- **Uso de recursos del sistema**: el monitor de precios se ejecuta junto a otros servicios. El sondeo agresivo compediría por CPU y memoria sin añadir valor.

El TTL en precios en caché se establece en 60 segundos - el doble del intervalo de sondeo. Si un sondeo falla, el precio anterior permanece válido para un ciclo más antes de expirar. Esto previene que un único sondeo fallido elimine un proveedor del libro de órdenes.

---

## El Patrón Adapter

Cada proveedor de energía tiene una API diferente. Diferentes puntos de conexión, diferentes métodos de autenticación, diferentes formatos de respuesta, diferentes códigos de error. El monitor de precios usa el patrón adapter para aislar estas diferencias de la lógica de sondeo central.

### La Interfaz del Proveedor

Cada adaptador de proveedor implementa una interfaz común:

```typescript
interface IEnergyProvider {
  name: string;

  // Fetch current pricing
  getPrices(): Promise<ProviderPriceResponse>;

  // Check if provider is operational
  healthCheck(): Promise<boolean>;

  // Get available inventory
  getAvailability(): Promise<AvailabilityResponse>;
}

interface ProviderPriceResponse {
  energyPricePerUnit: number;   // SUN per energy unit
  bandwidthPricePerUnit: number;
  minOrder: number;
  maxOrder: number;
  durations: Duration[];
  timestamp: number;
}
```

### Un Adaptador de Proveedor

Cada proveedor obtiene su propio archivo adaptador. Aquí hay un ejemplo simplificado de cómo se ve un adaptador de proveedor:

```typescript
// providers/tronsave/index.ts
import { IEnergyProvider, ProviderPriceResponse } from '../base';

export class TronSaveProvider implements IEnergyProvider {
  name = 'tronsave';
  private apiUrl: string;
  private apiKey: string;

  constructor(config: ProviderConfig) {
    this.apiUrl = config.apiUrl;
    this.apiKey = config.apiKey;
  }

  async getPrices(): Promise<ProviderPriceResponse> {
    const response = await fetch(`${this.apiUrl}/v1/prices`, {
      headers: { 'Authorization': `Bearer ${this.apiKey}` }
    });

    const data = await response.json();

    // Normalize provider-specific format to standard format
    return {
      energyPricePerUnit: this.normalizePrice(data.energy_price),
      bandwidthPricePerUnit: this.normalizePrice(data.bandwidth_price),
      minOrder: data.min_energy || 10000,
      maxOrder: data.max_energy || 10000000,
      durations: this.normalizeDurations(data.available_durations),
      timestamp: Date.now()
    };
  }

  async healthCheck(): Promise<boolean> {
    try {
      const response = await fetch(`${this.apiUrl}/health`);
      return response.ok;
    } catch {
      return false;
    }
  }

  // ... provider-specific normalization methods
}
```

### Agregar un Nuevo Proveedor

Uno de los beneficios clave de esta arquitectura es que agregar un nuevo proveedor requiere exactamente un nuevo archivo:

1. Crear `providers/newprovider/index.ts` implementando `IEnergyProvider`.
2. Registrar el proveedor en la configuración.
3. El monitor de precios automáticamente comienza a sondearlo.
4. El ejecutor de órdenes automáticamente lo incluye en decisiones de enrutamiento.

Sin cambios al monitor de precios, sin cambios al ejecutor de órdenes, sin cambios a la API. El patrón adapter asegura que la complejidad específica del proveedor esté encapsulada.

---

## El Bucle de Sondeo

El bucle central del monitor de precios es directo pero cuidadosamente diseñado para confiabilidad:

```typescript
class PriceMonitor {
  private providers: IEnergyProvider[];
  private redis: RedisClient;
  private pollInterval = 30_000; // 30 seconds

  async start() {
    // Initial poll on startup
    await this.pollAll();

    // Schedule recurring polls
    setInterval(() => this.pollAll(), this.pollInterval);
  }

  async pollAll() {
    const results = await Promise.allSettled(
      this.providers.map(provider => this.pollProvider(provider))
    );

    // Compute best price from successful results
    const validPrices = results
      .filter(r => r.status === 'fulfilled')
      .map(r => r.value);

    if (validPrices.length > 0) {
      await this.updateBestPrice(validPrices);
    }
  }

  async pollProvider(provider: IEnergyProvider) {
    const startTime = Date.now();

    try {
      const prices = await provider.getPrices();
      const responseTime = Date.now() - startTime;

      // Store in Redis with 60s TTL
      await this.redis.setex(
        `prices:${provider.name}`,
        60,
        JSON.stringify(prices)
      );

      // Publish price update event
      await this.redis.publish(
        'price-updates',
        JSON.stringify({
          provider: provider.name,
          prices,
          responseTime
        })
      );

      // Update health metrics
      await this.updateHealthMetrics(provider.name, {
        success: true,
        responseTime,
        timestamp: Date.now()
      });

      return { provider: provider.name, prices };

    } catch (error) {
      await this.updateHealthMetrics(provider.name, {
        success: false,
        error: error.message,
        timestamp: Date.now()
      });

      throw error; // Let Promise.allSettled handle it
    }
  }
}
```

### Decisiones de Diseño Clave

**Promise.allSettled, no Promise.all**: Un fallo de un solo proveedor no debe bloquear actualizaciones de otros proveedores. `allSettled` asegura que cada proveedor sea sondeado independientemente.

**TTL de 60 segundos**: Si un proveedor falla al responder durante dos ciclos consecutivos (60 segundos), su precio en caché expira automáticamente. El ejecutor de órdenes no enrutará a un proveedor sin precio en caché.

**Métricas de salud junto a precios**: Cada sondeo registra tiempo de respuesta y éxito/fallo. Estos datos alimentan la puntuación de confiabilidad del algoritmo de enrutamiento.

---

## Distribución de Redis Pub/Sub

El monitor de precios no sirve precios directamente a consumidores de API. En cambio, publica a Redis, y otros servicios se suscriben a las actualizaciones que necesitan.

### Estructura de Canales

```
Channel: price-updates
  -> All price update events (consumed by API service, WebSocket broadcast)

Channel: price-alerts
  -> Significant price changes (consumed by notification service)

Channel: provider-health
  -> Health status changes (consumed by admin dashboard)
```

### Por Qué Pub/Sub en Lugar de Llamadas Directas

El monitor de precios y el servicio de API son procesos separados (contenedores Docker separados, de hecho). Se comunican exclusivamente a través de Redis - sin importaciones directas, sin llamadas a funciones, sin memoria compartida. Este aislamiento significa:

- El monitor de precios puede reiniciarse sin afectar la API.
- La API puede escalar horizontalmente (múltiples instancias) y todas reciben las mismas actualizaciones de precios.
- Un error en el monitor de precios no puede bloquear el servicio de API.
- Cada servicio puede desplegarse independientemente.

### Transmisión WebSocket

El servicio de API se suscribe al canal `price-updates` y transmite actualizaciones a clientes WebSocket conectados:

```
Price Monitor -> Redis pub/sub -> API Service -> WebSocket -> Client

Latency: ~5ms from provider response to client notification
```

Los clientes suscritos al flujo WebSocket reciben actualizaciones de precios casi en tiempo real, habilitando paneles en vivo e interfaces de trading responsivas.

---

## Almacenamiento de Historial de Precios

Cada punto de datos de precio se almacena en PostgreSQL para análisis histórico. El esquema captura la instantánea de precio completa:

```sql
CREATE TABLE price_history (
  id BIGSERIAL PRIMARY KEY,
  provider VARCHAR(50) NOT NULL,
  energy_price_sun BIGINT NOT NULL,
  bandwidth_price_sun BIGINT NOT NULL,
  min_order INTEGER,
  max_order INTEGER,
  available_energy BIGINT,
  response_time_ms INTEGER,
  recorded_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_price_history_provider_time
  ON price_history(provider, recorded_at DESC);
```

### Volumen de Datos

A intervalos de 30 segundos en 7 proveedores:

```
7 providers x 2 polls/minute x 60 minutes x 24 hours = 20,160 rows/day
Monthly: ~604,800 rows
Yearly: ~7,257,600 rows
```

Cada fila es pequeña (aproximadamente 100 bytes), así que el almacenamiento anual es menos de 1 GB. PostgreSQL maneja este volumen trivialmente con indexación apropiada.

### Consultas de Análisis

El historial de precios habilita varios análisis valiosos:

```sql
-- Average price by provider over the last 24 hours
SELECT provider,
       AVG(energy_price_sun) as avg_price,
       MIN(energy_price_sun) as min_price,
       MAX(energy_price_sun) as max_price
FROM price_history
WHERE recorded_at > NOW() - INTERVAL '24 hours'
GROUP BY provider
ORDER BY avg_price;

-- Price trend for a specific provider
SELECT date_trunc('hour', recorded_at) as hour,
       AVG(energy_price_sun) as avg_price
FROM price_history
WHERE provider = 'tronsave'
  AND recorded_at > NOW() - INTERVAL '7 days'
GROUP BY hour
ORDER BY hour;
```

Estas consultas potencian el punto de conexión de API del historial de precios de MERX y el panel de administración.

---

## Manejo de Casos Especiales

### El Proveedor Devuelve Datos Inválidos

El monitor de precios valida cada respuesta antes de almacenarla en caché:

```typescript
function validatePrice(price: ProviderPriceResponse): boolean {
  // Price must be positive
  if (price.energyPricePerUnit <= 0) return false;

  // Price must be within reasonable bounds (10-500 SUN)
  if (price.energyPricePerUnit < 10_000_000) return false;  // < 10 SUN
  if (price.energyPricePerUnit > 500_000_000) return false;  // > 500 SUN

  // Must have valid timestamp
  if (price.timestamp > Date.now() + 60_000) return false;  // future
  if (price.timestamp < Date.now() - 300_000) return false;  // > 5min old

  return true;
}
```

Los datos inválidos se registran y se descartan. El precio válido anterior permanece en caché hasta que expira naturalmente.

### Cambios en la API del Proveedor

Las APIs del proveedor cambian ocasionalmente - nuevos campos, puntos de conexión deprecados, formatos de respuesta modificados. Porque cada proveedor tiene su propio adaptador, los cambios de API están aislados a un solo archivo. El adaptador se actualiza, se prueba y se implementa sin tocar ninguna otra parte del sistema.

### Particiones de Red

Si el monitor de precios pierde conectividad de red, todos los sondeos del proveedor fallan simultáneamente. El TTL de 60 segundos asegura que los precios en caché expiren dentro de un minuto, y el ejecutor de órdenes detiene el enrutamiento a todos los proveedores. Cuando la conectividad se restaura, el siguiente ciclo de sondeo repuebla el caché automáticamente.

### Desfase de Reloj

El monitor de precios se ejecuta en un solo servidor, así que el desfase de reloj entre servicios no es una preocupación para tiempos relativos. Las marcas de tiempo en historial de precios usan `NOW()` de PostgreSQL, asegurando consistencia. Para marcas de tiempo absolutas en respuestas de API, el servidor ejecuta NTP.

---

## Monitoreo del Monitor

El monitor de precios mismo se monitorea a través de varios mecanismos:

- **Punto de conexión de salud**: el servicio expone un punto de conexión `/health` que reporta la última hora de sondeo exitosa para cada proveedor.
- **Alertas**: si ninguna actualización de precio exitosa ha sido publicada durante 5 minutos, se dispara una alerta.
- **Métricas**: el recuento de sondeos, tasa de éxito, tiempo de respuesta promedio, y tasa de acierto de caché se rastrean.
- **Registros**: cada resultado de sondeo (éxito o fallo) se registra con JSON estructurado para análisis.

---

## Características de Rendimiento

```
Typical poll cycle (7 providers):
  Total time: 1-3 seconds (parallel HTTP requests)
  Redis writes: 8 (7 provider prices + 1 best price)
  Redis publishes: 7 (one per provider update)
  PostgreSQL inserts: 7 (one per provider)
  Memory usage: < 50 MB
  CPU usage: < 2% average
```

El monitor de precios es intencionalmente ligero. Hace una cosa - obtener y distribuir precios - y la hace eficientemente. El intervalo de 30 segundos significa que el servicio está inactivo el 90% del tiempo, dejando recursos disponibles para los servicios de ejecutor de órdenes y API más intensivos en cómputo.

---

## Conclusión

El monitor de precios es conceptualmente simple - sondear proveedores, normalizar datos, publicar actualizaciones. Pero los detalles importan. El patrón adapter hace agregar proveedores trivial. Redis pub/sub desacopla la recopilación de precios del consumo de precios. Los TTLs de 60 segundos automáticamente excluyen datos obsoletos. Las métricas de salud alimentan decisiones de enrutamiento. Y el historial de precios habilita análisis que ayuda a usuarios a tomar mejores decisiones.

Cada precio que ves en MERX, cada recomendación de mejor precio, cada decisión de enrutamiento automático se remonta al latido de 30 segundos del monitor de precios. Es la fundación que hace posible la agregación.

Explora precios en vivo y datos históricos en [https://merx.exchange](https://merx.exchange). Para acceso a API, ver la documentación en [https://merx.exchange/docs](https://merx.exchange/docs).

---

*Este artículo es parte de la serie técnica de MERX. MERX es el primer intercambio de recursos blockchain. Código fuente para SDKs: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js), [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python).*


## Pruébalo Ahora con IA

Agrega MERX a Claude Desktop o cualquier cliente compatible con MCP -- cero instalación, sin necesidad de clave API para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregunta a tu agente de IA: "¿Cuál es la energía de TRON más barata ahora mismo?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)