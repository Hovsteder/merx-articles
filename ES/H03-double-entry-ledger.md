# Libro Mayor de Doble Entrada para Blockchain: Arquitectura Contable de MERX

La contabilidad por partida doble fue inventada en la Italia del siglo XIII. Ha sobrevivido a cada innovación financiera desde entonces: moneda de papel, bolsas de valores, banca central, transferencias electrónicas de fondos, y ahora blockchain. Hay una razón por la que ha perdurado: funciona. Cada transacción produce un par equilibrado de asientos, y cualquier desequilibrio señala un error inmediatamente.

Cuando construimos el sistema contable para MERX —una plataforma que maneja depósitos de TRX, compras de energía, liquidaciones a proveedores y retiros— elegimos el diseño de libro mayor de doble entrada sin debate. La alternativa, el seguimiento de entrada única con actualizaciones de saldo corriente, es cómo la mayoría de plataformas criptográficas manejan la contabilidad. También es cómo la mayoría de plataformas criptográficas terminan con discrepancias de saldo inexplicables.

Este artículo explica la arquitectura del libro mayor detrás de MERX: por qué la doble entrada es importante para plataformas blockchain, cómo se diseña la tabla del libro mayor, por qué los registros son inmutables, cómo interactúan las cuentas de débito y crédito, y cómo reconciliamos saldos usando SELECT FOR UPDATE.

## Por qué Doble Entrada para Criptomonedas

Un sistema de entrada única funciona como un extracto bancario: una columna, un saldo corriente. Cuando un usuario deposita 100 TRX, sumas 100 a su saldo. Cuando gasta 5 TRX, restas 5. Simple.

Los problemas con la entrada única aparecen bajo estrés:

**Modificaciones concurrentes.** Dos órdenes se ejecutan simultáneamente. Ambas leen el saldo del usuario como 100 TRX. Ambas deducen 5 TRX. Ambas escriben 95 TRX. Al usuario se le ha cobrado una vez en lugar de dos. O peor: una escritura sobrescribe la otra, y la plataforma pierde completamente el rastro de una transacción.

**Falta de rastro de auditoría.** El saldo de un usuario es 47,3 TRX. ¿Cómo llegó allí? Con entrada única, tienes que reconstruir el saldo a partir de registros de transacciones individuales, que pueden o no estar completos, y que no tienen una verificación de integridad incorporada.

**Fallo de reconciliación.** La suma de todos los saldos de los usuarios debe ser igual a las tenencias de TRX de la plataforma. Con entrada única, verificar esto requiere agregar el saldo de cada usuario y compararlo con la tesorería. Si los números no coinciden, no hay una forma sistemática de encontrar la discrepancia.

La doble entrada resuelve todos estos problemas estructuralmente. Cada mutación de saldo crea dos asientos que deben sumar cero. La verificación de integridad está incorporada en cada operación.

## La Tabla del Libro Mayor

El libro mayor de MERX es una única tabla PostgreSQL:

```sql
CREATE TABLE ledger (
  id            BIGSERIAL PRIMARY KEY,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  account_id    UUID NOT NULL REFERENCES accounts(id),
  entry_type    TEXT NOT NULL,
  amount_sun    BIGINT NOT NULL,
  direction     TEXT NOT NULL CHECK (direction IN ('DEBIT', 'CREDIT')),
  reference_id  UUID,
  reference_type TEXT,
  description   TEXT,
  idempotency_key TEXT UNIQUE
);

CREATE INDEX idx_ledger_account ON ledger(account_id);
CREATE INDEX idx_ledger_reference ON ledger(reference_id);
CREATE INDEX idx_ledger_created ON ledger(created_at);
```

### Diseño de Columnas

**amount_sun**: Todos los montos se almacenan en SUN (1 TRX = 1.000.000 SUN). Usar la unidad más pequeña elimina la aritmética de punto flotante por completo. No hay montos decimales, no hay errores de redondeo, no hay pérdida de precisión. Cada cálculo es aritmética de enteros.

**direction**: Ya sea DEBIT o CREDIT. El significado depende del tipo de cuenta:

- Para cuentas de usuario: CREDIT aumenta el saldo, DEBIT lo disminuye
- Para cuentas de liquidación: DEBIT aumenta el saldo, CREDIT lo disminuye
- Para cuentas de ingresos: CREDIT aumenta el saldo

**entry_type**: Categoriza el asiento del libro mayor. Ejemplos:

```
DEPOSIT              El usuario deposita TRX en su cuenta de MERX
WITHDRAWAL           El usuario retira TRX de su cuenta de MERX
ORDER_PAYMENT        El usuario paga una orden de energía
ORDER_REFUND         La orden falla, el pago se devuelve al usuario
PROVIDER_SETTLEMENT  Pago al proveedor de energía por orden cumplida
X402_PAYMENT         Pago en cadena recibido vía protocolo x402
```

