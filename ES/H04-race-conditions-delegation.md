# Condiciones de carrera en delegación de energía: cómo las resolvimos

Esta es la historia de un error que costaba 19 TRX por transacción en lugar de ahorrar 1.43 TRX. Es la historia de una condición de carrera que parecía correcta en las pruebas, funcionaba en testnet y solo se reveló bajo condiciones de mainnet. Y es la historia de la solución: un mecanismo de sondeo que espera la confirmación en cadena antes de permitir que una transacción proceda.

## La configuración

MERX compra energía de proveedores en nombre de los usuarios. El flujo, en teoría, es directo:

1. El usuario solicita energía para su dirección
2. MERX selecciona el proveedor más barato
3. El proveedor delega energía a la dirección del usuario
4. El usuario ejecuta su transacción con la energía delegada
5. La transacción consume la energía alquilada en lugar de quemar TRX

La palabra crítica es "antes". La transacción del usuario debe ejecutarse después de que la delegación se confirme en cadena. Si la transacción se transmite antes de que la delegación llegue, la transacción seguirá siendo exitosa, pero quemará TRX según la tasa de quema del protocolo en lugar de consumir la energía alquilada. El usuario paga dos veces: una por el alquiler y otra a través de la quema de TRX.

## El error

La implementación inicial seguía un patrón secuencial:

```typescript
async function executeWithEnergy(
  targetAddress: string,
  energyAmount: number,
  transactionFn: () => Promise<string>
): Promise<string> {
  // Step 1: Buy energy
  const order = await merx.createOrder({
    energy_amount: energyAmount,
    target_address: targetAddress,
    duration: '1h'
  });

  // Step 2: Wait for order confirmation from MERX
  await waitForOrderStatus(order.id, 'FILLED');

  // Step 3: Execute the transaction
  const txHash = await transactionFn();

  return txHash;
}
```

Esto se ve correcto. Compra energía, espera a que el pedido se complete, luego ejecuta la transacción. La función `waitForOrderStatus` sondeaba la API de MERX hasta que el estado del pedido cambiaba a `FILLED`.

El problema es qué significa "FILLED". En el sistema MERX, un pedido se marca como `FILLED` cuando el proveedor confirma que ha iniciado la delegación. El proveedor devuelve una respuesta de éxito: "He presentado la transacción de delegación a la red TRON".

Pero "presentado" no es "confirmado". La delegación es una transacción TRON que debe incluirse en un bloque y ser procesada por la red. Entre la presentación y la confirmación hay una ventana: típicamente de 3 a 6 segundos, pero ocasionalmente más larga durante la congestión de la red.

Durante esa ventana, la dirección de destino aún no tiene la energía delegada.

### Qué sucedió en mainnet

El error se manifestó exactamente como predijo la condición de carrera:

```
Timeline:

  T+0.0s    Order created, sent to provider
  T+0.3s    Provider responds: delegation TX submitted
  T+0.4s    Order status -> FILLED
  T+0.5s    User transaction broadcast (USDT transfer)
  T+0.6s    User TX included in block N
  T+3.2s    Delegation TX included in block N+1

  Result:
    - User TX in block N: no delegation exists yet
    - Energy consumed: 0 (delegation not active)
    - TRX burned: 65,000 * 420 SUN = 27,300,000 SUN = 27.3 TRX
    - Energy rental cost: 1,820,000 SUN = 1.82 TRX
    - Total cost: 29.12 TRX (rental + burn)
    - Expected cost: 1.82 TRX (rental only)
```

El usuario pagó 29.12 TRX en lugar de 1.82 TRX. El alquiler de energía se desperdició porque la delegación llegó un bloque demasiado tarde.

### Los números

En mainnet de TRON, cada bloque toma aproximadamente 3 segundos. Una transacción de delegación presentada por el proveedor pasa por el mismo proceso de inclusión de bloque que cualquier otra transacción. Si la transacción del usuario y la transacción de delegación se presentan dentro de unos segundos la una de la otra, pueden terminar en bloques diferentes, sin garantía de orden.

La diferencia de costo es dramática:

```
Without delegation (TRX burn):
  65,000 energy * 420 SUN/energy = 27,300,000 SUN = 27.3 TRX

With delegation (rental):
  65,000 energy * 28 SUN/energy = 1,820,000 SUN = 1.82 TRX

Money wasted per race condition occurrence:
  27.3 + 1.82 = 29.12 TRX total cost
  vs.
  1.82 TRX expected cost

  Overpayment: 27.3 TRX per incident
```

En un día malo, esta condición de carrera podría ocurrir en el 10-20% de los pedidos donde el cliente se ejecutaba inmediatamente después de recibir el estado `FILLED`.

