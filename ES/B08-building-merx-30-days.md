# De la Idea a Producción: Construyendo MERX en 30 Días

MERX pasó de concepto a sistema de producción en vivo en 30 días. No una página de destino. No un prototipo. Un intercambio de recursos blockchain completamente operativo con siete integraciones de proveedores, agregación de precios en tiempo real, ejecución de órdenes en cadena, contabilidad de doble entrada, documentación completa, SDKs en dos idiomas y un servidor MCP con 55 herramientas para integración de agentes de IA.

Este artículo es la historia técnica de cómo sucedió - las decisiones de arquitectura, los problemas que resolvimos, los atajos que deliberadamente no tomamos, y las lecciones de construir una plataforma financiera a velocidad sin comprometer las cosas que importan.

---

## Día 0: La Declaración del Problema

El mercado de energía TRON está fragmentado. Siete o más proveedores ofrecen servicios de delegación de energía, cada uno con su propia API, precios y características de confiabilidad. Si quieres el mejor precio, necesitas integrarte con todos ellos. Si quieres redundancia, necesitas construir lógica de enrutamiento. Si quieres transparencia, necesitas construir monitoreo.

Cada negocio que envía USDT en TRON enfrenta este costo de integración. La solución es una capa de agregación - una única API que maneje enrutamiento multi-proveedor, selección del mejor precio y conmutación automática por error.

Nadie lo había construido todavía. Decidimos hacerlo.

---

## Semana 1: Fundación

### Arquitectura Primero

Antes de escribir una sola línea de código, pasamos dos días en arquitectura. El resultado fue un documento de arquitectura de 40 secciones cubriendo todo desde el esquema de base de datos hasta formatos de errores de API hasta códigos hexadecimales de colores. Este documento se convirtió en la única fuente de verdad para cada decisión de implementación.

Decisiones de arquitectura clave tomadas en esos dos días:

**Decisión 1: Microservicios desde el primer día.**

No porque los microservicios sean tendencia, sino porque los sistemas financieros necesitan aislamiento. El firmante del tesoro no debe ser accesible desde el servicio de API. El monitor de precios no debe tener acceso de escritura a los saldos de usuario. Los contenedores Docker proporcionan este aislamiento naturalmente.

```
services/
  api/              HTTP/WebSocket API
  price-monitor/    Polling de precios de proveedores
  order-executor/   Enrutamiento y ejecución de órdenes
  ledger/           Contabilidad de doble entrada
  deposit-monitor/  Detección de pagos entrantes
  treasury-signer/  Firma de transacciones (aislado)
```

**Decisión 2: PostgreSQL + Redis, sin bases de datos exóticas.**

PostgreSQL para todo lo que necesita garantías ACID (saldos, órdenes, entradas de libro mayor). Redis para todo lo que necesita velocidad (caché de precios, pub/sub, limitación de velocidad). Ambas están probadas en batalla, bien documentadas y operacionalmente simples.

**Decisión 3: Todos los montos en SUN.**

Cada valor financiero almacenado como un entero en SUN (1 TRX = 1.000.000 SUN). Sin punto flotante en ningún lugar de la ruta financiera. Esto eliminó una categoría completa de errores antes de escribir nuestra primera función.

**Decisión 4: Node.js + TypeScript para servicios, Go para el motor de coincidencia.**

TypeScript para la mayor parte del sistema - desarrollo rápido, tipado fuerte, excelente I/O asincrónico para API y cargas de trabajo de monitoreo. Go reservado para el motor de coincidencia donde el rendimiento bruto importa.

### Esquema de Base de Datos

Las migraciones de base de datos se escribieron el día 3. Cada tabla fue diseñada teniendo en mente la integridad financiera:

```sql
-- Principio central: cada mutación de saldo crea una entrada de libro mayor
CREATE TABLE ledger (
  id BIGSERIAL PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(id),
  type VARCHAR(50) NOT NULL,
  amount_sun BIGINT NOT NULL,
  direction VARCHAR(6) NOT NULL CHECK (direction IN ('DEBIT', 'CREDIT')),
  reference_type VARCHAR(50),
  reference_id UUID,
  balance_before BIGINT NOT NULL,
  balance_after BIGINT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Sin triggers UPDATE o DELETE - libro mayor es solo anexo
```

