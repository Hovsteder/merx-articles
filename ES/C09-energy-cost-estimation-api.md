# API de Estimación de Costos de Energía de MERX: Conoce el Costo Antes de Comprar

Cada transacción en TRON consume recursos. Una transferencia de USDT necesita aproximadamente 65,000 unidades de energía. Una aprobación de contrato inteligente quema alrededor de 15,000. Una interacción compleja de DeFi podría necesitar 200,000 o más. Los números exactos dependen del contrato, la operación y los parámetros actuales de la red.

Si estás construyendo un producto que maneja transacciones de TRON en nombre de los usuarios - una billetera, un procesador de pagos, un bot de trading - necesitas saber el costo antes de comprometerte. ¿Cuánta energía se requiere? ¿Cuánto costará alquilarla versus quemarla? ¿Cuánto ahorra el usuario?

MERX proporciona dos endpoints que responden estas preguntas con precisión: `POST /api/v1/estimate` para la estimación general de costos y `GET /api/v1/orders/preview` para la vista previa de costos específica del pedido. Juntos, te permiten mostrar a los usuarios costos exactos y ahorros antes de que un solo SUN cambie de manos.

## El Endpoint de Estimación

`POST /api/v1/estimate` calcula los requisitos de energía y ancho de banda para un tipo de transacción determinado y devuelve la comparación de costos entre alquilar energía a través de MERX y quemar TRX a nivel de protocolo.

### Solicitud Básica

```bash
curl -X POST https://merx.exchange/api/v1/estimate \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "trc20_transfer",
    "target_address": "TTargetAddressHere"
  }'
```

### Formato de Respuesta

```json
{
  "operation": "trc20_transfer",
  "target_address": "TTargetAddressHere",
  "energy_required": 64895,
  "bandwidth_required": 345,
  "costs": {
    "burn": {
      "trx_cost": 27370000,
      "trx_cost_readable": "27.37 TRX",
      "usd_equivalent": 2.19
    },
    "rental": {
      "best_provider": "sohu",
      "price_per_unit_sun": 22,
      "total_cost_sun": 1427690,
      "total_cost_trx": "1.43 TRX",
      "usd_equivalent": 0.11,
      "duration_hours": 1
    },
    "savings": {
      "trx_saved": "25.94 TRX",
      "percent": 94.8,
      "usd_saved": 2.08
    }
  },
  "address_resources": {
    "current_energy": 0,
    "current_bandwidth": 1200,
    "energy_deficit": 64895,
    "bandwidth_deficit": 0
  },
  "timestamp": "2026-03-30T10:00:00Z"
}
```

La respuesta te dice todo lo que necesitas saber para tomar una decisión sobre el costo de una transacción:

- **energy_required** - cuánta energía necesita la operación. Para una transferencia TRC-20 estándar, esto es aproximadamente 65,000 unidades, aunque el número exacto depende del contrato y del estado de la dirección de destino.
- **bandwidth_required** - cuánto ancho de banda consume la transacción. La mayoría de transferencias simples necesitan 300-400 puntos de ancho de banda. Las cuentas nuevas (activar una dirección por primera vez) necesitan más.
- **costs.burn** - qué cuesta si el usuario no hace nada y deja que el protocolo queme TRX. Este es el costo "predeterminado".
- **costs.rental** - la opción de alquiler más barata disponible a través de MERX. Incluye el proveedor, el precio por unidad, el costo total y la duración del alquiler.
- **costs.savings** - la diferencia entre quemar y alquilar, expresada como TRX ahorrado en términos absolutos, porcentaje ahorrado y equivalente en USD.
- **address_resources** - energía y ancho de banda actuales en la dirección de destino. Si la dirección ya tiene energía por stake o una delegación anterior, el déficit se reduce correspondientemente.

### Operaciones Soportadas

El campo `operation` acepta varios tipos predefinidos que cubren las transacciones de TRON más comunes:

#### trc20_transfer

La transferencia estándar de token TRC-20. Esta es la operación más común - enviar USDT, USDC, o cualquier otro token TRC-20 de una dirección a otra.

```json
{
  "operation": "trc20_transfer",
  "target_address": "TSenderAddressHere"
}
```

Energía requerida: aproximadamente 64,895 unidades para USDT en una transferencia estándar (dirección ya activada con saldo de USDT). Las primeras transferencias a una dirección que nunca ha tenido el token cuestan más - hasta 100,000 de energía - porque el contrato debe crear un nuevo espacio de almacenamiento.

#### trc20_approve