## Por qué las pruebas no lo detectaron

### Comportamiento de testnet

En testnet Shasta, los tiempos de bloque son similares a mainnet, pero la congestión de la red es mínima. Las transacciones de delegación generalmente se incluyen en el siguiente bloque. La ventana entre "presentado" y "confirmado" fue consistentemente inferior a 3 segundos en testnet, y nuestro arnés de pruebas tenía un retraso incorporado de 2 segundos entre pasos que enmascaraba la condición de carrera.

### Pruebas secuenciales

Nuestras pruebas de integración eran secuenciales. Una prueba compraría energía, esperaría, ejecutaría una transacción y verificaría. Nunca hubo carga concurrente, nunca una carrera entre delegación y ejecución, nunca la presión de tiempo que mainnet produjo.

### La trampa del estado del pedido

El aspecto más insidioso: el estado del pedido era técnicamente correcto. El pedido fue `FILLED`: el proveedor aceptó e inició la delegación. El error no estaba en el seguimiento del estado. Estaba en la suposición de que `FILLED` significaba "la energía está disponible en cadena".

## La solución

La solución tiene dos partes: un paso de verificación que comprueba los recursos en cadena de la dirección de destino, y un bucle de sondeo que espera hasta que la delegación sea realmente confirmada.

### Parte 1: check_address_resources

TRON proporciona una API para verificar los recursos (energía y bandwidth) actualmente disponibles para cualquier dirección:

```
GET https://api.trongrid.io/wallet/getaccountresource
```

Esto devuelve el límite de energía actual, energía utilizada, límite de bandwidth y bandwidth utilizado para una dirección. Críticamente, esto refleja el estado en cadena: si una delegación ha sido confirmada, el límite de energía lo reflejará. Si la delegación aún está pendiente, el límite de energía no la incluirá.

### Parte 2: Sondear hasta confirmar

La solución reemplaza la verificación única "esperar FILLED" con un bucle de sondeo que verifica los recursos en cadena:

```typescript
async function executeWithEnergy(
  targetAddress: string,
  energyAmount: number,
  transactionFn: () => Promise<string>
): Promise<string> {
  // Step 1: Check baseline resources
  const baseline = await checkAddressResources(targetAddress);
  const baselineEnergy = baseline.energy_limit - baseline.energy_used;

  // Step 2: Buy energy
  const order = await merx.createOrder({
    energy_amount: energyAmount,
    target_address: targetAddress,
    duration: '1h'
  });

  // Step 3: Wait for order to be filled by the provider
  await waitForOrderStatus(order.id, 'FILLED');

  // Step 4: Poll on-chain resources until delegation is confirmed
  const confirmed = await pollUntilDelegationConfirmed(
    targetAddress,
    baselineEnergy,
    energyAmount,
    { maxAttempts: 15, intervalMs: 2000 }
  );

  if (!confirmed) {
    throw new Error(
      'Delegation not confirmed on-chain within timeout. ' +
      'Do not execute transaction -- energy may not be available.'
    );
  }

  // Step 5: Execute the transaction (delegation is confirmed on-chain)
  const txHash = await transactionFn();

  return txHash;
}

async function pollUntilDelegationConfirmed(
  address: string,
  baselineEnergy: number,
  expectedIncrease: number,
  options: { maxAttempts: number; intervalMs: number }
): Promise<boolean> {
  for (let attempt = 0; attempt < options.maxAttempts; attempt++) {
    const resources = await checkAddressResources(address);
    const currentEnergy = resources.energy_limit - resources.energy_used;
    const increase = currentEnergy - baselineEnergy;

    if (increase >= expectedIncrease * 0.95) {
      // Allow 5% tolerance for rounding
      return true;
    }

    await sleep(options.intervalMs);
  }

  return false;
}

async function checkAddressResources(
  address: string
): Promise<{ energy_limit: number; energy_used: number }> {
  const response = await fetch(
    'https://api.trongrid.io/wallet/getaccountresource',
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        address: address,
        visible: true
      })
    }
  );

  const data = await response.json();
  return {
    energy_limit: data.EnergyLimit || 0,
    energy_used: data.EnergyUsed || 0
  };
}
```

### Por qué intervalos de 2 segundos

El intervalo de sondeo es de 2 segundos. Esto se elige basándose en el tiempo de bloque de 3 segundos de TRON:

- Sondear cada 1 segundo resultaría en muchas verificaciones redundantes entre bloques
- Sondear cada 3 segundos podría perder algunos bloques (desviación del reloj, latencia de red)
- Sondear cada 2 segundos asegura que verificamos dentro de cada ciclo de bloque

### Por qué 15 intentos

