# Implementación del Protocolo x402: Facturación, Pago, Verificación

El código de estado HTTP 402 -- "Payment Required" -- fue definido en la especificación original de HTTP/1.1 en 1997. La especificación lo marcó como "reservado para uso futuro". Veintinueve años después, el futuro ha llegado.

El protocolo x402 convierte ese código de estado reservado en un mecanismo de pago real. Habilita el comercio de pago por uso donde el pago se verifica en cadena en lugar de a través de cuentas, claves API o tarjetas de crédito. Cualquier entidad con una billetera blockchain -- humano, bot o agente autónomo -- puede pagar por un servicio en una única transacción sin crear una cuenta o establecer ninguna relación previa con el proveedor de servicios.

MERX implementa x402 para compras de energía TRON. Este artículo es una guía técnica completa de la implementación: creación de facturas, pago con memo, verificación en TronGrid (incluido el problema de coincidencia de direcciones en hexadecimal vs base58), el usuario del sistema x402, acreditación de saldo y ejecución de órdenes. Cubrimos cada paso, cada caso extremo y cada consideración de seguridad.

## Descripción General del Flujo x402

El protocolo tiene cinco pasos:

```
1. INVOICE   - El comprador solicita una cotización, el servidor devuelve las instrucciones de pago
2. PAY       - El comprador envía TRX con un memo que contiene el ID de la factura
3. VERIFY    - El servidor detecta el pago en cadena y lo valida
4. CREDIT    - El servidor acredita la cuenta del sistema x402 y crea entradas de libro mayor
5. EXECUTE   - El servidor ejecuta la orden de energía y delega al comprador
```

Cada paso está diseñado para ser confiable. El comprador nunca envía fondos a una dirección no verificada. El servidor nunca ejecuta una orden sin pago confirmado. El campo memo vincula el pago a una factura específica, previniendo ataques de pago cruzado.

## Paso 1: Creación de Factura

El comprador llama al endpoint `create_paid_order` con los parámetros de energía deseados:

```
POST /api/v1/orders/paid
Content-Type: application/json

{
  "energy_amount": 65000,
  "duration_hours": 1,
  "target_address": "TBuyerAddress..."
}
```

El servidor calcula el costo basándose en los mejores precios actuales de todos los proveedores y devuelve una factura:

```json
{
  "invoice": {
    "order_id": "xpay_a7f3c2d1",
    "amount_trx": 1.43,
    "amount_sun": 1430000,
    "pay_to": "TMerxTreasuryAddress...",
    "memo": "merx_xpay_a7f3c2d1",
    "expires_at": "2026-03-30T12:05:00Z",
    "energy_amount": 65000,
    "duration_hours": 1,
    "target_address": "TBuyerAddress..."
  }
}
```

### Decisiones de Diseño de Factura

**Expiración de 5 minutos.** La factura expira 5 minutos después de su creación. Este período es lo suficientemente largo para que el comprador revise, firme y transmita el pago. Es lo suficientemente corto para prevenir la explotación de precios obsoletos -- si los precios de energía cambian significativamente, el comprador debería solicitar una nueva factura al precio actual.

**Cantidad exacta requerida.** El pago debe coincidir exactamente con la cantidad de la factura. Ni más, ni menos. Esto previene ambigüedad en la coincidencia de pagos con facturas. Si un comprador envía 2 TRX para una factura de 1,43 TRX, el pago es rechazado (el exceso crearía complejidad contable sin ventaja).

**Memo único.** El campo memo contiene un identificador único que vincula el pago a esta factura específica. Este es el mecanismo crítico de seguridad -- más sobre esto abajo.

### Almacenamiento de Factura en el Servidor

```sql
INSERT INTO x402_invoices (
  order_id,
  amount_sun,
  pay_to,
  memo,
  expires_at,
  energy_amount,
  duration_hours,
  target_address,
  status
) VALUES (
  'xpay_a7f3c2d1',
  1430000,
  'TMerxTreasuryAddress...',
  'merx_xpay_a7f3c2d1',
  '2026-03-30T12:05:00Z',
  65000,
  1,
  'TBuyerAddress...',
  'PENDING'
);
```

