# 30 Prompts y 21 Recursos: Por qué MERX es el Único Servidor MCP de Protocolo Completo

## MCP Tiene Tres Primitivos, No Uno

El Model Context Protocol define tres tipos de capacidades que un servidor puede exponer a un cliente de IA:

1. **Herramientas** - acciones ejecutables (enviar una transacción, comprar energía, ejecutar un intercambio)
2. **Prompts** - plantillas preconstruidas que guían al modelo a través de flujos de trabajo complejos
3. **Recursos** - datos estructurados que el modelo puede leer (feeds de precios, parámetros de red, documentación)

La mayoría de servidores MCP implementan solo herramientas. Exponen un puñado de funciones llamables, dan por hecho que terminaron, y se comercializan a sí mismos como "compatibles con MCP". Esto es técnicamente correcto de la misma manera que un automóvil con motor pero sin volante es técnicamente un vehículo.

Las herramientas solas le dan al modelo acciones pero no orientación sobre cuándo o cómo usarlas. Sin prompts, el modelo debe descubrir flujos de trabajo complejos de múltiples pasos desde cero cada vez. Sin recursos, el modelo no tiene datos estructurados para consultar - debe hacer llamadas a herramientas solo para leer información que debería estar disponible pasivamente.

MERX implementa los tres primitivos: 55 herramientas, 30 prompts y 21 recursos. Este artículo explica qué hace cada primitivo, por qué los tres importan, y cómo MERX los utiliza para crear un servidor MCP que es cualitativamente diferente de las implementaciones que solo contienen herramientas.

## Primitivo 1: Herramientas (Acciones)

Las herramientas son el primitivo más intuitivo. Una herramienta es una función que el modelo puede llamar para realizar una acción. Tiene un nombre, una descripción, parámetros tipados y devuelve una respuesta estructurada.

MERX expone 55 herramientas que cubren el alcance completo de operaciones de blockchain de TRON:

### Operaciones de Cartera
- `set_private_key` - Configurar la cartera con una clave privada (deriva la dirección automáticamente)
- `get_trx_balance` - Verificar el saldo de TRX de cualquier dirección
- `get_trc20_balance` - Verificar saldo de token TRC-20
- `transfer_trx` - Enviar TRX a una dirección
- `transfer_trc20` - Enviar tokens TRC-20 a una dirección
- `check_address_resources` - Ver asignación de energía y bandwidth

### Mercado de Energía
- `get_prices` - Precios actuales de todos los proveedores
- `get_best_price` - Encontrar la oferta de energía más barata
- `create_order` - Comprar energía del mercado
- `create_paid_order` - Comprar energía a través de x402 (sin necesidad de cuenta)
- `ensure_resources` - Comprar automáticamente los recursos necesarios
- `list_orders` - Ver historial de órdenes
- `get_order` - Obtener detalles de una orden específica

### Comercio en DEX
- `get_swap_quote` - Obtener una cotización de SunSwap V2 con simulación exacta
- `execute_swap` - Ejecutar un intercambio con gestión automática de recursos
- `approve_trc20` - Establecer aprobación de gasto de token

### Automatización
- `create_standing_order` - Configurar compras automáticas de energía
- `list_standing_orders` - Ver órdenes activas recurrentes
- `create_monitor` - Configurar monitores de delegación o saldo
- `list_monitors` - Ver monitores activos

### Avanzado
- `execute_intent` - Ejecutar planes de transacción de múltiples pasos
- `estimate_contract_call` - Simular cualquier llamada de contrato para estimación de energía

Cada herramienta tiene una definición de JSON Schema para sus parámetros, una descripción en lenguaje natural, y tipos de retorno estructurados. El modelo sabe exactamente qué hace cada herramienta, qué parámetros necesita, y qué devolverá.

Pero las herramientas solas no son suficientes.

## Primitivo 2: Prompts (Plantillas)

