# Construir un Bot de Pagos USDT con MERX SDK

Enviar USDT en TRON debería ser simple. Tienes una dirección de destinatario, una cantidad y una billetera con fondos. Pero si envías USDT sin energía, el protocolo TRON quema aproximadamente 27 TRX de tu billetera para cubrir el costo de la transacción. A escala —cientos o miles de transferencias por día— ese costo de quema se convierte en un gasto significativo.

Este tutorial construye un bot completo de pagos USDT que elimina ese costo. El bot recibe solicitudes de pago, renta energía a través de MERX a una fracción del precio de quema, envía el USDT con la energía alquilada e informa los ahorros. Maneja errores, admite reintentos e integra notificaciones webhook para confiabilidad en producción.

Al final, tendrás un bot de pagos funcional que ahorra más del 90 por ciento en cada transferencia de USDT.

## Descripción General de la Arquitectura

El bot sigue un pipeline sencillo:

1. Recibir una solicitud de pago (dirección del destinatario + cantidad).
2. Verificar el saldo de tu cuenta MERX.
3. Estimar la energía requerida para la transferencia.
4. Crear una orden de energía a través de MERX.
5. Esperar a que se complete la delegación de energía.
6. Enviar la transferencia USDT usando la energía delegada.
7. Informar la transacción y los ahorros.

Cada paso está aislado y es reintentable. Si la orden de energía falla, el USDT nunca se envía. Si la transferencia USDT falla, aún tienes la energía (expira después del período de alquiler, pero no se pierden fondos).

## Requisitos Previos

Antes de construir, necesitas:

- **Node.js 18 o posterior** - el bot usa módulos ES y características modernas de JavaScript.
- **Una cuenta MERX con saldo** - regístrate en [merx.exchange](https://merx.exchange) y deposita TRX.
- **Una clave API de MERX** - crea una con permisos `create_orders`, `view_orders` y `view_balance`.
- **Una billetera TRON** - con saldo de USDT para los pagos y una pequeña cantidad de TRX para bandwidth.
- **TronWeb** - para firmar y transmitir la transacción de transferencia USDT.

### Configuración del Proyecto

```bash
mkdir usdt-payment-bot
cd usdt-payment-bot
npm init -y
npm install merx-sdk tronweb dotenv uuid
```

Crea un archivo `.env` (nunca confirmes esto en control de versiones):

```bash
MERX_API_KEY=merx_live_your_key_here
TRON_PRIVATE_KEY=your_tron_wallet_private_key
TRON_FULL_HOST=https://api.trongrid.io
USDT_CONTRACT=TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t
MIN_SAVINGS_PERCENT=50
```

## Paso 1: Inicializar los Clientes

Crea el cliente MERX e instancia de TronWeb:

```javascript
// src/clients.js
import { MerxClient } from 'merx-sdk';
import TronWeb from 'tronweb';
import dotenv from 'dotenv';

dotenv.config();

export const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
  baseUrl: 'https://merx.exchange/api/v1',
});

export const tronWeb = new TronWeb({
  fullHost: process.env.TRON_FULL_HOST,
  privateKey: process.env.TRON_PRIVATE_KEY,
});

export const USDT_CONTRACT = process.env.USDT_CONTRACT;
export const SENDER_ADDRESS = tronWeb.defaultAddress.base58;
```

## Paso 2: Verificar Saldo de MERX

Antes de hacer cualquier cosa, verifica que tengas saldo suficiente para alquilar energía:

```javascript
// src/balance.js
import { merx } from './clients.js';

export async function checkMerxBalance(requiredSun) {
  const balance = await merx.account.getBalance();

  const availableSun = balance.available_sun;

  if (availableSun < requiredSun) {
    throw new Error(
      `Insufficient MERX balance. ` +
        `Available: ${(availableSun / 1_000_000).toFixed(2)} TRX, ` +
        `Required: ${(requiredSun / 1_000_000).toFixed(2)} TRX`
    );
  }

  return {
    available: availableSun,
    availableTRX: (availableSun / 1_000_000).toFixed(2),
  };
}
```

## Paso 3: Estimar Costo de Energía

Usa la API de estimación de MERX para determinar cuánta energía necesita la transferencia y cuánto costará:

```javascript
// src/estimate.js
import { merx, SENDER_ADDRESS } from './clients.js';

export async function estimateTransferCost() {
  const estimate = await merx.estimate({
    operation: 'trc20_transfer',
    target_address: SENDER_ADDRESS,
  });

  const savings = estimate.costs.savings;
  const rental = estimate.costs.rental;
  const burn = estimate.costs.burn;

  return {
    energyRequired: estimate.energy_required,
    bandwidthRequired: estimate.bandwidth_required,
    burnCostTRX: burn.trx_cost_readable,
    burnCostSun: burn.trx_cost,
    rentalCostTRX: rental.total_cost_trx,
    rentalCostSun: rental.total_cost_sun,
    bestProvider: rental.best_provider,
    savingsPercent: savings.percent,
    savingsTRX: savings.trx_saved,
    durationHours: rental.duration_hours,
  };
}
```

## Paso 4: Crear la Orden de Energía

Ordena energía a través de MERX con una clave de idempotencia para reintentos seguros:

```javascript
// src/order.js
import { merx } from './clients.js';
import { randomUUID } from 'crypto';

export async function createEnergyOrder(energyAmount, targetAddress, durationHours = 1) {
  const idempotencyKey = randomUUID();

  const order = await merx.orders.create(
    {
      energy_amount: energyAmount,
      target_address: targetAddress,
      duration_hours: durationHours,
    },
    {
      idempotencyKey,
    }
  );

  return {
    orderId: order.id,
    status: order.status,
    provider: order.provider,
    totalCostSun: order.total_cost_sun,
    idempotencyKey,
  };
}
```

## Paso 5: Esperar Delegación de Energía

Después de crear la orden, espera a que la energía se delegue en cadena. El bot sondea el estado de la orden con un tiempo de espera:

```javascript
// src/wait.js
import { merx } from './clients.js';

export async function waitForFill(orderId, timeoutMs = 60000) {
  const startTime = Date.now();
  const pollInterval = 2000; // 2 segundos

  while (Date.now() - startTime < timeoutMs) {
    const order = await merx.orders.get(orderId);

    if (order.status === 'filled') {
      return {
        status: 'filled',
        provider: order.provider,
        txHash: order.delegation_tx_hash,
        filledAt: order.filled_at,
      };
    }

    if (order.status === 'failed' || order.status === 'cancelled') {
      throw new Error(
        `Order ${orderId} ${order.status}: ${order.failure_reason || 'unknown reason'}`
      );
    }

    // Aún pendiente o procesándose - espera y sondea de nuevo
    await new Promise((resolve) => setTimeout(resolve, pollInterval));
  }

  throw new Error(`Order ${orderId} timed out after ${timeoutMs / 1000}s`);
}
```

Para sistemas en producción, considera usar webhooks en lugar de sondeo (cubierto más adelante en este artículo).

## Paso 6: Enviar la Transferencia USDT

Con energía delegada a tu dirección, envía la transferencia USDT. La energía se consume en lugar de quemar TRX:

```javascript
// src/transfer.js
import { tronWeb, USDT_CONTRACT, SENDER_ADDRESS } from './clients.js';

export async function sendUSDT(recipientAddress, amountUSDT) {
  // USDT tiene 6 decimales en TRON
  const amountSun = Math.floor(amountUSDT * 1_000_000);

  // Validar dirección del destinatario
  if (!tronWeb.isAddress(recipientAddress)) {
    throw new Error(`Invalid TRON address: ${recipientAddress}`);
  }

  // Construir la transacción de transferencia TRC-20
  const contract = await tronWeb.contract().at(USDT_CONTRACT);

  const tx = await contract.methods
    .transfer(recipientAddress, amountSun)
    .send({
      from: SENDER_ADDRESS,
      feeLimit: 100_000_000, // límite de tarifa de 100 TRX (límite de seguridad)
    });

  return {
    txHash: tx,
    recipient: recipientAddress,
    amount: amountUSDT,
    amountSun,
  };
}
```

## Paso 7: Ponerlo Todo Junto

La función principal del bot orquesta todos los pasos:

```javascript
// src/bot.js
import { checkMerxBalance } from './balance.js';
import { estimateTransferCost } from './estimate.js';
import { createEnergyOrder } from './order.js';
import { waitForFill } from './wait.js';
import { sendUSDT } from './transfer.js';
import { SENDER_ADDRESS } from './clients.js';

const MIN_SAVINGS_PERCENT = parseFloat(process.env.MIN_SAVINGS_PERCENT || '50');

export async function processPayment(recipientAddress, amountUSDT) {
  const startTime = Date.now();

  console.log(`--- Payment Request ---`);
  console.log(`Recipient: ${recipientAddress}`);
  console.log(`Amount: ${amountUSDT} USDT`);

  // Paso 1: Estimar costos
  console.log('\n[1/5] Estimating energy cost...');
  const estimate = await estimateTransferCost();
  console.log(`  Energy required: ${estimate.energyRequired}`);
  console.log(`  Burn cost: ${estimate.burnCostTRX}`);
  console.log(`  Rental cost: ${estimate.rentalCostTRX}`);
  console.log(`  Savings: ${estimate.savingsPercent}%`);

  // Paso 2: Decidir si el alquiler es rentable
  if (estimate.savingsPercent < MIN_SAVINGS_PERCENT) {
    console.log(`  Savings below threshold (${MIN_SAVINGS_PERCENT}%). Skipping energy rental.`);
    const tx = await sendUSDT(recipientAddress, amountUSDT);
    return { ...tx, energyRented: false, savings: null };
  }

  // Paso 3: Verificar saldo de MERX
  console.log('\n[2/5] Checking MERX balance...');
  const balance = await checkMerxBalance(estimate.rentalCostSun);
  console.log(`  Available: ${balance.availableTRX} TRX`);

  // Paso 4: Crear orden de energía
  console.log('\n[3/5] Creating energy order...');
  const order = await createEnergyOrder(
    estimate.energyRequired,
    SENDER_ADDRESS,
    estimate.durationHours
  );
  console.log(`  Order ID: ${order.orderId}`);
  console.log(`  Provider: ${order.provider}`);
  console.log(`  Status: ${order.status}`);

  // Paso 5: Esperar delegación de energía
  console.log('\n[4/5] Waiting for energy delegation...');
  const fill = await waitForFill(order.orderId);
  console.log(`  Filled by: ${fill.provider}`);
  console.log(`  Delegation TX: ${fill.txHash}`);

  // Paso 6: Enviar USDT con energía delegada
  console.log('\n[5/5] Sending USDT...');
  const tx = await sendUSDT(recipientAddress, amountUSDT);
  console.log(`  TX Hash: ${tx.txHash}`);

  const elapsed = ((Date.now() - startTime) / 1000).toFixed(1);

  console.log(`\n--- Payment Complete ---`);
  console.log(`  Total time: ${elapsed}s`);
  console.log(`  Energy cost: ${estimate.rentalCostTRX}`);
  console.log(`  Saved: ${estimate.savingsTRX} (${estimate.savingsPercent}%)`);

  return {
    txHash: tx.txHash,
    recipient: recipientAddress,
    amount: amountUSDT,
    energyRented: true,
    energyOrderId: order.orderId,
    rentalCost: estimate.rentalCostTRX,
    burnCostAvoided: estimate.burnCostTRX,
    savingsPercent: estimate.savingsPercent,
    savingsTRX: estimate.savingsTRX,
    elapsedSeconds: parseFloat(elapsed),
  };
}
```

### Ejecutar el Bot

```javascript
// index.js
import { processPayment } from './src/bot.js';

const recipient = process.argv[2];
const amount = parseFloat(process.argv[3]);

if (!recipient || !amount) {
  console.error('Usage: node index.js <recipient_address> <amount_usdt>');
  process.exit(1);
}

try {
  const result = await processPayment(recipient, amount);
  console.log('\nResult:', JSON.stringify(result, null, 2));
} catch (err) {
  console.error('\nPayment failed:', err.message);
  process.exit(1);
}
```

```bash
node index.js TRecipientAddressHere 100
```

Salida de ejemplo:

```
--- Payment Request ---
Recipient: TRecipientAddressHere
Amount: 100 USDT

[1/5] Estimating energy cost...
  Energy required: 64895
  Burn cost: 27.37 TRX
  Rental cost: 1.43 TRX
  Savings: 94.8%

[2/5] Checking MERX balance...
  Available: 50.00 TRX

[3/5] Creating energy order...
  Order ID: ord_7k3m9x2p
  Provider: sohu
  Status: pending

[4/5] Waiting for energy delegation...
  Filled by: sohu
  Delegation TX: 5a8f2c1e...

[5/5] Sending USDT...
  TX Hash: 3d7b4e9a...

--- Payment Complete ---
  Total time: 8.2s
  Energy cost: 1.43 TRX
  Saved: 25.94 TRX (94.8%)
```

## Manejo de Errores

Los sistemas de pagos en producción necesitan manejo de errores sólido. Estos son los modos de fallo y cómo manejar cada uno.

### Saldo Insuficiente en MERX

Si tu cuenta MERX se agota, el bot debería alertarte y opcionalmente recurrir a quemar TRX:

```javascript
try {
  await checkMerxBalance(estimate.rentalCostSun);
} catch (err) {
  if (err.message.includes('Insufficient MERX balance')) {
    console.warn('MERX balance low. Sending without energy rental.');
    // Opcionalmente: enviar alerta al equipo de operaciones
    // await alertOps('MERX balance low', err.message);
    const tx = await sendUSDT(recipientAddress, amountUSDT);
    return { ...tx, energyRented: false, fallbackReason: 'low_balance' };
  }
  throw err;
}
```

### Tiempo de Espera de Llenado de Orden

Si el proveedor de energía tarda demasiado en delegar, cancela e intenta de nuevo con un proveedor diferente o recurre:

```javascript
try {
  const fill = await waitForFill(order.orderId, 30000); // tiempo de espera de 30 segundos
} catch (err) {
  if (err.message.includes('timed out')) {
    console.warn('Energy delegation timed out. Sending without energy.');
    const tx = await sendUSDT(recipientAddress, amountUSDT);
    return { ...tx, energyRented: false, fallbackReason: 'delegation_timeout' };
  }
  throw err;
}
```

### Fallo de Transferencia USDT

Si la transferencia USDT en sí falla (saldo insuficiente, error de contrato), la energía no se desperdicia —permanece delegada durante la duración del alquiler. Puedes reintentar la transferencia:

```javascript
async function sendUSDTWithRetry(recipientAddress, amountUSDT, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await sendUSDT(recipientAddress, amountUSDT);
    } catch (err) {
      if (attempt === maxRetries) throw err;
      console.warn(`Transfer attempt ${attempt} failed: ${err.message}`);
      await new Promise((r) => setTimeout(r, 2000 * attempt));
    }
  }
}
```

## Notificaciones Webhook

El sondeo para cambios de estado de orden funciona para bots simples, pero los sistemas en producción deben usar webhooks. MERX envía notificaciones HTTP POST a tu URL webhook cuando el estado de la orden cambia.

### Configurar Webhooks

Configura tu endpoint webhook en el panel de control de MERX o a través de la API:

```bash
curl -X POST https://merx.exchange/api/v1/webhooks \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-server.com/webhooks/merx",
    "events": ["order.filled", "order.failed"]
  }'
```

### Manejar Eventos Webhook

```javascript
// webhook-handler.js
import express from 'express';

const app = express();
app.use(express.json());

// Almacenar devoluciones de llamada de pagos pendientes
const pendingPayments = new Map();

app.post('/webhooks/merx', (req, res) => {
  const event = req.body;

  // Verificar firma webhook (requisito de producción)
  // const isValid = verifyWebhookSignature(req);
  // if (!isValid) return res.status(401).json({ error: 'Invalid signature' });

  if (event.type === 'order.filled') {
    const callback = pendingPayments.get(event.data.order_id);
    if (callback) {
      callback.resolve(event.data);
      pendingPayments.delete(event.data.order_id);
    }
  }

  if (event.type === 'order.failed') {
    const callback = pendingPayments.get(event.data.order_id);
    if (callback) {
      callback.reject(new Error(`Order failed: ${event.data.reason}`));
      pendingPayments.delete(event.data.order_id);
    }
  }

  res.status(200).json({ received: true });
});

// Reemplazar sondeo con espera basada en webhook
export function waitForFillWebhook(orderId, timeoutMs = 60000) {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => {
      pendingPayments.delete(orderId);
      reject(new Error(`Order ${orderId} webhook timed out`));
    }, timeoutMs);

    pendingPayments.set(orderId, {
      resolve: (data) => {
        clearTimeout(timer);
        resolve(data);
      },
      reject: (err) => {
        clearTimeout(timer);
        reject(err);
      },
    });
  });
}
```

Los webhooks eliminan el bucle de sondeo, reducen llamadas API y responden más rápido a los llenados de órdenes (notificación subsecundo versus intervalos de sondeo de 2 segundos).

## Consideraciones de Producción

### Concurrencia

Si tu bot procesa múltiples pagos simultáneamente, cada pago debe tener su propia clave de idempotencia y operar de forma independiente. Usa una cola (Bull, BullMQ o similar) para gestionar pagos concurrentes:

```javascript
import Queue from 'bull';

const paymentQueue = new Queue('payments', {
  redis: { host: '127.0.0.1', port: 6379 },
});

paymentQueue.process(5, async (job) => {
  // Procesar hasta 5 pagos concurrentemente
  const { recipient, amount } = job.data;
  return await processPayment(recipient, amount);
});

// Agregar un pago a la cola
await paymentQueue.add({
  recipient: 'TRecipientAddressHere',
  amount: 100,
});
```

### Monitoreo y Alertas

Realiza un seguimiento de métricas clave para visibilidad operacional:

- **Pagos por hora** - seguimiento del rendimiento.
- **Porcentaje de ahorros promedio** - si los ahorros caen por debajo del 80 por ciento, investiga los precios del proveedor.
- **Saldo de MERX** - alerta cuando el saldo caiga por debajo de un umbral (por ejemplo, suficiente para 100 transferencias).
- **Tiempo de llenado** - si la delegación de energía toma más de 30 segundos, los proveedores pueden estar congestionados.
- **Tasa de fallo** - porcentaje de intentos de pago que fallan en cualquier paso.

Realiza un seguimiento de estos en tu herramienta de monitoreo preferida (Prometheus, Datadog o incluso un simple registro JSON).

### Límites de Velocidad

La API de MERX limita la creación de órdenes a 10 solicitudes por minuto. Si tu bot necesita enviar más de 10 pagos por minuto, tienes dos opciones:

1. **Cola de pagos** y procesarlos a una velocidad dentro del límite.
2. **Compras de energía por lotes** - compra suficiente energía para múltiples transferencias en una sola orden, luego envía las transferencias secuencialmente mientras la energía está activa.

La opción 2 es más eficiente para escenarios de alto volumen:

```javascript
async function processBatch(payments) {
  // Calcular energía total para todos los pagos
  const totalEnergy = payments.length * 65000; // ~65K por transferencia USDT

  // Orden única de energía para todo el lote
  const order = await createEnergyOrder(totalEnergy, SENDER_ADDRESS, 1);
  await waitForFill(order.orderId);

  // Enviar todas las transferencias USDT mientras la energía está activa
  const results = [];
  for (const payment of payments) {
    try {
      const tx = await sendUSDT(payment.recipient, payment.amount);
      results.push({ success: true, ...tx });
    } catch (err) {
      results.push({ success: false, error: err.message, ...payment });
    }
  }

  return results;
}
```

### Lista de Verificación de Seguridad

Antes de desplegar en producción:

- Almacena la clave privada de TRON en un gestor de secretos, no en `.env` en disco.
- Ejecuta el bot en un entorno restringido sin acceso a red entrante excepto el endpoint webhook.
- Usa una billetera TRON dedicada para el bot solo con el USDT necesario para pagos a corto plazo. No mantengas toda tu tesorería en la billetera del bot.
- Implementa límites de retiro - el bot no debería poder drenar la billetera en una sola ejecución.
- Registra cada pago con detalles completos para auditoría.
- Configura alertas para actividad inusual (montos de pago por encima del umbral, fallos sucesivos rápidos).

## Estructura Completa del Proyecto

```
usdt-payment-bot/
  .env                  # Nunca confirmar
  .gitignore
  package.json
  index.js              # Punto de entrada
  src/
    clients.js          # Inicialización de MERX + TronWeb
    balance.js          # Verificación de saldo
    estimate.js         # Estimación de costos
    order.js            # Creación de orden de energía
    wait.js             # Sondeo de llenado de orden
    transfer.js         # Ejecución de transferencia USDT
    bot.js              # Orquestación principal
    webhook-handler.js  # Receptor webhook (opcional)
```

Cada archivo tiene una única responsabilidad. La base de código total tiene menos de 300 líneas. Simula el cliente MERX para pruebas unitarias, usa la testnet Shasta para pruebas de integración.

## Conclusión

Este bot de pagos demuestra el patrón de integración MERX principal: estimar, ordenar, esperar, transaccionar. El mismo patrón se aplica ya sea que estés construyendo un procesador de pagos, una billetera, un sistema de retiro de intercambio o cualquier aplicación que envíe tokens TRC-20.

La conclusión clave es la matemática de ahorros. Con un 94 por ciento de ahorros por transferencia, un bot que procesa 1,000 transferencias USDT por día ahorra aproximadamente 26,000 TRX diarios —más de 750,000 TRX por mes. Eso no es una optimización. Eso es un cambio fundamental en la estructura de costos.

Comienza en la testnet Shasta (`TRON_FULL_HOST=https://api.shasta.trongrid.io`) para validar el flujo sin arriesgar fondos reales, luego cambia a mainnet cuando todo funciona.

- Plataforma MERX: [merx.exchange](https://merx.exchange)
- Documentación de API: [merx.exchange/docs](https://merx.exchange/docs)
- SDK de JavaScript: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- SDK de Python: [github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)
- Servidor MCP para agentes IA: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)


## Pruébalo Ahora con IA

Agrega MERX a Claude Desktop o cualquier cliente compatible con MCP —sin instalación, sin clave API necesaria para herramientas de solo lectura:

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