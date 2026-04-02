# SUN, TRX, energia, ancho de banda: todas las unidades de TRON explicadas

Si alguna vez ha intentado desarrollar en TRON o simplemente enviar USDT, probablemente se ha encontrado con una serie confusa de unidades: TRX, SUN, energia, ancho de banda, saldo congelado, recursos en staking. La documentacion dispersa estos conceptos a lo largo de docenas de paginas, y la mayoria de los tutoriales pasan por alto las distinciones que realmente importan cuando se escribe codigo de produccion o se calculan costos.

Este articulo es la guia de referencia que desearia que existiera cuando comence a trabajar con TRON. Cubriremos cada unidad en el ecosistema TRON, como se relacionan entre si, y proporcionaremos ejemplos concretos de conversion que puede usar de inmediato.

---

## TRX: la moneda base

TRX (Tronix) es la criptomoneda nativa de la blockchain TRON. Cumple el mismo rol que ETH cumple en Ethereum: es la unidad de cuenta, el medio para pagar comisiones de transaccion y el activo que se pone en staking para adquirir recursos de red.

A principios de 2026, TRX cotiza en el rango de $0.20 a $0.30 USD, aunque esto fluctua. Lo que no fluctua es su rol en el modelo de recursos: cada operacion en TRON se remonta en ultima instancia a TRX, ya sea que lo gaste directamente (quemando) o lo bloquee (staking).

Datos clave sobre TRX:

- El suministro total esta limitado y es deflacionario debido a mecanismos de quema.
- TRX puede enviarse, ponerse en staking o quemarse.
- La unidad minima divisible es 1 SUN (mas sobre esto a continuacion).
- Todas las comisiones en cadena se denominan en TRX o sus sub-unidades.

---

## SUN: la unidad mas pequena

SUN es para TRX lo que wei es para ETH o lo que satoshi es para BTC. Es la unidad indivisible mas pequena de la moneda TRON.

**1 TRX = 1,000,000 SUN**

Esto no es negociable, no es aproximado y no esta sujeto a condiciones de mercado. Es una conversion fija definida a nivel de protocolo.

### Por que SUN importa para los desarrolladores

Toda aplicacion seria de TRON trabaja en SUN internamente. La razon es simple: la aritmetica de punto flotante es el enemigo del software financiero. Cuando representa 1.5 TRX como `1500000` SUN, elimina los errores de redondeo por completo. La matematica entera es exacta.

Considere este ejemplo:

```typescript
// Incorrecto - punto flotante
const amount = 1.1 + 0.2; // 1.3000000000000003

// Correcto - enteros en SUN
const amountSun = 1_100_000 + 200_000; // 1_300_000 (exactamente 1.3 TRX)
```

En MERX, cada saldo, cada entrada del libro mayor, cada cotizacion de precio se almacena y transmite en SUN. La conversion a TRX solo ocurre en la capa de visualizacion, nunca en la logica de negocio.

### Conversiones comunes

| TRX | SUN |
|-----|-----|
| 0.000001 | 1 |
| 0.01 | 10,000 |
| 1 | 1,000,000 |
| 100 | 100,000,000 |
| 1,000 | 1,000,000,000 |
| 36,000 | 36,000,000,000 |

### Conversion en codigo

```typescript
// TRX a SUN
function trxToSun(trx: number): bigint {
  return BigInt(Math.round(trx * 1_000_000));
}

// SUN a TRX (solo para visualizacion)
function sunToTrx(sun: bigint): string {
  const trx = Number(sun) / 1_000_000;
  return trx.toFixed(6);
}
```

Note el uso de `BigInt` para valores en SUN. El tipo `number` estandar de JavaScript puede representar de forma segura enteros hasta 2^53, que es aproximadamente 9 cuatrillones de SUN o 9 mil millones de TRX. Para la mayoria de aplicaciones esto es suficiente, pero `BigInt` elimina cualquier riesgo.

---

## Energia: combustible para contratos inteligentes

La energia es el recurso consumido al ejecutar contratos inteligentes en TRON. Cada instruccion en la Maquina Virtual de TRON (TVM) cuesta una cantidad especifica de energia, similar a como funciona el gas en Ethereum.

### Que consume energia

- Llamar a cualquier funcion de contrato inteligente (incluyendo transferencias de tokens TRC-20 como USDT)
- Desplegar nuevos contratos inteligentes
- Cualquier operacion que involucre computacion en cadena

### Que NO consume energia

- Transferencias simples de TRX (estas usan solo ancho de banda)
- Creacion de cuentas
- Votar por super representantes
- Staking y retiro de staking

### Cuanta energia cuesta una transferencia de USDT

Una transferencia estandar de USDT (TRC-20) cuesta aproximadamente **65,000 de energia**. Esta es la operacion mas comun en TRON, representando la mayoria de todas las llamadas a contratos inteligentes en la red.

