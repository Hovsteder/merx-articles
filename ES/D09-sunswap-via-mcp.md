# SunSwap a través de MCP: Agentes de IA operando en un DEX

## El Puente Faltante

Los intercambios descentralizados han sido accesibles a través de interfaces web, billeteras móviles y SDK programáticos durante años. Lo que no ha sido accesible es el lenguaje natural. Un agente de IA no puede hacer clic en un botón de intercambio. No puede navegar una interfaz web. Y aunque un agente técnicamente puede llamar a un contrato inteligente mediante la construcción de transacciones sin procesar, esto requiere que el agente entienda la codificación ABI, las direcciones del contrato enrutador, la mecánica de los fondos de liquidez y el modelo de recursos de TRON.

MERX cierra esta brecha. A través del servidor MCP, un agente de IA puede solicitar una cotización de intercambio, entender el resultado esperado y los costos, y ejecutar la operación, todo mediante llamadas de herramientas estructuradas que abstraen la complejidad a nivel de protocolo mientras preservan total transparencia en lo que está sucediendo en la cadena.

Este artículo cubre la integración completa de SunSwap: obtener cotizaciones, ejecutar intercambios, manejar aprobaciones, simular costos de energía y entender los números reales de operaciones en la red principal.

## Obtener una Cotización de Intercambio

El primer paso en cualquier operación es entender qué obtendrás. La herramienta `get_swap_quote` consulta el contrato enrutador de SunSwap V2 para calcular el resultado esperado para una entrada dada:

```
Tool: get_swap_quote
Input: {
  "from_token": "TRX",
  "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "amount": "100000",
  "from_address": "TYourAddress..."
}

Response:
{
  "input_token": "TRX",
  "input_amount": "0.1",
  "input_amount_sun": 100000,
  "output_token": "USDT",
  "output_amount": "0.032847",
  "output_amount_raw": "32847",
  "minimum_received": "0.032519",
  "price_impact": "0.001%",
  "route": ["WTRX", "USDT"],
  "router": "TKzxdSv2FZKQrEkFPRQi4qeCokAFkrntVM",
  "energy_required": 223354,
  "energy_cost_estimate_trx": 11.76,
  "liquidity": {
    "pool_address": "TPool...",
    "trx_reserve": "45,234,891.23",
    "usdt_reserve": "14,832,445.67"
  }
}
```

Varios detalles importantes en esta respuesta:

### Impacto de Precio

El campo `price_impact` muestra cuánto tu operación mueve el precio del fondo. Para el intercambio de 0,1 TRX anterior, el impacto es de 0,001%, negligible. Para operaciones más grandes, este número crece:

```
Intercambio de 0,1 TRX:      0,001% impacto
Intercambio de 1.000 TRX:    0,012% impacto
Intercambio de 100.000 TRX:  1,2% impacto
Intercambio de 1.000.000 TRX: 11,8% impacto
```

El impacto de precio no es una tarifa, es una consecuencia estructural de la mecánica AMM de producto constante. Cuanto mayor sea tu operación en relación con la liquidez del fondo, peor será tu precio de ejecución.

### Mínimo Recibido

El campo `minimum_received` tiene en cuenta la protección contra deslizamiento. Por defecto, MERX calcula una tolerancia de deslizamiento del 1%, lo que significa que el intercambio se revertirá si el resultado es más del 1% por debajo de la cantidad cotizada. Esto protege contra front-running y movimientos rápidos de precios entre la cotización y la ejecución.

### Ruta

El campo `route` muestra la ruta de tokens. Para TRX a USDT, la ruta pasa a través de WTRX (TRX envuelto) porque los pares de SunSwap V2 están entre tokens TRC20, y TRX nativo debe ser envuelto primero. Para intercambios de token a token sin un par directo, la ruta puede incluir tokens intermedios:

```
Token A -> WTRX -> Token B  (ruta de dos saltos)
```

Las rutas de múltiples saltos consumen más energía debido a interacciones adicionales con contratos.

### Energía Requerida

Esta es la estimación exacta de energía de la simulación `triggerConstantContract`. No es una constante codificada, no es un rango, es las unidades de energía precisas que este intercambio específico consumirá con estos parámetros específicos en el estado actual de la cadena de bloques.

