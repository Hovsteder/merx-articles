# Widget de Precios MERX: Incrustar Precios de Energía TRON en Vivo en Cualquier Sitio Web

Los precios de energía TRON cambian constantemente. Los proveedores ajustan las tarifas según la demanda, las condiciones de la red y la capacidad disponible. Si tu sitio web sirve a usuarios de TRON —ya sea un monedero, una dApp, un explorador de blockchain o una guía de recursos— mostrar precios de energía en vivo añade un valor inmediato y práctico para tus visitantes.

MERX proporciona un widget de precios incrustable que muestra los precios de energía en tiempo real de todos los proveedores principales, ordenados por costo. Requiere dos líneas de HTML, se actualiza automáticamente cada 60 segundos y hereda un tema oscuro profesional que se adapta a la mayoría de sitios web relacionados con blockchain sin modificación.

Este artículo cubre cómo incrustar el widget, cómo funciona internamente y cómo personalizarlo para tu caso de uso.

## Dos Líneas de HTML

La integración más simple posible. Añade estas dos líneas en cualquier lugar de tu HTML:

```html
<div id="merx-prices"></div>
<script src="https://merx.exchange/widget/prices.js"></script>
```

Eso es todo. El script se inicializa automáticamente, obtiene los precios actuales de la API pública de MERX y renderiza una tabla estilizada dentro del `div`. No se requiere clave de API. Sin pasos de construcción. Sin dependencias de frameworks.

El widget funciona en cualquier página HTML —sitios estáticos, WordPress, Webflow, Squarespace (mediante bloques de código personalizado) o cualquier framework que renderice en el navegador. Se carga de forma asincrónica y no bloquea el renderizado de la página.

## Qué Muestra el Widget

El widget renderiza una tabla compacta que muestra todos los proveedores activos de energía TRON con sus precios actuales. Cada fila incluye:

- **Nombre del proveedor** —el proveedor de energía (Sohu, CatFee, NETTs, TronSave, Feee, iTRX, PowerSun).
- **Precio por unidad** —precio actual de energía en SUN. Este es el costo por unidad de energía para una delegación estándar.
- **Orden mín/máx** —la cantidad mínima y máxima de energía que el proveedor actualmente acepta.
- **Duración** —duraciones de alquiler disponibles (1 hora, 3 horas, 1 día, etc.).
- **Estado** —si el proveedor está actualmente en línea y aceptando órdenes.

Los proveedores se ordenan por precio, el más barato primero. La ordenación se actualiza con cada actualización, así que si un proveedor baja su precio, se mueve hacia arriba automáticamente.

### Apariencia Predeterminada

El widget utiliza un tema oscuro de forma predeterminada:

- Fondo: `#0a0a0a` (casi negro)
- Texto: `#e0e0e0` (gris claro)
- Bordes de tabla: `#1a1a1a` (bordes oscuros sutiles)
- Color de acento: `#00d4aa` (verde MERX, utilizado para resaltar el precio más barato)
- Fuente: IBM Plex Mono (cargada desde Google Fonts si no está disponible)

El estilo visual es deliberadamente minimalista. Sin esquinas redondeadas, sin gradientes, sin sombras. Se adapta a la estética de la mayoría de interfaces de criptomonedas y blockchain.

## Cómo Funciona Internamente

Entender el funcionamiento interno del widget ayuda con la personalización y la solución de problemas.

### Fuente de Datos

El widget obtiene datos del endpoint de precios público de MERX:

```
GET https://merx.exchange/api/v1/prices
```

Este endpoint es público —no se requiere autenticación, no se necesita clave de API. Devuelve precios actuales de todos los proveedores conectados:

```json
{
  "prices": [
    {
      "provider": "sohu",
      "energy_price_sun": 22,
      "min_energy": 32000,
      "max_energy": 10000000,
      "durations": [1, 3, 24],
      "available": true,
      "updated_at": "2026-03-30T10:30:00Z"
    },
    {
      "provider": "catfee",
      "energy_price_sun": 25,
      "min_energy": 10000,
      "max_energy": 5000000,
      "durations": [1, 24],
      "available": true,
      "updated_at": "2026-03-30T10:30:15Z"
    }
  ],
  "timestamp": "2026-03-30T10:30:20Z"
}
```

### Ciclo de Actualización

