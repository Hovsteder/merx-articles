# Tres protocolos, un agente TRON: MCP, A2A y ACP en MERX

El ecosistema de agentes de IA se esta fragmentando entre protocolos. Anthropic tiene MCP. Google lanzo A2A. BeeAI construyo ACP. Cada protocolo resuelve la comunicacion entre agentes de manera diferente, y cada uno tiene su propio ecosistema de frameworks y orquestadores. Para los desarrolladores que construyen aplicaciones TRON, esto significa que elegir un protocolo tambien implica elegir que frameworks pueden acceder a los recursos TRON. MERX elimina esa eleccion al soportar los tres protocolos desde una unica plataforma -- el primero y unico agente TRON en hacerlo.

## El panorama de protocolos en 2026

Tres protocolos dominan ahora la infraestructura de agentes de IA:

**MCP (Model Context Protocol)** -- creado por Anthropic. Un protocolo basado en herramientas donde los agentes descubren y llaman funciones. 55 herramientas, 30 prompts, 21 recursos. Utilizado por Claude, Cursor, Windsurf y cientos de clientes compatibles con MCP. Es el protocolo mas maduro para la interaccion directa entre agente y herramienta.

**A2A (Agent-to-Agent Protocol)** -- creado por Google, ahora bajo la Linux Foundation. Un protocolo basado en tareas donde los orquestadores envian tareas a agentes especialistas y reciben resultados de forma asincrona. Utilizado por LangChain, CrewAI, Vertex AI Agent Builder, AutoGen y Mastra. Disenado para sistemas multiagente donde un agente delega trabajo a otro.

**ACP (Agent Communication Protocol)** -- creado por BeeAI (IBM). Un protocolo basado en ejecuciones para orquestadores empresariales. ACP se esta fusionando con A2A bajo la Linux Foundation, pero el endpoint del protocolo sigue siendo util para los clientes ACP existentes.

Cada protocolo tiene un mecanismo de descubrimiento diferente, un modelo de ejecucion diferente y un conjunto diferente de frameworks compatibles. Un agente TRON que solo habla MCP es invisible para LangChain. Un agente que solo habla A2A es invisible para Claude. Hasta ahora, ningun proyecto TRON soportaba mas de uno de estos protocolos.

## Lo que MERX soporta ahora

Desde abril de 2026, MERX soporta los tres protocolos desde un unico despliegue:

| Protocolo | Descubrimiento | Ejecucion | Frameworks compatibles |
|----------|-----------|-----------|----------------------|
| MCP | `merx.exchange/mcp/sse` | Llamadas a herramientas (solicitud-respuesta) | Claude, Cursor, Windsurf, cualquier cliente MCP |
| A2A | `merx.exchange/.well-known/agent.json` | Tareas (asincronas, streaming SSE) | LangChain, CrewAI, Vertex AI, AutoGen, Mastra |
| ACP | `merx.exchange/.well-known/agent-manifest.json` | Ejecuciones (asincronas, long-polling) | BeeAI, IBM watsonx, frameworks ACP |

Los tres protocolos comparten el mismo backend. Cuando una tarea A2A llama a `buy_energy`, ejecuta exactamente la misma logica de enrutamiento de ordenes que una llamada a la herramienta MCP `create_order`. Los protocolos son diferentes puntos de entrada al mismo motor de agregacion MERX.

## MCP: 55 herramientas para integracion directa

MCP es el punto de integracion mas profundo. El servidor MCP de MERX proporciona 55 herramientas organizadas en 15 categorias:

- **Inteligencia de precios** (5 herramientas): precios en tiempo real de los 7 proveedores, enrutamiento al mejor precio, datos historicos, analisis de mercado
- **Comercio de recursos** (4 herramientas): crear ordenes, listar ordenes, obtener detalles de ordenes, asegurar recursos
- **Operaciones con tokens** (4 herramientas): enviar TRX, enviar tokens TRC-20, aprobar asignaciones, consultar informacion de tokens
- **Swaps en DEX** (3 herramientas): obtener cotizaciones de SunSwap, ejecutar swaps, consultar precios de tokens
- **Ordenes permanentes** (4 herramientas): automatizacion del lado del servidor con disparadores de precio, programaciones, alertas de saldo
- **Consultas on-chain** (5 herramientas): informacion de cuentas, saldos, transacciones, datos de bloques
- Ademas de 28 herramientas mas en estimacion, conveniencia, contratos, red, incorporacion, pagos, ejecucion de intenciones y gestion de sesiones

