# Cómo Funciona la Delegación de Energía en TRON

La delegación de energía es el mecanismo que hace posible todo el mercado de alquiler de energía en TRON. Sin ella, la energía sería intransferible: los stakers solo podrían usar su propia energía y no habría forma de venderla o compartirla. Comprender cómo funciona la delegación a nivel de protocolo es esencial para cualquiera que esté construyendo en TRON o evaluando proveedores de energía.

Este artículo explica en detalle el mecanismo de delegación de Stake 2.0: cómo los proveedores delegan energía a los compradores, qué sucede durante y después del período de delegación, y cómo MERX gestiona el ciclo de vida completo de la delegación entre múltiples proveedores.

---

## La Base de Stake 2.0

TRON introdujo Stake 2.0 (también llamado Stake v2) como reemplazo del mecanismo original de congelación de recursos. La innovación clave de Stake 2.0 es la separación entre el staking y el uso de recursos. Bajo el modelo original (Stake 1.0), el TRX congelado producía energía que solo podía ser utilizada por la dirección del que lo congelaba. Stake 2.0 introdujo la delegación, permitiendo que una dirección dirija sus recursos stakeados a cualquier otra dirección en la red.

### Las Tres Operaciones

La delegación de Stake 2.0 implica tres operaciones en cadena:

**1. Stake (freezeBalanceV2)**

El proveedor bloquea TRX en el contrato de staking. Esto produce energía proporcional a la cantidad stakeada y al total de stake de la red. La energía se regenera continuamente durante una ventana de 24 horas.

```
El proveedor stakeea 36.000 TRX
La red asigna ~65.000 energía/día al proveedor
```

**2. Delegate (delegateResource)**

El proveedor dirige una parte de su energía a una dirección de destino. Esta es la operación de delegación real. Después de que se confirme esta transacción, la dirección de destino puede usar la energía delegada como si fuera propia.

```json
{
  "owner_address": "TProviderAddress...",
  "receiver_address": "TBuyerAddress...",
  "resource": "ENERGY",
  "balance": 36000000000,
  "lock": true,
  "lock_period": 3
}
```

Parámetros clave:
- `owner_address`: la dirección del proveedor (el delegador)
- `receiver_address`: la dirección del comprador (el receptor)
- `resource`: ENERGY o BANDWIDTH
- `balance`: la cantidad de TRX (en SUN) respaldando esta delegación
- `lock`: si la delegación está bloqueada por un período mínimo
- `lock_period`: la duración del bloqueo en días (si lock es verdadero)

**3. Undelegate (undelegateResource)**

Cuando el período de delegación expira, el proveedor reclama el recurso mediante la anulación de la delegación. Después de que se confirme esta transacción, la energía deja de fluir hacia la dirección del comprador y regresa al conjunto del proveedor.

```json
{
  "owner_address": "TProviderAddress...",
  "receiver_address": "TBuyerAddress...",
  "resource": "ENERGY",
  "balance": 36000000000
}
```

---

## El Ciclo de Vida de la Delegación

Una delegación completa atraviesa varias fases. Comprender cada fase es crítico para construir sistemas confiables alrededor del alquiler de energía.

### Fase 1: Colocación de Orden

El comprador solicita energía para una dirección específica y duración. Esto ocurre fuera de la cadena: el comprador paga al proveedor (directamente o a través de un agregador como MERX), y el proveedor pone en cola la delegación.

### Fase 2: Delegación En Cadena

El proveedor transmite la transacción `delegateResource`. Esto típicamente se confirma dentro de 3-6 segundos en TRON (un bloque). Una vez confirmado, la energía está inmediatamente disponible en la dirección del comprador.

```
Energía efectiva del comprador = energía_stakeada_propia + toda_energía_delegada
```

Importante: el comprador puede usar energía delegada inmediatamente. No hay período de calentamiento. Tan pronto como se confirme la transacción de delegación, la siguiente llamada de contrato inteligente del comprador consumirá energía delegada.

### Fase 3: Período de Delegación Activa

Durante el período de delegación, el comprador utiliza la energía para sus operaciones. Comportamientos clave durante esta fase:

- **Regeneración**: la energía delegada se regenera a la misma velocidad que la energía stakeada propia, de forma continua durante 24 horas.
- **Prioridad de consumo**: cuando un comprador tiene tanto energía stakeada propia como delegada, TRON no distingue entre ellas. Toda la energía se agrupa.
- **Múltiples delegaciones**: un comprador puede recibir delegaciones de múltiples proveedores simultáneamente. Los montos de energía son aditivos.

### Fase 4: Vencimiento y Reclamación

Cuando el período de bloqueo expira, el proveedor puede ejecutar la transacción `undelegateResource` para reclamar sus recursos. Esto no es automático: el proveedor debe llamar activamente a esta función.

```
Cronología:
  T+0:   delegateResource confirmado (energía activa)
  T+3d:  El período de bloqueo expira
  T+3d+: El proveedor llama a undelegateResource (energía eliminada)
```