El widget consulta el endpoint de precios cada 60 segundos. Cada actualización es silenciosa —sin spinner de carga, sin parpadeo de contenido vacío. La tabla se actualiza en su lugar. Si una obtención falla (problema de red, tiempo de espera del servidor), el widget retiene los últimos datos obtenidos exitosamente e intenta de nuevo en el próximo ciclo.

Una pequeña marca de tiempo en el pie de página del widget muestra cuándo se actualizaron los datos por última vez, para que los usuarios puedan saber de un vistazo si los precios son actuales.

### Carga de Script

El script `prices.js` se sirve desde la CDN de MERX con almacenamiento en caché agresivo (1 hora) y compresión gzip. El tiempo de carga típico es inferior a 50ms en conexiones de banda ancha. El script es aproximadamente 8 KB minificado y comprimido con gzip.

Al cargar, el script:

1. Encuentra el elemento `#merx-prices` (o un destino personalizado si está configurado).
2. Inyecta estilos CSS con alcance (prefijados para evitar conflictos con los estilos de tu página).
3. Realiza la primera llamada API.
4. Renderiza la tabla.
5. Configura el intervalo de actualización de 60 segundos.

Todos los estilos tienen alcance bajo `.merx-widget` para evitar conflictos CSS con tus estilos existentes.

## Opciones de Personalización

El widget acepta configuración a través de atributos de datos en el `div` contenedor u a través de un objeto de configuración JavaScript.

### Configuración mediante Atributos de Datos

```html
<div
  id="merx-prices"
  data-refresh="30"
  data-providers="sohu,catfee,netts"
  data-duration="1"
  data-theme="light"
  data-max-rows="5"
></div>
<script src="https://merx.exchange/widget/prices.js"></script>
```

Atributos de datos disponibles:

| Atributo | Predeterminado | Descripción |
|----------|---------|-------------|
| `data-refresh` | `60` | Intervalo de actualización en segundos (mínimo 15) |
| `data-providers` | todos | Lista de proveedores a mostrar separada por comas |
| `data-duration` | todos | Filtrar a duración específica (1, 3, 24 horas) |
| `data-theme` | `dark` | `dark` u `light` |
| `data-max-rows` | todos | Número máximo de proveedores a mostrar |
| `data-show-header` | `true` | Mostrar u ocultar el encabezado "MERX Energy Prices" |
| `data-show-footer` | `true` | Mostrar u ocultar el pie de página con marca de tiempo |
| `data-compact` | `false` | Modo compacto —menos columnas, texto más pequeño |

### Configuración mediante JavaScript

Para mayor control, inicializa el widget de forma programática:

```html
<div id="energy-prices"></div>
<script src="https://merx.exchange/widget/prices.js"></script>
<script>
  MerxWidget.init({
    container: '#energy-prices',
    refresh: 30,
    providers: ['sohu', 'catfee', 'netts', 'tronsave'],
    duration: 1,
    theme: 'dark',
    maxRows: 5,
    showHeader: true,
    showFooter: true,
    compact: false,
    onUpdate: function (prices) {
      console.log('Precios actualizados:', prices.length, 'proveedores');
    },
    onError: function (error) {
      console.error('Error del widget:', error.message);
    },
  });
</script>
```

Los callbacks `onUpdate` y `onError` te permiten reaccionar a eventos del widget en tu propio código. El callback `onUpdate` recibe el array de precios analizado en cada actualización exitosa.

### Tema Claro

Para sitios web con fondo claro:

```html
<div id="merx-prices" data-theme="light"></div>
<script src="https://merx.exchange/widget/prices.js"></script>
```

Colores del tema claro:

- Fondo: `#ffffff`
- Texto: `#1a1a1a`
- Bordes de tabla: `#e0e0e0`
- Acento: `#00a88a` (verde más oscuro para fondos claros)

### Estilo Personalizado

Las clases CSS del widget son estables y están documentadas. Sobrescríbelas en tu propia hoja de estilos:

```css
/* Hacer que el widget tenga ancho completo */
.merx-widget {
  width: 100%;
  max-width: none;
}

/* Fuente personalizada */
.merx-widget table {
  font-family: 'JetBrains Mono', monospace;
  font-size: 13px;
}

/* Color de acento personalizado */
.merx-widget .merx-best-price {
  color: #ff6b00;
}

/* Ocultar columnas específicas */
.merx-widget .merx-col-duration {
  display: none;
}
```

