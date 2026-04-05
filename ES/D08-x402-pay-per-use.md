# x402 Pago por Uso: Compra Energía TRON sin Cuenta

## El Problema del Registro

Todo servicio blockchain sigue el mismo patrón: crear una cuenta, verificar tu correo electrónico, generar una clave API, financiar un saldo interno, luego comenzar a usar el servicio. Para un usuario humano, esto es un inconveniente menor. Para un agente de IA autónomo, es un punto de ruptura insalvable.

Los agentes no tienen direcciones de correo electrónico. No quieren que terceros gestionen saldos internos. No quieren confiarle a una plataforma la custodia de sus fondos. Lo que los agentes quieren es simple: pagar por un servicio, recibir el servicio, continuar. Sin relación. Sin estado. Sin confianza.

El protocolo x402 hace esto posible. Inspirado en el código de estado HTTP 402 "Payment Required" (definido en 1997 pero nunca ampliamente implementado), x402 permite el comercio de pago por uso donde el pago se verifica en cadena en lugar de a través de cuentas y claves API.

MERX implementa x402 para compras de energía. Cualquier entidad con una cartera TRON - humana, agente o contrato inteligente - puede comprar energía en una sola transacción sin crear una cuenta, sin depositar fondos, y sin relación previa alguna con la plataforma MERX.

## Cómo Funciona

El flujo x402 tiene cinco pasos. Cada paso es verificable en cadena, y en ningún momento el comprador necesita confiarle a MERX la custodia de sus fondos.

### Paso 1: Solicitar una Factura

El comprador llama a `create_paid_order` con los parámetros de energía deseados:

```
Tool: create_paid_order
Input: {
  "energy_amount": 65000,
  "duration_hours": 1,
  "target_address": "TBuyerAddress..."
}

Response:
{
  "invoice": {
    "order_id": "xpay_abc123",
    "amount_trx": 1.43,
    "amount_sun": 1430000,
    "pay_to": "TMerxTreasuryAddress...",
    "memo": "merx_xpay_abc123",
    "expires_at": "2026-03-30T12:05:00Z",
    "energy_amount": 65000,
    "duration_hours": 1,
    "target_address": "TBuyerAddress..."
  }
}
```

La factura es una cotización, no un compromiso. No se han movido fondos. El comprador puede inspeccionar el precio, compararlo con alternativas, y decidir si proceder. La factura expira después de 5 minutos - si no se paga dentro de ese período, el precio cotizado ya no está garantizado.

### Paso 2: Firmar el Pago Localmente

El comprador construye una transacción de transferencia de TRX desde su cartera a la dirección del tesorero de MERX. El detalle crítico es el campo memo, que debe contener la cadena exacta de la factura.

```javascript
const tx = await tronWeb.transactionBuilder.sendTrx(
  invoice.pay_to,        // TMerxTreasuryAddress
  invoice.amount_sun,    // 1430000 (1.43 TRX en SUN)
  buyerAddress
);

// Agregar memo a los datos de la transacción
const txWithMemo = await tronWeb.transactionBuilder.addUpdateData(
  tx,
  invoice.memo,          // "merx_xpay_abc123"
  'utf8'
);

// Firmar localmente - la clave privada nunca sale de la máquina del comprador
const signedTx = await tronWeb.trx.sign(txWithMemo);
```

El pago se firma con la clave privada del comprador en la máquina del comprador. La clave privada nunca se comparte con MERX, nunca se transmite por la red, y nunca se almacena en ningún lugar fuera del control del comprador.

### Paso 3: Transmitir el Pago

La transacción firmada se transmite a la red TRON:

```javascript
const result = await tronWeb.trx.sendRawTransaction(signedTx);
const txHash = result.txid;
```

En este punto, el pago está en cadena. Es visible para cualquiera que consulte la blockchain de TRON. El TRX se ha movido de la dirección del comprador a la dirección del tesorero de MERX, y el campo memo contiene el identificador de la factura.

