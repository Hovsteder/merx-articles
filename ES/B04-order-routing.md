# Enrutamiento de Órdenes MERX: Cómo Tu Orden Obtiene el Mejor Precio

Cuando envías una orden de energy a MERX, una secuencia de decisiones ocurre en milisegundos: qué proveedor tiene el mejor precio, ¿pueden completar esta orden?, qué sucede si fallan, cómo verificamos que la delegación se registró en la cadena. Este es el motor de enrutamiento de órdenes - el componente que convierte una simple llamada a API en una entrega de energy optimizada, tolerante a fallos y verificada.

Este artículo recorre el ciclo de vida completo de una orden MERX, desde el momento en que llamas a `createOrder` hasta el momento en que el energy delegado aparece en tu dirección TRON.

---

## El Ciclo de Vida de la Orden

Cada orden pasa por seis etapas:

```
1. Validación    -> ¿La orden está bien formada?
2. Pricing       -> ¿Cuál es el mejor precio en este momento?
3. Enrutamiento  -> ¿Qué proveedor(es) la completarán?
4. Ejecución     -> Enviar al/los proveedor(es)
5. Verificación  -> Confirmar delegación en la cadena
6. Liquidación   -> Debitar saldo, registrar en libro mayor
```

Recorramos cada etapa.

---

## Etapa 1: Validación

Antes de que se ejecute cualquier lógica de enrutamiento, la orden se valida:

```typescript
// Validación de entrada con Zod
const OrderSchema = z.object({
  energy: z.number().int().min(10000).max(100000000),
  targetAddress: z.string().refine(isValidTronAddress),
  duration: z.enum(['1h', '1d', '3d', '7d', '14d', '30d']),
  maxPrice: z.number().optional(),  // SUN por unidad de energy
  idempotencyKey: z.string().optional()
});
```

Validaciones clave:

- **Cantidad de energy**: debe ser un entero positivo dentro del rango soportado.
- **Dirección de destino**: debe ser una dirección TRON válida y activada. El sistema valida el formato de la dirección y opcionalmente verifica en la cadena que la cuenta existe.
- **Duración**: debe ser uno de los períodos de delegación soportados.
- **Precio máximo**: límite superior opcional. Si se establece, la orden solo se ejecuta si el mejor precio disponible está en o por debajo de este umbral.
- **Clave de idempotencia**: si se proporciona, los envíos duplicados con la misma clave devuelven la orden original en lugar de crear una nueva.

### Idempotencia

La clave de idempotencia es crítica para integraciones en producción. Los problemas de red pueden causar que un cliente reintente una solicitud, potencialmente creando órdenes duplicadas. Con una clave de idempotencia, la segunda solicitud devuelve el resultado de la primera:

```typescript
const order = await client.createOrder({
  energy: 65000,
  targetAddress: 'TBuyerAddress...',
  duration: '1h',
  idempotencyKey: 'payment-123-energy'
});

// Si se llama nuevamente con la misma clave, devuelve la misma orden
// Sin delegación duplicada, sin cargos dobles
```

---

## Etapa 2: Pricing

El ejecutor de órdenes lee los precios actuales del caché de Redis. El monitor de precios actualiza estos cada 30 segundos, así que los datos tienen a lo sumo 30 segundos de antigüedad.

```typescript
async function getBestPrices(
  energyAmount: number,
  duration: string
): Promise<ProviderPrice[]> {

  const allPrices = await redis.keys('prices:*');
  const validPrices = [];

  for (const key of allPrices) {
    const price = JSON.parse(await redis.get(key));

    // Filtro: debe soportar la duración solicitada
    if (!price.durations.includes(duration)) continue;

    // Filtro: debe tener suficiente disponibilidad
    if (price.availableEnergy < energyAmount) continue;

    // Filtro: debe pasar el umbral de salud
    const health = await getProviderHealth(price.provider);
    if (health.fillRate < 0.90) continue;

    validPrices.push(price);
  }

  // Ordenar por precio efectivo (considerando confiabilidad)
  return validPrices.sort((a, b) => {
    const effectiveA = a.energyPricePerUnit / a.fillRate;
    const effectiveB = b.energyPricePerUnit / b.fillRate;
    return effectiveA - effectiveB;
  });
}
```