Si el proveedor no anula la delegación, la energía continúa fluyendo hacia el comprador indefinidamente. Los proveedores están incentivados a reclamar prontamente porque el TRX stakeado respaldando la delegación no puede ser utilizado para otros clientes hasta que se anule la delegación.

### Fase 5: Postanulación

Después de que se confirme la anulación de la delegación, el TRX del proveedor regresa a su conjunto "indelegable". Luego pueden delegarlo al siguiente cliente. La energía disponible del comprador disminuye por la cantidad no delegada.

---

## Mecánica del Período de Bloqueo

El parámetro `lock` y `lock_period` en la transacción de delegación merecen atención especial.

### Delegaciones Bloqueadas

Cuando `lock: true`, el proveedor se compromete a mantener la delegación durante al menos `lock_period` días. Durante este tiempo, el proveedor no puede anular la delegación. Esto da al comprador una garantía de disponibilidad de recursos.

```
lock_period = 3 días

Día 0: Delegación creada, bloqueo comienza
Día 1: El proveedor NO PUEDE anular la delegación
Día 2: El proveedor NO PUEDE anular la delegación
Día 3: El bloqueo expira, el proveedor PUEDE anular la delegación
```

### Delegaciones Desbloqueadas

Cuando `lock: false`, el proveedor puede anular la delegación en cualquier momento, incluso inmediatamente después de delegar. Esto es arriesgado para los compradores porque el proveedor podría reclamar la energía antes de que el comprador la haya usado.

En la práctica, los proveedores reputados siempre usan delegaciones bloqueadas. MERX verifica que todas las delegaciones estén correctamente bloqueadas durante la duración comprada.

### Granularidad del Período de Bloqueo

El período de bloqueo de TRON se especifica en días (bloques técnicamente, pero el protocolo se asigna a períodos de aproximadamente 24 horas). El bloqueo práctico mínimo es de 1 día. Duraciones comunes en el mercado:

| Período de Bloqueo | Caso de Uso Típico |
|------------------|------------------|
| 0 (desbloqueado) | No recomendado |
| 1 día | Corto plazo / pruebas |
| 3 días | Alquiler estándar |
| 7 días | Operaciones semanales |
| 14 días | Cobertura quincenal |
| 30 días | Contratos mensuales |

---

## Verificación de Entrega de Energía

¿Cómo sabes que la delegación realmente sucedió? Hay varios métodos de verificación.

### Verificación En Cadena

Consultar directamente la red TRON:

```typescript
// Usando TronWeb
const accountResources = await tronWeb.trx.getAccountResources(buyerAddress);
console.log('Límite total de energía:', accountResources.EnergyLimit);
console.log('Energía usada:', accountResources.EnergyUsed);
console.log('Disponible:', accountResources.EnergyLimit - accountResources.EnergyUsed);
```

El campo `EnergyLimit` refleja la energía total de todas las fuentes: stakeada propia y delegada. Un aumento en `EnergyLimit` después de que se confirme una delegación verifica la entrega.

### Búsqueda de Registro de Delegación

TRON proporciona APIs para consultar registros de delegación para una dirección:

```typescript
// Obtener todas las delegaciones recibidas por una dirección
const delegations = await tronWeb.trx.getDelegatedResourceV2(
  buyerAddress,
  providerAddress
);
```

Esto devuelve los montos específicos de delegación y períodos de bloqueo de un proveedor dado al comprador.

### Verificación de MERX

MERX realiza verificación automatizada después de cada orden:

1. Se coloca y paga la orden.
2. El proveedor ejecuta la transacción de delegación.
3. MERX monitorea la blockchain para la confirmación de la transacción de delegación.
4. MERX consulta los recursos de la cuenta del comprador para verificar el aumento de energía.
5. Si la verificación falla (energía no recibida dentro del tiempo límite), la orden se marca y al comprador se le reembolsa.

```typescript
import { MerxClient } from 'merx-sdk';

const client = new MerxClient({ apiKey: 'your-key' });

// Colocar orden
const order = await client.createOrder({
  energy: 65000,
  targetAddress: 'TBuyerAddress...',
  duration: '1d'
});

// Verificar estado de la orden (incluye verificación de delegación)
const status = await client.getOrder(order.id);
console.log(status.delegationStatus);
// 'pending' | 'confirmed' | 'verified' | 'failed'
```

---

## Delegación de Múltiples Proveedores

Una única dirección puede recibir delegaciones de múltiples proveedores simultáneamente. Así es como los agregadores como MERX pueden dividir órdenes grandes entre múltiples proveedores.

### Cómo Funciona

```
Dirección del comprador: TBuyer123...

Delegación 1: Proveedor A -> 200.000 energía (65.000 x 3)
Delegación 2: Proveedor B -> 130.000 energía (65.000 x 2)
Delegación 3: Proveedor C ->  65.000 energía (65.000 x 1)

Energía delegada total: 395.000/día
Más energía stakeada propia del comprador: 0
Energía total disponible: 395.000/día
```

Todas las delegaciones son independientes. El proveedor A puede anular la delegación sin afectar la delegación del proveedor B o C. La energía total del comprador es simplemente la suma de todas las delegaciones activas más su energía stakeada propia.