## Ejecutar el Intercambio

Una vez que el agente ha revisado la cotización y decidido continuar, la herramienta `execute_swap` maneja el flujo de ejecución completo:

```
Tool: execute_swap
Input: {
  "from_token": "TRX",
  "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "amount": "100000",
  "slippage": 1.0,
  "deadline_minutes": 5
}

Response:
{
  "status": "completed",
  "tx_hash": "9c7b3a2f...",
  "input": {
    "token": "TRX",
    "amount": "0.1"
  },
  "output": {
    "token": "USDT",
    "amount": "0.032847"
  },
  "energy_used": 223354,
  "energy_purchased": 225000,
  "energy_cost_trx": 11.84,
  "block": 58234892,
  "gas_saved_trx": 41.16
}
```

### Qué Sucede Dentro de execute_swap

La herramienta `execute_swap` orquesta el flujo de transacción completo consciente de recursos:

1. **Simular el intercambio** - `triggerConstantContract` con los parámetros exactos del intercambio para obtener el requisito de energía preciso (223.354 en este caso)

2. **Verificar recursos actuales** - Consultar la dirección del remitente para obtener energía disponible y bandwidth

3. **Comprar déficit de energía** - Encontrar el mejor precio del proveedor, hacer el pedido y esperar la confirmación de delegación

4. **Construir la transacción de intercambio** - Construir la llamada al enrutador de SunSwap V2 con el selector de función correcto, parámetros y valor de llamada

5. **Firmar localmente** - La clave privada firma la transacción en la máquina del agente

6. **Difundir** - La transacción firmada se envía a la red TRON

7. **Verificar** - Encuestar la confirmación de transacción y analizar el resultado

El agente llama a una herramienta. Siete pasos se ejecutan detrás de escenas. El agente recibe una sola respuesta con el resultado completo.

## Aprobaciones de Tokens

Al intercambiar tokens TRC20 (en lugar de intercambiar TRX nativo), el enrutador de SunSwap necesita permiso para gastar tokens en tu nombre. Este es el mecanismo estándar de TRC20 `approve`.

MERX maneja esto automáticamente. Antes de ejecutar un intercambio TRC20 a TRC20 o TRC20 a TRX, la herramienta execute_swap verifica si el contrato enrutador tiene una asignación suficiente para el token de entrada:

```
Verificación de aprobación:
  Token: USDT (TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t)
  Gastador: Enrutador SunSwap V2 (TKzxdSv2FZKQrEkFPRQi4qeCokAFkrntVM)
  Asignación actual: 0
  Requerida: 100.000.000 (100 USDT)

  -> Aprobación necesaria
```

Si se necesita aprobación, MERX:

1. Simula la transacción de aprobación para obtener su costo de energía
2. Agrega la energía de aprobación a la energía del intercambio para una compra combinada
3. Ejecuta la transacción de aprobación primero
4. Espera la confirmación
5. Luego ejecuta el intercambio

```
Requisito de energía combinada:
  Aprobación: 46.312 energía
  Intercambio: 223.354 energía
  Total: 269.666 energía

  Comprada: 270.000 energía a 14,20 TRX
```

El agente no necesita saber sobre aprobaciones. Si se necesita una, sucede. Si el token ya está aprobado, el paso se omite.

### Cantidad de Aprobación

Por defecto, MERX aprueba el valor máximo uint256 (aprobación ilimitada). Esto significa que futuros intercambios del mismo token no requerirán otra transacción de aprobación, ahorrando energía y tiempo. Si el agente prefiere aprobaciones exactas (aprobar solo la cantidad específica necesaria), esto puede especificarse:

```
Tool: approve_trc20
Input: {
  "token_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "spender": "TKzxdSv2FZKQrEkFPRQi4qeCokAFkrntVM",
  "amount": "100000000"
}
```

Las aprobaciones exactas son más seguras (limitan la exposición si el contrato enrutador se ve comprometido) pero cuestan energía en cada intercambio.

## Intercambio Real: 0,1 TRX a USDT

Aquí están los detalles completos de un intercambio real en la red principal ejecutado a través del servidor MERX MCP:

### Estado Previo al Intercambio

```
Dirección: TWallet...
Saldo TRX: 523,456789 TRX
Saldo USDT: 0,00 USDT
Energía Disponible: 0
Bandwidth Disponible: 1.420 (libre)
```