### Cumplimiento del Precio Máximo

Si el comprador especificó un `maxPrice`, la orden se rechaza si ningún proveedor puede cumplirlo:

```typescript
if (maxPrice && bestPrice.energyPricePerUnit > maxPrice) {
  throw new OrderError({
    code: 'PRICE_EXCEEDED',
    message: `Best available price (${bestPrice.energyPricePerUnit} SUN) exceeds your maximum (${maxPrice} SUN)`,
    details: { bestAvailable: bestPrice.energyPricePerUnit, maxPrice }
  });
}
```

Esto previene cargos inesperados durante picos de precio.

---

## Etapa 3: Enrutamiento

El motor de enrutamiento decide qué proveedor(es) completarán la orden. Esta es la inteligencia central del sistema.

### Caso Simple: Completado por Un Único Proveedor

Para órdenes dentro de la capacidad de un único proveedor:

```
Orden: 65,000 energy
Mejor proveedor: itrx a 85 SUN/unidad, 500,000 disponibles

Enrutamiento: 100% a itrx
```

### Caso Dividido: Completado por Múltiples Proveedores

Para órdenes grandes o cuando el proveedor más barato tiene stock limitado:

```
Orden: 500,000 energy

Proveedor A: 200,000 disponibles a 85 SUN
Proveedor B: 180,000 disponibles a 87 SUN
Proveedor C: 300,000 disponibles a 92 SUN

Plan de enrutamiento:
  Tramo 1: Proveedor A -> 200,000 energy a 85 SUN
  Tramo 2: Proveedor B -> 180,000 energy a 87 SUN
  Tramo 3: Proveedor C -> 120,000 energy a 92 SUN

Tasa combinada: (200K*85 + 180K*87 + 120K*92) / 500K = 87.28 SUN
```

El enrutador se abastece del más barato al más caro, tomando la mayor cantidad posible de cada proveedor antes de pasar al siguiente.

### La Cadena de Respaldo

Todo plan de enrutamiento incluye una cadena de respaldo - una lista ordenada de proveedores alternativos para intentar si el primario falla:

```
Primario:   Proveedor A (85 SUN)
Respaldo 1: Proveedor B (87 SUN)
Respaldo 2: Proveedor C (92 SUN)
Respaldo 3: Proveedor D (95 SUN)
```

Si el Proveedor A falla en ejecutar (error de API, tiempo de espera agotado, fondos insuficientes), el ejecutor automáticamente se mueve al Proveedor B sin ninguna acción del comprador.

---

## Etapa 4: Ejecución

El ejecutor envía la orden a el/los proveedor(es) seleccionado(s) y monitorea la finalización.

### Flujo de Ejecución

```typescript
async function executeOrder(
  order: Order,
  routingPlan: RoutingPlan
): Promise<ExecutionResult> {

  const results: LegResult[] = [];

  for (const leg of routingPlan.legs) {
    try {
      const result = await executeLeg(leg, order);
      results.push(result);
    } catch (error) {
      // Proveedor primario falló - intentar respaldo
      const failoverResult = await executeWithFailover(
        leg,
        order,
        routingPlan.failoverChain
      );
      results.push(failoverResult);
    }
  }

  return {
    orderId: order.id,
    legs: results,
    totalEnergy: results.reduce((sum, r) => sum + r.energy, 0),
    totalCostSun: results.reduce((sum, r) => sum + r.costSun, 0)
  };
}

async function executeWithFailover(
  failedLeg: RoutingLeg,
  order: Order,
  failoverChain: Provider[]
): Promise<LegResult> {

  for (const provider of failoverChain) {
    try {
      const result = await executeLeg(
        { ...failedLeg, provider: provider.name },
        order
      );
      return result;
    } catch (error) {
      // Registrar y continuar al siguiente respaldo
      continue;
    }
  }

  throw new OrderError({
    code: 'ALL_PROVIDERS_FAILED',
    message: 'Order could not be filled by any available provider'
  });
}
```

