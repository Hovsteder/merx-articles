# Monitores de Delegación: Nunca Dejes que tu Energía TRON Expire

## El Problema de la Expiración

Las delegaciones de energía TRON tienen una duración fija. Cuando compras energía por 1 hora, obtienes exactamente 1 hora. Cuando esa hora termina, la delegación se revoca y tu dirección vuelve a cero energía delegada. Si tu aplicación envía una transferencia USDT 61 minutos después de que comenzó la delegación, esa transacción quema TRX al precio completo.

Para aplicaciones que funcionan 24/7, esto crea una carga de gestión. Necesitas rastrear cuándo expira cada delegación, comprar un reemplazo antes de que la actual caduque y manejar la brecha entre la expiración y la llegada de la nueva delegación. Si pierdes una renovación por apenas unos segundos, las transacciones ejecutadas durante esa ventana incurren en la quema completa de TRX.

Los monitores de delegación MERX eliminan este problema. Observan tus delegaciones, rastrean los tiempos de expiración y renuevan automáticamente antes de que se agote tu energía. El monitor se ejecuta en el servidor MERX 24/7 - no depende de que tu aplicación esté en línea, de que tu cliente MCP esté conectado o de que tu agente de IA esté activo.

## Tipos de Monitor

El sistema de monitoreo MERX admite tres tipos de monitores, cada uno diseñado para una preocupación operativa diferente.

### delegation_expiry

Observa las delegaciones de energía activas y toma medidas antes de que expiren.

```
Tool: create_monitor
Input: {
  "type": "delegation_expiry",
  "config": {
    "address": "TYourAddress...",
    "renew_before_minutes": 10,
    "auto_renew": true,
    "max_price_per_unit": 0.00006,
    "renewal_amount": 500000,
    "renewal_duration_hours": 24
  },
  "notifications": {
    "channels": ["webhook"],
    "webhook_url": "https://your-app.com/hooks/energy-monitor"
  }
}
```

Este monitor rastrea todas las delegaciones activas a la dirección especificada. Cuando cualquier delegación está a 10 minutos de expirar, el monitor toma medidas:

1. Si `auto_renew` es verdadero y el precio actual mejor es igual o inferior a `max_price_per_unit`, coloca automáticamente una nueva orden por `renewal_amount` de energía con duración de `renewal_duration_hours`.
2. Si el precio excede `max_price_per_unit`, el monitor envía una notificación en lugar de comprar, alertándote de que los precios son demasiado altos para la renovación automática.
3. Si `auto_renew` es falso, el monitor solo envía notificaciones, dándote tiempo para actuar manualmente.

El parámetro `renew_before_minutes` es crítico. Establécelo lo suficientemente alto para que la nueva delegación pueda colocarse, confirmarse y activarse antes de que expire la anterior. Una ventana de 10 minutos proporciona tiempo amplio para la colocación de orden (segundos), procesamiento del proveedor (1-2 minutos) y confirmación de delegación (3-6 segundos), con margen para congestión de red.

### balance_threshold

Monitorea el saldo de energía en cadena y se activa cuando cae por debajo de un nivel especificado.

```
Tool: create_monitor
Input: {
  "type": "balance_threshold",
  "config": {
    "address": "TYourAddress...",
    "energy_threshold": 200000,
    "bandwidth_threshold": 2000,
    "action": "ensure_resources",
    "energy_target": 500000,
    "bandwidth_target": 5000
  },
  "notifications": {
    "channels": ["telegram"],
    "telegram_chat_id": "-1001234567890"
  }
}
```

A diferencia de `delegation_expiry`, que se basa en el tiempo, `balance_threshold` se basa en el consumo. Se activa cuando el energía disponible de tu dirección cae por debajo de 200,000, independientemente del motivo. Esto cubre escenarios que el monitoreo de expiración no puede:

- Múltiples transacciones consumiendo energía más rápido de lo esperado
- Una delegación siendo revocada anticipadamente por el proveedor
- Energía en staking disminuyendo debido a un cambio en los parámetros de staking

Cuando se activa, la acción `ensure_resources` verifica el saldo actual, calcula el déficit para alcanzar el objetivo y compra solo lo necesario.

### price_alert

Monitorea los precios del mercado de energía y notifica cuando se cumplen las condiciones.

```
Tool: create_monitor
Input: {
  "type": "price_alert",
  "config": {
    "condition": "below",
    "threshold": 0.000040,
    "resource_type": "energy",
    "duration_hours": 24,
    "cooldown_minutes": 360
  },
  "notifications": {
    "channels": ["webhook", "telegram"],
    "webhook_url": "https://your-app.com/hooks/price-alert",
    "telegram_chat_id": "-1001234567890"
  }
}
```

Las alertas de precio son solo informativas por defecto - notifican sin comprar. Esto es útil para equipos que desean comprar energía a precios históricamente bajos pero prefieren una persona revisando las decisiones de compra.

El parámetro `cooldown_minutes` previene la fatiga de alertas. Si el precio se mantiene por debajo del umbral durante horas, recibes una alerta cada 6 horas en lugar de una cada 30 segundos.

## Renovación Automática en Detalle