### Paso 4: MERX Verifica En Cadena

MERX monitorea su dirección de tesorero buscando transacciones entrantes. Cuando llega una transacción, MERX:

1. Lee el campo memo
2. Lo compara con facturas pendientes
3. Verifica que el monto coincida con la factura (cantidad exacta requerida)
4. Verifica que la factura no haya expirado
5. Verifica que la factura no haya sido pagada ya (previene doble reclamación)

```
Verificación:
  Hash TX: 7f3a2b...
  De: TBuyerAddress...
  Para: TMerxTreasuryAddress...
  Monto: 1,430,000 SUN (1.43 TRX) - COINCIDE
  Memo: "merx_xpay_abc123" - COINCIDE con factura
  Estado de factura: NO PAGADA - OK
  Vencimiento de factura: 2026-03-30T12:05:00Z - NO EXPIRADA

  Resultado: VERIFICADA
```

### Paso 5: Delegación de Energía

Con el pago verificado, MERX realiza la orden de energía a través de su red de proveedores:

```
Orden realizada:
  Energía: 65,000
  Duración: 1 hora
  Destino: TBuyerAddress...
  Proveedor: sohu (mejor precio en el momento de la orden)

Delegación confirmada después de 4.1 segundos
Energía disponible en TBuyerAddress: 65,000
```

El comprador ahora tiene 65,000 de energía delegados a su dirección durante 1 hora. Pueden usarla para transferencias de USDT, swaps de DEX, o cualquier otra interacción de contrato inteligente.

## El Flujo Completo del Agente

Aquí es cómo un agente de IA ejecuta todo el flujo x402 a través del servidor MCP de MERX:

```
Agente: "Necesito enviar 100 USDT a TRecipient. No tengo una cuenta de MERX."

Paso 1: Agente llama a create_paid_order
  -> Recibe factura: 1.43 TRX, memo "merx_xpay_abc123"

Paso 2: Agente llama a transfer_trx
  -> Envía 1.43 TRX a TMerxTreasury con memo "merx_xpay_abc123"
  -> Hash TX: 7f3a2b...

Paso 3: Agente espera verificación (automática)
  -> MERX detecta pago, verifica en cadena
  -> Energía delegada a dirección del agente

Paso 4: Agente llama a transfer_trc20
  -> Envía 100 USDT a TRecipient
  -> Usa energía delegada en lugar de quemar TRX
  -> Costo: 1.43 TRX en lugar de ~27 TRX

Total de interacciones con MERX: 2 llamadas a herramientas
Cuenta creada: No
Clave API utilizada: No
Fondos depositados: No
Confianza requerida: Mínima (pago verificado en cadena)
```

## Seguridad: Por Qué el Memo es Importante

El campo memo es el eje de la seguridad x402. Sin él, cualquier transferencia de TRX al tesorero de MERX podría reclamarse como pago para cualquier factura. Esto crea dos vectores de ataque:

### Ataque de Pago Cruzado

Sin verificación de memo, un atacante podría:
1. Solicitar una factura por 65,000 de energía (1.43 TRX)
2. Esperar a que alguien más envíe 1.43 TRX al tesorero de MERX por una razón no relacionada
3. Reclamar ese pago no relacionado como pago de su factura
4. Recibir energía gratuita

El memo previene esto. Cada factura tiene una cadena de memo única. Un pago solo se compara con una factura si el campo memo contiene la cadena exacta. Las transferencias aleatorias de TRX al tesorero - que ocurren en cualquier dirección activa - se ignoran porque carecen de un memo válido.

### Ataque de Reproducción

Sin expiración y cumplimiento de un solo uso, un atacante podría:
1. Pagar una factura legítimamente
2. Hacer referencia a la misma transacción de pago para una segunda factura
3. Recibir energía dos veces por un pago

MERX previene esto con dos mecanismos:
- Cada factura se marca como PAGADA después de la primera verificación exitosa. Un segundo pago que reclama el mismo memo se rechaza.
- Cada transacción de pago solo se puede comparar con una factura. El hash de la transacción se registra y no se puede reutilizar.