### Manejo de Tiempo de Espera

Cada ejecución de proveedor tiene un tiempo de espera estricto. Si el proveedor no reconoce la orden dentro de la ventana de tiempo de espera (típicamente 30 segundos), el ejecutor se mueve a la cadena de respaldo:

```
T+0:    Enviar orden al Proveedor A
T+30s:  Sin respuesta -> tiempo de espera agotado, respaldo al Proveedor B
T+31s:  Enviar orden al Proveedor B
T+35s:  Proveedor B reconoce -> proceder a verificación
```

---

## Etapa 5: Verificación

El reconocimiento del proveedor no es suficiente. MERX verifica que la delegación de energy realmente aparezca en la cadena de bloques TRON.

### Proceso de Verificación En Cadena

```typescript
async function verifyDelegation(
  targetAddress: string,
  expectedEnergy: number,
  delegationTxHash: string
): Promise<VerificationResult> {

  // Paso 1: Verificar que la transacción de delegación existe
  const tx = await tronWeb.trx.getTransaction(delegationTxHash);
  if (!tx || tx.ret[0].contractRet !== 'SUCCESS') {
    return { verified: false, reason: 'Transaction not found or failed' };
  }

  // Paso 2: Verificar recursos de la dirección de destino
  const resources = await tronWeb.trx.getAccountResources(targetAddress);
  const currentEnergy = resources.EnergyLimit || 0;

  // Paso 3: Verificar que el energy aumentó por la cantidad esperada (con tolerancia)
  const tolerance = expectedEnergy * 0.02; // Tolerancia del 2%
  if (currentEnergy < expectedEnergy - tolerance) {
    return { verified: false, reason: 'Energy amount below expected' };
  }

  return {
    verified: true,
    txHash: delegationTxHash,
    energyDelivered: currentEnergy,
    verifiedAt: new Date()
  };
}
```

### Por Qué Tolerancia del 2%

Los montos de delegación de energy se basan en ratios de conversión de TRX a energy que pueden cambiar ligeramente entre la colocación de la orden y la confirmación de delegación. Una tolerancia del 2% explica esto sin aceptar montos grossamente incorrectos.

### Tiempo de Verificación

```
T+0:    Proveedor reconoce la orden
T+3-6s: Transacción de delegación confirma en TRON (1 bloque)
T+10s:  MERX consulta recursos de dirección de destino
T+10s:  Verificación completa, comprador notificado
```

Todo el proceso desde envío de orden hasta entrega verificada típicamente toma 15-45 segundos.

---

## Etapa 6: Liquidación

Después de verificación, ocurre la liquidación financiera:

### Deducción de Saldo

```sql
-- Deducción y verificación de saldo atómica
BEGIN;

SELECT balance_sun FROM accounts
WHERE user_id = $1
FOR UPDATE;

-- Verificar saldo suficiente
-- Si insuficiente, ROLLBACK y retornar error

UPDATE accounts
SET balance_sun = balance_sun - $2
WHERE user_id = $1;

COMMIT;
```

`SELECT FOR UPDATE` asegura que no haya condición de carrera entre verificación de saldo y deducción. Si dos órdenes se procesan simultáneamente, la segunda esperará a que la primera se complete antes de verificar el saldo.

### Entrada de Libro Mayor

Cada liquidación crea una entrada inmutable de libro mayor:

```sql
INSERT INTO ledger (
  user_id, type, amount_sun,
  reference_type, reference_id,
  balance_before, balance_after,
  created_at
) VALUES (
  $1, 'ORDER_PAYMENT', $2,
  'order', $3,
  $4, $5,
  NOW()
);
```

Las entradas de libro mayor son solo adiciones. Nunca se actualizan o se eliminan. Esto crea un historial completo y auditable de cada operación financiera.