Un prompt es una plantilla preconstruida que el modelo puede usar para manejar un tipo específico de solicitud. Piénsalo como una receta: "Cuando el usuario pregunte por X, aquí está el enfoque paso a paso, incluyendo qué herramientas llamar, en qué orden, y cómo interpretar los resultados".

MERX proporciona 30 prompts organizados en 10 categorías.

### Primeros Pasos (3 prompts)

**setup_wallet**
```
Plantilla: Guiar al usuario a través de la configuración de la cartera
Pasos:
  1. Explicar seguridad de clave privada
  2. Llamar a set_private_key
  3. Llamar a get_trx_balance para verificar
  4. Llamar a check_address_resources para línea base
  5. Recomendar próximos pasos según el saldo
```

**first_energy_purchase**
```
Plantilla: Guiar a compradores de energía por primera vez
Pasos:
  1. Explicar qué es la energía y por qué importa
  2. Llamar a get_prices para mostrar descripción general del mercado
  3. Ayudar a elegir cantidad y duración
  4. Llamar a create_order o create_paid_order
  5. Verificar delegación con check_address_resources
```

**platform_overview**
```
Plantilla: Explicar capacidades de MERX
Contenido: Descripción general estructurada de todas las herramientas, prompts y recursos
  con ejemplos de cuándo usar cada una
```

### Gestión de Energía (4 prompts)

**buy_cheapest_energy**
```
Plantilla: Encontrar y comprar energía al mejor precio
Pasos:
  1. Llamar a get_best_price con parámetros del usuario
  2. Mostrar comparación de precios entre proveedores
  3. Confirmar compra con usuario
  4. Llamar a create_order
  5. Sondear confirmación de delegación
```

**compare_providers**
```
Plantilla: Comparación detallada de proveedores
Pasos:
  1. Llamar a get_prices para todos los proveedores
  2. Calcular costo para necesidades específicas del usuario
  3. Presentar tabla de comparación
  4. Incluir métricas de confiabilidad y velocidad
  5. Recomendar opción mejor con razonamiento
```

**optimize_costs**
```
Plantilla: Analizar gasto y sugerir optimizaciones
Pasos:
  1. Revisar historial de órdenes
  2. Analizar patrones de precio
  3. Identificar oportunidades para órdenes recurrentes
  4. Calcular ahorros potenciales
  5. Presentar plan de optimización
```

**ensure_resources_for_tx**
```
Plantilla: Preparar recursos para una transacción específica
Pasos:
  1. Estimar energía para la transacción planeada
  2. Verificar recursos actuales
  3. Calcular déficit
  4. Comprar si es necesario
  5. Confirmar disponibilidad
```

### Ejecución de Transacciones (4 prompts)

**send_usdt**
```
Plantilla: Transferencia USDT completa con optimización de recursos
Pasos:
  1. Validar dirección del destinatario
  2. Estimar energía
  3. Asegurar recursos
  4. Ejecutar transferencia
  5. Verificar en cadena
```

**send_trx**
```
Plantilla: Transferencia de TRX (más simple - no requiere energía)
Pasos:
  1. Validar dirección del destinatario
  2. Verificar bandwidth
  3. Ejecutar transferencia
  4. Verificar en cadena
```

**swap_tokens**
```
Plantilla: Intercambio en DEX con pipeline completo
Pasos:
  1. Obtener cotización
  2. Revisar impacto de precio y deslizamiento
  3. Verificar si se necesita aprobación
  4. Asegurar recursos para aprobación + intercambio
  5. Ejecutar intercambio
  6. Verificar resultado
```

**execute_complex_intent**
```
Plantilla: Plan de transacción de múltiples pasos
Pasos:
  1. Ayudar al usuario a definir pasos
  2. Simular todos los pasos
  3. Revisar costos totales
  4. Elegir estrategia de recursos
  5. Ejecutar intención
  6. Reportar resultados
```

### Automatización (4 prompts)