La factura se almacena con estado PENDING. Pasará a PAID cuando se verifique, o EXPIRED cuando el TTL expire sin pago.

## Paso 2: Pago con Memo

El comprador construye y firma una transacción de transferencia de TRX. El detalle crítico de implementación es el campo memo.

```javascript
const tronWeb = new TronWeb({
  fullHost: 'https://api.trongrid.io'
});

// Construir la transacción base
const tx = await tronWeb.transactionBuilder.sendTrx(
  invoice.pay_to,       // TMerxTreasuryAddress
  invoice.amount_sun,   // 1430000
  buyerAddress           // TBuyerAddress
);

// Agregar el memo
const txWithMemo = await tronWeb.transactionBuilder.addUpdateData(
  tx,
  invoice.memo,         // "merx_xpay_a7f3c2d1"
  'utf8'
);

// Firmar localmente
const signedTx = await tronWeb.trx.sign(txWithMemo, privateKey);

// Transmitir
const result = await tronWeb.trx.sendRawTransaction(signedTx);
console.log('TX hash:', result.txid);
```

### Por Qué Memo, No Cantidad

Un diseño anterior consideró usar cantidades únicas (p. ej., 1.430.017 SUN en lugar de 1.430.000 SUN) para identificar pagos. Este enfoque es frágil:

- Las colisiones de cantidad son posibles (dos facturas podrían tener el mismo precio)
- Requiere que el comprador pague una cantidad no redonda
- No funciona cuando múltiples facturas tienen parámetros idénticos

El campo memo proporciona un identificador inequívoco sin riesgo de colisión.

### Seguridad de la Clave Privada

La clave privada del comprador nunca sale de su dispositivo. La transacción se construye, firma y transmite completamente en la máquina del comprador. MERX nunca ve, solicita o tiene acceso a la clave privada del comprador. Esta es una propiedad de seguridad fundamental del protocolo x402.

## Paso 3: Verificación en TronGrid

Después de que el pago se transmite, MERX debe verificarlo en cadena. Aquí es donde la implementación se vuelve interesante -- y donde surge un desafío técnico significativo.

### El Bucle de Monitoreo

El monitor de depósitos de MERX observa continuamente la dirección del tesoro para detectar transacciones entrantes:

```typescript
async function monitorTreasuryForX402Payments(): Promise<void> {
  const treasuryAddress = process.env.TREASURY_ADDRESS;

  while (true) {
    const transactions = await tronWeb.trx.getTransactionsRelated(
      treasuryAddress,
      'to',
      { limit: 50, only_confirmed: true }
    );

    for (const tx of transactions) {
      await processIncomingTransaction(tx);
    }

    await sleep(3000); // Verificar cada 3 segundos (un bloque)
  }
}
```

### El Problema de Dirección Hexadecimal vs Base58

Aquí está el desafío técnico que consumió más tiempo de depuración que cualquier otra parte de la implementación de x402.

Las direcciones TRON existen en dos formatos:

- **Base58**: `TJRabPrwbZy45sbavfcjinPJC18kjpRTv8` (legible por humanos, comienza con T)
- **Hex**: `415a523b449890854c8fc460ab602df9f31fe4293f` (hex con prefijo 41, usado internamente)

Cuando consultas TronGrid para obtener detalles de la transacción, la respuesta usa direcciones hexadecimales. Cuando tu factura almacena la dirección del comprador y la dirección del tesoro, están en base58. Si los comparas directamente, nunca coincidirán.

```typescript
// Transacción de la API de TronGrid
const txData = {
  owner_address: '415a523b449890854c8fc460ab602df9f31fe4293f',  // hex
  to_address: '41e552f6487585c2b58bc2c9bb4492bc1f17132cd0',    // hex
  amount: 1430000
};

// Factura de la base de datos
const invoice = {
  pay_to: 'TJRabPrwbZy45sbavfcjinPJC18kjpRTv8',  // base58
  target_address: 'TBuyerAddressBase58...',         // base58
  amount_sun: 1430000
};

// La comparación directa FALLA
txData.to_address === invoice.pay_to  // false (hex vs base58)
```