### Cotización

```
get_swap_quote:
  Entrada: 0,1 TRX (100.000 SUN)
  Salida: 0,032847 USDT
  Impacto de precio: 0,001%
  Energía requerida: 223.354
  Ruta: WTRX -> USDT
```

### Ejecución

```
execute_swap:
  Energía comprada: 225.000 a 11,84 TRX (proveedor: sohu)
  Delegación confirmada: 4,1 segundos
  Aprobación: No necesaria (TRX -> USDT, TRX nativo no requiere aprobación)
  TX de intercambio difundida: 9c7b3a2f...
  TX de intercambio confirmada: Bloque 58.234.892
```

### Estado Posterior al Intercambio

```
Dirección: TWallet...
Saldo TRX: 511,516789 TRX  (523,456789 - 0,1 intercambio - 11,84 energía)
Saldo USDT: 0,032847 USDT
Energía Disponible: 1.646  (225.000 comprada - 223.354 usada)
Bandwidth Disponible: 1.000 (libre, parcialmente consumido por TX)
```

### Desglose de Costos

```
Compra de energía:       11,84 TRX
Entrada de intercambio:   0,10 TRX
Total gastado:           11,94 TRX

Sin compra de energía (quema de TRX):
  Quema de energía de intercambio: ~53,00 TRX
  Entrada de intercambio:           0,10 TRX
  Total:                           ~53,10 TRX

Ahorros:                 41,16 TRX (77,5%)
```

## Estimación de Impacto de Precio

Para agentes que realizan operaciones más grandes, entender el impacto de precio es crítico. La herramienta `get_swap_quote` proporciona esta estimación, pero entender cómo se calcula ayuda a los agentes a tomar mejores decisiones.

SunSwap V2 usa la fórmula de producto constante: `x * y = k`, donde x e y son las reservas de tokens en el fondo. Cuando cambias dx de token X por dy de token Y:

```
dy = y * dx / (x + dx)
```

El precio efectivo que recibes (dy/dx) siempre es peor que el precio spot (y/x) porque tu operación aumenta la oferta de X y disminuye la oferta de Y.

Para un fondo con 45 millones de TRX y 14,8 millones de USDT:

```
Precio spot: 14.800.000 / 45.000.000 = 0,328889 USDT/TRX

Intercambio de 100 TRX:
  Salida: 14.800.000 * 100 / (45.000.000 + 100) = 0,032888 USDT/TRX
  Impacto: 0,0003%

Intercambio de 100.000 TRX:
  Salida: 14.800.000 * 100.000 / (45.000.000 + 100.000) = 32.797 USDT
  Precio efectivo: 0,32797 USDT/TRX
  Impacto: 0,28%

Intercambio de 1.000.000 TRX:
  Salida: 14.800.000 * 1.000.000 / (45.000.000 + 1.000.000) = 321.739 USDT
  Precio efectivo: 0,32174 USDT/TRX
  Impacto: 2,17%
```

MERX incluye el cálculo de este impacto en cada cotización para que el agente pueda evaluar si el tamaño de la operación es apropiado para la liquidez disponible.

## Intercambios de Múltiples Saltos

No todos los pares de tokens tienen fondos de liquidez directa. Para tokens que operan contra WTRX pero no entre sí, SunSwap V2 enruta a través de un token intermedio:

```
Token A -> WTRX -> Token B
```

Esto requiere dos operaciones de intercambio en una sola transacción (manejada por el contrato enrutador). El consumo de energía es mayor:

```
Intercambio directo (p. ej., TRX -> USDT): ~223.000 energía
Intercambio de dos saltos (p. ej., USDC -> USDT vía WTRX): ~340.000 energía
Intercambio de tres saltos: ~460.000 energía
```

La herramienta `get_swap_quote` encuentra automáticamente la ruta óptima e informa el requisito de energía total para la ruta completa.

## Estrategias de Intercambio para Agentes

### Promedio de Costo en Dólares

Un agente encargado de acumular USDT puede usar órdenes permanentes combinadas con ejecución de intercambios:

```
Orden permanente:
  Disparador: programar "0 */4 * * *" (cada 4 horas)
  Acción: Ejecutar intercambio 100 TRX -> USDT a precio de mercado
  Restricción: máximo 600 TRX/día
```