**reference_id** y **reference_type**: Vinculan el asiento del libro mayor al objeto comercial que lo causó (una orden, un depósito, un retiro). Esto crea un rastro de auditoría bidireccional: desde el asiento del libro mayor puedes encontrar la orden, y desde la orden puedes encontrar sus asientos en el libro mayor.

**idempotency_key**: Previene asientos duplicados. Si la misma operación se procesa dos veces (debido a un reintento, un tiempo de espera de red, un webhook duplicado), la restricción única en idempotency_key asegura que solo se cree un asiento.

## La Regla de Inmutabilidad

Los registros del libro mayor nunca se actualizan. Nunca se eliminan. Esto se fuerza a nivel de base de datos:

```sql
-- Prevenir cualquier actualización de registros del libro mayor
CREATE OR REPLACE FUNCTION prevent_ledger_update()
RETURNS TRIGGER AS $$
BEGIN
  RAISE EXCEPTION 'Los registros del libro mayor no pueden ser actualizados';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_ledger_update
  BEFORE UPDATE ON ledger
  FOR EACH ROW
  EXECUTE FUNCTION prevent_ledger_update();

-- Prevenir cualquier eliminación del libro mayor
CREATE OR REPLACE FUNCTION prevent_ledger_delete()
RETURNS TRIGGER AS $$
BEGIN
  RAISE EXCEPTION 'Los registros del libro mayor no pueden ser eliminados';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_ledger_delete
  BEFORE DELETE ON ledger
  FOR EACH ROW
  EXECUTE FUNCTION prevent_ledger_delete();
```

Estos disparadores hacen que la regla de inmutabilidad sea inquebrantable a nivel de base de datos. Ni siquiera un administrador ejecutando SQL directo puede modificar o eliminar un asiento del libro mayor sin antes deshabilitar el disparador, una operación que sería visible en los registros de auditoría de la base de datos.

### Por qué la Inmutabilidad es Importante

**Integridad de auditoría.** Si los registros del libro mayor pueden ser modificados, un atacante (o un bug) puede alterar el historial financiero de la plataforma. Los registros inmutables significan que el historial es permanente y resistente a manipulaciones.

**Cumplimiento regulatorio.** Las regulaciones de mantenimiento de registros financieros universalmente requieren que los registros de transacciones se preserven. Eliminarlos o alterarlos es una violación de cumplimiento.

**Depuración.** Cuando algo sale mal —y en un sistema que procesa dinero real, las cosas saldrán mal— los registros inmutables proporcionan una línea de tiempo completa e inalterada de eventos. Puedes reproducir el historial exactamente como sucedió.

### Correcciones y Reversiones

Si un asiento del libro mayor necesita ser "corregido" (por ejemplo, un reembolso), no actualizas el asiento original. Creas un nuevo asiento con la dirección opuesta:

```sql
-- Pago de orden original
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($user_id, 'ORDER_PAYMENT', 1820000, 'DEBIT', $order_id);

INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($provider_settlement, 'ORDER_PAYMENT', 1820000, 'CREDIT', $order_id);

-- La orden falló, emitir reembolso (nuevos asientos, los originales permanecen)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($user_id, 'ORDER_REFUND', 1820000, 'CREDIT', $order_id);

INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($provider_settlement, 'ORDER_REFUND', 1820000, 'DEBIT', $order_id);
```

Después del reembolso, los asientos de pago originales siguen existiendo. El saldo calculado del usuario refleja tanto el pago como el reembolso: neto cero. El rastro de auditoría muestra exactamente qué sucedió y cuándo.

## Cuentas de Débito y Crédito

MERX usa varios tipos de cuenta, cada uno con su propio rol en el sistema de doble entrada:

### Cuentas de Usuario

Cada usuario de MERX tiene una cuenta. Su saldo se calcula a partir de asientos del libro mayor:

```sql
SELECT
  COALESCE(SUM(CASE WHEN direction = 'CREDIT' THEN amount_sun ELSE 0 END), 0) -
  COALESCE(SUM(CASE WHEN direction = 'DEBIT' THEN amount_sun ELSE 0 END), 0)
  AS balance_sun
FROM ledger
WHERE account_id = $user_id;
```

Los créditos aumentan el saldo (depósitos, reembolsos). Los débitos lo disminuyen (pagos de órdenes, retiros).

### Cuenta de Liquidación de Proveedores

Cuando un usuario compra energía, el pago necesita llegar al proveedor. La cuenta de liquidación de proveedores rastrea lo que MERX debe a cada proveedor:

