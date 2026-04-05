# Ejecución de Intenciones: Planes Multi-Paso para Agentes de IA en TRON

## El Problema con Uno-a-Uno

Los agentes de IA que interactúan con TRON enfrentan un problema estructural cuando las tareas implican múltiples operaciones en cadena. Considera un escenario simple: un agente necesita enviar 100 USDT a Alice y luego intercambiar 50 TRX por USDT. Cada operación requiere su propia compra de energy, su propio tiempo de espera de delegación y su propio ciclo de transmisión.

Con el enfoque uno-a-uno, el agente realiza al menos 8 llamadas de herramienta:

1. Estimar energy para transferencia USDT
2. Comprar energy para transferencia USDT
3. Esperar delegación
4. Ejecutar transferencia USDT
5. Estimar energy para intercambio
6. Comprar energy para intercambio
7. Esperar delegación
8. Ejecutar intercambio

Cada compra de energy es una interacción de mercado separada con su propio overhead de transacción. Cada espera de delegación agrega 3-6 segundos de latencia. El tiempo total puede superar 30 segundos para lo que debería ser una simple tarea de dos pasos.

MERX resuelve esto con ejecución de intenciones - un sistema que toma un plan multi-paso de un agente de IA, simula cada paso, optimiza compras de recursos en todos los pasos, y ejecuta el plan completo en secuencia.

## Qué es una Intención

En el sistema MERX, una intención es una descripción declarativa de lo que el agente desea lograr, expresada como una lista ordenada de acciones. El agente especifica el resultado deseado, y MERX maneja la mecánica de ejecución.

Una intención difiere de una secuencia de llamadas de herramienta en tres formas importantes:

1. **Optimización de recursos** - MERX puede agrupar compras de energy en todos los pasos, comprando el energy total necesario en una sola orden en lugar de paso a paso.

2. **Pre-validación** - Cada paso se simula antes de que cualquier paso se ejecute. Si el paso 3 de un plan de 5 pasos fallara, el agente lo sabe antes de que el paso 1 se transmita.

3. **Planificación atómica** - El agente envía el plan completo de una vez, dando a MERX visibilidad del alcance completo del trabajo. Esto permite optimizaciones que son imposibles cuando los pasos se envían individualmente.

## La Herramienta execute_intent

El servidor MCP expone la ejecución de intenciones a través de la herramienta `execute_intent`:

```
Tool: execute_intent
Input: {
  "steps": [
    {
      "action": "transfer_trc20",
      "params": {
        "to": "TAliceAddress...",
        "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "100000000"
      }
    },
    {
      "action": "swap",
      "params": {
        "from_token": "TRX",
        "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "50000000",
        "slippage": 0.5
      }
    }
  ],
  "resource_strategy": "batch_cheapest"
}
```

La respuesta incluye los resultados de simulación para cada paso, el costo total de recursos, y el estado de ejecución de cada paso después de completarse.

## Acciones Soportadas

El sistema de intenciones soporta los siguientes tipos de acción:

### transfer_trx

Enviar TRX a una dirección. Esta es una transferencia nativa que consume bandwidth pero no energy.

```json
{
  "action": "transfer_trx",
  "params": {
    "to": "TRecipient...",
    "amount_sun": 1000000
  }
}
```

### transfer_trc20

Enviar un token TRC-20 (USDT, USDC, etc.) a una dirección. Consume energy para la llamada de contrato inteligente.

```json
{
  "action": "transfer_trc20",
  "params": {
    "to": "TRecipient...",
    "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "amount": "100000000"
  }
}
```

### swap

Ejecutar un intercambio de token en SunSwap V2. Incluye simulación exacta de energy para los parámetros de intercambio específicos.

```json
{
  "action": "swap",
  "params": {
    "from_token": "TRX",
    "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "amount": "100000000",
    "slippage": 0.5
  }
}
```

### approve

Establecer una aprobación de gasto para un token TRC-20. Requerido antes de intercambiar tokens (no necesario para intercambiar TRX).

```json
{
  "action": "approve",
  "params": {
    "token_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    "spender": "TRouterAddress...",
    "amount": "unlimited"
  }
}
```

### call_contract

Ejecutar una llamada de contrato inteligente arbitraria. Esta es la puerta de escape para operaciones no cubiertas por los tipos de acción específicos.

```json
{
  "action": "call_contract",
  "params": {
    "contract_address": "TContractAddress...",
    "function_selector": "stake(uint256)",
    "parameters": [{ "type": "uint256", "value": "1000000" }],
    "call_value": 0
  }
}
```

### buy_resource

Comprar energy o bandwidth como un paso en el plan. Útil cuando el agente desea control explícito sobre el timing de recursos.