**setup_standing_order**
```
Plantilla: Configurar compras automáticas de energía
Pasos:
  1. Entender necesidades del usuario (volumen, tiempo, presupuesto)
  2. Recomendar tipo de disparador
  3. Establecer restricciones apropiadas
  4. Crear orden recurrente
  5. Explicar monitoreo y gestión
```

**setup_delegation_monitor**
```
Plantilla: Configurar renovación automática para delegaciones de energía
Pasos:
  1. Revisar delegaciones actuales
  2. Establecer ventana de renovación
  3. Configurar protección de precio
  4. Establecer canales de notificación
  5. Crear monitor
```

**setup_balance_monitor**
```
Plantilla: Configurar alertas y acciones basadas en saldo
Pasos:
  1. Analizar patrones actuales de uso de recursos
  2. Recomendar niveles de umbral
  3. Configurar acción (comprar, asegurar o notificar)
  4. Establecer restricciones de presupuesto
  5. Crear monitor
```

**review_automation**
```
Plantilla: Auditar y optimizar automatización existente
Pasos:
  1. Listar todas las órdenes recurrentes y monitores
  2. Revisar historial de ejecución
  3. Identificar reglas de bajo rendimiento
  4. Sugerir mejoras
  5. Aplicar cambios si se aprueba
```

### Análisis (4 prompts)

**analyze_prices**
```
Plantilla: Análisis de precios del mercado
Pasos:
  1. Extraer precios actuales de todos los proveedores
  2. Extraer historial de precios
  3. Identificar tendencias y patrones
  4. Calcular tiempos de compra óptimos
  5. Presentar análisis con gráficos
```

**calculate_savings**
```
Plantilla: Calcular ahorros del uso de energía delegada
Pasos:
  1. Estimar energía para tipos de transacciones del usuario
  2. Calcular costo con quema de TRX
  3. Calcular costo con compra de energía
  4. Mostrar porcentaje de ahorro
  5. Proyectar ahorros anuales a volumen dado
```

**estimate_transaction_cost**
```
Plantilla: Estimación de costos para cualquier tipo de transacción
Pasos:
  1. Identificar tipo de transacción
  2. Simular con parámetros exactos
  3. Mostrar requisitos de energía y bandwidth
  4. Mostrar costo con y sin delegación
  5. Recomendar enfoque óptimo
```

**portfolio_overview**
```
Plantilla: Descripción general completa de cuenta
Pasos:
  1. Verificar todos los saldos (TRX, USDT, otros tokens)
  2. Verificar asignación de recursos
  3. Revisar delegaciones activas
  4. Resumir reglas de automatización activas
  5. Presentar descripción general financiera
```

### Gestión de Cuenta (3 prompts)

**account_setup**, **deposit_guide**, **withdrawal_guide** - Plantillas paso a paso para operaciones del ciclo de vida de la cuenta.

### x402 (2 prompts)

**x402_purchase** - Guía a través del flujo de compra sin registro.
**x402_explain** - Explicar cómo funciona x402 y cuándo usarlo.

### Información de Red (2 prompts)

**explain_energy**, **explain_bandwidth** - Plantillas educativas que explican el modelo de recursos de TRON usando datos de recursos.

### Solución de Problemas (2 prompts)

**transaction_failed**, **delegation_not_arrived** - Plantillas de diagnóstico que ayudan a identificar y resolver problemas comunes.

### Integración (2 prompts)

**sdk_quickstart**, **api_integration** - Plantillas para desarrolladores integrando MERX en sus aplicaciones.

### Por Qué Los Prompts Importan

Sin prompts, un modelo que recibe la solicitud "ayúdame a comprar energía" necesitaría:
1. Descubrir qué herramientas son relevantes
2. Determinar el orden correcto de operaciones
3. Decidir qué información recopilar primero
4. Manejar casos especiales que podría no conocer

Con el prompt `buy_cheapest_energy`, el modelo tiene un flujo de trabajo probado y optimizado que maneja casos especiales, presenta información en el formato correcto, y sigue la secuencia correcta de operaciones. La diferencia es la misma que darle a alguien un conjunto de herramientas versus darle herramientas más un manual.

