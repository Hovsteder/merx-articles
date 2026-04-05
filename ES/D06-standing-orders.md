# Órdenes Permanentes: Automatiza Compras de Energía TRON 24/7

## El Enfoque Manual No Escala

Si estás comprando energía TRON manualmente, lo estás haciendo mal. No porque el proceso sea difícil - colocar una orden toma segundos - sino porque el mercado de energía es dinámico. Los precios fluctúan a lo largo del día. Las delegaciones expiran en horarios fijos. Las necesidades de energía de tu aplicación cambian según el volumen de transacciones. Gestionar todo esto manualmente significa o vigilancia constante o dinero desperdiciado.

Las órdenes permanentes resuelven esto dejándote definir reglas que se ejecutan automáticamente, 24 horas al día, 7 días a la semana. Cuando se cumple una condición - el precio cae por debajo de tu umbral, se dispara un horario, o tu saldo de energía es demasiado bajo - MERX coloca la orden en tu nombre con los parámetros que definiste con anticipación.

Este artículo cubre el sistema completo de órdenes permanentes: tipos de disparadores, tipos de acciones, controles presupuestarios, y los patrones prácticos que ahorran más dinero.

## Cómo Funcionan las Órdenes Permanentes

Una orden permanente es una regla persistente almacenada en la base de datos PostgreSQL de MERX. Consta de tres componentes:

1. **Disparador** - la condición que activa la orden
2. **Acción** - qué sucede cuando se dispara el disparador
3. **Restricciones** - límites presupuestarios, frecuencia de ejecución y expiración

El backend de MERX evalúa los disparadores continuamente. Los disparadores basados en precio se verifican cada vez que llega una nueva cotización de precio de cualquier proveedor (típicamente cada 30 segundos). Los disparadores basados en horario usan expresiones cron evaluadas por el servidor. Los disparadores basados en saldo se verifican periódicamente mediante sondeo del estado en cadena.

Cuando se dispara un disparador y se cumplen todas las restricciones, la acción se ejecuta inmediatamente sin intervención manual alguna.

## Crear una Orden Permanente

El servidor MCP expone órdenes permanentes a través de la herramienta `create_standing_order`:

```
Tool: create_standing_order
Input: {
  "trigger": {
    "type": "price_below",
    "threshold": 0.00005,
    "resource_type": "energy"
  },
  "action": {
    "type": "buy_resource",
    "params": {
      "energy_amount": 500000,
      "duration_hours": 24,
      "target_address": "TYourAddress..."
    }
  },
  "constraints": {
    "max_daily_spend_trx": 100,
    "max_executions_per_day": 3,
    "expires_at": "2026-06-30T00:00:00Z"
  }
}
```

Esta orden comprará 500.000 energía por 24 horas siempre que el precio de mercado caiga por debajo de 0.00005 TRX por unidad de energía, hasta 3 veces por día, gastando no más de 100 TRX diarios.

## Tipos de Disparadores

### price_below

Se dispara cuando el mejor precio de energía disponible cae por debajo de tu umbral especificado.

```json
{
  "type": "price_below",
  "threshold": 0.00005,
  "resource_type": "energy"
}
```

Este es el disparador más común para optimización de costos. Los precios de energía en TRON fluctúan según la oferta y demanda. Durante horas fuera de pico (típicamente 00:00-06:00 UTC), los precios frecuentemente caen 15-25% por debajo de los niveles pico. Un disparador `price_below` te permite comprar energía automáticamente durante estas ventanas sin monitorear precios tú mismo.

**Consejo práctico:** Establece tu umbral en el percentil 25 de los precios de los últimos 7 días. Esto significa que compras cuando los precios están en el cuarto inferior de su rango reciente - lo suficientemente barato para ahorrar dinero, lo suficientemente frecuente para mantener tu dirección abastecida.

### schedule

Se dispara en un horario cron, independientemente de las condiciones del mercado.

```json
{
  "type": "schedule",
  "cron": "0 */6 * * *"
}
```

Este disparador se activa cada 6 horas. Útil para aplicaciones con necesidades de energía predecibles - si sabes que tu aplicación procesa la mayoría de transacciones durante horario comercial, programa compras de energía para que lleguen justo antes del apogeo.

Patrones cron comunes:

```
"0 */6 * * *"    - Cada 6 horas
"0 8 * * 1-5"    - Cada día laboral a las 8:00 UTC
"*/30 * * * *"   - Cada 30 minutos
"0 0 * * *"      - Diariamente a medianoche UTC
```

### balance_below

Se dispara cuando la energía disponible de tu dirección cae por debajo de un umbral.

```json
{
  "type": "balance_below",
  "energy_threshold": 100000,
  "check_address": "TYourAddress..."
}
```

Este disparador es reactivo - asegura que tu dirección nunca se quede sin energía reabasteciendo automáticamente cuando el saldo es bajo. Combinado con una acción `buy_resource`, crea un suministro de energía que se mantiene automáticamente.

**Cómo funciona:** MERX sondea el saldo de recursos en cadena de la dirección especificada a intervalos regulares (cada 60 segundos por defecto). Cuando la energía disponible cae por debajo del umbral, se dispara el disparador.