Seis intercambios por día, cada uno de 100 TRX, suavizando la volatilidad de precios.

### Operación Basada en Umbral

```
Orden permanente:
  Disparador: Precio TRX/USDT por encima de 0,35
  Acción: Intercambiar 1.000 TRX -> USDT
  Restricción: máximo 1 ejecución/día
```

El agente vende TRX cuando el precio es favorable, acumulando USDT para gastos operativos.

### Reequilibrio

Un agente que gestiona una cartera puede usar intenciones para reequilibrar:

```
execute_intent:
  Paso 1: Intercambiar 500 TRX -> USDT (si asignación de TRX > 60%)
  Paso 2: Intercambiar 200 USDT -> TRX (si asignación de TRX < 40%)
  Estrategia de recursos: batch_cheapest
```

## Comparación: TronWeb sin Procesar vs MERX MCP

### TronWeb sin Procesar (sin MERX)

```javascript
// 1. Estimar energía (manual)
const estimation = await tronWeb.transactionBuilder.triggerConstantContract(
  routerAddress, 'swapExactETHForTokens(uint256,address[],address,uint256)',
  { callValue: 100000 },
  [{ type: 'uint256', value: 0 },
   { type: 'address[]', value: [wtrxAddr, usdtAddr] },
   { type: 'address', value: myAddr },
   { type: 'uint256', value: deadline }],
  myAddr
);
// 2. Comprar energía en algún lugar (integración separada)
// 3. Esperar delegación (encuesta manual)
// 4. Construir transacción de intercambio
// 5. Firmar
// 6. Difundir
// 7. Verificar
// ~50 líneas de código, múltiples integraciones API
```

### MERX MCP (una llamada de herramienta)

```
Tool: execute_swap
Input: {
  "from_token": "TRX",
  "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
  "amount": "100000",
  "slippage": 1.0
}
// Una llamada. Todo manejado internamente.
```

El agente no necesita saber sobre direcciones de enrutador, envolvimiento de WTRX, codificación ABI, mercados de energía, mecánica de delegación o construcción de transacciones. Expresa intención ("intercambiar TRX por USDT") y recibe un resultado.

## Riesgos y Limitaciones

### Deslizamiento

Incluso con protección contra deslizamiento, los movimientos rápidos de precios pueden causar que los intercambios se reviertan. Si el intercambio se revierte, la energía utilizada para la transacción revertida se pierde (aunque los tokens no). El agente puede reintentar con una tolerancia de deslizamiento más alta o una cantidad menor.

### MEV y Front-Running

El modelo de producción de bloques de TRON es diferente al de Ethereum, y el panorama de MEV es menos desarrollado. Sin embargo, los intercambios grandes en SunSwap todavía pueden ser adelantados por actores sofisticados que monitorean el mempool. Para operaciones grandes, considera dividir en múltiples intercambios más pequeños en diferentes bloques.

### Liquidez

La liquidez de SunSwap V2 varía significativamente entre pares de tokens. Los pares principales (TRX/USDT, TRX/USDC) tienen liquidez profunda. Los tokens más pequeños pueden tener fondos delgados donde incluso operaciones moderadas causan un impacto de precio significativo. Siempre verifica la cotización antes de ejecutar.

## Conclusión

El operador DEX a través de un agente de IA ya no es una capacidad teórica. MERX la hace operacional con dos llamadas de herramienta: una para cotizar, una para ejecutar. La simulación de energía es exacta. La compra de recursos es automática. Las aprobaciones de tokens se manejan de forma transparente.

Para agentes de IA que operan en TRON, SunSwap a través de MCP es la ruta más rápida hacia el operador en la cadena. Sin interfaz web. Sin construcción de transacciones manual. Sin sobrecarga de gestión de recursos.

Cotiza. Ejecuta. Hecho.

---

**Enlaces:**
- Plataforma MERX: [https://merx.exchange](https://merx.exchange)
- Servidor MCP (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- Servidor MCP (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)


## Pruébalo Ahora con IA

Agrega MERX a Claude Desktop o cualquier cliente compatible con MCP -- sin instalación, sin clave API necesaria para herramientas de solo lectura:

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