## Primitivo 3: Recursos (Datos)

Los recursos son datos estructurados que el modelo puede leer sin hacer una llamada de acción. A diferencia de las herramientas, que hacen algo, los recursos proporcionan algo - hacen información disponible pasivamente.

MERX proporciona 21 recursos: 14 estáticos y 7 plantillas dinámicas.

### Recursos Estáticos (14)

Los recursos estáticos proporcionan datos de referencia fijos que no cambian entre sesiones:

```
merx://docs/getting-started
merx://docs/energy-explained
merx://docs/bandwidth-explained
merx://docs/resource-model
merx://docs/providers-overview
merx://docs/x402-protocol
merx://docs/standing-orders-guide
merx://docs/monitors-guide
merx://docs/intent-execution-guide
merx://docs/api-reference
merx://docs/sdk-js
merx://docs/sdk-python
merx://docs/faq
merx://docs/troubleshooting
```

Cuando el modelo lee `merx://docs/energy-explained`, recibe un documento estructurado explicando el modelo de energía de TRON - qué es la energía, cómo se consume, cómo funciona la delegación, y cómo se relaciona con costos de TRX. Esta información está disponible inmediatamente, sin hacer ninguna llamada API o invocación de herramienta.

### Recursos de Plantilla Dinámicos (7)

Los recursos de plantilla aceptan parámetros y devuelven datos específicos para la solicitud:

```
merx://prices/current
merx://prices/history/{period}
merx://providers/{provider_name}
merx://account/{address}/resources
merx://account/{address}/balances
merx://network/parameters
merx://network/chain-info
```

Por ejemplo, `merx://prices/current` devuelve los precios actuales de energía de todos los proveedores en un formato estructurado:

```json
{
  "timestamp": "2026-03-30T12:00:00Z",
  "prices": [
    {
      "provider": "sohu",
      "energy_1h": 0.0000526,
      "energy_24h": 0.0000498,
      "available": true,
      "min_amount": 65000
    },
    {
      "provider": "catfee",
      "energy_1h": 0.0000540,
      "energy_24h": 0.0000510,
      "available": true,
      "min_amount": 65000
    }
  ]
}
```

Y `merx://account/{address}/resources` devuelve la asignación de recursos actual para cualquier dirección:

```json
{
  "address": "TYourAddress...",
  "energy": {
    "available": 487231,
    "total": 500000,
    "used": 12769,
    "delegated_from": [
      {
        "amount": 500000,
        "provider": "sohu",
        "expires_at": "2026-03-31T02:00:00Z"
      }
    ]
  },
  "bandwidth": {
    "available": 1342,
    "total": 1500,
    "free_remaining": 1342
  }
}
```

### Por Qué Los Recursos Importan

Los recursos dan al modelo contexto sin la sobrecarga de llamadas a herramientas. Cuando un usuario pregunta "¿cuál es mi saldo de energía?", el modelo puede verificar el recurso `merx://account/{address}/resources` directamente. Cuando el modelo necesita referenciar documentación sobre órdenes recurrentes antes de crear una, lee `merx://docs/standing-orders-guide`.

Sin recursos, cada pieza de información requiere una llamada a una herramienta. El modelo necesitaría llamar a `get_prices` para ver precios actuales, incluso aunque esa información podría estar disponible pasivamente. La distinción está entre un modelo que debe solicitar activamente cada pieza de datos (solo herramientas) y un modelo que tiene un entorno de información rico disponible para consulta (herramientas + recursos).

## La Ventaja del Protocolo Completo

Cuando los tres primitivos trabajan juntos, el modelo opera a un nivel fundamentalmente diferente:

### Escenario: El usuario pregunta "Ayúdame a configurar automatización de energía"