```json
{
  "action": "buy_resource",
  "params": {
    "resource_type": "energy",
    "amount": 130000,
    "duration_hours": 1
  }
}
```

## Estrategias de Recursos

El parámetro `resource_strategy` controla cómo MERX maneja las compras de energy en los pasos de la intención.

### batch_cheapest

Esta es la estrategia predeterminada y recomendada. MERX simula todos los pasos, suma el energy total requerido, resta los recursos disponibles, y realiza una única compra de energy para la intención completa.

```
Paso 1 (transfer_trc20): 64,895 energy
Paso 2 (swap):           223,354 energy
Total necesario:         288,249 energy
Disponible actualmente:  0 energy
Compra:                  290,000 energy (redondeado a unidad de orden)
```

Una compra. Una espera de delegación. Luego todos los pasos se ejecutan secuencialmente usando el energy agrupado.

Beneficios:
- Interacción de mercado única (menor overhead)
- Espera de delegación única (menor latencia)
- Posible descuento por volumen en órdenes más grandes
- Manejo de fallos más simple

### per_step

Cada paso compra su propio energy independientemente. Usa esto cuando los pasos son condicionales o cuando necesitas minimizar riesgo (si el paso 1 falla, no has comprado energy para el paso 2).

```
Paso 1: comprar 65,000 energy -> esperar -> ejecutar transferencia
Paso 2: comprar 225,000 energy -> esperar -> ejecutar intercambio
```

Esta estrategia es más lenta (dos esperas de delegación) pero desperdicia menos energy si la ejecución se detiene a mitad del plan.

## Simulación Stateful

El motor de simulación del sistema de intenciones mantiene estado en todos los pasos. Esto es crítico para planes donde los pasos posteriores dependen de los resultados de pasos anteriores.

Considera esta intención: "Intercambiar 50 TRX por USDT, luego enviar el USDT recibido a Alice."

El motor de simulación:

1. Simula el paso 1 (intercambio). Resultado: el agente recibe 16.42 USDT.
2. Actualiza el estado simulado para reflejar el nuevo balance de USDT.
3. Simula el paso 2 (transferir 16.42 USDT a Alice) contra el estado actualizado.
4. Confirma que el paso 2 tendría éxito con el balance del paso 1.

Sin simulación stateful, el paso 2 se simularía contra el balance actual del agente (que podría no incluir el USDT del intercambio). La simulación reportaría incorrectamente que el paso 2 fallaría debido a balance insuficiente.

```
Tool: execute_intent
Input: {
  "steps": [
    {
      "action": "swap",
      "params": {
        "from_token": "TRX",
        "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "50000000",
        "slippage": 0.5
      }
    },
    {
      "action": "transfer_trc20",
      "params": {
        "to": "TAliceAddress...",
        "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "use_previous_output"
      }
    }
  ],
  "resource_strategy": "batch_cheapest"
}
```

El parámetro `use_previous_output` le dice al sistema de intenciones que use la cantidad de salida del paso anterior como la cantidad de entrada para este paso.

## Respuesta de Simulación

Antes de que la ejecución comience, el sistema de intenciones devuelve un resumen de simulación:

```json
{
  "simulation": {
    "steps": [
      {
        "action": "transfer_trc20",
        "energy_required": 64895,
        "bandwidth_required": 345,
        "simulated_success": true,
        "estimated_cost_trx": 3.42
      },
      {
        "action": "swap",
        "energy_required": 223354,
        "bandwidth_required": 420,
        "simulated_success": true,
        "estimated_output": "16.42 USDT",
        "estimated_cost_trx": 11.76
      }
    ],
    "total_energy": 288249,
    "total_bandwidth": 765,
    "total_cost_trx": 15.18,
    "resource_purchase": {
      "energy": 290000,
      "price": 15.24,
      "provider": "sohu"
    }
  },
  "status": "ready_to_execute"
}
```

El agente ve el plan completo con costos antes de que se realice cualquier acción en cadena. Si los costos son inaceptables o un paso fallaría, el agente puede modificar el plan sin gastar nada.

## Ejecución y Manejo de Errores

Una vez que el agente confirma el plan (o si la auto-ejecución está habilitada), la intención se ejecuta paso a paso:

```json
{
  "execution": {
    "steps": [
      {
        "action": "transfer_trc20",
        "status": "completed",
        "tx_hash": "abc123...",
        "energy_used": 64895,
        "block": 58234567
      },
      {
        "action": "swap",
        "status": "completed",
        "tx_hash": "def456...",
        "energy_used": 223354,
        "output_amount": "16.42",
        "block": 58234568
      }
    ],
    "total_energy_used": 288249,
    "total_energy_purchased": 290000,
    "energy_wasted": 1751,
    "status": "all_steps_completed"
  }
}
```

