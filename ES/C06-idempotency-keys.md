# Claves de Idempotencia: Reintentos Seguros para Órdenes de Energía en TRON

Las solicitudes de red fallan. Las conexiones se agotan. Los equilibradores de carga pierden paquetes. Los clientes móviles pierden señal a mitad de la solicitud. En cualquier sistema distribuido, la pregunta no es si habrá fallos, sino cómo los manejarás cuando ocurran.

Para la mayoría de las llamadas a API, la respuesta es directa: reintenta la solicitud. Pero cuando la solicitud implica dinero—crear una orden que débita tu saldo, o iniciar un retiro que mueve fondos en cadena—un reintento ingenuo puede ser catastrófico. Envías la solicitud, el servidor la procesa, pero la respuesta nunca te llega. Reintentas. El servidor la procesa nuevamente. Acabas de pagar dos veces por la misma orden, o peor, desencadenaste dos retiros en cadena.

Este es el problema que las claves de idempotencia resuelven. MERX implementa idempotencia en cada endpoint financiero, haciendo seguro reintentar cualquier solicitud sin riesgo de ejecución duplicada.

## Qué Significa la Idempotencia en la Práctica

Una operación es idempotente si realizarla múltiples veces produce el mismo resultado que realizarla una sola vez. HTTP GET es naturalmente idempotente—obtener la misma URL diez veces devuelve los mismos datos. HTTP DELETE es idempotente por convención—eliminar un recurso ya eliminado es una operación sin efecto.

HTTP POST no es idempotente. Cada POST a `/api/v1/orders` crea una nueva orden. Envíalo tres veces, obtén tres órdenes, paga tres veces.

Las claves de idempotencia hacen que las solicitudes POST se comporten de manera idempotente. El cliente genera un identificador único y lo envía con la solicitud. El servidor usa esta clave para detectar duplicados. Si la misma clave llega nuevamente, el servidor devuelve el resultado de la primera ejecución en lugar de procesar la solicitud nuevamente.

La distinción es importante: el servidor no simplemente rechaza duplicados. Devuelve la respuesta original. Desde la perspectiva del cliente, el reintento se comporta exactamente como un primer intento exitoso. Esto es crítico para la automatización—tu código no necesita distinguir entre "primera llamada exitosa" y "reintento exitoso".

## Por Qué Esto Importa para el Comercio de Energía en TRON

Las órdenes de energía en TRON involucran dinero real moviéndose a través de sistemas reales. Cuando creas una orden a través de MERX, varias cosas suceden en secuencia:

1. El saldo de tu cuenta se débita.
2. La orden se enruta al proveedor más económico disponible.
3. El proveedor delega energía en cadena a tu dirección de destino.
4. El estado de la orden se actualiza y los webhooks se disparan.

Si la conexión se cae después del paso 1 pero antes de recibir confirmación, no tienes forma de saber si la orden fue creada. Sin idempotencia, tus opciones son malas:

- **No reintentar.** Podrías tener una orden pendiente que no conoces, o podrías haber perdido una solicitud que nunca llegó al servidor. Tienes que consultar la lista de órdenes para descubrirlo, añadiendo complejidad.
- **Reintentar ciegamente.** Si la primera solicitud pasó, ahora tienes dos órdenes y dos débitos de saldo. Para un sistema automatizado procesando cientos de órdenes por día, esto se suma rápidamente.
- **Reintentar con clave de idempotencia.** El servidor reconoce el duplicado, devuelve la orden existente, y tu código continúa normalmente. Sin gasto duplicado. Sin reconciliación manual.

La misma lógica se aplica a los retiros. Una solicitud de retiro duplicada podría mover fondos en cadena dos veces. Con claves de idempotencia, el reintento devuelve el registro de retiro existente.

## Cómo MERX Implementa la Idempotencia

MERX soporta el encabezado `Idempotency-Key` en dos endpoints:

- `POST /api/v1/orders` - creación de órdenes de energía y bandwidth
- `POST /api/v1/withdraw` - retiros de saldo

La implementación sigue un ciclo de vida directo.

### Formato de Clave y Generación

La clave de idempotencia es una cadena generada por el cliente, de hasta 64 caracteres. MERX no impone un formato específico, pero la mejor práctica es usar UUIDs (v4) o identificadores estructurados que codifiquen contexto:

```
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

O con contexto de negocio:

```
Idempotency-Key: order-user42-2026-03-30-batch7-item3
```

La clave debe ser única por operación distinta. Reutilizar una clave con parámetros de solicitud diferentes (cantidad diferente, dirección de destino diferente) devuelve un error en lugar de ignorar silenciosamente los nuevos parámetros.

### Comportamiento del Lado del Servidor

Cuando MERX recibe una solicitud con un encabezado `Idempotency-Key`, se ejecuta la siguiente lógica:

1. **Primera solicitud con esta clave.** El servidor procesa la solicitud normalmente—valida parámetros, débita saldo, crea la orden. Almacena la clave, el hash de la solicitud, y la respuesta. Devuelve la respuesta con HTTP 201.

2. **Solicitud duplicada con la misma clave y los mismos parámetros.** El servidor omite todo procesamiento y devuelve la respuesta almacenada de la primera ejecución. La respuesta es idéntica, incluyendo el mismo ID de orden, estado, y marcas de tiempo. Devuelve HTTP 200 (no 201) para que el cliente pueda distinguir si es necesario.

3. **Solicitud duplicada con la misma clave pero parámetros diferentes.** El servidor devuelve HTTP 409 Conflict. Esto previene un error sutil donde una colisión de clave causa que una orden no relacionada sea devuelta.

4. **Solicitud mientras la primera aún se está procesando.** El servidor devuelve HTTP 409 con un mensaje indicando que la solicitud original aún está en progreso. Esto maneja la condición de carrera donde un reintento llega antes de que la primera solicitud termine.

### Expiración de Clave

Las claves de idempotencia se almacenan por 24 horas. Después de eso, la misma clave puede reutilizarse para una nueva solicitud. En la práctica, los reintentos suceden dentro de segundos o minutos, no días. La ventana de 24 horas es lo suficientemente generosa para cubrir cualquier escenario de reintento realista mientras se previene el crecimiento de almacenamiento ilimitado.

## Ejemplos de Código

### SDK de JavaScript - Creación de Orden Segura para Reintentos

El SDK de JavaScript de MERX maneja automáticamente las claves de idempotencia cuando pasas la opción `idempotencyKey`. Aquí hay un ejemplo completo con lógica de reintento:

```javascript
import { MerxClient } from 'merx-sdk';
import { randomUUID } from 'crypto';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
  baseUrl: 'https://merx.exchange/api/v1',
});

async function createOrderSafe(params, maxRetries = 3) {
  const idempotencyKey = randomUUID();

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const order = await merx.orders.create(
        {
          energy_amount: params.energyAmount,
          target_address: params.targetAddress,
          duration_hours: params.durationHours,
        },
        {
          idempotencyKey,
        }
      );

      console.log(`Order created: ${order.id}, status: ${order.status}`);
      return order;
    } catch (err) {
      if (err.status === 409 && err.code === 'IDEMPOTENCY_CONFLICT') {
        // Same key with different params - this is a bug in our code
        throw new Error('Idempotency key conflict - parameters mismatch');
      }

      if (attempt === maxRetries) {
        throw err;
      }

      const delay = Math.min(1000 * Math.pow(2, attempt - 1), 10000);
      console.log(`Attempt ${attempt} failed, retrying in ${delay}ms...`);
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }
}

// Usage
const order = await createOrderSafe({
  energyAmount: 65000,
  targetAddress: 'TTargetAddressHere',
  durationHours: 1,
});
```

Puntos clave en esta implementación:

- La clave de idempotencia se genera una sola vez, antes del bucle de reintento. Cada reintento envía la misma clave.
- El backoff exponencial previene sobrecargar el servidor durante fallos transitorios.
- El conflicto 409 en desajuste de parámetros se trata como un error no reintentalble porque indica un error lógico, no un problema de red.

### SDK de Python - Retiro Seguro para Reintentos

El SDK de Python sigue el mismo patrón. Aquí hay un ejemplo para retiros:

```python
import uuid
import time
from merx_sdk import MerxClient, MerxAPIError

client = MerxClient(
    api_key="your_api_key",
    base_url="https://merx.exchange/api/v1",
)


def withdraw_safe(address: str, amount_sun: int, max_retries: int = 3):
    idempotency_key = str(uuid.uuid4())

    for attempt in range(1, max_retries + 1):
        try:
            result = client.withdraw(
                address=address,
                amount=amount_sun,
                idempotency_key=idempotency_key,
            )
            print(f"Withdrawal initiated: {result['id']}")
            return result

        except MerxAPIError as e:
            if e.status_code == 409:
                raise ValueError(
                    "Idempotency conflict - check parameters"
                ) from e

            if attempt == max_retries:
                raise

            delay = min(2 ** (attempt - 1), 10)
            print(f"Attempt {attempt} failed: {e}. Retrying in {delay}s...")
            time.sleep(delay)

    raise RuntimeError("All retry attempts exhausted")


# Usage
result = withdraw_safe(
    address="TWithdrawalAddressHere",
    amount_sun=100_000_000,  # 100 TRX
)
```

### HTTP Raw - Llamada Directa a API con curl

Si no estás usando un SDK, el encabezado `Idempotency-Key` es un encabezado HTTP estándar:

```bash
IDEMPOTENCY_KEY=$(uuidgen)

curl -X POST https://merx.exchange/api/v1/orders \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $IDEMPOTENCY_KEY" \
  -d '{
    "energy_amount": 65000,
    "target_address": "TTargetAddressHere",
    "duration_hours": 1
  }'