**Servidor MCP solo con herramientas:**
1. El modelo adivina qué herramientas existen para automatización
2. Llama a herramientas en una secuencia de prueba y error
3. Podría perder opciones de configuración importantes
4. Sin orientación sobre mejores prácticas
5. Sin material de referencia de fondo

**Servidor MCP MERX de protocolo completo:**
1. El modelo lee `merx://docs/standing-orders-guide` para material de referencia
2. El modelo activa el prompt `setup_standing_order` para flujo de trabajo paso a paso
3. El modelo lee `merx://prices/history/7d` para recomendar umbrales de precio
4. El modelo llama a la herramienta `create_standing_order` con parámetros optimizados
5. El modelo lee `merx://docs/monitors-guide` y sugiere monitores complementarios
6. El modelo llama a la herramienta `create_monitor` para protección de expiración de delegación
7. Resultado: Automatización completa y bien configurada con decisiones respaldadas por documentación

El enfoque de protocolo completo no es solo marginalmente mejor - produce resultados cualitativamente diferentes porque el modelo tiene acceso a orientación (prompts), conocimiento (recursos) y capacidad (herramientas) simultáneamente.

## Lo Que Otros Servidores MCP Están Perdiendo

Una encuesta de servidores MCP relacionados con blockchain a principios de 2026 muestra un patrón consistente:

| Servidor | Herramientas | Prompts | Recursos | Protocolo Completo |
|---|---|---|---|---|
| ETH MCP Genérico | 5-8 | 0 | 0 | No |
| Solana MCP | 10-12 | 0 | 0 | No |
| Bitcoin MCP | 3-4 | 0 | 0 | No |
| MCP Multicadena | 15-20 | 0 | 0 | No |
| MERX | 52 | 30 | 21 | Sí |

Ningún otro servidor MCP de blockchain implementa prompts o recursos. Todos son solo herramientas, lo cual significa:

- Sin orientación de flujo de trabajo para operaciones de múltiples pasos
- Sin documentación disponible en tiempo de inferencia del modelo
- Sin acceso pasivo a datos para contexto y referencia
- Cada interacción comienza desde cero sin conocimiento estructural

MERX es el único servidor MCP donde el modelo tiene un entorno operacional completo - la capacidad de actuar (herramientas), el conocimiento de cómo actuar (prompts), y los datos para informar decisiones (recursos).

## Los Números

MERX por los números:

- **55 herramientas** en 5 categorías (cartera, mercado de energía, DEX, automatización, avanzado)
- **30 prompts** en 10 categorías (primeros pasos, energía, transacciones, automatización, análisis, cuentas, x402, red, solución de problemas, integración)
- **21 recursos** (14 documentación estática, 7 plantillas de datos dinámicos)
- **72 capacidades totales** expuestas a través de un único servidor MCP

Para un agente de IA conectado a MERX, esto significa:
- Puede realizar cualquier operación de TRON (herramientas)
- Conoce el mejor enfoque para cualquier escenario (prompts)
- Tiene datos en tiempo real y documentación siempre disponibles (recursos)

## Conclusión

MCP es un protocolo con tres primitivos, no uno. Implementar solo herramientas es como construir una API con puntos finales pero sin documentación y sin feeds de datos. Funciona, técnicamente, pero obliga al consumidor a descubrir todo por su cuenta.

MERX implementa el protocolo completo porque el protocolo completo es cómo MCP fue diseñado para ser usado. Herramientas para acción. Prompts para orientación. Recursos para conocimiento. Los tres trabajando juntos para crear un entorno donde un agente de IA puede operar en la blockchain de TRON con la misma profundidad de capacidad que un desarrollador humano con años de experiencia.

Treinta prompts. Veintiuno recursos. Veinticinco herramientas. Ningún otro servidor MCP de blockchain se acerca.

---

**Enlaces:**
- Plataforma MERX: [https://merx.exchange](https://merx.exchange)
- Servidor MCP (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- Servidor MCP (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)


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

Pregunta a tu agente de IA: "¿Cuál es la energía de TRON más barata ahora mismo?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)