```
El usuario paga por la orden:
  Cuenta de usuario:                DEBIT  1.820.000 SUN
  Liquidación de proveedor (Feee): CREDIT 1.820.000 SUN

MERX liquida con el proveedor:
  Liquidación de proveedor (Feee): DEBIT  1.820.000 SUN
  Tesorería:                        CREDIT 1.820.000 SUN
```

En cualquier momento, el saldo de la cuenta de liquidación del proveedor muestra el monto total que MERX debe a ese proveedor y aún no ha liquidado.

### Cuenta de Tesorería

La cuenta de tesorería representa las tenencias de TRX en cadena de MERX. Los depósitos acreditan la tesorería (TRX recibido). Los retiros y las liquidaciones de proveedores debitan la tesorería (TRX enviado).

### La Ecuación Fundamental

En todo momento:

```
Suma de todos los CRÉDITOS = Suma de todos los DÉBITOS
```

Si esta ecuación falla, el sistema tiene un bug. MERX ejecuta una verificación de reconciliación periódicamente:

```sql
SELECT
  SUM(CASE WHEN direction = 'CREDIT' THEN amount_sun ELSE 0 END) AS total_credits,
  SUM(CASE WHEN direction = 'DEBIT' THEN amount_sun ELSE 0 END) AS total_debits
FROM ledger;

-- total_credits DEBE ser igual a total_debits
-- Si no, alertar inmediatamente
```

## El Flujo de Transacción Completo

Aquí está cómo una compra típica de energía fluye a través del libro mayor:

### 1. El Usuario Deposita TRX

El monitor de depósito detecta una transferencia de TRX entrante a la dirección de depósito de MERX:

```sql
BEGIN;

-- Acreditar la cuenta del usuario (su saldo aumenta)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type, idempotency_key)
VALUES ($user_id, 'DEPOSIT', 100000000, 'CREDIT', $deposit_id, 'DEPOSIT', $tx_hash);

-- Debitar la tesorería (TRX recibido, tesorería reconoce la responsabilidad)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type, idempotency_key)
VALUES ($treasury_id, 'DEPOSIT', 100000000, 'DEBIT', $deposit_id, 'DEPOSIT', $tx_hash || '_treasury');

COMMIT;
```

### 2. El Usuario Compra Energía

El usuario coloca una orden de 65.000 energía a 28 SUN/unidad:

```sql
BEGIN;

-- Verificar saldo con bloqueo de fila
SELECT balance_sun FROM account_balances
WHERE account_id = $user_id
FOR UPDATE;

-- Verificar saldo suficiente
-- 65.000 * 28 = 1.820.000 SUN = 1,82 TRX

-- Debitar la cuenta del usuario (el saldo disminuye)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type, idempotency_key)
VALUES ($user_id, 'ORDER_PAYMENT', 1820000, 'DEBIT', $order_id, 'ORDER', $idempotency_key);

-- Acreditar la liquidación del proveedor (MERX ahora debe al proveedor)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type, idempotency_key)
VALUES ($provider_settlement_id, 'ORDER_PAYMENT', 1820000, 'CREDIT', $order_id, 'ORDER', $idempotency_key || '_settlement');

COMMIT;
```

### 3. La Orden Falla (Reembolso)

Si el proveedor no delega la energía, la orden se reembolsa:

```sql
BEGIN;

-- Acreditar la cuenta del usuario (saldo restaurado)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type)
VALUES ($user_id, 'ORDER_REFUND', 1820000, 'CREDIT', $order_id, 'ORDER');

-- Debitar la liquidación del proveedor (MERX ya no debe al proveedor)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type)
VALUES ($provider_settlement_id, 'ORDER_REFUND', 1820000, 'DEBIT', $order_id, 'ORDER');

COMMIT;
```

Los asientos de pago originales permanecen. Los asientos de reembolso son nuevos registros. El cambio de saldo neto del usuario para esta orden es cero: 1.820.000 SUN debitados, luego 1.820.000 SUN acreditados.

## Reconciliación de Saldos con SELECT FOR UPDATE

El momento más peligroso en cualquier sistema financiero es la verificación de saldo antes de una deducción. Sin bloqueo adecuado, las solicitudes concurrentes pueden pasar ambas la verificación de saldo y ambas deducir, resultando en un saldo negativo.

### La Condición de Carrera

```
Hilo A: SELECT balance WHERE user_id = 1  -> 10 TRX
Hilo B: SELECT balance WHERE user_id = 1  -> 10 TRX
Hilo A: Deducir 8 TRX, nuevo saldo = 2 TRX
Hilo B: Deducir 8 TRX, nuevo saldo = 2 TRX
Resultado: El usuario tenía 10 TRX, gastó 16 TRX, el saldo muestra 2 TRX
```