La transacción de aprobación TRC-20, utilizada para permitir que un contrato inteligente gaste tokens en tu nombre. Se requiere antes de interactuar con contratos de DEX, protocolos de préstamo y la mayoría de aplicaciones DeFi.

```json
{
  "operation": "trc20_approve",
  "target_address": "TApproverAddressHere"
}
```

Energía requerida: aproximadamente 15,000-18,000 unidades, significativamente menos que una transferencia.

#### trx_transfer

Una transferencia simple de TRX. Estas consumen principalmente ancho de banda, no energía, pero el endpoint de estimación las maneja por completitud.

```json
{
  "operation": "trx_transfer",
  "target_address": "TSenderAddressHere"
}
```

Energía requerida: 0 (las transferencias de TRX no consumen energía). Ancho de banda requerido: aproximadamente 270 bytes.

#### custom

Para llamadas de contrato inteligente que no caben en los tipos predefinidos. Proporciona la dirección del contrato y el selector de función, y MERX estima el consumo de energía simulando la llamada.

```json
{
  "operation": "custom",
  "target_address": "TCallerAddressHere",
  "contract_address": "TContractAddressHere",
  "function_selector": "stake(uint256)",
  "parameters": [
    {
      "type": "uint256",
      "value": "1000000"
    }
  ]
}
```

La operación custom ejecuta una simulación contra la red TRON para determinar el consumo real de energía y ancho de banda. Este es el método más preciso para transacciones no estándar pero tarda un poco más (200-500ms versus 50ms para tipos predefinidos).

## El Endpoint de Vista Previa del Pedido

Mientras que `POST /estimate` te proporciona requisitos de recursos y comparaciones de costos, `GET /api/v1/orders/preview` te muestra exactamente cómo se vería un pedido de MERX - incluyendo qué proveedor sería seleccionado, el débito exacto de tu balance y cualquier tarifa aplicable.

### Solicitud

```bash
curl "https://merx.exchange/api/v1/orders/preview?energy_amount=65000&target_address=TTargetAddressHere&duration_hours=1" \
  -H "X-API-Key: your_api_key"
```

### Respuesta

```json
{
  "preview": {
    "energy_amount": 65000,
    "target_address": "TTargetAddressHere",
    "duration_hours": 1,
    "provider": "sohu",
    "price_per_unit_sun": 22,
    "subtotal_sun": 1430000,
    "fee_sun": 14300,
    "total_sun": 1444300,
    "total_trx": "1.44 TRX",
    "your_balance_sun": 50000000,
    "balance_after_sun": 48555700,
    "estimated_fill_time_seconds": 5
  },
  "alternatives": [
    {
      "provider": "catfee",
      "price_per_unit_sun": 25,
      "total_sun": 1641250,
      "total_trx": "1.64 TRX"
    },
    {
      "provider": "netts",
      "price_per_unit_sun": 28,
      "total_sun": 1838200,
      "total_trx": "1.84 TRX"
    }
  ]
}
```

La vista previa incluye:

- **provider** - a qué proveedor MERX dirigiría el pedido a los precios actuales.
- **subtotal_sun** - el costo bruto de la energía a la tasa del proveedor.
- **fee_sun** - la tarifa de la plataforma MERX.
- **total_sun** - la cantidad total que sería debitada de tu balance.
- **balance_after_sun** - tu balance después del pedido, para que puedas verificar que puedas pagarlo.
- **estimated_fill_time_seconds** - cuánto tiempo típicamente tarda la delegación con este proveedor.
- **alternatives** - otros proveedores que podrían completar el pedido, ordenados por precio. Útil para mostrar a los usuarios sus opciones.

### Diferencia Entre Estimación y Vista Previa

| Característica | POST /estimate | GET /orders/preview |
|---------|---------------|-------------------|
| Propósito | Análisis de costos general | Planificación exacta del pedido |
| Autenticación requerida | Sí | Sí |
| Muestra ahorros vs quemar | Sí | No |
| Muestra tu balance | No | Sí |
| Muestra tarifas de plataforma | No | Sí |
| Muestra alternativas | No | Sí |
| Muestra recursos de dirección | Sí | No |
| Soporta contratos personalizados | Sí | No |

Usa `estimate` cuando necesites decirle a un usuario "esta transacción costará X si quemas TRX, o Y si alquilas energía - ahorrándote un Z por ciento." Usa `preview` cuando el usuario ha decidido comprar energía y quieres mostrar el pedido exacto antes de que confirme.

## Ejemplos del Mundo Real con Números

### Ejemplo 1: Procesador de Pagos USDT

Ejecutas un procesador de pagos que envía USDT a comerciantes. Antes de cada pago, estimas el costo:

```javascript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
});

async function estimatePayoutCost(senderAddress) {
  const estimate = await merx.estimate({
    operation: 'trc20_transfer',
    target_address: senderAddress,
  });

  console.log(`Energy needed: ${estimate.energy_required}`);
  console.log(`Burn cost: ${estimate.costs.burn.trx_cost_readable}`);
  console.log(`Rental cost: ${estimate.costs.rental.total_cost_trx}`);
  console.log(`Savings: ${estimate.costs.savings.percent}%`);

  // Decide whether to rent or burn based on savings threshold
  if (estimate.costs.savings.percent > 50) {
    return { method: 'rent', cost: estimate.costs.rental };
  } else {
    return { method: 'burn', cost: estimate.costs.burn };
  }
}
```

Salida típica:

```
Energy needed: 64895
Burn cost: 27.37 TRX
Rental cost: 1.43 TRX
Savings: 94.8%
```

A los precios actuales, alquilar energía casi siempre ahorra más del 90 por ciento en comparación con quemar.

### Ejemplo 2: Pantalla de Costos de Billetera

Construyes una billetera TRON y quieres mostrar al usuario el costo de la transacción antes de que confirme un envío:

```python
from merx_sdk import MerxClient

client = MerxClient(api_key="your_api_key")


def get_transfer_cost(sender_address: str, token: str = "usdt") -> dict:
    """Get the cost to send a TRC-20 token, accounting for existing resources."""

    estimate = client.estimate(
        operation="trc20_transfer",
        target_address=sender_address,
    )

    resources = estimate["address_resources"]
    costs = estimate["costs"]

    # If the address already has enough energy, the transfer is free
    if resources["energy_deficit"] == 0:
        return {
            "cost": "0 TRX",
            "note": "Address has sufficient energy",
        }

    return {
        "without_energy": costs["burn"]["trx_cost_readable"],
        "with_energy": costs["rental"]["total_cost_trx"],
        "savings": f'{costs["savings"]["percent"]}%',
        "energy_deficit": resources["energy_deficit"],
    }
```

La interfaz de usuario de la billetera podría mostrar algo como:

```
Send 100 USDT to TRecipient...

Transaction cost:
  Without energy rental:  27.37 TRX (burned)
  With MERX energy:       1.43 TRX (rented)
  You save:               94.8%

[ Rent Energy and Send ]    [ Send Without Energy ]
```

### Ejemplo 3: Proyección de Costos de Transferencia en Lote

Necesitas enviar USDT a 500 direcciones y quieres saber el costo total de antemano:

```javascript
async function estimateBatchCost(addresses) {
  let totalBurnCost = 0;
  let totalRentalCost = 0;
  let totalEnergy = 0;

  // Estimate for a representative sample (first 10 addresses)
  // Most TRC-20 transfers cost the same energy
  const sampleSize = Math.min(10, addresses.length);
  const sample = addresses.slice(0, sampleSize);

  for (const address of sample) {
    const estimate = await merx.estimate({
      operation: 'trc20_transfer',
      target_address: address,
    });

    totalBurnCost += estimate.costs.burn.trx_cost;
    totalRentalCost += estimate.costs.rental.total_cost_sun;
    totalEnergy += estimate.energy_required;
  }

  // Extrapolate to full batch
  const avgBurnCost = totalBurnCost / sampleSize;
  const avgRentalCost = totalRentalCost / sampleSize;
  const avgEnergy = totalEnergy / sampleSize;

  const projectedBurnTRX = (avgBurnCost * addresses.length) / 1_000_000;
  const projectedRentalTRX = (avgRentalCost * addresses.length) / 1_000_000;
  const projectedEnergy = avgEnergy * addresses.length;

  console.log(`Batch size: ${addresses.length} transfers`);
  console.log(`Total energy needed: ${projectedEnergy.toLocaleString()}`);
  console.log(`Cost without MERX: ${projectedBurnTRX.toFixed(2)} TRX`);
  console.log(`Cost with MERX: ${projectedRentalTRX.toFixed(2)} TRX`);
  console.log(
    `Total savings: ${(projectedBurnTRX - projectedRentalTRX).toFixed(2)} TRX`
  );
}
```

Para 500 transferencias a tasas de mercado actuales:

```
Batch size: 500 transfers
Total energy needed: 32,447,500
Cost without MERX: 13,685.00 TRX
Cost with MERX: 715.00 TRX
Total savings: 12,970.00 TRX
```

A aproximadamente 0.08 USD por TRX, eso es un ahorro de más de 1,000 USD en un solo lote.