```

Reintenta el mismo comando curl (con el mismo valor `IDEMPOTENCY_KEY`) y obtendrás la orden original en lugar de una nueva.

## Mejores Prácticas para Sistemas en Producción

### Genera Claves en el Nivel Correcto

La clave de idempotencia debe representar la intención de negocio, no la solicitud HTTP. Si un usuario hace clic en "Comprar Energía" una sola vez, esa es una operación de negocio con una sola clave—independientemente de cuántos reintentos HTTP sean necesarios.

No generes una nueva clave en cada reintento. Eso derrota completamente el propósito.

### Almacena Claves Antes de Enviar

En un sistema que procesa órdenes en una cola, escribe la clave de idempotencia en tu base de datos antes de hacer la llamada a API. Si tu proceso se bloquea y se reinicia, recupera la misma clave de la base de datos e intenta nuevamente de manera segura.

```javascript
// Write the intent to your database first
const intent = await db.orderIntents.create({
  idempotencyKey: randomUUID(),
  energyAmount: 65000,
  targetAddress: 'TTargetAddressHere',
  status: 'pending',
});

// Now make the API call with the stored key
const order = await merx.orders.create(
  { energy_amount: intent.energyAmount, target_address: intent.targetAddress },
  { idempotencyKey: intent.idempotencyKey }
);

// Update the intent with the result
await db.orderIntents.update(intent.id, {
  status: 'completed',
  orderId: order.id,
});
```

### Maneja Todos los Códigos de Respuesta

Tu lógica de reintento debe manejar estos casos:

| Código HTTP | Significado | Acción |
|------------|-------------|--------|
| 201 | Creación exitosa por primera vez | Almacena resultado, continúa |
| 200 | Clave duplicada, mismos parámetros | Trata como éxito (mismo resultado) |
| 409 | Conflicto de clave o aún procesando | No reintentar—investigar |
| 429 | Límite de velocidad alcanzado | Reintentar después de retraso |
| 500+ | Error del servidor | Reintentar con backoff |

### Usa Claves Estructuradas para Depuración

Aunque los UUIDs funcionan perfectamente, las claves estructuradas facilitan la depuración:

```
order-{userId}-{date}-{sequenceNumber}
withdraw-{userId}-{timestamp}-{nonce}
```

Cuando investigues un caso de soporte, una clave estructurada te dice inmediatamente quién inició la operación y cuándo.

### Establece Límites de Reintento Razonables

Tres a cinco reintentos con backoff exponencial cubren la gran mayoría de fallos transitorios. Si el servidor genuinamente está caído, reintentar 50 veces no ayudará y solo generará ruido en tus registros.

Un límite sensato: reintentar hasta 3 veces, con retrasos de 1 segundo, 2 segundos y 4 segundos. Si los tres fallan, escala el error a un sistema de monitoreo en lugar de reintentar indefinidamente.

## Errores Comunes

**Generar una nueva clave en cada reintento.** Este es el error más común. Si cada reintento tiene una nueva clave, el servidor trata cada uno como una solicitud única. Obtienes órdenes duplicadas.

**Compartir claves entre diferentes operaciones.** Cada operación de negocio distinta necesita su propia clave. Si usas la misma clave para dos órdenes diferentes (cantidades diferentes, direcciones diferentes), la segunda fallará con un 409.

**No manejar la distinción 200 vs 201.** Aunque ambos indican éxito, el código de estado te dice si esta fue la primera ejecución o una repetición. Esto puede ser útil para registros y métricas—saber con qué frecuencia los reintentos resultan en duplicados te dice algo sobre tu confiabilidad de red.

**Ignorar idempotencia para webhooks.** MERX envía webhooks en cambios de estado de orden. Tu manejador de webhook también debe ser idempotente—si recibes el mismo evento `order.filled` dos veces, procesarlo dos veces no debe causar problemas. Usa el ID de orden como clave de idempotencia natural para tu propio procesamiento.

## Conclusión

Las claves de idempotencia son una pequeña adición a tus llamadas a API—un encabezado, un UUID—pero eliminan una clase completa de errores financieros. Para cualquier sistema que cree órdenes de energía en TRON o inicie retiros programáticamente, no son opcionales. Son la diferencia entre un sistema que maneja fallos de red graciosamente y uno que crea tiquetes de soporte cada vez que cae una conexión.

MERX soporta claves de idempotencia en todos los endpoints financieros. El SDK de JavaScript y el SDK de Python ambos proporcionan soporte integrado. Comienza a usarlas desde el primer día—añadir idempotencia retroactivamente a un sistema en producción que ya ha experimentado órdenes duplicadas es considerablemente más doloroso que construirlo desde el principio.

- Plataforma MERX: [merx.exchange](https://merx.exchange)
- Documentación de API: [merx.exchange/docs](https://merx.exchange/docs)
- SDK de JavaScript: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- SDK de Python: [github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)


## Pruébalo Ahora con AI

Añade MERX a Claude Desktop o cualquier cliente compatible con MCP—sin instalación, sin clave de API necesaria para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregúntale a tu agente AI: "¿Cuál es la energía de TRON más barata en este momento?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)