**Advertencia importante:** Hay latencia inherente entre el saldo cayendo y la nueva energía llegando (intervalo de sondeo + colocación de orden + confirmación de delegación). Establece el umbral lo suficientemente alto para que tu dirección no se quede sin energía durante esta ventana. Si tu aplicación consume 50.000 energía por minuto, un umbral de 100.000 te da aproximadamente 2 minutos de tiempo de ejecución - suficiente para que MERX reabastezcas.

## Tipos de Acciones

### buy_resource

Compra energía o bandwidth del mercado.

```json
{
  "type": "buy_resource",
  "params": {
    "energy_amount": 500000,
    "duration_hours": 24,
    "target_address": "TYourAddress..."
  }
}
```

Esta es la acción estándar para procuración de energía. MERX selecciona automáticamente el proveedor más barato disponible en el momento de la ejecución. La `target_address` recibe la energía delegada.

### ensure_resources

Una acción de nivel superior que verifica recursos actuales antes de comprar.

```json
{
  "type": "ensure_resources",
  "params": {
    "target_address": "TYourAddress...",
    "energy_target": 500000,
    "bandwidth_target": 5000
  }
}
```

A diferencia de `buy_resource`, que siempre compra la cantidad especificada, `ensure_resources` primero verifica lo que la dirección ya tiene y solo compra el déficit. Si la dirección ya tiene 300.000 energía y el objetivo es 500.000, compra 200.000.

Esta es la acción más segura para disparadores `balance_below`, ya que previene sobre-compra cuando múltiples disparadores se activan en rápida sucesión.

### notify_only

Envía una notificación sin tomar ninguna acción en cadena.

```json
{
  "type": "notify_only",
  "params": {
    "channels": ["webhook", "telegram"],
    "message": "El precio de energía cayó por debajo del umbral"
  }
}
```

Usa esto cuando quieras conciencia sin automatización. La orden permanente monitorea la condición y te alerta, pero tú tomas la decisión de compra manualmente. Este es un buen punto de partida para equipos que aún no están cómodos con compras completamente automatizadas.

## Límites Presupuestarios y Restricciones

Las órdenes permanentes sin restricciones son peligrosas. Un disparador `price_below` sin límite de gasto podría drenar tu saldo completo durante una caída de precio sostenida. MERX proporciona varios mecanismos de restricción:

### max_daily_spend_trx

El máximo total de TRX que la orden permanente puede gastar en un período de 24 horas deslizante.

```json
{
  "max_daily_spend_trx": 200
}
```

Una vez que se alcanza este límite, el disparador continúa disparándose pero la acción se suprime hasta que la ventana de 24 horas avanza.

### max_executions_per_day

El número máximo de veces que la acción puede ejecutarse en un período de 24 horas deslizante.

```json
{
  "max_executions_per_day": 5
}
```

Esto previene ejecución rápida durante períodos volátiles. Incluso si el precio rebota arriba y abajo del umbral 20 veces en una hora, la acción se ejecuta como máximo 5 veces por día.

### min_interval_minutes

El tiempo mínimo entre ejecuciones consecutivas.

```json
{
  "min_interval_minutes": 60
}
```

Esto hace cumplir un período de enfriamiento. Después de que la acción se ejecuta, el disparador se suprime por 60 minutos independientemente de las condiciones.

### expires_at

La orden permanente se desactiva automáticamente después de esta marca de tiempo.

```json
{
  "expires_at": "2026-06-30T00:00:00Z"
}
```

Siempre establece una expiración en órdenes permanentes. Una orden permanente huérfana que corre indefinidamente después de que la has olvidado es una responsabilidad.

### Presupuesto Total

Un límite máximo en el gasto total a lo largo de la vida útil de la orden permanente.

```json
{
  "max_total_spend_trx": 5000
}
```

Una vez que el gasto de vida útil alcanza este límite, la orden permanente se desactiva permanentemente.

## Canales de Notificación

Las órdenes permanentes pueden enviar notificaciones en activación de disparador, ejecución exitosa, fallo de ejecución y límite presupuestario alcanzado.

### Webhook

```json
{
  "notification_channels": [
    {
      "type": "webhook",
      "url": "https://your-app.com/hooks/merx",
      "headers": { "Authorization": "Bearer your-secret" }
    }
  ]
}
```

El webhook recibe un payload JSON con los detalles del disparador, acción tomada y resultado de la ejecución.

### Telegram

```json
{
  "notification_channels": [
    {
      "type": "telegram",
      "chat_id": "-1001234567890"
    }
  ]
}
```

MERX envía un mensaje formateado al chat de Telegram especificado. Útil para equipos que monitorean operaciones a través de grupos de Telegram.

## Gestionar Órdenes Permanentes

### Listar Órdenes Activas

```
Tool: list_standing_orders

Response:
{
  "standing_orders": [
    {
      "id": "so_abc123",
      "trigger": { "type": "price_below", "threshold": 0.00005 },
      "action": { "type": "buy_resource", "energy_amount": 500000 },
      "status": "active",
      "executions_today": 1,
      "total_spent_trx": 247.50,
      "created_at": "2026-03-15T10:00:00Z",
      "last_executed": "2026-03-30T02:15:00Z"
    }
  ]
}
```