El `max-width` predeterminado del widget es 640px. Configurarlo a `100%` permite que llene su contenedor.

## Ejemplo de Página HTML Completa

Aquí hay una página HTML completa y autónoma con el widget incrustado. Cópiala, abrela en un navegador y tendrás un rastreador de precios de energía TRON en vivo:

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Precios de Energía TRON - En Vivo</title>
  <style>
    body {
      background: #0a0a0a;
      color: #e0e0e0;
      font-family: 'IBM Plex Mono', monospace;
      display: flex;
      justify-content: center;
      padding: 40px 20px;
      margin: 0;
    }
    .container {
      max-width: 720px;
      width: 100%;
    }
    h1 {
      font-family: 'Cormorant Garamond', serif;
      font-size: 28px;
      font-weight: 400;
      margin-bottom: 8px;
    }
    p {
      color: #888;
      font-size: 14px;
      margin-bottom: 32px;
    }
  </style>
  <link
    href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:wght@400;600&family=IBM+Plex+Mono:wght@400;500&display=swap"
    rel="stylesheet"
  />
</head>
<body>
  <div class="container">
    <h1>Precios de Energía TRON</h1>
    <p>Precios en vivo de todos los proveedores principales. Se actualiza cada 60 segundos.</p>

    <div id="merx-prices" data-refresh="60" data-compact="false"></div>
    <script src="https://merx.exchange/widget/prices.js"></script>
  </div>
</body>
</html>
```

## Construir tu Propio Widget con la API Pública

Si el widget precompilado no se adapta a tus necesidades, puedes construir el tuyo usando el endpoint de precios público directamente. Esto te da control completo sobre la interfaz de usuario.

### Ejemplo de JavaScript Vanilla

```javascript
async function fetchMerxPrices() {
  const response = await fetch('https://merx.exchange/api/v1/prices');
  const data = await response.json();
  return data.prices
    .filter((p) => p.available)
    .sort((a, b) => a.energy_price_sun - b.energy_price_sun);
}

function renderPriceTable(prices) {
  const table = document.getElementById('custom-price-table');

  const rows = prices
    .map(
      (p) =>
        `<tr>
      <td>${p.provider}</td>
      <td>${p.energy_price_sun} SUN</td>
      <td>${(p.min_energy / 1000).toFixed(0)}K</td>
      <td>${p.available ? 'En línea' : 'Sin conexión'}</td>
    </tr>`
    )
    .join('');

  table.innerHTML = `
    <thead>
      <tr>
        <th>Proveedor</th>
        <th>Precio</th>
        <th>Orden Mín</th>
        <th>Estado</th>
      </tr>
    </thead>
    <tbody>${rows}</tbody>
  `;
}

// Carga inicial y actualización automática
async function refreshPrices() {
  try {
    const prices = await fetchMerxPrices();
    renderPriceTable(prices);
  } catch (err) {
    console.error('Error al obtener precios:', err);
  }
}

refreshPrices();
setInterval(refreshPrices, 60000);
```

### Ejemplo de Componente React

```jsx
import { useState, useEffect } from 'react';

function MerxPrices({ refreshInterval = 60000 }) {
  const [prices, setPrices] = useState([]);
  const [lastUpdated, setLastUpdated] = useState(null);

  useEffect(() => {
    async function fetchPrices() {
      try {
        const res = await fetch('https://merx.exchange/api/v1/prices');
        const data = await res.json();
        const sorted = data.prices
          .filter((p) => p.available)
          .sort((a, b) => a.energy_price_sun - b.energy_price_sun);
        setPrices(sorted);
        setLastUpdated(new Date());
      } catch (err) {
        console.error('Error en obtención de precios:', err);
      }
    }

    fetchPrices();
    const interval = setInterval(fetchPrices, refreshInterval);
    return () => clearInterval(interval);
  }, [refreshInterval]);

  return (
    <div className="merx-prices">
      <table>
        <thead>
          <tr>
            <th>Proveedor</th>
            <th>Precio (SUN)</th>
            <th>Mín</th>
            <th>Máx</th>
          </tr>
        </thead>
        <tbody>
          {prices.map((p) => (
            <tr key={p.provider}>
              <td>{p.provider}</td>
              <td>{p.energy_price_sun}</td>
              <td>{(p.min_energy / 1000).toFixed(0)}K</td>
              <td>{(p.max_energy / 1000000).toFixed(1)}M</td>
            </tr>
          ))}
        </tbody>
      </table>
      {lastUpdated && (
        <small>Actualizado: {lastUpdated.toLocaleTimeString()}</small>
      )}
    </div>
  );
}