### Por Qué Esto Es Importante para la Agregación

Cuando un comprador ordena 500.000 energía a través de MERX, es posible que ningún proveedor individual tenga esa cantidad disponible. MERX puede dividir la orden:

```
Orden: 500.000 energía para TBuyer123...

Enrutamiento:
  Proveedor A: 200.000 energía a 82 SUN/unidad  = 16.400.000 SUN
  Proveedor B: 180.000 energía a 85 SUN/unidad  = 15.300.000 SUN
  Proveedor C: 120.000 energía a 88 SUN/unidad  = 10.560.000 SUN

Costo total: 42.260.000 SUN
Tasa efectiva: 84,52 SUN/unidad
```

El comprador ve una única orden con una tasa combinada. Detrás de escenas, tres transacciones de delegación separadas se ejecutan en cadena.

---

## Cómo MERX Gestiona el Ciclo de Vida de la Delegación

MERX maneja el ciclo de vida completo de cada delegación, desde la orden hasta el vencimiento, entre todos los proveedores integrados.

### Ejecución de Orden

1. **Verificación de precio**: Sondear todos los proveedores por el precio actual más favorable.
2. **Enrutamiento**: Seleccionar el(los) proveedor(es) más económico(s) que puedan llenar la orden.
3. **Ejecución**: Enviar orden al(os) proveedor(es) seleccionado(s) a través de su API.
4. **Monitoreo**: Observar la transacción de delegación en cadena.
5. **Verificación**: Confirmar que la energía llegó a la dirección de destino.
6. **Notificación**: Informar al comprador a través de webhook o WebSocket.

### Durante el Período Activo

- **Monitoreo de salud**: Verificar periódicamente que las delegaciones sigan activas.
- **Seguimiento de recursos**: Monitorear el consumo de energía del comprador para detectar problemas.
- **Alertas**: Notificar al comprador si la energía se está agotando o si los patrones de uso sugieren que necesita más.

### En el Vencimiento

- **Notificación de cuenta atrás**: Alertar al comprador antes de que venza la delegación.
- **Opción de renovación**: Ofrecer renovación automática al precio actual más favorable.
- **Verificación post-vencimiento**: Confirmar que la energía fue debidamente reclamada por el proveedor.

### Manejo de Errores

- **Fallo de delegación**: Si el proveedor no logra delegar dentro del período esperado, MERX enruta automáticamente al siguiente proveedor más económico.
- **Llenado parcial**: Si un proveedor solo puede llenar parte de una orden, MERX llena el resto de otros proveedores.
- **Tiempo de inactividad del proveedor**: Si un proveedor es inaccesible, sus precios se eliminan del libro de órdenes y las órdenes se enrutan a proveedores disponibles.

---

## Casos Especiales Comunes

### Energía Usada Antes de que Venza la Delegación

El comprador no está obligado a "devolver" la energía no utilizada. Cuando vence la delegación y el proveedor anula la delegación, la energía simplemente deja de estar disponible. No hay nada que devolver.

### Proveedor Delega Más de lo Comprado

Ocasionalmente, un proveedor puede delegar más energía de la que el comprador pagó (debido a diferencias en la contabilidad interna). El comprador se beneficia de la energía extra durante el período de delegación. MERX rastrea los montos exactos ordenados y entregados.

### Múltiples Órdenes para la Misma Dirección

Si un comprador coloca múltiples órdenes para la misma dirección de destino en diferentes momentos, se acumulan. Cada delegación es independiente y vence según su propio período de bloqueo.

### Dirección No Existe

TRON permite delegación a cualquier dirección válida, incluso si nunca ha sido activada. La energía estará disponible cuando se active la dirección. Sin embargo, la mayoría de proveedores y MERX validan que la dirección de destino exista y esté activada antes de procesar órdenes.

---

## Conclusión

La delegación de energía es la base de la economía de recursos de TRON. El mecanismo de Stake 2.0 transforma la energía de un recurso no transferible en una mercancía negociable, permitiendo todo el ecosistema de proveedores. Comprender cómo funcionan las delegaciones (la mecánica del bloqueo, el proceso de verificación, la composición de múltiples proveedores) es esencial para cualquiera que construya sistemas de producción en TRON.

MERX abstrae la complejidad de gestionar delegaciones entre múltiples proveedores en una única llamada a API, pero conocer qué ocurre bajo el capó ayuda a tomar mejores decisiones sobre duración, volumen y selección de proveedores.

Explora la API de MERX y comienza a gestionar delegaciones programáticamente en [https://merx.exchange/docs](https://merx.exchange/docs).

---

*Este artículo forma parte de la serie de conocimiento de MERX sobre infraestructura de TRON. MERX es el primer intercambio de recursos blockchain. GitHub: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js).*


## Pruébalo Ahora con IA

Añade MERX a Claude Desktop o cualquier cliente compatible con MCP: sin instalación, sin necesidad de clave de API para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregunta a tu agente de IA: "¿Cuál es la energía de TRON más barata en este momento?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)