El costo exacto puede variar ligeramente dependiendo del estado del contrato; por ejemplo, la primera transferencia a una nueva direccion puede costar mas porque crea un nuevo espacio de almacenamiento. Pero 65,000 es el numero que debe usar para planificacion.

### Como obtener energia

Existen exactamente dos formas de obtener energia:

**1. Staking de TRX (Stake 2.0)**

Usted bloquea TRX en un contrato de staking y recibe energia en proporcion a su participacion relativa al total de staking en la red. La formula es:

```
su_energia = (su_trx_en_staking / total_trx_en_staking_red) * pool_total_energia
```

A principios de 2026, necesita aproximadamente **36,000 TRX** en staking para recibir alrededor de **65,000 de energia por dia**, suficiente para una transferencia de USDT diaria. La proporcion exacta fluctua a medida que cambia el staking total de la red.

Importante: la energia en staking se regenera en 24 horas. Si usa toda su energia de una vez, toma un dia completo recuperarla. La energia del staking no se puede acumular.

**2. Alquilar de un proveedor (o agregador como MERX)**

Un proveedor hace staking de su propio TRX y delega la energia resultante a su direccion por una tarifa. Este es el modelo de pago por uso. Usted paga una fraccion del costo en TRX sin bloquear capital.

Los precios tipicos de alquiler a principios de 2026 varian de 80 a 120 SUN por unidad de energia, dependiendo del proveedor, la duracion y el volumen.

### Unidades de energia

La energia es adimensional, es simplemente un numero. No tiene sub-unidad. Usted tiene 65,000 de energia o no la tiene. No esta denominada en TRX o SUN; es su propio recurso.

---

## Ancho de banda: asignacion de datos de transaccion

El ancho de banda es el recurso consumido por los datos brutos de cualquier transaccion en TRON. Cada transaccion, ya sea una simple transferencia de TRX o una llamada compleja a un contrato inteligente, usa ancho de banda proporcional a su tamano en bytes.

### Como se mide el ancho de banda

El ancho de banda se mide en bytes. Una transaccion tipica de transferencia de TRX es de aproximadamente 250-300 bytes. Una transferencia de USDT (llamada a contrato inteligente) es de aproximadamente 350-400 bytes.

### Ancho de banda gratuito

Cada cuenta de TRON recibe **600 puntos de ancho de banda gratuitos por dia**. Estos se regeneran en 24 horas, similar a la energia. Para transferencias simples de TRX (aproximadamente 270 bytes cada una), esto significa que obtiene aproximadamente dos transferencias gratuitas por dia.

### Que sucede cuando se agota

Cuando su ancho de banda se agota, la red quema TRX de su cuenta para cubrir el costo de ancho de banda. La tasa de quema fluctua pero es generalmente modesta: unos pocos TRX como maximo para una transaccion estandar. La formula es:

```
costo_ancho_banda_trx = bytes_transaccion * precio_ancho_banda_por_byte
```

Para la mayoria de usuarios que realizan transferencias ocasionales, los 600 de ancho de banda gratuitos son suficientes. Para aplicaciones de alto volumen, los costos de ancho de banda se convierten en una partida que vale la pena rastrear.

### Ancho de banda vs energia: la diferencia clave

Cada transaccion usa ancho de banda. Solo las transacciones de contratos inteligentes usan energia. Una transferencia simple de TRX usa ancho de banda pero cero energia. Una transferencia de USDT usa tanto ancho de banda como energia.

| Operacion | Ancho de banda | Energia |
|-----------|-----------|--------|
| Transferencia de TRX | ~270 bytes | 0 |
| Transferencia de USDT | ~350 bytes | ~65,000 |
| Despliegue de contrato | variable | variable |
| Voto | ~270 bytes | 0 |

---

## Como se relacionan todas las unidades

Aqui esta el panorama completo de como se conectan las unidades de TRON:

```
TRX (moneda)
  |
  |-- 1 TRX = 1,000,000 SUN (sub-unidad)
  |
  |-- Staking TRX --> Energia (combustible de contratos inteligentes)
  |                  |
  |                  |-- Medida en unidades (adimensional)
  |                  |-- ~65,000 necesarias por transferencia de USDT
  |                  |-- Se regenera en 24 horas
  |
  |-- Staking TRX --> Ancho de banda (asignacion de datos)
  |                  |
  |                  |-- Medido en bytes
  |                  |-- 600 gratuitos diarios por cuenta
  |                  |-- Usado por TODAS las transacciones
  |
  |-- Quemar TRX --> Pagar ancho de banda/energia directamente
```

La idea fundamental: hacer staking de TRX le da recursos renovables (se regeneran diariamente), mientras que quemar TRX es un pago unico. Para usuarios de alto volumen, el staking o el alquiler es dramaticamente mas barato que quemar.

---

## Ejemplo practico de costos

Recorramos un escenario concreto: necesita enviar 100 transferencias de USDT por dia.

