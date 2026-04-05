# MERX Dashboard: Comprando Energía sin Escribir Código

No todo comprador de energía es desarrollador. No todo desarrollador quiere escribir código para una compra única. Y no todas las organizaciones tienen la capacidad de ingeniería para integrar una API antes de poder comenzar a ahorrar en costos de transacciones de TRON.

El dashboard de MERX en [merx.exchange](https://merx.exchange) proporciona una interfaz web completa para comprar energía, comparar precios de proveedores, gestionar tu saldo, rastrear pedidos y generar claves API -- todo sin escribir una sola línea de código. Este artículo recorre todas las características del dashboard y muestra cómo utilizarlo de manera efectiva.

## Primeros Pasos

Navega a [merx.exchange](https://merx.exchange) y crea una cuenta. El registro requiere una dirección de correo electrónico y contraseña. Sin KYC. Sin verificación de identidad. Sin tiempo de espera. Tu cuenta está activa inmediatamente.

Una vez que inicies sesión, llegarás a la vista principal del dashboard. La interfaz sigue un tema oscuro con tipografía de alto contraste -- sin ruido visual, sin decoración innecesaria, solo la información que necesitas para tomar decisiones de compra.

## El Panel de Precios

Lo primero que ves en el dashboard es el panel de precios en vivo. Este muestra los precios actuales de energía de los siete proveedores integrados, actualizados cada 30 segundos.

```
Provider      1h Price    1d Price    Available
-------------------------------------------------
Feee          28 SUN      22 SUN      Yes
Netts         31 SUN      24 SUN      Yes
itrx          32 SUN      23 SUN      Yes
CatFee        30 SUN      25 SUN      Yes
PowerSun      33 SUN      26 SUN      Yes
TronSave      35 SUN      28 SUN      Yes
SoHu          34 SUN      27 SUN      Yes
```

El panel resalta el proveedor más económico para cada categoría de duración. Los precios se muestran en SUN por unidad de energía -- la denominación estándar que permite comparación directa entre proveedores sin importar cómo cotizan internamente sus tarifas.

### Qué Significan los Precios

Cada precio representa el costo por unidad de energía para la duración de alquiler especificada. Para calcular el costo total de tu compra:

```
Total cost = energy_amount * price_per_unit

Example:
  65,000 energy * 28 SUN/unit = 1,820,000 SUN = 1.82 TRX
```

El dashboard realiza este cálculo automáticamente cuando ingresas un pedido. Ves el costo total tanto en SUN como en TRX antes de confirmar.

### Historial de Precios

Debajo de los precios en vivo, un gráfico histórico muestra cómo se han movido los precios durante las últimas 24 horas, 7 días o 30 días. Esto te ayuda a identificar patrones -- los precios tienden a ser más bajos durante las horas de menor actividad (00:00-08:00 UTC) y más altos durante los períodos de mayor transacción.

El gráfico no es solo informativo. Si estás realizando un pedido grande y existe flexibilidad de tiempo, esperar algunas horas para una caída de precio puede ahorrar un porcentaje significativo.

## Creando un Pedido

El formulario de creación de pedidos es la función central del dashboard. Aquí está el proceso:

### Paso 1: Ingresar Parámetros del Pedido

Completa tres campos:

- **Energy amount**: El número de unidades de energía que necesitas. Si no estás seguro, el dashboard proporciona botones de selección rápida para cantidades comunes:
  - 32,000 (transferencia USDT mínima)
  - 65,000 (transferencia USDT estándar)
  - 200,000 (intercambio DEX)
  - 500,000 (interacción de contrato compleja)
  - Cantidad personalizada

- **Target address**: La dirección de TRON que recibirá la delegación de energía. Esta es la dirección que ejecutará la transacción que necesita energía. El dashboard valida el formato de la dirección antes de continuar.

- **Duration**: Cuánto tiempo necesitas la energía. Las opciones típicamente incluyen 1 hora, 1 día, 3 días, 7 días, 14 días y 30 días.

### Paso 2: Revisar la Cotización

Después de ingresar tus parámetros, el dashboard muestra una cotización detallada:

```
Order Summary
-------------------------------------------------
Energy:           65,000 units
Duration:         1 hour
Target:           TYourAddress...
Best provider:    Feee
Price per unit:   28 SUN
Total cost:       1,820,000 SUN (1.82 TRX)
Account balance:  50.00 TRX
Balance after:    48.18 TRX
```

La cotización muestra exactamente qué proveedor cumplirá el pedido, el precio por unidad, el costo total y el impacto en tu saldo de cuenta. No hay comisiones ocultas ni recargos.

### Paso 3: Confirmar y Ejecutar

Haz clic en el botón de confirmación. El pedido se envía al backend de MERX, que lo enruta al proveedor seleccionado para su ejecución. La delegación típicamente se completa en segundos.

Una vez que se completa el pedido, el dashboard se actualiza para mostrar el estado del pedido, el hash de transacción de delegación en cadena y el tiempo restante en el alquiler.

## Gestionando Tu Saldo

### Depositando Fondos

Antes de poder comprar energía, necesitas un saldo de TRX en MERX. El proceso de depósito:

1. Navega a la sección de Saldo
2. Haz clic en "Depositar"
3. El dashboard muestra tu dirección de depósito única
4. Envía TRX a esa dirección desde cualquier billetera de TRON
5. El depósito se acredita automáticamente después de la confirmación en cadena

El servicio de monitoreo de depósitos observa las transacciones entrantes continuamente. Los créditos típicamente aparecen dentro de 1-2 minutos de que la transacción sea confirmada en cadena.

### Viendo Historial de Saldo

La sección de saldo muestra un historial completo de todos los cambios de saldo:

```
Date/Time            Type          Amount       Balance
-----------------------------------------------------------
2026-03-28 14:30     Deposit       +100.00 TRX  100.00 TRX
2026-03-28 14:35     Order #1247   -1.82 TRX    98.18 TRX
2026-03-28 16:20     Order #1248   -1.95 TRX    96.23 TRX
2026-03-29 09:00     Deposit       +50.00 TRX   146.23 TRX
2026-03-29 09:15     Order #1249   -5.40 TRX    140.83 TRX
```

Cada entrada corresponde a un registro de libro mayor de doble entrada en el sistema de contabilidad de MERX. La suma de todas las entradas siempre se reconcilia con tu saldo actual -- esto es verificable y auditable.

### Retiros

Si necesitas retirar TRX de tu cuenta de MERX:

1. Navega a la sección de Saldo
2. Haz clic en "Retirar"
3. Ingresa la dirección de TRON de destino
4. Ingresa la cantidad a retirar
5. Confirma el retiro

Los retiros se procesan mediante el servicio de treasury-signer y se transmiten a la red de TRON. El tiempo de procesamiento depende de las condiciones de la red pero típicamente se completa en algunos minutos.

## Historial de Pedidos

La vista de Historial de Pedidos proporciona una lista buscable y filtrable de todos tus pedidos anteriores. Cada entrada muestra:

- **Order ID**: Identificador único para referencia
- **Date/Time**: Cuándo se realizó el pedido
- **Energy Amount**: Cuántas unidades de energía se compraron
- **Duration**: Período de alquiler
- **Provider**: Qué proveedor cumplió el pedido
- **Price**: Precio por unidad en SUN
- **Total Cost**: Cantidad total cobrada en TRX
- **Status**: Pendiente, Activo, Completado o Fallido
- **Target Address**: Dónde se delegó la energía

### Filtrado y Búsqueda

Puedes filtrar pedidos por:

- Rango de fechas
- Estado (activo, completado, todos)
- Proveedor
- Dirección de destino

Esto es particularmente útil para organizaciones que compran energía para múltiples direcciones y necesitan rastrear costos por billetera o por aplicación.

### Detalles del Pedido

Hacer clic en cualquier pedido abre una vista de detalles con información completa:

```
Order #1247
-------------------------------------------------
Status:           Completed
Created:          2026-03-28 14:35:22 UTC
Energy:           65,000 units
Duration:         1 hour (expired)
Provider:         Feee
Price:            28 SUN/unit
Total:            1.82 TRX
Target:           TYourAddress...
TX Hash:          7f3a2b...
Delegation Start: 2026-03-28 14:35:30 UTC
Delegation End:   2026-03-28 15:35:30 UTC
```

El hash de transacción es un enlace al registro de delegación en cadena, verificable mediante cualquier explorador de bloques de TRON.

## Generando Claves API

Cuando estés listo para integrar MERX en tu aplicación de manera programática, el dashboard te permite generar claves API sin necesidad de contactar con soporte.

1. Navega a la sección de Configuración o API
2. Haz clic en "Generar Clave API"
3. Proporciona una etiqueta para la clave (p. ej., "Servidor de producción", "Entorno de staging")
4. La clave se muestra una vez -- cópiala inmediatamente
5. La clave se almacena hash en el servidor; no se puede recuperar nuevamente

Puedes gestionar múltiples claves API, cada una con su propia etiqueta. Si una clave se ve comprometida, revócala desde el dashboard y genera una nueva. Revocar una clave es inmediato -- todas las solicitudes que usen la clave revocada fallarán con un error de autenticación.

### Seguridad de Claves API

Las claves API autentican tus solicitudes a la API REST de MERX, feeds de WebSocket y SDKs. Trátalas como contraseñas:

- Nunca confirmes claves API en control de versiones
- Almacénalas en variables de entorno o gestores de secretos
- Usa claves separadas para desarrollo, staging y producción
- Rota claves periódicamente

El dashboard muestra la marca de tiempo de último uso para cada clave, facilitando la identificación y revocación de claves sin usar.

## Monitoreando Delegaciones Activas

El dashboard incluye una vista en tiempo real de tus delegaciones de energía activas:

```
Active Delegations
-------------------------------------------------
Target            Energy     Expires           Remaining
TAddr1...         65,000     2026-03-30 15:00  2h 30m
TAddr2...         200,000    2026-03-31 09:00  20h 30m
TAddr3...         500,000    2026-04-02 12:00  3d 2h 30m
```

Para cada delegación activa, puedes:

- Ver la hora exacta de vencimiento
- Ver el tiempo restante
- Ver cuánta energía está disponible
- Extender la delegación realizando un nuevo pedido para la misma dirección

El sistema actualmente no soporta cancelar una delegación activa antes de tiempo (la delegación de energía en TRON es una operación en cadena que se ejecuta hasta su duración especificada), pero siempre puedes extender una delegación existente comprando energía adicional para la misma dirección de destino.

## Herramienta de Estimación de Energía

El dashboard incluye una herramienta integrada de estimación de energía. Si no estás seguro de cuánta energía necesitará tu transacción, puedes simularla directamente desde el dashboard:

1. Ingresa la dirección del contrato (p. ej., el contrato USDT)
2. Selecciona la función (p. ej., transferencia)
3. Ingresa los parámetros (dirección del destinatario, cantidad)
4. Haz clic en "Estimar"

La herramienta llama a `triggerConstantContract` internamente y devuelve la energía exacta requerida para tu transacción específica contra el estado actual del contrato. Esto elimina conjeturas y previene la compra excesiva o insuficiente de energía.

## Para Quién Es Este Dashboard

### Operadores de Negocio

Si diriges un negocio que envía pagos USDT -- nóminas, pagos a proveedores, remesas -- no necesitas un desarrollador para integrar una API. Abre el dashboard, deposita TRX y comienza a comprar energía antes de cada lote de transferencias. Los ahorros de costos son inmediatos y significativos.

### Desarrolladores Evaluando MERX

Antes de comprometerte con una integración API, utiliza el dashboard para probar el servicio. Realiza algunos pedidos, observa los precios, verifica que las delegaciones lleguen en cadena como se espera. Una vez que estés satisfecho, genera una clave API y pasa al acceso programático.

### Equipos de Finanzas

Las vistas de historial de pedidos y saldos proporcionan los reportes que los equipos de finanzas necesitan: qué se gastó, cuándo, en qué y de qué proveedor. Exporta estos datos para reconciliación con tus sistemas de contabilidad internos.

### Usuarios Ocasionales

Si realizas transacciones de TRON ocasionalmente -- algunas pocas por semana o por mes -- el dashboard probablemente es todo lo que necesitas. Sin integración, sin código, sin mantenimiento. Solo inicia sesión, compra energía y ahorra el 90% en comisiones de transacción.

## Del Dashboard a la API

El dashboard y la API comparten el mismo backend. Cada acción que realizas en el dashboard -- verificar precios, realizar pedidos, ver historial -- se asigna directamente a un endpoint de API. Cuando estés listo para automatizar:

```
Dashboard action          API equivalent
-----------------------------------------------------------
View prices               GET /api/v1/prices
Create order              POST /api/v1/orders
View order                GET /api/v1/orders/:id
View balance              GET /api/v1/balance
Estimate energy           POST /api/v1/estimate
```

La transición de usuario de dashboard a usuario de API es perfecta. Tu cuenta, saldo e historial de pedidos se transfieren. La única adición es la clave API que generas desde el mismo dashboard.

Documentación completa: [https://merx.exchange/docs](https://merx.exchange/docs)
Plataforma: [https://merx.exchange](https://merx.exchange)


## Pruébalo Ahora con IA

Agrega MERX a Claude Desktop o a cualquier cliente compatible con MCP -- sin instalación, sin clave API necesaria para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregúntale a tu agente de IA: "¿Cuál es la energía de TRON más económica en este momento?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)