---

## Rastreando Tu Orden

La API de MERX proporciona estado de orden en tiempo real a través de REST y WebSocket:

```typescript
import { MerxClient } from 'merx-sdk';

const client = new MerxClient({ apiKey: 'your-key' });

// REST: consultar estado de la orden
const order = await client.getOrder('ord_abc123');
console.log(order.status);
// 'pending' | 'executing' | 'verifying' | 'completed' | 'failed'

// WebSocket: actualizaciones en tiempo real
client.onOrderUpdate('ord_abc123', (update) => {
  console.log(`Order ${update.orderId}: ${update.status}`);
  if (update.status === 'completed') {
    console.log(`Delegation TX: ${update.delegationTxHash}`);
  }
});
```

### Flujo de Estado de Orden

```
pending -> executing -> verifying -> completed
                |                       |
                v                       v
              failed              partially_filled
```

- **pending**: Orden recibida, esperando ejecución.
- **executing**: Enviada al proveedor, esperando reconocimiento.
- **verifying**: Proveedor reconoció, esperando confirmación en cadena.
- **completed**: Energy verificado en dirección de destino.
- **failed**: Todos los proveedores fallaron. Saldo no cargado.
- **partially_filled**: Algunos tramos completados, otros fallidos. Saldo cargado solo por porción completada.

---

## Manejo de Errores

### Errores de Proveedor

Cada proveedor puede fallar de formas únicas. El ejecutor de órdenes normaliza todos los errores de proveedor en códigos de error estándar:

```json
{
  "error": {
    "code": "PROVIDER_UNAVAILABLE",
    "message": "Primary provider could not fill the order. Failover to secondary provider.",
    "details": {
      "primaryProvider": "tronsave",
      "primaryError": "timeout",
      "filledBy": "feee",
      "priceImpact": "+2 SUN/unit"
    }
  }
}
```

### Completados Parciales

Para órdenes multi-tramo, algunos tramos pueden tener éxito mientras otros fallan. MERX maneja esto:

1. Completando tramos exitosos normalmente.
2. Intentando respaldo para tramos fallidos.
3. Si el respaldo también falla, retornando un resultado de completado parcial.
4. Cobrando al comprador solo por el energy que fue realmente entregado.

---

## Rendimiento

Tiempos típicos de ejecución de orden:

```
Validación:    < 5ms
Pricing:       < 10ms (lectura de Redis)
Enrutamiento:  < 5ms
Ejecución:     5-30 segundos (API de proveedor + cadena de bloques)
Verificación:  3-10 segundos (confirmación en cadena de bloques)

Total: 10-45 segundos desde llamada a API hasta entrega verificada
```

El cuello de botella siempre es la cadena de bloques. El procesamiento interno de MERX agrega menos de 50ms de sobrecarga. El resto es esperar al proveedor y la red TRON.

---

## Conclusión

El enrutamiento de órdenes es donde el valor de MERX es más tangible. Una sola llamada a API desencadena una cascada de optimizaciones: selección de mejor precio, división multi-proveedor, respaldo automático, verificación en cadena, y liquidación atómica. Cada paso está diseñado para asegurar que obtengas el energy más barato disponible, entregado confiablemente, con completa auditabilidad.

La complejidad es real, pero es complejidad de MERX para gestionar, no la tuya. Tu integración sigue siendo una sola llamada a API.

Comienza a enrutar órdenes en [https://merx.exchange](https://merx.exchange). Referencia completa de API en [https://merx.exchange/docs](https://merx.exchange/docs).

---

*Este artículo es parte de la serie técnica de MERX. MERX es el primer intercambio de recursos de cadena de bloques. SDKs: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) y [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python). Servidor MCP: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp).*


## Pruébalo Ahora con IA

Agrega MERX a Claude Desktop o cualquier cliente compatible con MCP -- sin instalación, sin API key necesaria para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregunta a tu agente de IA: "¿Cuál es el energy de TRON más barato en este momento?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)