### Opcion 1: quemar todo

Cada transferencia de USDT sin energia cuesta aproximadamente 27-30 TRX en comisiones quemadas (cubriendo tanto los costos de energia como de ancho de banda). Para 100 transferencias:

```
100 transferencias x 28 TRX = 2,800 TRX/dia
A $0.25/TRX = $700/dia = $21,000/mes
```

### Opcion 2: staking de TRX para energia

Necesita aproximadamente 65,000 de energia por transferencia, asi que 6,500,000 de energia por dia. A las proporciones actuales de staking, esto requiere aproximadamente 3,600,000 TRX en staking (~$900,000 de capital bloqueado).

Obviamente impractico para la mayoria de usuarios.

### Opcion 3: alquilar energia via MERX

A una tasa de alquiler de aproximadamente 90 SUN por unidad de energia:

```
65,000 energia x 90 SUN = 5,850,000 SUN = 5.85 TRX por transferencia
100 transferencias x 5.85 TRX = 585 TRX/dia
A $0.25/TRX = $146.25/dia = $4,388/mes
```

El ahorro frente a quemar: aproximadamente 80%.

Esta es la razon por la que existe el mercado de alquiler de energia. La brecha entre los costos de quema y los costos de alquiler es enorme, y escala linealmente con el volumen.

---

## Unidades en la API de MERX

Al trabajar con la API de MERX, todos los valores siguen convenciones consistentes:

```json
{
  "price_per_unit": 88000000,
  "energy_amount": 65000,
  "total_cost_sun": 5720000000,
  "total_cost_trx": 5720.0
}
```

- `price_per_unit`: precio en SUN por unidad de energia
- `energy_amount`: unidades de energia (entero adimensional)
- `total_cost_sun`: costo total en SUN (entero)
- `total_cost_trx`: costo total en TRX (conveniencia de visualizacion)

El SDK gestiona las conversiones:

```typescript
import { MerxClient } from 'merx-sdk';

const client = new MerxClient({ apiKey: 'your-key' });
const prices = await client.getPrices({ energy: 65000 });

console.log(prices.bestPrice.perUnit);    // SUN por unidad de energia
console.log(prices.bestPrice.totalSun);   // Total en SUN
console.log(prices.bestPrice.totalTrx);   // Total en TRX
```

Documentacion completa del SDK: [https://merx.exchange/docs](https://merx.exchange/docs)

---

## Errores comunes

**Error 1: mezclar TRX y SUN en calculos.** Siempre elija uno y mantengalo. Internamente, use SUN. Convierta a TRX solo para visualizacion.

**Error 2: asumir que la energia tiene precio en TRX.** Los precios de alquiler de energia se cotizan tipicamente en SUN por unidad de energia. Un precio de "88" usualmente significa 88 SUN, no 88 TRX. Confundir esto por un factor de 1,000,000 arruinara su dia.

**Error 3: olvidar que el ancho de banda existe.** Incluso con cobertura completa de energia, cada transaccion aun consume ancho de banda. Para aplicaciones de alto volumen, los costos de ancho de banda se acumulan.

**Error 4: tratar la energia como acumulable.** La energia en staking se regenera en 24 horas. No puede guardar energia de dias tranquilos para usar en dias ocupados. La energia alquilada, sin embargo, esta disponible inmediatamente durante la duracion del alquiler.

**Error 5: usar punto flotante para valores en SUN.** Use enteros o BigInt. Siempre. Sin excepciones.

---

## Resumen

El modelo de recursos de TRON es mas matizado que el de la mayoria de las cadenas, pero no es complicado una vez que entiende los cuatro conceptos fundamentales:

- **TRX** es dinero.
- **SUN** es la unidad mas pequena de dinero (1 TRX = 1,000,000 SUN).
- **Energia** es combustible para contratos inteligentes, obtenida mediante staking de TRX o alquiler.
- **Ancho de banda** es asignacion de datos de transaccion, parcialmente gratuito, obtenido mediante staking o quema.

Para desarrolladores que construyen en TRON, la conclusion mas importante es trabajar siempre en SUN internamente y tratar la energia como un costo a optimizar. La diferencia entre quemar y alquilar energia puede ser de 5x a 10x en costo, que es exactamente la razon por la que existen plataformas como MERX.

Explore los precios actuales de energia y comience a ahorrar en sus operaciones de TRON en [https://merx.exchange](https://merx.exchange).

---

*Este articulo es parte de la serie de conocimiento de MERX sobre infraestructura TRON. MERX es el primer exchange de recursos blockchain, agregando todos los principales proveedores de energia en una sola API. Mas informacion en [https://merx.exchange](https://merx.exchange).*

## Try It Now with AI

Add MERX to Claude Desktop or any MCP-compatible client -- zero install, no API key needed for read-only tools:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Ask your AI agent: "What is the cheapest TRON energy right now?" and get live prices from all connected providers.

Full MCP documentation: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)