export default MerxPrices;
```

## Límites de Velocidad para el Endpoint Público

El endpoint `GET /api/v1/prices` es público y no requiere autenticación, pero tiene un límite de velocidad de 300 solicitudes por minuto por dirección IP. Para un widget que se actualiza cada 60 segundos, estás bien dentro de este límite.

Si construyes una solución personalizada que consulta más agresivamente —por ejemplo, un backend que agrega precios cada 5 segundos— considera almacenar en caché los resultados y servirlos a tu frontend desde tu propio servidor. Esto mantiene bajo tu uso de la API de MERX y mejora los tiempos de respuesta para tus usuarios.

```javascript
// Ejemplo de almacenamiento en caché del lado del servidor (Node.js/Express)
let cachedPrices = null;
let cacheTimestamp = 0;

app.get('/api/energy-prices', async (req, res) => {
  const now = Date.now();

  if (!cachedPrices || now - cacheTimestamp > 15000) {
    const response = await fetch('https://merx.exchange/api/v1/prices');
    cachedPrices = await response.json();
    cacheTimestamp = now;
  }

  res.json(cachedPrices);
});
```

Esto almacena en caché las respuestas de MERX durante 15 segundos en tu servidor, permitiendo que tu frontend consulte tan frecuentemente como quiera sin aumentar la carga en la API de MERX.

## Consideraciones de SEO

El widget se renderiza del lado del cliente mediante JavaScript, lo que significa que los motores de búsqueda que no ejecutan JavaScript no indexarán los datos de precios. Si SEO es importante para tu página de precios, considera renderizar del lado del servidor los datos de precios iniciales e hidratar luego con el widget para actualizaciones en vivo.

Alternativamente, estructura la página con contenido estático sobre precios de energía TRON y usa el widget como un elemento complementario en vivo. El texto circundante proporciona valor de SEO mientras que el widget proporciona utilidad en tiempo real.

## Preguntas Frecuentes

**¿El widget ralentiza mi página?**
No. El script es aproximadamente 8 KB comprimido con gzip y se carga de forma asincrónica. No bloquea el renderizado. La llamada API inicial añade una solicitud de red, pero los precios típicamente responden dentro de 100ms.

**¿Puedo mostrar solo proveedores específicos?**
Sí. Usa `data-providers="sohu,catfee"` para filtrar la visualización a proveedores específicos.

**¿Qué sucede si la API de MERX está caída?**
El widget muestra los últimos datos obtenidos con éxito. Si nunca ha cargado exitosamente (carga de primera página mientras la API está caída), muestra un mensaje indicando que los precios están temporalmente no disponibles.

**¿Es el widget responsive para dispositivos móviles?**
Sí. En modo compacto (`data-compact="true"`), funciona bien en pantallas tan estrechas como 320px. El modo predeterminado requiere un ancho mínimo de aproximadamente 500px.

**¿Puedo usar el widget comercialmente?**
Sí. El widget y la API de precios público subyacente son gratuitos para usar. La atribución es apreciada pero no obligatoria.

## Conclusión

Añadir precios de energía TRON en vivo a tu sitio web toma dos líneas de HTML y cero trabajo de backend. El widget de precios de MERX maneja la obtención de datos, ordenación, estilo y actualización automática desde el inicio. Para implementaciones personalizadas, la API de precios público está disponible sin autenticación.

Ya sea que ejecutes un monedero TRON, un blog de blockchain o una herramienta de gestión de energía, mostrar precios de proveedores en tiempo real proporciona a tus usuarios información práctica que no pueden encontrar fácilmente en otro lugar.

- Plataforma MERX: [merx.exchange](https://merx.exchange)
- Documentación de API: [merx.exchange/docs](https://merx.exchange/docs)
- Ejemplos de código completos: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)

## Pruébalo Ahora con IA

Añade MERX a Claude Desktop o cualquier cliente compatible con MCP —sin instalación, sin clave de API necesaria para herramientas de solo lectura:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Pregunta a tu agente de IA: "¿Cuál es la energía TRON más barata ahora mismo?" y obtén precios en vivo de todos los proveedores conectados.

Documentación completa de MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)