Un archivo por migración, nombrado `YYYYMMDD_descripcion.sql`. Al final de los 30 días, había 14 archivos de migración, cada uno aditivo, ninguno destructivo.

### Interfaz de Proveedor

La interfaz `IEnergyProvider` fue definida el día 4. Este era el contrato que cada adaptador de proveedor implementaría:

```typescript
interface IEnergyProvider {
  name: string;
  getPrices(): Promise<ProviderPriceResponse>;
  createOrder(params: OrderParams): Promise<OrderResult>;
  getOrderStatus(orderId: string): Promise<OrderStatus>;
  healthCheck(): Promise<boolean>;
}
```

Esta interfaz nunca cambió. Siete proveedores fueron integrados contra ella en las siguientes semanas, cada uno en su propio archivo, ninguno requiriendo cambios al sistema central.

---

## Semana 2: Servicios Centrales

### Monitor de Precios

El monitor de precios fue el primer servicio en estar en vivo. Realiza polling de cada proveedor cada 30 segundos, normaliza precios, publica a Redis y almacena historial en PostgreSQL. La implementación es aproximadamente 180 líneas de TypeScript en tres archivos.

La parte más difícil no fue la lógica de polling - fue la normalización. Cada proveedor devuelve precios en formatos ligeramente diferentes:

- Proveedor A: SUN por unidad de energía
- Proveedor B: TRX total para una cantidad de energía fija
- Proveedor C: SUN por unidad de energía, pero con un orden mínimo diferente
- Proveedor D: precios escalonados basados en volumen

Cada adaptador traduce el formato de su proveedor a la respuesta estándar `ProviderPriceResponse`. El monitor de precios no se preocupa por los peculiaridades del proveedor; solo ve datos normalizados.

### Ejecutor de Órdenes

El ejecutor de órdenes es el servicio más complejo. Lee precios de Redis, determina enrutamiento óptimo, envía órdenes a proveedores, monitorea confirmación en cadena y publica eventos de liquidación.

La cadena de conmutación por error fue el elemento de diseño crítico. Si el Proveedor A falla, intenta con el Proveedor B. Si B falla, intenta con C. La llamada de API del comprador tiene éxito mientras cualquier proveedor sea operativo.

```
Orden recibida -> Leer precios -> Seleccionar más barato
  -> Ejecutar en Proveedor A
    -> ¿Éxito? Verificar en cadena -> Liquidar
    -> ¿Error? Intentar Proveedor B
      -> ¿Éxito? Verificar en cadena -> Liquidar
      -> ¿Error? Intentar Proveedor C
        -> ... y así sucesivamente
```

### Servicio de Libro Mayor

El servicio de libro mayor refuerza la restricción de doble entrada. Cada mutación de saldo crea entradas pareadas. El servicio ejecuta una verificación de reconciliación cada hora:

```sql
SELECT SUM(CASE direction
  WHEN 'DEBIT' THEN amount_sun
  WHEN 'CREDIT' THEN -amount_sun
END) FROM ledger;
-- Debe ser 0. Si no: alertar inmediatamente.
```

En 30 días de desarrollo y pruebas, esta verificación nunca se activó. La restricción nunca fue violada porque la arquitectura hizo las violaciones estructuralmente imposibles, no solo improbables.

---

## Semana 3: API, Frontend y Verificación En Cadena

### Diseño de API

La API sigue convenciones REST con versionado estricto (`/api/v1/...`). Cada endpoint fue diseñado antes de la implementación:

```
GET    /api/v1/prices          Precios actuales de todos los proveedores
GET    /api/v1/prices/best     Mejor precio actual
POST   /api/v1/orders          Crear una nueva orden
GET    /api/v1/orders/:id      Obtener estado de orden
GET    /api/v1/balance         Obtener saldo de cuenta
POST   /api/v1/deposit         Obtener dirección de depósito
POST   /api/v1/withdraw        Solicitar retiro
```