El flujo de renovación automática para monitores `delegation_expiry` funciona de la siguiente manera:

### Paso 1: Rastrear Delegaciones Activas

MERX mantiene un registro de todas las órdenes de energía colocadas a través de la plataforma. Cada orden incluye la hora de inicio de delegación, duración y hora de expiración. El monitor verifica estos registros contra la hora actual.

```
Active delegations for TYourAddress:
  Order #1234: 500,000 energy, expires 2026-03-30T14:00:00Z (47 min remaining)
  Order #1235: 200,000 energy, expires 2026-03-30T15:30:00Z (137 min remaining)
```

### Paso 2: Evaluar Ventana de Renovación

Cuando cualquier delegación entra en la ventana de renovación (p. ej., 10 minutos antes de expirar):

```
Order #1234: 500,000 energy, expires in 9 minutes 42 seconds
-> Within renewal window (10 minutes)
-> Initiating renewal check
```

### Paso 3: Verificación de Precio

El monitor consulta los precios de mercado actuales:

```
Best price for 500,000 energy / 24 hours:
  Provider: sohu
  Price: 26.30 TRX
  Price per unit: 0.0000526

Max allowed: 0.00006 per unit
  0.0000526 <= 0.00006: PASS
```

### Paso 4: Colocar Orden de Renovación

```
Order placed:
  Amount: 500,000 energy
  Duration: 24 hours
  Provider: sohu
  Cost: 26.30 TRX
  Target: TYourAddress...
  Status: CONFIRMED
```

### Paso 5: Verificar Delegación

El monitor sondea la dirección para confirmar que la nueva delegación ha sido recibida:

```
Delegation confirmed:
  Previous energy: 487,231 (remaining from expiring delegation)
  New energy: 987,231 (previous + new delegation)
```

### Paso 6: Notificar

```
Webhook sent:
{
  "event": "delegation_renewed",
  "address": "TYourAddress...",
  "old_order": "1234",
  "new_order": "1236",
  "energy_amount": 500000,
  "cost_trx": 26.30,
  "provider": "sohu",
  "new_expiry": "2026-03-31T13:50:18Z"
}
```

El flujo completo se completa en menos de 30 segundos. La dirección nunca experimentó una brecha en la cobertura de energía.

## Protección de Precio

El parámetro `max_price_per_unit` en monitores de renovación automática es un mecanismo de seguridad crítico. Los precios de energía pueden aumentar durante períodos de alta demanda. Sin protección de precio, una renovación automática durante un pico de precio podría costar 2-3 veces la tasa normal.

Cuando el precio de mercado excede el máximo:

```
Best price for 500,000 energy / 24 hours:
  Provider: catfee
  Price: 42.50 TRX
  Price per unit: 0.0000850

Max allowed: 0.00006 per unit
  0.0000850 > 0.00006: FAIL - Price exceeds maximum

Action: Notification sent instead of purchase
  "Energy delegation expiring in 8 minutes. Auto-renewal skipped:
   market price 0.0000850 exceeds maximum 0.00006. Manual action required."
```

La notificación te da la opción de:
- Aceptar el precio más alto y colocar una orden manual
- Esperar a que los precios se normalicen y aceptar una brecha breve en la cobertura
- Ajustar el max_price_per_unit en el monitor

### Establecer el Precio Máximo Correcto

Para establecer un precio máximo efectivo:

1. Consulta el recurso `get_price_history` para los últimos 30 días
2. Identifica el precio del percentil 95 (el precio que 95% de las cotizaciones estuvieron en o por debajo)
3. Establece tu máximo en o ligeramente por encima de este nivel

Este enfoque detecta fluctuaciones normales mientras rechaza picos de precio genuinos.

## Ejecutándose 24/7 Sin un Agente

Esta es la diferenciación clave de los monitores MERX en comparación con la lógica del lado del agente. Un agente de IA se ejecuta durante una sesión de conversación. Cuando la sesión termina, el agente se detiene. Si implementaras el rastreo de delegación en el código de tu agente, solo funcionaría mientras el agente está activo.

Los monitores MERX se ejecutan en la infraestructura del servidor MERX:

- **Persistencia PostgreSQL** - Las configuraciones de monitor se almacenan en la base de datos y sobreviven a los reinicios del servidor
- **Evaluación del lado del servidor** - Los disparadores se evalúan por el proceso backend de MERX, no por ningún cliente
- **Independiente de conexiones MCP** - Ningún cliente necesita estar conectado para que los monitores funcionen
- **Recuperación de fallos** - Si el servicio MERX se reinicia, los monitores se reanudan automáticamente desde su último estado conocido

Un agente crea un monitor una vez. Ese monitor se ejecuta indefinidamente (o hasta su fecha de expiración) independientemente de si el agente se conecta nuevamente.

```
Day 1: Agent creates delegation_expiry monitor
Day 2: Agent is offline. Monitor renews delegation at 14:00.
Day 3: Agent is offline. Monitor renews delegation at 13:55.
Day 7: Agent reconnects. Checks monitor history:
  - 6 successful auto-renewals
  - 0 gaps in energy coverage
  - Total spent: 157.80 TRX
  - Average price: 0.0000526 per energy unit
```