La plataforma acaba de perder 6 TRX.

### La Solución: SELECT FOR UPDATE

```sql
BEGIN;

-- Bloquear la fila. Cualquier otra transacción que intente leer esta fila
-- con FOR UPDATE se bloqueará hasta que esta transacción se complete.
SELECT balance_sun FROM account_balances
WHERE account_id = $user_id
FOR UPDATE;

-- Ahora tenemos un bloqueo exclusivo. Verificar el saldo de forma segura.
-- Si insuficiente, ROLLBACK.
-- Si suficiente, proceder con asientos del libro mayor.

INSERT INTO ledger ...;

COMMIT;
-- Bloqueo liberado. La siguiente transacción esperando puede proceder.
```

Con `FOR UPDATE`, el escenario se convierte en:

```
Hilo A: SELECT ... FOR UPDATE  -> 10 TRX (fila bloqueada)
Hilo B: SELECT ... FOR UPDATE  -> BLOQUEADO (esperando el bloqueo de A)
Hilo A: Deducir 8 TRX, COMMIT  -> saldo = 2 TRX (bloqueo liberado)
Hilo B: SELECT ... FOR UPDATE  -> 2 TRX (bloqueo adquirido)
Hilo B: ¿Deducir 8 TRX?         -> SALDO INSUFICIENTE, ROLLBACK
```

Sin gasto excesivo. Sin fondos perdidos. La garantía de serialización de `SELECT FOR UPDATE` asegura que las verificaciones de saldo y las deducciones sean atómicas.

### Implicaciones de Rendimiento

`SELECT FOR UPDATE` serializa transacciones por cuenta. Dos usuarios pueden realizar transacciones simultáneamente sin bloquearse mutuamente (bloquean filas diferentes). Pero dos órdenes concurrentes del mismo usuario deben esperar en la cola.

En la práctica, esto no es un cuello de botella. Los usuarios individuales raramente envían solicitudes verdaderamente concurrentes. Cuando lo hacen (p. ej., un bot mal configurado), la serialización es el comportamiento correcto: quieres que esas solicitudes se procesen secuencialmente, no en paralelo.

## Reconciliación Periódica

Más allá de la integridad por transacción, MERX ejecuta una reconciliación periódica que verifica todo el libro mayor:

```sql
-- 1. Verificar ecuación de saldo global
SELECT
  SUM(CASE WHEN direction = 'CREDIT' THEN amount_sun ELSE 0 END) AS credits,
  SUM(CASE WHEN direction = 'DEBIT' THEN amount_sun ELSE 0 END) AS debits
FROM ledger;
-- credits DEBE ser igual a debits

-- 2. Verificar saldos por cuenta contra estado en cadena
-- La suma de todos los saldos de usuario debe ser igual a las tenencias de tesorería
-- menos liquidaciones de proveedores pendientes

-- 3. Verificar referencias huérfanas
-- Cada reference_id debe apuntar a una orden, depósito o retiro válido
SELECT l.reference_id, l.reference_type
FROM ledger l
LEFT JOIN orders o ON l.reference_id = o.id AND l.reference_type = 'ORDER'
LEFT JOIN deposits d ON l.reference_id = d.id AND l.reference_type = 'DEPOSIT'
WHERE o.id IS NULL AND d.id IS NULL AND l.reference_type IS NOT NULL;
```

Si alguna verificación falla, el sistema alerta inmediatamente. La respuesta es investigación y corrección (a través de nuevos asientos del libro mayor), nunca modificación de registros existentes.

## Resumen

El libro mayor de doble entrada de MERX proporciona:

1. **Integridad**: Cada transacción es un par equilibrado. Los desequilibrios se detectan inmediatamente.
2. **Inmutabilidad**: Los registros no pueden ser modificados ni eliminados. El historial es permanente.
3. **Seguridad de concurrencia**: SELECT FOR UPDATE previene condiciones de carrera en verificaciones de saldo.
4. **Auditabilidad**: Historial financiero completo con referencias bidireccionales.
5. **Reconciliación**: Verificaciones periódicas verifican todo el estado del sistema.

Esto no es novedoso. Es una técnica contable de 700 años aplicada a una plataforma blockchain. La novedad es que la mayoría de plataformas criptográficas lo omiten —y pagan el precio en fondos perdidos, discrepancias inexplicables, y pesadillas de auditoría.

Plataforma: [https://merx.exchange](https://merx.exchange)
Documentación: [https://merx.exchange/docs](https://merx.exchange/docs)


## Pruébalo Ahora con IA

Añade MERX a Claude Desktop o cualquier cliente compatible con MCP —sin instalación, sin clave API necesaria para herramientas de solo lectura:

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