15 intentos a intervalos de 2 segundos = tiempo de espera máximo de 30 segundos. En la práctica, la delegación se confirma dentro de 1-2 sondeos (3-6 segundos). El tiempo de espera de 30 segundos maneja casos extremos:

- Congestión de red retrasando la inclusión de bloques
- Proveedor presentando la transacción de delegación tarde
- Retraso temporal del nodo RPC

Si la delegación no se confirma después de 30 segundos, algo genuinamente está mal, y es más seguro fallar estrepitosamente que proceder y quemar TRX.

## Datos reales de mainnet

Después de desplegar la solución, medimos el comportamiento del sondeo durante una semana de tráfico de producción:

```
Delegation confirmation timing:

  Confirmed on first poll (0-2s):     12%
  Confirmed on second poll (2-4s):    61%
  Confirmed on third poll (4-6s):     22%
  Confirmed on fourth poll (6-8s):    4%
  Confirmed on fifth+ poll (8s+):     1%
  Timeout (not confirmed in 30s):     0.04%

Average time from order FILLED to on-chain confirmation: 3.1 seconds
Median: 2.8 seconds
P99: 8.2 seconds
```

Los datos confirman la ventana de condición de carrera: 3.1 segundos en promedio entre la respuesta "FILLED" del proveedor y la confirmación en cadena. Sin la solución de sondeo, cualquier transacción ejecutada dentro de esa ventana habría quemado TRX.

### Impacto de costos

```
Before fix (30 days):
  Orders affected by race condition:  ~180
  Average overpayment per incident:   ~19 TRX
  Total cost of race conditions:      ~3,420 TRX

After fix (30 days):
  Race condition incidents:           0
  Timeout failures (delegation never confirmed): 2
    Both caught by the timeout, transaction not executed
    Orders refunded automatically
```

La solución eliminó la condición de carrera completamente. Los dos fallos por tiempo de espera fueron problemas genuinos del lado del proveedor donde la transacción de delegación nunca fue presentada, exactamente los casos donde deseas fallar en lugar de proceder.

## Lecciones aprendidas

### "Presentado" no es "confirmado"

Esta es la lección central. En cualquier sistema que interactúe con una blockchain, siempre hay una brecha entre presentar una transacción y que esa transacción sea confirmada en cadena. Cualquier lógica que trate la presentación como confirmación eventualmente fallará.

### Verifica la cadena, no el servicio

El proveedor dice que la delegación está hecha. MERX dice que el pedido está completo. Pero ninguna de estas afirmaciones significa que la delegación existe en cadena en este momento. La única fuente autorizada es la blockchain misma. Verifica la cadena.

### Testnet oculta errores de tiempo

Las testnets tienen menor carga, inclusión más rápida y comportamiento más predecible. Los errores sensibles al tiempo que nunca se disparan en testnet aparecerán en mainnet. Si tu lógica depende del tiempo entre dos eventos en cadena, pruébalo bajo condiciones de carga realistas.

### Falla estrepitosamente

Cuando el bucle de sondeo agota el tiempo de espera, la solución lanza un error e impide que la transacción se ejecute. Este es el comportamiento correcto. La alternativa, ejecutar de todas formas y esperar que la delegación haya llegado, cuesta 19 TRX por fallo. Un mensaje de error no cuesta nada.

## La implementación de MERX

La herramienta `ensure_resources` en el servidor MCP de MERX implementa este patrón. Cuando un agente de IA llama a `ensure_resources` antes de ejecutar una llamada de contrato, la herramienta:

1. Verifica los recursos actuales en cadena
2. Calcula el déficit
3. Compra exactamente la energía necesaria
4. Sondea hasta que la delegación se confirme en cadena
5. Devuelve éxito solo cuando los recursos se verifican

El agente nunca tiene que implementar lógica de sondeo él mismo. La condición de carrera se maneja a nivel de plataforma.

```
Tool: ensure_resources
Input: {
  "address": "TYourAddress...",
  "energy_needed": 65000
}

Response: {
  "status": "confirmed",
  "energy_available": 65000,
  "confirmation_time_ms": 4200,
  "order_id": "ord_abc123"
}
```

El campo `confirmation_time_ms` le indica al agente cuánto tiempo tomó el sondeo. El `status: "confirmed"` significa que la energía está en cadena y es segura para usar.

Plataforma: [https://merx.exchange](https://merx.exchange)
Documentación: [https://merx.exchange/docs](https://merx.exchange/docs)
Servidor MCP: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)


## Pruébalo ahora con IA

Añade MERX a Claude Desktop o cualquier cliente compatible con MCP — sin instalación, sin necesidad de clave API para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregunta a tu agente de IA: "¿Cuál es la energía de TRON más barata en este momento?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)