### La Solución: Convertir Antes de Comparar

Cada comparación de dirección debe convertir ambos lados al mismo formato:

```typescript
function normalizeAddress(address: string): string {
  if (address.startsWith('41') && address.length === 42) {
    // Formato hexadecimal -- convertir a base58
    return tronWeb.address.fromHex(address);
  }
  if (address.startsWith('T') && address.length === 34) {
    // Ya está en base58
    return address;
  }
  throw new Error(`Formato de dirección TRON inválido: ${address}`);
}

function addressesMatch(a: string, b: string): boolean {
  return normalizeAddress(a) === normalizeAddress(b);
}
```

Esta función de normalización se usa en cada comparación de dirección en toda la tubería de verificación de x402. Perder una única comparación crearía una vulnerabilidad.

### Lógica Completa de Verificación

```typescript
async function verifyX402Payment(tx: TronTransaction): Promise<void> {
  // 1. Extraer memo de los datos de la transacción
  const memo = extractMemo(tx);
  if (!memo || !memo.startsWith('merx_xpay_')) {
    return; // No es un pago x402, omitir
  }

  // 2. Encontrar factura coincidente
  const invoice = await findInvoiceByMemo(memo);
  if (!invoice) {
    console.warn(`No se encontró factura para memo: ${memo}`);
    return;
  }

  // 3. Verificar estado de factura
  if (invoice.status !== 'PENDING') {
    console.warn(`Factura ${invoice.order_id} ya está ${invoice.status}`);
    return; // Previene reclamación duplicada
  }

  // 4. Verificar expiración
  if (new Date() > new Date(invoice.expires_at)) {
    await markInvoiceExpired(invoice.order_id);
    console.warn(`Factura ${invoice.order_id} expirada`);
    return;
  }

  // 5. Verificar cantidad (coincidencia exacta requerida)
  if (tx.amount !== invoice.amount_sun) {
    console.warn(
      `Cantidad no coincide: TX=${tx.amount}, factura=${invoice.amount_sun}`
    );
    return;
  }

  // 6. Verificar destinatario (comparación segura hex vs base58)
  if (!addressesMatch(tx.to_address, invoice.pay_to)) {
    console.warn('Dirección del destinatario no coincide');
    return;
  }

  // Todos los controles pasaron -- el pago es válido
  await processValidPayment(invoice, tx);
}
```

### Extracción de Memo

El memo se almacena en el campo `raw_data.data` de la transacción como una cadena codificada en hexadecimal:

```typescript
function extractMemo(tx: TronTransaction): string | null {
  try {
    const hexData = tx.raw_data?.data;
    if (!hexData) return null;

    // Decodificar hex a UTF-8
    const memo = Buffer.from(hexData, 'hex').toString('utf8');
    return memo;
  } catch {
    return null;
  }
}
```

## Paso 4: El Usuario del Sistema x402

Cuando se verifica un pago x402, MERX necesita acreditar el pago y ejecutar la orden. Pero los pagos x402 son sin cuenta -- el comprador no tiene una cuenta en MERX. ¿Cómo creas entradas de libro mayor sin una cuenta?

La solución es el usuario del sistema x402. Esta es una cuenta interna especial que representa todas las transacciones x402:

```sql
INSERT INTO accounts (id, email, type)
VALUES (
  'x402-system-00000000-0000-0000-0000-000000000000',
  'x402@system.merx.exchange',
  'SYSTEM'
);
```

### Acreditación de Saldo

Cuando se verifica un pago x402, el sistema:

1. Acredita la cuenta del sistema x402 (el saldo aumenta)
2. Debita la cuenta del tesoro (TRX recibido)
3. Inmediatamente debita la cuenta del sistema x402 (pago de orden)
4. Acredita la cuenta de liquidación del proveedor (pago al proveedor)