### Ejemplo 4: Estimación de Llamada de Contrato Personalizado

Interactúas con un contrato de stake personalizado y necesitas estimar el costo de una llamada `stake()`:

```bash
curl -X POST https://merx.exchange/api/v1/estimate \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "custom",
    "target_address": "TCallerAddressHere",
    "contract_address": "TStakingContractHere",
    "function_selector": "stake(uint256)",
    "parameters": [
      {
        "type": "uint256",
        "value": "50000000"
      }
    ]
  }'
```

La respuesta incluye el consumo de energía simulado, que para un contrato de stake complejo podría ser 120,000 a 200,000 unidades - significativamente más que una transferencia simple.

## Integración de Estimaciones en el Flujo de Pedidos

Los endpoints de estimación y vista previa encajan naturalmente en un flujo de pedidos visible para el usuario:

1. **El usuario inicia una transacción** (por ejemplo, enviar USDT).
2. **Llama POST /estimate** para obtener requisitos de energía y comparación de costos.
3. **Muestra el costo de quemar versus el costo de alquiler** con porcentaje de ahorro.
4. **El usuario elige alquilar energía.**
5. **Llama GET /orders/preview** para mostrar detalles exactos del pedido con tarifas.
6. **El usuario confirma.**
7. **Llama POST /orders** con una clave de idempotencia para crear el pedido.
8. **Sondea o espera webhook** confirmando la delegación de energía.
9. **Ejecuta la transacción original** con la energía delegada.

Los pasos 2-3 son informativos. No se mueve dinero. El usuario ve precios transparentes antes de comprometerse. Esto genera confianza y reduce solicitudes de soporte de usuarios sorprendidos por los costos.

## Manejo de Errores

Ambos endpoints devuelven respuestas de error estándar de MERX:

```json
{
  "error": {
    "code": "INVALID_ADDRESS",
    "message": "Target address is not a valid TRON address",
    "details": {
      "address": "invalid_address_here"
    }
  }
}
```

Errores comunes:

| Código | Causa | Resolución |
|------|-------|-----------|
| `INVALID_ADDRESS` | La dirección de destino no pasa la validación de dirección TRON | Verifica el formato de dirección (prefijo T-, base58) |
| `INVALID_OPERATION` | Tipo de operación no reconocido | Usa uno de: trc20_transfer, trc20_approve, trx_transfer, custom |
| `SIMULATION_FAILED` | La simulación de llamada de contrato personalizado falló | Verifica la dirección del contrato y el selector de función |
| `NO_PROVIDERS` | No hay proveedores disponibles para la cantidad de energía requerida | Intenta más tarde o reduce la cantidad de energía |
| `INSUFFICIENT_BALANCE` | Balance demasiado bajo para el pedido vista previa (solo vista previa) | Deposita más TRX en tu cuenta de MERX |

## Consideraciones de Caché

Los resultados de estimación son válidos para una ventana corta. Los precios de energía cambian cuando los proveedores ajustan las tarifas, y los parámetros de la red pueden cambiar el costo de quemar. Para la mayoría de casos de uso:

- **Cachea estimaciones durante 30-60 segundos** si estás mostrando costos en una interfaz de usuario. Los precios no cambian más rápido que MERX sondea a los proveedores (cada 30 segundos).
- **Siempre obtén una vista previa nueva** inmediatamente antes de la creación del pedido. La vista previa refleja el costo exacto en ese momento.
- **No cachees simulaciones de contratos personalizados** si el estado del contrato cambia frecuentemente. El resultado de la simulación depende del estado en cadena en el momento de la ejecución.

## Conclusión

Los endpoints de estimación y vista previa eliminan las conjeturas de la compra de energía en TRON. En lugar de alquilar una cantidad fija de energía y esperar que sea suficiente, sabes exactamente cuánto necesitas. En lugar de aceptar cualquier precio disponible, ves cada opción clasificada por costo.

Para desarrolladores que construyen productos en TRON, estos endpoints transforman la energía de un costo impredecible a un elemento de línea conocido y optimizable. Verifica el costo, muestra los ahorros, deja que el usuario decida, luego ejecuta con confianza.

- Plataforma MERX: [merx.exchange](https://merx.exchange)
- Documentación completa de API: [merx.exchange/docs](https://merx.exchange/docs)
- SDK de JavaScript: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- SDK de Python: [github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)


## Pruébalo Ahora con IA

Agrega MERX a Claude Desktop o cualquier cliente compatible con MCP -- cero instalación, sin clave API necesaria para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregúntale a tu agente de IA: "¿Cuál es la energía de TRON más barata en este momento?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)