### Fallo en Medio de la Ejecución

Si un paso falla durante la ejecución (no durante la simulación), el sistema de intenciones se detiene y reporta el fallo:

```json
{
  "execution": {
    "steps": [
      {
        "action": "transfer_trc20",
        "status": "completed",
        "tx_hash": "abc123..."
      },
      {
        "action": "swap",
        "status": "failed",
        "error": "SLIPPAGE_EXCEEDED",
        "message": "Output 15.89 USDT below minimum 16.34 USDT"
      }
    ],
    "status": "partial_execution",
    "completed_steps": 1,
    "failed_step": 2,
    "remaining_energy": 223354
  }
}
```

El paso 1 ya se ha comprometido en cadena y no puede revertirse. El agente recibe el balance de energy restante y puede decidir cómo proceder - reintentar el paso fallido con parámetros ajustados, ejecutar una acción diferente, o dejar que el energy expire.

## Ejemplo del Mundo Real: Rebalanceo de Tesorería

Aquí hay una intención realista multi-paso que un agente podría ejecutar para gestión de tesorería:

"Intercambiar 1,000 TRX por USDT, enviar 300 USDT a la billetera de operaciones, enviar 200 USDT a la billetera de marketing, mantener el resto."

```
Tool: execute_intent
Input: {
  "steps": [
    {
      "action": "swap",
      "params": {
        "from_token": "TRX",
        "to_token": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "1000000000",
        "slippage": 1.0
      }
    },
    {
      "action": "transfer_trc20",
      "params": {
        "to": "TOpsWallet...",
        "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "300000000"
      }
    },
    {
      "action": "transfer_trc20",
      "params": {
        "to": "TMarketingWallet...",
        "contract_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "amount": "200000000"
      }
    }
  ],
  "resource_strategy": "batch_cheapest"
}
```

Simulación:

```
Paso 1 (intercambio):  223,354 energy
Paso 2 (transferencia): 29,631 energy  (OpsWallet ya tiene USDT)
Paso 3 (transferencia): 64,895 energy  (MarketingWallet es nuevo en USDT)
Total:                 317,880 energy

Compra por lotes: 320,000 energy a 16.83 TRX de catfee

Sin agrupamiento de intenciones: 3 compras separadas = ~18.20 TRX
Con agrupamiento de intenciones: 1 compra = 16.83 TRX
Ahorros por agrupamiento: 1.37 TRX + latencia reducida (1 espera vs 3)
```

## Cuándo Usar Intenciones vs Herramientas Individuales

Usa `execute_intent` cuando:

- La tarea implica dos o más operaciones en cadena
- Los pasos tienen dependencias (el paso 2 usa la salida del paso 1)
- Quieres minimizar costos totales de recursos a través del agrupamiento
- Necesitas pre-validación de todo el plan antes de comprometerse

Usa herramientas individuales cuando:

- La tarea es una sola operación
- El agente necesita tomar decisiones entre pasos basadas en entrada externa
- Los pasos están separados por brechas de tiempo significativas
- El agente quiere control máximo sobre cada etapa de ejecución

## Intenciones y Autonomía del Agente

El sistema de intenciones está diseñado para la autonomía del agente. Un agente que recibe una instrucción de alto nivel como "rebalancea la tesorería" puede descomponerla en pasos concretos, construir una intención, simularla, revisar los costos, y ejecutarla - todo sin intervención humana en ninguna etapa.

El paso de simulación sirve como verificación de seguridad del agente. Antes de comprometer fondos, el agente puede verificar que cada paso tendrá éxito, el costo total está dentro del presupuesto, y los resultados esperados coinciden con el resultado deseado. Esto es equivalente a un humano revisando una transacción antes de hacer clic en "confirmar", pero ejecutado programáticamente por el agente mismo.

Combinado con órdenes permanentes para compras recurrentes de recursos y monitores para alertas de balance, el sistema de intenciones permite operaciones en cadena totalmente autónomas que se ejecutan 24/7 sin supervisión humana.

## Conclusión

La ejecución de un solo paso son las ruedas de entrenamiento de la automatización blockchain. Los flujos de trabajo reales del agente implican múltiples operaciones, dependencias entre pasos, y optimización de recursos en el plan completo.

La ejecución de intenciones MERX da a los agentes de IA la capacidad de pensar en planes en lugar de acciones individuales. Simula todo. Optimiza recursos en el alcance completo. Ejecuta con confianza de que cada paso ha sido pre-validado.

El blockchain no es un entorno de operación única. Tu agente tampoco debería serlo.

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

Pregunta a tu agente de IA: "¿Cuál es el energy de TRON más barato en este momento?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)