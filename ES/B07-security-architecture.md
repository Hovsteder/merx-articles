# Arquitectura de Seguridad de MERX: Cómo Protegemos Tus Fondos

Cuando una plataforma maneja transacciones financieras, la seguridad no es una característica - es la base sobre la que se asienta todo lo demás. Una única vulnerabilidad puede destruir la confianza del usuario de forma permanente. En MERX, las consideraciones de seguridad moldearon cada decisión arquitectónica desde el primer día, no como una ocurrencia posterior añadida apresuradamente.

Este artículo detalla la arquitectura de seguridad de MERX: cómo se protegen los fondos, cómo se mantiene la integridad de los datos, cómo el sistema se defiende contra vectores de ataque comunes, y los principios de diseño que hacen la plataforma resiliente.

---

## Principio 1: Sin Custodia de Claves Privadas

MERX nunca retiene, almacena, ni tiene acceso a tus claves privadas de TRON. Esta es una decisión de diseño fundamental que elimina toda una clase de ataques.

### Cómo Funciona

Cuando usas MERX para comprar energy, la delegación ocurre desde la dirección del proveedor hacia tu dirección. MERX orquesta esta transacción pero nunca necesita acceso a tu billetera. Tu clave privada permanece en tu dispositivo, en tu billetera de hardware, o dondequiera que la gestiones.

El flujo:

```
1. Le dices a MERX: "Delega 65.000 energy a TMyAddress"
2. MERX le dice al proveedor: "Delega 65.000 energy a TMyAddress"
3. El proveedor delega de TProviderAddress a TMyAddress
4. MERX verifica la delegación en cadena
5. Tu clave privada nunca fue involucrada
```

### Por Qué Importa

Si MERX fuera comprometida, los atacantes no podrían robar tu TRX u tokens porque MERX no tiene tus claves. Compara esto con plataformas que te requieren depositar tokens a una dirección controlada por la plataforma - esas plataformas retienen tus claves (o claves para tus fondos), creando un único punto de falla.

### La Excepción del Tesoro

MERX sí gestiona su propia dirección de tesoro para recibir depósitos y procesar retiros. La clave privada del tesoro se almacena como un secreto de Docker, accesible solo para el servicio `treasury-signer`. Nunca se expone al servicio API, la interfaz web del frontend, o cualquier otro componente. Más sobre este aislamiento abajo.

---

## Principio 2: Libro Mayor de Doble Entrada

Cada operación financiera en MERX crea una entrada de libro mayor emparejada. Este es el mismo principio contable utilizado por cada banco e institución financiera durante los últimos 700 años. Funciona.

### Cómo Funciona

Cada transacción crea dos entradas: un débito y un crédito. La suma de todos los débitos siempre es igual a la suma de todos los créditos. Si no lo son, algo está mal, y el sistema lo detecta inmediatamente.

```sql
-- Ejemplo de pago de orden
INSERT INTO ledger (account_id, type, amount_sun, direction)
VALUES
  ($user_id, 'ORDER_PAYMENT', 5525000, 'DEBIT'),
  ($provider_settlement, 'ORDER_PAYMENT', 5525000, 'CREDIT');
```

### Inmutabilidad

Los registros del libro mayor nunca se actualizan ni se eliminan. Si una transacción necesita ser invertida (por ejemplo, un reembolso), se crea una nueva entrada en el libro mayor con la dirección opuesta:

```sql
-- Reembolso: nuevas entradas, las entradas originales permanecen
INSERT INTO ledger (account_id, type, amount_sun, direction, reference_id)
VALUES
  ($user_id, 'REFUND', 5525000, 'CREDIT', $original_order_id),
  ($provider_settlement, 'REFUND', 5525000, 'DEBIT', $original_order_id);
```

La entrada de débito original nunca se modifica. La entrada de crédito de reembolso explícitamente hace referencia al original, creando un registro de auditoría completo.

### Por Qué Importa

Si un atacante compromete la capa de aplicación e intenta inflar el balance de un usuario, las entradas del libro mayor no equilibrarán. Las verificaciones de reconciliación regulares detectan esto inmediatamente:

```sql
-- Consulta de reconciliación: siempre debería retornar 0
SELECT SUM(CASE direction
  WHEN 'DEBIT' THEN amount_sun
  WHEN 'CREDIT' THEN -amount_sun
END) as imbalance
FROM ledger;
```

Cualquier resultado distinto de cero dispara una alerta inmediata e investigación.

---

## Principio 3: Operaciones de Balance Atómicas

Cada mutación de balance utiliza `SELECT FOR UPDATE` para prevenir condiciones de carrera. Esto no es opcional - se aplica a nivel de base de datos.

### El Problema de Condición de Carrera

Sin bloqueo adecuado, un usuario con un balance de 10 TRX podría enviar simultáneamente dos órdenes de 8 TRX cada una:

```
Hilo 1: SELECT balance WHERE user_id = 1    -> 10 TRX
Hilo 2: SELECT balance WHERE user_id = 1    -> 10 TRX
Hilo 1: ¿balance (10) >= orden (8)? SÍ      -> proceder
Hilo 2: ¿balance (10) >= orden (8)? SÍ      -> proceder
Hilo 1: UPDATE balance = 10 - 8 = 2 TRX
Hilo 2: UPDATE balance = 10 - 8 = 2 TRX

Resultado: El usuario gastó 16 TRX con solo 10 TRX de balance
```

### La Solución

```sql
BEGIN;

-- Bloquea la fila - la segunda transacción espera aquí
SELECT balance_sun FROM accounts
WHERE user_id = $1
FOR UPDATE;

-- Verifica el balance
-- Si insuficiente: ROLLBACK
-- Si suficiente: proceder

UPDATE accounts
SET balance_sun = balance_sun - $order_amount
WHERE user_id = $1
  AND balance_sun >= $order_amount;  -- Doble verificación en UPDATE

COMMIT;
```

`FOR UPDATE` adquiere un bloqueo a nivel de fila. La segunda transacción se bloquea hasta que la primera se confirma o se revierte. Después de que la primera transacción se confirma (reduciendo el balance a 2 TRX), la segunda transacción lee el balance actualizado (2 TRX) y rechaza correctamente la orden de fondos insuficientes.

---

## Principio 4: Validación de Entrada

Todas las entradas se validan con esquemas Zod antes del procesamiento. Esto incluye solicitudes de API, cargas útiles de webhook, respuestas de proveedores y mensajes de servicio internos.

### Validación de Entrada de API

```typescript
const CreateOrderSchema = z.object({
  energy: z.number()
    .int('Energy must be an integer')
    .min(10000, 'Minimum order is 10,000 energy')
    .max(100000000, 'Maximum order is 100,000,000 energy'),

  targetAddress: z.string()
    .regex(/^T[1-9A-HJ-NP-Za-km-z]{33}$/, 'Invalid TRON address format')
    .refine(isValidTronAddress, 'Invalid TRON address checksum'),

  duration: z.enum(['1h', '1d', '3d', '7d', '14d', '30d']),

  maxPrice: z.number()
    .positive()
    .optional(),

  idempotencyKey: z.string()
    .max(255)
    .optional()
});
```

Cada campo está tipado, acotado y validado. Ninguna entrada de usuario sin procesar llega a la lógica de negocio ni a consultas de base de datos.

### Prevención de Inyección SQL

Todas las consultas de base de datos utilizan declaraciones parametrizadas. La concatenación de cadenas nunca se usa para construir SQL:

```typescript
// Nunca esto:
const query = `SELECT * FROM users WHERE id = '${userId}'`;  // SQL injection

// Siempre esto:
const query = 'SELECT * FROM users WHERE id = $1';
const result = await db.query(query, [userId]);
```

Esto se aplica mediante revisión de código y reglas de linting. No hay mecanismo para interpolación de cadena SQL sin procesar en la base de código.

### Validación de Dirección TRON

Las direcciones TRON se validan en múltiples niveles:

1. **Verificación de formato**: debe coincidir con la expresión regular de dirección TRON (comienza con T, 34 caracteres, base58).
2. **Verificación de suma de comprobación**: la dirección incluye una suma de comprobación que detecta errores tipográficos.
3. **Verificación en cadena** (opcional): confirma que la dirección existe y está activada.

Enviar energy a una dirección inválida desperdicia recursos y no puede invertirse en cadena. La validación estricta previene esto.

---

## Principio 5: Aislamiento de Servicios

MERX se ejecuta como un conjunto de contenedores Docker aislados, cada uno con permisos mínimos y sin acceso innecesario.

### Arquitectura de Contenedores

```
Red Docker:
  |
  |-- api           (puerto 3000, acceso público)
  |-- price-monitor (sin puertos externos)
  |-- order-executor (sin puertos externos)
  |-- ledger        (sin puertos externos)
  |-- treasury-signer (sin puertos externos, acceso a secretos de Docker)
  |-- deposit-monitor (sin puertos externos)
  |-- withdrawal-executor (sin puertos externos)
  |
  |-- postgresql    (puerto 5432, solo interno)
  |-- redis         (puerto 6379, solo interno)
```

### Propiedades Clave de Aislamiento

- **El servicio API no puede acceder a la clave privada del tesoro.** Solo el contenedor `treasury-signer` puede leer el secreto de Docker que contiene la clave.
- **El monitor de precios no puede modificar balances.** Solo tiene acceso de lectura a APIs de proveedores y acceso de escritura a canales de precios de Redis.
- **El ejecutor de órdenes no puede modificar directamente el libro mayor.** Publica eventos de liquidación a Redis, que el servicio del libro mayor consume.
- **PostgreSQL y Redis no se exponen externamente.** Solo son accesibles desde dentro de la red Docker.

### Por Qué Importa

Si un atacante compromete el servicio API (el componente más expuesto), no puede:
- Acceder a la clave privada del tesoro (contenedor diferente, secreto de Docker).
- Modificar directamente entradas del libro mayor (servicio diferente, sin acceso de escritura a tablas del libro mayor).
- Eludir verificaciones de balance (se aplican a nivel de base de datos con FOR UPDATE).

El radio de explosión de cualquier compromiso de servicio único está limitado por diseño.

---

## Principio 6: Límite de Velocidad y Prevención de Abuso

### Límites de Velocidad de API

Cada punto final de API tiene límites de velocidad apropiados para su propósito:

```
Puntos finales públicos (precios, salud):          100 solicitudes/minuto
Lecturas autenticadas (órdenes, balance):         60 solicitudes/minuto
Escrituras autenticadas (crear orden):            30 solicitudes/minuto
Retiros:                                          5 solicitudes/minuto
```

Los límites de velocidad se aplican por clave de API, rastreados en Redis con ventanas deslizantes.

### Salvaguardas de Retiro

Los retiros son la operación de mayor riesgo (mover activos reales fuera de la plataforma). Las salvaguardas adicionales incluyen:

- **Límite de velocidad**: máximo 5 solicitudes de retiro por minuto.
- **Límites de cantidad**: límites de retiro diario por cuenta.
- **Retraso de confirmación**: los retiros grandes disparan un período de enfriamiento.
- **Verificación de balance**: `SELECT FOR UPDATE` asegura balance suficiente.
- **Idempotencia**: solicitudes de retiro duplicadas (misma clave de idempotencia) retornan el resultado original.

---

## Principio 7: Seguridad de Webhook

MERX envía notificaciones de webhook para actualizaciones de estado de orden, depósitos y otros eventos. Los webhooks se firman con HMAC-SHA256 para prevenir falsificación.

### Cómo Funcionan los Webhooks HMAC

```
1. MERX calcula: HMAC-SHA256(webhook_body, tu_secreto_webhook)
2. MERX incluye la firma en el encabezado X-Merx-Signature
3. Tu servidor recalcula el HMAC con el mismo secreto
4. Si las firmas coinciden: webhook genuino. Si no: falsificado, descartar.
```