Las respuestas de error usan un formato consistente:

```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "El saldo de la cuenta (5.2 TRX) es insuficiente para esta orden (8.1 TRX)",
    "details": {
      "balance": 5200000,
      "required": 8100000
    }
  }
}
```

Ningún endpoint fue publicado sin validación Zod en todas las entradas.

### Frontend

El frontend es una aplicación Next.js con un sistema de diseño estricto: solo tema oscuro, sin esquinas redondeadas superiores a 2px, sin gradientes, sin sombras, Cormorant Garamond para encabezados, IBM Plex Mono para todo lo demás. La identidad visual fue definida en el documento de arquitectura e implementada fielmente.

### Verificación En Cadena

Cada orden es verificada en la blockchain TRON. El servicio de verificación observa transacciones de delegación y confirma que la energía llegó a la dirección objetivo. Esta fue la integración más desafiante porque los tiempos de confirmación de blockchain son variables y los formatos de transacción del proveedor difieren.

Ocho transacciones de mainnet fueron verificadas durante la fase de pruebas, confirmando que el flujo de extremo a extremo - desde la llamada de API hasta la delegación en cadena - funcionó correctamente con TRX real y proveedores reales.

---

## Semana 4: SDKs, Servidor MCP y Documentación

### SDK de JavaScript

El SDK de JavaScript fue construido para entornos Node.js y navegador:

```typescript
import { MerxClient } from 'merx-sdk';

const client = new MerxClient({ apiKey: 'your-key' });
const prices = await client.getPrices({ energy: 65000 });
const order = await client.createOrder({
  energy: 65000,
  targetAddress: 'TAddress...',
  duration: '1h'
});
```

Fuente: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)

### SDK de Python

El SDK de Python refleja la superficie de API del SDK de JavaScript:

```python
from merx import MerxClient

client = MerxClient(api_key='your-key')
prices = client.get_prices(energy=65000)
order = client.create_order(
    energy=65000,
    target_address='TAddress...',
    duration='1h'
)
```

Fuente: [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)

### Servidor MCP: 55 Herramientas

El servidor MCP (Protocolo de Contexto de Modelo) fue quizás el componente más visionario. Expone funcionalidad de MERX como herramientas que los agentes de IA pueden usar directamente.

El servidor MCP creció de 7 herramientas en su versión inicial a 55 herramientas al final de los 30 días:

```
Gestión de cuenta:        create_account, login, get_balance, get_deposit_info
Datos de precios:         get_prices, get_best_price, compare_providers, analyze_prices
Gestión de órdenes:       create_order, get_order, list_orders, create_standing_order
Monitoreo de recursos:    check_address_resources, estimate_transaction_cost
Utilidades TRON:          validate_address, convert_address, get_trx_balance
Operaciones en cadena:    transfer_trx, transfer_trc20, approve_trc20
Análisis:                 calculate_savings, get_price_history, suggest_duration
... y 30 más
```

Fuente: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)

### Documentación