```sql
BEGIN;

-- Acreditar cuenta del sistema x402 (pago recibido)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($x402_system_id, 'X402_PAYMENT', 1430000, 'CREDIT', $order_id);

-- Debitar tesoro (TRX recibido en cadena)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($treasury_id, 'X402_PAYMENT', 1430000, 'DEBIT', $order_id);

-- Debitar cuenta del sistema x402 (pago de orden)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($x402_system_id, 'ORDER_PAYMENT', 1430000, 'DEBIT', $order_id);

-- Acreditar liquidación del proveedor (MERX debe al proveedor)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($provider_settlement_id, 'ORDER_PAYMENT', 1430000, 'CREDIT', $order_id);

COMMIT;
```

El saldo de la cuenta del sistema x402 debe estar siempre en cero o cercano a cero: cada crédito (pago recibido) se compensa inmediatamente con un débito (orden ejecutada). Si el saldo crece, significa que se están recibiendo pagos pero las órdenes no se están ejecutando -- una condición de alerta.

## Paso 5: Ejecución de la Orden

Después de que el pago se acredita, la orden se ejecuta a través de la tubería de órdenes estándar de MERX:

```typescript
async function processValidPayment(
  invoice: X402Invoice,
  tx: TronTransaction
): Promise<void> {
  // Marcar factura como PAID
  await updateInvoiceStatus(invoice.order_id, 'PAID', tx.txid);

  // Crear entradas de libro mayor (como se muestra arriba)
  await createX402LedgerEntries(invoice, tx);

  // Ejecutar la orden de energía
  const order = await executeOrder({
    energy_amount: invoice.energy_amount,
    duration_hours: invoice.duration_hours,
    target_address: invoice.target_address,
    source: 'x402',
    reference_tx: tx.txid
  });

  // Actualizar factura con resultado de la orden
  await updateInvoiceWithOrder(invoice.order_id, order);
}
```

La ejecución de la orden sigue la misma ruta que cualquier otra orden de MERX: enrutamiento al mejor precio, selección de proveedor, delegación y verificación en cadena (incluida la corrección de sondeo para condiciones de carrera descrita en nuestro artículo anterior).

## Seguridad: La Verificación de Memo Previene Pago Cruzado

El campo memo es el pasador de la seguridad x402. Sin él, existe un vector de ataque crítico.

### El Ataque de Pago Cruzado

Imagina x402 sin memos. Dos usuarios solicitan facturas simultáneamente:

```
Alice solicita 65.000 de energía para TAliceAddress. Factura: 1,43 TRX a TMerxTreasury.
Bob solicita 65.000 de energía para TBobAddress. Factura: 1,43 TRX a TMerxTreasury.
```

Ambas facturas tienen la misma cantidad y la misma dirección de pago. Si Bob paga su factura, ¿cómo sabe MERX si delegar energía a TAliceAddress o TBobAddress? Sin un memo, el pago es ambiguo.

Peor aún: Bob podría pagar una vez y reclamar ambas facturas. O Alice podría reclamar el pago de Bob para su propia factura.

### Cómo los Memos Previenen Esto

```
Factura de Alice: memo = "merx_xpay_alice123"
Factura de Bob:   memo = "merx_xpay_bob456"

Pago de Alice TX: 1,43 TRX a TMerxTreasury, memo = "merx_xpay_alice123"
Pago de Bob TX:   1,43 TRX a TMerxTreasury, memo = "merx_xpay_bob456"

Verificación:
  Memo de TX de Alice coincide con factura de Alice -> delegar a TAliceAddress
  Memo de TX de Bob coincide con factura de Bob -> delegar a TBobAddress
```

Cada pago está vinculado inequívocamente a su factura. No hay forma de reclamar cruzadamente.

### Controles de Seguridad Adicionales

Más allá de la coincidencia de memo, la tubería de verificación incluye:

**Prevención de pago duplicado.** Una vez que una factura se marca como PAID, los pagos posteriores con el mismo memo se rechazan. El pagador tendría que contactar al soporte para un reembolso (o el sistema devolvería los fondos automáticamente si la cantidad excede la factura).

**Exactitud de cantidad.** El pago debe coincidir exactamente con la cantidad de la factura. Esto previene pagos parciales (que requerirían lógica compleja de llenado parcial) y pagos excesivos (que requerirían lógica de reembolso).

**Aplicación de expiración.** Los pagos recibidos después de que la factura expira no se procesan. Esto previene la explotación de precios obsoletos donde un comprador solicita una factura durante un período de bajo precio, espera a que los precios suban y luego paga la factura antigua.

**Verificación de dirección.** El pago debe ir a la dirección del tesoro correcta. Si un usuario de alguna manera paga a una dirección diferente (error al copiar-pegar, phishing), el pago no será detectado por el monitor.

## Manejo de Errores

### Pago Sin Factura

Si una transferencia de TRX llega a la dirección del tesoro con un memo que no coincide con ninguna factura (error tipográfico, factura expirada, transacción de prueba), el pago se registra pero no se procesa. Los fondos permanecen en el tesoro. En un sistema de producción, esto activaría una alerta de soporte para revisión manual y posible reembolso.

### Falla del Proveedor Después del Pago

Si el proveedor de energía falla al delegar después de un pago verificado:

```typescript
try {
  const order = await executeOrder(invoice);
} catch (error) {
  // Orden falló -- reembolsar la cuenta del sistema x402
  await createRefundLedgerEntries(invoice);

  // Marcar factura como REFUND_REQUIRED
  await updateInvoiceStatus(invoice.order_id, 'REFUND_REQUIRED');

  // Alertar al equipo de ops para reembolso manual de TRX a la dirección del pagador
  await alertOps({
    type: 'X402_REFUND_REQUIRED',
    invoice: invoice.order_id,
    payer_address: extractSenderFromTx(tx),
    amount_sun: invoice.amount_sun
  });
}
```

El reembolso crea nuevas entradas de libro mayor (nunca modifica las existentes) y marca la factura para procesamiento manual de reembolso.

### Congestión de Red

Durante congestión de red alta, la brecha entre transmisión de pago y confirmación de pago puede extenderse más allá de la ventana de factura de 5 minutos. El sistema maneja esto verificando la marca de tiempo de la transacción (cuándo se transmitió) en lugar de la marca de tiempo de confirmación (cuándo se incluyó en un bloque). Si la transacción se transmitió antes de que la factura expirara, se acepta incluso si la confirmación llega después de la expiración.

## Resumen

La implementación de x402 en MERX demuestra que los pagos confiables sin cuenta son prácticos hoy en día. Las decisiones clave de diseño:

1. **Factura con memo único** -- vinculación inequívoca de pago a orden
2. **Coincidencia de cantidad exacta** -- elimina la complejidad de pago parcial/excesivo
3. **Expiración de 5 minutos** -- previene explotación de precios obsoletos
4. **Normalización de hexadecimal a base58** -- resuelve el problema de formato de dirección de TronGrid
5. **Usuario del sistema x402** -- habilita contabilidad de doble entrada sin cuentas del comprador
6. **Entradas de libro mayor inmutables** -- pista de auditoría completa para cada transacción x402

El protocolo convierte HTTP 402 de un marcador de posición de 29 años en un mecanismo de pago funcional. Para agentes de IA que no pueden crear cuentas ni administrar claves API, x402 hace que la energía TRON sea accesible a través de una única transacción en cadena.

Plataforma: [https://merx.exchange](https://merx.exchange)
Documentación: [https://merx.exchange/docs](https://merx.exchange/docs)
Servidor MCP: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)


## Pruébalo Ahora con IA

Agrega MERX a Claude Desktop o cualquier cliente compatible con MCP -- sin instalación, sin clave API requerida para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregúntale a tu agente de IA: "¿Cuál es la energía TRON más barata en este momento?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)