### Verificación de Monto

El monto del pago debe coincidir exactamente con el monto de la factura. Enviar 1.42 TRX en lugar de 1.43 TRX resulta en una verificación fallida. Esto previene ataques donde un atacante envía un monto mínimo (por ejemplo, 0.000001 TRX) con un memo válido para reclamar una factura a una fracción del precio.

## Transacción Real de Mainnet

Aquí hay una transacción x402 verificada de la red principal de TRON:

```
Factura:
  ID de Orden: xpay_m7k2p9
  Energía: 65,000
  Duración: 1 hora
  Precio: 1.43 TRX
  Memo: merx_xpay_m7k2p9

Transacción de Pago:
  Hash: 8d4f1a7b3c2e...
  Bloque: 58,234,891
  De: TWallet...
  Para: TMerxTreasury...
  Monto: 1,430,000 SUN
  Memo: merx_xpay_m7k2p9
  Estado: CONFIRMADA

Verificación:
  Marca de tiempo: 2026-03-28T14:22:17Z
  Coincidencia: Monto OK, Memo OK, No expirada, No pagada previamente
  Resultado: VERIFICADA

Delegación de Energía:
  Delegada: 65,000 de energía
  Proveedor: catfee
  TX de delegación: 3a8b2c...
  Confirmada en: 2026-03-28T14:22:23Z
  Expira en: 2026-03-28T15:22:23Z
```

De solicitud de factura a delegación de energía: 23 segundos. Sin cuenta. Sin clave API. Sin relación previa.

## Cuándo Usar x402 vs Acceso Basado en Cuenta

x402 es ideal para:

- **Agentes de IA** que operan autónomamente y no deberían depender de cuentas gestionadas
- **Compras únicas** donde el costo de creación de cuenta excede el valor de la transacción
- **Usuarios conscientes de la privacidad** que no quieren crear cuentas o proporcionar información identificativa
- **Pruebas y evaluación** donde los desarrolladores quieren probar el servicio antes de comprometerse con una cuenta
- **Agentes multiplataforma** que interactúan con muchos servicios y no deberían mantener cuentas separadas para cada uno

El acceso basado en cuenta es mejor para:

- **Operaciones de alto volumen** donde el costo por transacción de x402 (creación de factura + transmisión de pago + verificación) importa
- **Órdenes permanentes y monitores** que requieren estado persistente del lado del servidor vinculado a una cuenta
- **Facturación basada en saldo** donde el crédito prepagado es más eficiente que pagos por transacción
- **Operaciones de equipo** donde múltiples usuarios comparten una cuenta con acceso basado en roles

Muchos usuarios comienzan con x402 para evaluación y migran al acceso basado en cuenta a medida que su uso crece. Los dos modelos son complementarios, no competitivos.

## x402 y el Futuro del Comercio de Agentes

El protocolo x402 representa un cambio más amplio en cómo se consumen los servicios. La facturación tradicional de SaaS - suscripciones mensuales, precios escalonados, límites de uso - asume un cliente humano que crea una cuenta, evalúa niveles de precios, y toma una decisión de compra. Este modelo se desmorona cuando el cliente es un agente de IA que necesita tomar decisiones de compra autónomamente, en tiempo real, para tareas específicas.

x402 se alinea con la economía de agentes:

- **Sin registro** - Los agentes no tienen identidades en el sentido tradicional. x402 requiere solo una dirección de cartera.
- **Precios por uso** - Los agentes deberían pagar por lo que usan, cuándo lo usan, en montos proporcionales a la tarea en cuestión.
- **Verificación en cadena** - La confianza se establece a través de prueba criptográfica, no a través de credenciales de cuenta o relaciones comerciales.
- **Componibilidad** - Un agente puede usar x402 para pagar energía de MERX, luego usar esa energía para interactuar con cualquier contrato inteligente TRON. El pago y el servicio están desacoplados.