### Verificación en Código

```typescript
import crypto from 'crypto';

function verifyWebhook(body: string, signature: string, secret: string): boolean {
  const computed = crypto
    .createHmac('sha256', secret)
    .update(body)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(computed)
  );
}
```

Ten en cuenta el uso de `timingSafeEqual` para prevenir ataques de temporización. Una comparación de cadena ingenua (`===`) filtraría información sobre la firma correcta a través de variaciones en el tiempo de respuesta.

---

## Principio 8: Gestión de Secretos

Ningún secreto se codifica nunca en el código fuente. Todos los valores sensibles se gestionan a través de variables de entorno y secretos de Docker.

### Variables de Entorno

```
# .env (nunca confirmado en git)
DATABASE_URL=postgresql://...
REDIS_URL=redis://...
API_JWT_SECRET=...
WEBHOOK_SIGNING_SECRET=...
TRON_API_KEY=...
```

### Secretos de Docker (para Valores de Alta Sensibilidad)

La clave privada del tesoro es demasiado sensible para variables de entorno (que pueden registrarse o filtrarse a través de inspección de procesos). Se almacena como un secreto de Docker:

```yaml
# docker-compose.yml
services:
  treasury-signer:
    secrets:
      - treasury_private_key

secrets:
  treasury_private_key:
    file: /run/secrets/treasury_key
```

Los secretos de Docker se montan como archivos dentro del contenedor, legibles solo por el proceso de servicio. No aparecen en listados de variables de entorno, salida de inspección de contenedor, ni registros.

### Protección de Git

El archivo `.gitignore` excluye todos los archivos sensibles:

```
.env
*.key
*.pem
secrets/
```

Esto se configura antes del primer commit, no después.

---

## Monitoreo y Respuesta a Incidentes

### Alertas Automatizadas

Las siguientes condiciones disparan alertas inmediatas:

- Desequilibrio del libro mayor detectado (desajuste débito-crédito).
- Balance del tesoro cae por debajo del umbral.
- Los intentos de autenticación fallidos exceden el umbral (10/minuto por IP).
- Proveedor API retorna errores inesperados.
- Tasa de falla de ejecución de orden excede 5%.

### Registro de Auditoría

Cada operación relevante para la seguridad se registra con datos estructurados:

```json
{
  "event": "withdrawal_requested",
  "user_id": "usr_abc123",
  "amount_sun": 10000000,
  "destination": "TAddress...",
  "ip": "203.0.113.45",
  "timestamp": "2026-03-30T12:00:00Z"
}
```

Los registros se retienen para análisis forense y cumplimiento normativo. Son de solo agregar y se almacenan separadamente de los datos de aplicación.

---

## Conclusión

La seguridad en MERX no es una característica única sino un conjunto de principios interconectados: sin custodia de claves, contabilidad de doble entrada, operaciones de balance atómicas, validación estricta de entrada, aislamiento de servicios, limitación de velocidad, webhooks firmados y gestión apropiada de secretos. Cada principio aborda un vector de amenaza específico, y juntos crean una arquitectura de defensa en profundidad donde comprometer cualquier componente individual no compromete el sistema.

Ningún sistema es invulnerable. Pero al diseñar la seguridad en la arquitectura desde el principio - en lugar de parcharla después - MERX minimiza la superficie de ataque y maximiza el costo que debe pagar un atacante para causar daño.

Revisa los componentes de código abierto: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js), [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python), [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp).

Comienza a usar MERX en [https://merx.exchange](https://merx.exchange).

---

*Este artículo es parte de la serie técnica de MERX. MERX es el primer intercambio de recursos de blockchain, construido con seguridad como un requisito fundamental, no una ocurrencia posterior.*


## Pruébalo Ahora con IA

Añade MERX a Claude Desktop o cualquier cliente compatible con MCP -- sin instalación, sin clave de API necesaria para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregunta a tu agente de IA: "¿Cuál es el energy de TRON más barato ahora mismo?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)