El servidor MCP tambien proporciona 30 prompts para flujos de trabajo guiados y 21 recursos para acceso estructurado a datos.

### Conectar en una sola linea

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Sin instalacion. Sin API keys para herramientas de solo lectura. 22 herramientas disponibles inmediatamente.

## A2A: 6 habilidades para frameworks de orquestacion

A2A expone a MERX como un agente especialista al que los orquestadores pueden delegar tareas. La Agent Card en `/.well-known/agent.json` anuncia 7 habilidades:

| Habilidad | Descripcion | Autenticacion requerida |
|-------|-------------|---------------|
| `buy_energy` | Comprar energia delegada del mercado agregado | Si |
| `get_prices` | Precios actuales de energia de los 7 proveedores | No |
| `analyze_prices` | Analisis de mercado con tendencias y recomendaciones | No |
| `check_balance` | Saldo de cuenta y asignacion de recursos on-chain | Opcional |
| `ensure_resources` | Aprovisionamiento declarativo de recursos (comprar solo el deficit) | Si |
| `create_standing_order` | Reglas de automatizacion del lado del servidor | Si |

### Como funciona A2A

El protocolo A2A utiliza un modelo basado en tareas. Un orquestador envia una tarea, MERX la procesa de forma asincrona y el orquestador recupera el resultado.

**Paso 1: Descubrir el agente**

```bash
curl https://merx.exchange/.well-known/agent.json
```

Esto devuelve la Agent Card con las 7 habilidades, sus esquemas de entrada, modos soportados y requisitos de autenticacion.

**Paso 2: Enviar una tarea**

```bash
curl -X POST https://merx.exchange/a2a/tasks/send \
  -H "Content-Type: application/json" \
  -d '{
    "id": "task-001",
    "message": {
      "role": "user",
      "parts": [{
        "type": "data",
        "data": { "action": "get_prices" }
      }]
    }
  }'
```

La respuesta se devuelve inmediatamente con estado `submitted`. La tarea se procesa en segundo plano.

**Paso 3: Obtener el resultado**

```bash
curl https://merx.exchange/a2a/tasks/task-001
```

La respuesta incluye el estado de la tarea (`completed`, `failed`, etc.) y los artefactos del resultado con datos de precios de los 7 proveedores.

**Paso 4: Transmitir eventos (opcional)**

```bash
curl -N https://merx.exchange/a2a/tasks/task-001/events
```

El stream SSE entrega transiciones de estado en tiempo real: `submitted` a `working` a `completed`.

### Enrutamiento de habilidades

Las tareas A2A pueden usar datos estructurados o lenguaje natural. El procesador de tareas enruta automaticamente:

- **Estructurado**: `{ "action": "buy_energy", "energy_amount": 65000, "target_address": "T..." }` se enruta directamente a la habilidad buy_energy
- **Lenguaje natural**: "What is the current energy price?" coincide con el patron de palabras clave y se enruta a get_prices

## ACP: Ejecucion basada en runs para empresas

ACP utiliza un modelo basado en ejecuciones similar a A2A pero con una superficie de API diferente. El manifiesto en `/.well-known/agent-manifest.json` declara las mismas 7 capacidades.

```bash
# Crear una ejecucion
curl -X POST https://merx.exchange/acp/v1/agents/merx-tron-agent/runs \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "merx-tron-agent",
    "input": [{
      "role": "user",
      "parts": [{
        "contentType": "application/json",
        "content": "{\"action\":\"get_prices\"}"
      }]
    }]
  }'

# Consultar resultado (con long-polling)
curl "https://merx.exchange/acp/v1/runs/{runId}?wait=true"
```

El parametro `?wait=true` habilita long-polling: la solicitud se bloquea hasta 30 segundos esperando a que la ejecucion termine, reduciendo la necesidad de consultas repetidas.

Nota: ACP se esta fusionando con A2A bajo la Linux Foundation. El endpoint seguira operando para clientes existentes, pero las nuevas integraciones deberian usar A2A.

## Arquitectura: Un backend, tres puntos de entrada

Los tres protocolos comparten la misma ruta de ejecucion:

```
MCP Tool Call ─┐
               ├──► MERX API ──► Provider Router ──► 7 Energy Providers
A2A Task ──────┤                                     (Netts, CatFee, TEM,
               ├──► MERX API                          ITRX, TronSave, Feee,
ACP Run ───────┘                                      PowerSun)
```

Los manejadores de A2A y ACP se ejecutan dentro del servicio API existente (`services/api/src/agent-protocols/`). Realizan llamadas HTTP internas a los mismos endpoints REST que utilizan el servidor MCP y el panel de control web. Esto significa:

- **Mismos precios**: todos los protocolos ven los mismos datos de proveedores en tiempo real
- **Mismo enrutamiento**: las ordenes pasan por la misma logica de proveedor mas economico
- **Misma autenticacion**: X-API-Key funciona en todos los protocolos
- **Misma fiabilidad**: la logica de failover y reintentos se aplica por igual

El estado de tareas y ejecuciones se almacena en Redis con un TTL de 24 horas. No se requieren escrituras en base de datos para las operaciones del protocolo.

## Por que importa el soporte multiprotocolo

### Para desarrolladores

Estas construyendo una integracion TRON. Tu framework de orquestacion usa LangChain (A2A). El bot de tu companero usa Claude (MCP). Tu cliente empresarial requiere BeeAI (ACP). Con un agente de un solo protocolo, necesitas tres integraciones diferentes al mismo servicio subyacente.

Con MERX, los tres se conectan a la misma plataforma. Una API key. Un conjunto de documentacion. Un canal de soporte.

### Para constructores de agentes

Los sistemas multiagente se estan convirtiendo en estandar. Un agente de planificacion se coordina con un agente de trading, un agente de monitoreo y un agente de reportes. Estos agentes pueden ejecutarse en diferentes frameworks. Un crew de CrewAI podria delegar compras de energia a MERX via A2A mientras un agente Claude monitorea precios via MCP.

MERX maneja ambos sin que los agentes conozcan la eleccion de protocolo del otro.

### Para el ecosistema TRON

Mayor cobertura de protocolos significa mas integraciones potenciales. Cada framework de IA que soporta A2A ahora puede acceder a los mercados de energia TRON. Cada cliente MCP puede optimizar los costos de transaccion TRON. El mercado total direccionable para servicios de energia TRON se expande con cada protocolo soportado.

## Donde esta listado MERX

MERX es el unico proyecto TRON con presencia en los directorios de MCP y A2A:

**Registros MCP:**
- [Glama](https://glama.ai/mcp/servers/Hovsteder/merx-mcp)
- [Smithery](https://smithery.ai/servers/powersun/merx)
- [Registro oficial MCP](https://registry.modelcontextprotocol.io/servers/exchange.merx/mcp)
- mcp.so
- PulseMCP

**Directorios A2A:**
- [awesome-a2a](https://github.com/pab1it0/awesome-a2a) (seccion de Servicios Financieros)
- [a2aregistry.in](https://a2aregistry.in)

## Primeros pasos

### MCP (Claude, Cursor, Windsurf)

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

### A2A (LangChain, CrewAI, Vertex AI, AutoGen)

URL de descubrimiento: `https://merx.exchange/.well-known/agent.json`

### ACP (BeeAI)

URL de descubrimiento: `https://merx.exchange/.well-known/agent-manifest.json`

### Documentacion

- [Descripcion general de protocolos de agentes](https://merx.exchange/agents)
- [Servidor MCP (55 herramientas)](https://merx.exchange/mcp)
- [Documentacion del protocolo A2A](https://merx.exchange/docs/tools/a2a)
- [Documentacion del protocolo ACP](https://merx.exchange/docs/tools/acp)
- [GitHub](https://github.com/Hovsteder/merx-mcp)

---

*Tags: tron mcp server, a2a protocol, acp protocol, tron ai agent, ai agent tron energy, langchain tron, crewai tron, multi-protocol agent, merx exchange*

---

**Pruebalo ahora con IA**

Agrega a tu cliente MCP:

```json
{ "merx": { "url": "https://merx.exchange/mcp/sse" } }
```

O descubre via A2A:

```bash
curl https://merx.exchange/.well-known/agent.json
```

Luego pregunta: "What are the current TRON energy prices across all providers?"