La documentación fue reconstruida de 5 páginas a 36 páginas, cubriendo la referencia completa de API, guías de SDK, conceptos de TRON y tutoriales de integración. La documentación se encuentra en [https://merx.exchange/docs](https://merx.exchange/docs).

Además, 4 páginas de guías SEO y 7 páginas de comparación de proveedores fueron publicadas, llevando el mapa del sitio a 53 URLs.

---

## Lo Que No Comprometimos

La velocidad crea presión para cortar esquinas. Aquí hay esquinas que explícitamente no cortamos:

### Sin Punto Flotante para Dinero

Usar enteros (SUN) para todos los valores financieros añadió complejidad en formateo de pantalla pero eliminó errores de redondeo por completo. Cada caso de prueba coincidió con valores esperados exactamente.

### Sin Concatenación de Strings para SQL

Cada consulta de base de datos usa sentencias parametrizadas. Esta fue una regla innegociable desde el primer día. La inyección de SQL es un problema resuelto, y lo mantuvimos resuelto.

### Sin Secretos Codificados

Variables de entorno desde el primer día. Secretos de Docker para la clave del tesoro. `.gitignore` configurado antes del primer commit.

### Sin Servicios Compartiendo Estado Directamente

Los servicios se comunican a través de Redis pub/sub o llamadas de API REST. Sin importaciones directas entre servicios. Esto hizo posible la implementación independiente y previno fallos en cascada.

### Sin Mutaciones de Libro Mayor

Libro mayor solo de anexo desde la primera migración. Sin UPDATE o DELETE en tablas de libro mayor. Las correcciones crean nuevas entradas, no modificaciones.

---

## Lo Que Aprendimos

### Lección 1: Los Documentos de Arquitectura Se Pagan Solos

Los dos días dedicados a la arquitectura ahorraron semanas de refabricación. Cada pregunta de desarrollador fue respondida por el documento. Cada desacuerdo de diseño fue resuelto haciendo referencia a la especificación. Las 40 secciones no eran gastos generales burocráticos; eran una función forzada para pensar en problemas antes de que se convirtieran en errores.

### Lección 2: Las APIs de Proveedores Son Poco Confiables

De los siete proveedores integrados, al menos dos experimentaron tiempo de inactividad durante el período de construcción de 30 días. La cadena de conmutación por error no era una delicadeza teórica - fue utilizada dentro de la primera semana de pruebas.

### Lección 3: El Patrón Adaptador Vale la Pena el Código Repetitivo

Escribir siete adaptadores que todos implementen la misma interfaz se sintió repetitivo. Pero cuando el Proveedor C cambió su formato de respuesta de API el día 22, actualizamos un archivo y nada más cambió. Los 10 minutos dedicados a actualizar el adaptador versus los días que habríamos dedicado a actualizar cada sitio de llamada hicieron que el valor del patrón fuera obvio.

### Lección 4: MCP Es el Futuro de la Integración de Servicios

El servidor MCP fue inicialmente un experimento. Pero ver agentes de IA usar herramientas de MERX para gestionar autónomamente la adquisición de energía fue una revelación. Así es como los servicios serán consumidos en el futuro - no a través de desarrolladores humanos escribiendo código de integración, sino a través de agentes de IA llamando APIs de herramientas directamente.

### Lección 5: Límite de 200 Líneas por Archivo Es una Característica

Aplicamos un límite estricto de 200 líneas por archivo en todo el proyecto. Esto forzó descomposición constante. Las funciones se mantuvieron pequeñas. Las responsabilidades se mantuvieron claras. Cuando un archivo se acercaba a 200 líneas, era hora de dividir, y la división siempre mejoraba la claridad.

---

## Por los Números

```
Documento de arquitectura:    40 secciones
Servicios:                    9 contenedores Docker
Integraciones de proveedores: 7
Migraciones de base de datos: 14
Endpoints de API:             20+
Herramientas MCP:             55 (desde 7 iniciales)
Idiomas del SDK:              2 (JavaScript, Python)
Páginas de documentación:     36 (desde 5 iniciales)
URLs del mapa del sitio:      53
Transacciones en mainnet:     8 verificadas
Tasa de comisión:             0%
Días a producción:            30
```

---

## Lo Que Viene Después

La plataforma está en vivo en [https://merx.exchange](https://merx.exchange). El enfoque inmediato es pruebas, optimización e incorporación de los primeros usuarios de producción. La fundación es sólida - la arquitectura soporta escalado horizontal, nuevos proveedores pueden ser añadidos en horas, y el modelo de comisión cero elimina fricción de adopción.

El mercado de agregación de energía en TRON está esperando una plataforma que lo haga simple. MERX es esa plataforma.

---

*MERX es el primer intercambio de recursos blockchain. Explora la plataforma en [https://merx.exchange](https://merx.exchange). Documentación en [https://merx.exchange/docs](https://merx.exchange/docs). SDKs de código abierto y servidor MCP en GitHub.*


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