MERX es el primer intercambio de energía que implementa x402. A medida que el protocolo madura, esperamos que otros servicios blockchain adopten modelos de pago por uso similares optimizados para consumo de agentes.

## Detalles de Implementación para Desarrolladores

### Construir un Cliente x402

Si está construyendo una aplicación que usa MERX x402 sin el servidor MCP, aquí está la implementación mínima:

```javascript
const TronWeb = require('tronweb');

async function buyEnergyX402(tronWeb, energyAmount, durationHours, targetAddress) {
  // Paso 1: Obtener factura
  const invoiceRes = await fetch('https://merx.exchange/api/v1/x402/invoice', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      energy_amount: energyAmount,
      duration_hours: durationHours,
      target_address: targetAddress
    })
  });
  const invoice = await invoiceRes.json();

  // Paso 2: Construir transacción de pago
  const tx = await tronWeb.transactionBuilder.sendTrx(
    invoice.pay_to,
    invoice.amount_sun,
    tronWeb.defaultAddress.base58
  );

  const txWithMemo = await tronWeb.transactionBuilder.addUpdateData(
    tx, invoice.memo, 'utf8'
  );

  // Paso 3: Firmar y transmitir
  const signedTx = await tronWeb.trx.sign(txWithMemo);
  const result = await tronWeb.trx.sendRawTransaction(signedTx);

  // Paso 4: Sondear finalización de orden
  let order;
  for (let i = 0; i < 20; i++) {
    await new Promise(r => setTimeout(r, 3000));
    const orderRes = await fetch(
      `https://merx.exchange/api/v1/x402/order/${invoice.order_id}`
    );
    order = await orderRes.json();
    if (order.status === 'completed') break;
  }

  return order;
}
```

### Manejo de Errores

Modos de fallo comunes y sus resoluciones:

| Error | Causa | Resolución |
|---|---|---|
| `INVOICE_EXPIRED` | Pago no enviado dentro de 5 minutos | Solicitar nueva factura |
| `AMOUNT_MISMATCH` | El monto de pago difiere de la factura | Enviar monto exacto; solicitar nueva factura si el precio cambió |
| `MEMO_NOT_FOUND` | Pago sin campo memo | Reenviar con memo correcto; los fondos del intento fallido no se reembolsan automáticamente |
| `ALREADY_PAID` | La factura ya fue cumplida | Verificar estado de orden; esto no es un error si está sondeando |
| `PROVIDER_UNAVAILABLE` | Ningún proveedor puede llenar la orden | Reintentar después de unos minutos; el monto de pago será acreditado a un saldo de MERX para retiro manual |

### Política de Reembolso

Si MERX no puede cumplir la orden después de que el pago se verifica (por ejemplo, todos los proveedores no están disponibles temporalmente), el monto de pago se acredita a un saldo reclamable asociado con la dirección TRON del pagador. El pagador puede reclamar este saldo a través de un punto final separado sin crear una cuenta - la reclamación se verifica firmando un mensaje con la misma clave privada que envió el pago original.

## Conclusión

El código de estado HTTP 402 fue definido hace casi 30 años con la visión de pagos nativos en internet. Esa visión fue adelantada a su tiempo. Las blockchains finalmente proporcionan la infraestructura para hacerla real.

MERX x402 convierte la compra de energía en un flujo atómico único: solicitar, pagar, recibir. Sin cuentas. Sin claves API. Sin suposiciones de confianza más allá de lo que la blockchain proporciona.

Para agentes de IA, este es el modelo de compra natural. Para desarrolladores, es el camino de integración más simple. Para el ecosistema, es una prueba de concepto de cómo se consumirán los servicios blockchain en la economía de agentes.

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

Pregunta a tu agente de IA: "¿Cuál es la energía TRON más barata en este momento?" y obtén precios en vivo de todos los proveedores conectados.

Documentación MCP completa: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)