## Combinar Monitores para Cobertura Robusta

Un único tipo de monitor no puede cubrir todos los modos de fallo. La configuración recomendada para uso en producción combina dos o tres tipos de monitor:

### Configuración Recomendada

```
Monitor 1: delegation_expiry
  Purpose: Proactive renewal before expiry
  Config:
    renew_before_minutes: 10
    auto_renew: true
    max_price: 0.00006
    renewal_amount: 500,000
    renewal_duration: 24 hours

Monitor 2: balance_threshold
  Purpose: Catch unexpected energy depletion
  Config:
    energy_threshold: 100,000
    action: ensure_resources
    energy_target: 500,000

Monitor 3: price_alert
  Purpose: Opportunity buying at low prices
  Config:
    condition: below
    threshold: 0.000035
    cooldown: 360 minutes
    action: notify_only
```

Monitor 1 maneja el caso normal - renovaciones programadas a precios aceptables. Monitor 2 maneja casos anormales - picos de consumo de energía repentinos, revocaciones anticipadas de delegación o Monitor 1 siendo bloqueado por protección de precio. Monitor 3 te alerta sobre excepcionales oportunidades para compra manual a granel.

Juntos, estos tres monitores proporcionan:
- Cobertura de energía sin brechas en condiciones normales
- Respaldo automático cuando los precios suben temporalmente
- Alertas para oportunidades de optimización de costos

## Gestionar Monitores

### Listar Monitores Activos

```
Tool: list_monitors

Response:
{
  "monitors": [
    {
      "id": "mon_abc123",
      "type": "delegation_expiry",
      "status": "active",
      "address": "TYourAddress...",
      "last_triggered": "2026-03-30T02:00:00Z",
      "total_renewals": 14,
      "total_spent_trx": 368.20,
      "next_expiry": "2026-03-31T02:00:00Z"
    },
    {
      "id": "mon_def456",
      "type": "balance_threshold",
      "status": "active",
      "address": "TYourAddress...",
      "last_triggered": "2026-03-28T15:42:00Z",
      "total_triggers": 2,
      "total_spent_trx": 52.60
    }
  ]
}
```

### Historial de Monitor

Cada monitor mantiene un registro de ejecución detallado:

```json
{
  "history": [
    {
      "timestamp": "2026-03-30T02:00:12Z",
      "trigger_reason": "Delegation expiring in 9m48s",
      "action_taken": "auto_renew",
      "order_id": "ord_xyz789",
      "energy_purchased": 500000,
      "cost_trx": 26.30,
      "provider": "sohu",
      "status": "success"
    },
    {
      "timestamp": "2026-03-29T01:55:33Z",
      "trigger_reason": "Delegation expiring in 9m27s",
      "action_taken": "notification_only",
      "reason": "Price 0.0000780 exceeds max 0.0000600",
      "status": "skipped"
    }
  ]
}
```

Este historial proporciona trazabilidad completa. Puedes ver exactamente cuándo ocurrió cada renovación, cuánto costó, qué proveedor se utilizó y por qué se omitieron las renovaciones.

## Economía de Nunca Expirar

Considera una aplicación que procesa 500 transferencias USDT por día. Cada transferencia requiere aproximadamente 65,000 energía.

Sin monitores (gestión manual con brechas ocasionales):

```
Average gaps per week: 3 (each lasting ~15 minutes)
Transactions during gaps: ~15
TRX burned during gaps: ~15 x 27 = 405 TRX/week
Annual burn from gaps: ~21,060 TRX (~$5,475)
```

Con monitores de delegación MERX:

```
Gaps per week: 0
TRX burned from gaps: 0
Monitor cost (auto-renewal): ~0 additional (same energy would be purchased anyway)
Annual savings: ~$5,475
```

Los monitores no cuestan extra. Estarías comprando la misma energía de cualquier forma - los monitores solo aseguran que no haya brechas entre compras. El ahorro proviene enteramente de eliminar la quema de TRX durante ventanas de expiración sin gestionar.

## Conclusión

Las delegaciones de energía expiran. Este es un hecho de la red TRON que no puede evitarse. Lo que puede evitarse es el costo de dejar que expiren sin un reemplazo listo.

Los monitores de delegación MERX convierten un proceso manual y propenso a errores en un sistema automatizado y confiable. Se ejecutan en el servidor, independientemente de cualquier conexión de cliente. Renuevan delegaciones antes de que expiren. Respetan tus límites de precio. Te notifican sobre excepciones.

Configúralos una vez. Nunca pienses en la expiración de delegación nuevamente.

---

**Enlaces:**
- Plataforma MERX: [https://merx.exchange](https://merx.exchange)
- Servidor MCP (GitHub): [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
- Servidor MCP (npm): [https://www.npmjs.com/package/merx-mcp](https://www.npmjs.com/package/merx-mcp)


## Pruébalo Ahora con IA

Añade MERX a Claude Desktop o cualquier cliente compatible con MCP -- sin instalación, sin clave API necesaria para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregunta a tu agente de IA: "¿Cuál es la energía TRON más barata en este momento?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)