### Pausar y Reanudar

Las órdenes permanentes pueden pausarse sin eliminarlas. Una orden pausada retiene su configuración e historial de ejecución pero no se dispara.

### Eliminar

Elimina permanentemente la orden permanente. El historial de ejecución se retiene para propósitos de auditoría.

## Patrones Prácticos

### Patrón 1: Cobertura Optimizada por Costo 24/7

Para aplicaciones que necesitan cobertura continua de energía al costo más bajo posible:

```
Orden Permanente 1:
  Disparador: price_below 0.000045 TRX/energía
  Acción: compra 1.000.000 energía por 24 horas
  Restricción: máx 2 ejecuciones/día

Orden Permanente 2:
  Disparador: balance_below 200.000 energía
  Acción: ensure_resources a 500.000 energía (1 hora)
  Restricción: máx 100 TRX/día
```

La Orden 1 compra grandes cantidades de energía barata cuando los precios caen, proporcionando cobertura de 24 horas. La Orden 2 es la red de seguridad - si los precios nunca caen lo suficientemente bajo para disparar la Orden 1, la Orden 2 asegura que la dirección nunca se seque comprando a precio de mercado cuando el saldo es críticamente bajo.

### Patrón 2: Optimización de Horas Comerciales

Para aplicaciones con picos de hora predecibles:

```
Orden Permanente 1:
  Disparador: schedule "0 7 * * 1-5" (7 AM UTC, días laborales)
  Acción: compra 2.000.000 energía por 12 horas
  Restricción: máx 200 TRX/día

Orden Permanente 2:
  Disparador: schedule "0 19 * * 1-5" (7 PM UTC, días laborales)
  Acción: compra 500.000 energía por 14 horas
  Restricción: máx 50 TRX/día
```

Energía pesada antes de horario comercial, energía ligera para la noche. Las órdenes de fin de semana pueden ser separadas con cantidades más bajas.

### Patrón 3: Alerta de Precio Solamente

Para equipos que quieren toma de decisiones humana con monitoreo automatizado:

```
Orden Permanente:
  Disparador: price_below 0.000040 TRX/energía
  Acción: notify_only
  Canales: webhook + telegram
  Mensaje: "Energía en mínimo histórico - considera compra manual a granel"
```

Sin compras automatizadas. El equipo recibe alertas cuando los precios son excepcionalmente bajos y puede decidir si colocar una orden manual grande.

### Patrón 4: Auto-Escalado para Carga Variable

Para aplicaciones donde el volumen de transacciones varía impredeciblemente:

```
Orden Permanente 1:
  Disparador: balance_below 500.000 energía
  Acción: ensure_resources a 1.000.000 energía (1 hora)
  Restricción: min_interval 15 minutos, máx 500 TRX/día

Orden Permanente 2:
  Disparador: balance_below 100.000 energía
  Acción: ensure_resources a 500.000 energía (1 hora)
  Restricción: min_interval 5 minutos, máx 1000 TRX/día
```

La Orden 1 maneja carga normal con top-ups moderados. La Orden 2 es el soporte de alta urgencia para picos de tráfico, con un intervalo más corto y presupuesto más alto.

## Órdenes Permanentes e Integración de Agentes

Para agentes de IA, las órdenes permanentes son el mecanismo para comportamiento persistente. Una sesión de agente es temporal - cuando la conexión MCP se cierra, el agente ya no puede monitorear precios o comprar energía. Las órdenes permanentes se ejecutan en el servidor MERX, independientemente de cualquier conexión de cliente.

Un agente puede establecer órdenes permanentes durante una sesión y beneficiarse de ellas en todas las sesiones futuras:

```
Sesión 1 (configuración):
  El agente crea órdenes permanentes para compra basada en precio
  El agente crea monitores de saldo con renovación automática

Sesión 2 (días después):
  El agente verifica el historial de ejecución de órdenes permanentes
  La energía ha sido comprada automáticamente 14 veces
  Todas las transacciones ejecutadas con energía delegada
  Cero intervención manual requerida
```

Este es el camino hacia operaciones blockchain verdaderamente autónomas. El agente define las reglas una vez, y las reglas se ejecutan continuamente.

## Conclusión

La gestión manual de energía es un problema resuelto. Las órdenes permanentes reemplazan el monitoreo humano con reglas automatizadas que se ejecutan más rápido, más consistentemente, y a menor costo que cualquier proceso manual.

Define tus disparadores. Establece tus restricciones. Deja que MERX maneje el resto.

Tu aplicación se ejecuta 24/7. Tu procuración de energía también debería.

---

**Enlaces:**
- Plataforma MERX: [https://merx.exchange](https://merx.exchange)
- Servidor MCP (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- Servidor MCP (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)


## Pruébalo Ahora con IA

Añade MERX a Claude Desktop o cualquier cliente compatible con MCP -- cero instalación, sin API key necesaria para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregunta a tu agente IA: "¿Cuál es la energía TRON más barata en este momento?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)