# Виджет цен MERX: встраивание актуальных цен energy TRON на любой сайт

TRON energy prices change constantly. Providers adjust rates based on demand, network conditions, and available capacity. If your website serves TRON users - whether it is a wallet, a dApp, a blockchain explorer, or a resource guide - showing live energy prices adds immediate, practical value for your visitors.

MERX provides an embeddable price widget that displays real-time energy prices from all major providers, sorted by cost. It requires two lines of HTML, auto-refreshes every 60 seconds, and inherits a professional dark theme that fits most blockchain-related websites without modification.

This article covers how to embed the widget, how it works under the hood, and how to customize it for your use case.

## Two Lines of HTML

The simplest possible integration. Add these two lines anywhere in your HTML:

```html
<div id="merx-prices"></div>
<script src="https://merx.exchange/widget/prices.js"></script>
```

That is it. The script initializes automatically, fetches current prices from the MERX public API, and renders a styled table inside the `div`. No API key required. No build step. No framework dependency.

The widget works in any HTML page - static sites, WordPress, Webflow, Squarespace (via custom code blocks), or any framework that renders to the browser. It loads asynchronously and does not block page rendering.

## Что отображает виджет

The widget renders a compact table showing all active TRON energy providers with their current pricing. Each row includes:

- **Provider name** - the energy provider (Sohu, CatFee, NETTs, TronSave, Feee, iTRX, PowerSun).
- **Price per unit** - current energy price in SUN. This is the cost per unit of energy for a standard delegation.
- **Min/Max order** - the minimum and maximum energy amount the provider currently accepts.
- **Duration** - available rental durations (1 hour, 3 hours, 1 day, etc.).
- **Status** - whether the provider is currently online and accepting orders.

Providers are sorted by price, cheapest first. The sorting updates with each refresh, so if a provider drops their price, they move up automatically.

### Default Appearance

The widget uses a dark theme by default:

- Background: `#0a0a0a` (near-black)
- Text: `#e0e0e0` (light gray)
- Table borders: `#1a1a1a` (subtle dark borders)
- Accent color: `#00d4aa` (MERX green, used for the cheapest price highlight)
- Font: IBM Plex Mono (loaded from Google Fonts if not already available)

The visual style is deliberately minimal. No rounded corners, no gradients, no shadows. It fits the aesthetic of most cryptocurrency and blockchain interfaces.

## Как это работает Under the Hood

Understanding the widget's internals helps with customization and troubleshooting.

### Data Source

The widget fetches data from the MERX public prices endpoint:

```
GET https://merx.exchange/api/v1/prices
```

This endpoint is public - no authentication required, no API key needed. It returns current prices from all connected providers:

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

### Refresh Cycle

The widget polls the prices endpoint every 60 seconds. Each refresh is silent - no loading spinner, no flash of empty content. The table updates in place. If a fetch fails (network issue, server timeout), the widget retains the last successful data and tries again on the next cycle.

A small timestamp in the widget footer shows when data was last updated, so users can tell at a glance if prices are current.

### Script Loading

The `prices.js` script is served from the MERX CDN with aggressive caching (1 hour) and gzip compression. Typical load time is under 50ms on broadband connections. The script is approximately 8 KB minified and gzipped.

On load, the script:

1. Finds the `#merx-prices` element (or a custom target if configured).
2. Injects scoped CSS styles (prefixed to avoid conflicts with your page styles).
3. Makes the first API call.
4. Renders the table.
5. Sets up the 60-second refresh interval.

All styles are scoped under `.merx-widget` to prevent CSS conflicts with your existing styles.

## Параметры настройки

The widget accepts configuration via data attributes on the container `div` or through a JavaScript configuration object.

### Data Attribute Configuration

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

Available data attributes:

| Attribute | Default | Description |
|----------|---------|-------------|
| `data-refresh` | `60` | Refresh interval in seconds (minimum 15) |
| `data-providers` | all | Comma-separated list of providers to show |
| `data-duration` | all | Filter to specific duration (1, 3, 24 hours) |
| `data-theme` | `dark` | `dark` or `light` |
| `data-max-rows` | all | Maximum number of providers to display |
| `data-show-header` | `true` | Show or hide the "MERX Energy Prices" header |
| `data-show-footer` | `true` | Show or hide the timestamp footer |
| `data-compact` | `false` | Compact mode - fewer columns, smaller text |

### JavaScript Configuration

For more control, initialize the widget programmatically:

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
      console.log('Prices updated:', prices.length, 'providers');
    },
    onError: function (error) {
      console.error('Widget error:', error.message);
    },
  });
</script>
```

The `onUpdate` and `onError` callbacks let you react to widget events in your own code. The `onUpdate` callback receives the parsed price array on every successful refresh.

### Light Theme

For websites with a light background:

```html
<div id="merx-prices" data-theme="light"></div>
<script src="https://merx.exchange/widget/prices.js"></script>
```

Light theme colors:

- Background: `#ffffff`
- Text: `#1a1a1a`
- Table borders: `#e0e0e0`
- Accent: `#00a88a` (darker green for light backgrounds)

### Custom Styling

The widget's CSS classes are stable and documented. Override them in your own stylesheet:

```css
/* Make the widget full-width */
.merx-widget {
  width: 100%;
  max-width: none;
}

/* Custom font */
.merx-widget table {
  font-family: 'JetBrains Mono', monospace;
  font-size: 13px;
}

/* Custom accent color */
.merx-widget .merx-best-price {
  color: #ff6b00;
}

/* Hide specific columns */
.merx-widget .merx-col-duration {
  display: none;
}
```

The widget's default `max-width` is 640px. Setting it to `100%` lets it fill its container.

## Full HTML Page Example

Here is a complete, self-contained HTML page with the widget embedded. Copy it, open it in a browser, and you have a live TRON energy price tracker:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>TRON Energy Prices - Live</title>
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
    <h1>TRON Energy Prices</h1>
    <p>Live prices from all major providers. Updates every 60 seconds.</p>

    <div id="merx-prices" data-refresh="60" data-compact="false"></div>
    <script src="https://merx.exchange/widget/prices.js"></script>
  </div>
</body>
</html>
```

## Building Your Own Widget with the Public API

If the pre-built widget does not fit your needs, you can build your own using the public prices endpoint directly. This gives you complete control over the UI.

### Vanilla JavaScript Example

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
      <td>${p.available ? 'Online' : 'Offline'}</td>
    </tr>`
    )
    .join('');

  table.innerHTML = `
    <thead>
      <tr>
        <th>Provider</th>
        <th>Price</th>
        <th>Min Order</th>
        <th>Status</th>
      </tr>
    </thead>
    <tbody>${rows}</tbody>
  `;
}

// Initial load and auto-refresh
async function refreshPrices() {
  try {
    const prices = await fetchMerxPrices();
    renderPriceTable(prices);
  } catch (err) {
    console.error('Failed to fetch prices:', err);
  }
}

refreshPrices();
setInterval(refreshPrices, 60000);
```

### React Component Example

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
        console.error('Price fetch failed:', err);
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
            <th>Provider</th>
            <th>Price (SUN)</th>
            <th>Min</th>
            <th>Max</th>
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
        <small>Updated: {lastUpdated.toLocaleTimeString()}</small>
      )}
    </div>
  );
}

export default MerxPrices;
```

## Rate Limits for the Public Endpoint

The `GET /api/v1/prices` endpoint is public and does not require authentication, but it is rate-limited to 300 requests per minute per IP address. For a widget refreshing every 60 seconds, you are well within this limit.

If you build a custom solution that polls more aggressively - for example, a backend that aggregates prices every 5 seconds - consider caching the results and serving them to your frontend from your own server. This keeps your MERX API usage low and improves response times for your users.

```javascript
// Server-side caching example (Node.js/Express)
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

This caches MERX responses for 15 seconds on your server, allowing your frontend to poll as frequently as it wants without increasing load on the MERX API.

## Вопросы SEO

The widget renders client-side via JavaScript, which means search engines that do not execute JavaScript will not index the price data. If SEO is important for your price page, consider server-side rendering the initial price data and then hydrating with the widget for live updates.

Alternatively, structure the page with static content about TRON energy pricing and use the widget as a supplementary live element. The surrounding text provides SEO value while the widget provides real-time utility.

## Часто задаваемые вопросы

**Does the widget slow down my page?**
No. The script is approximately 8 KB gzipped and loads asynchronously. It does not block rendering. The initial API call adds one network request, but prices typically respond within 100ms.

**Can I show only specific providers?**
Yes. Use `data-providers="sohu,catfee"` to filter the display to specific providers.

**What happens if the MERX API is down?**
The widget displays the last successfully fetched data. If it has never successfully loaded (first page load while the API is down), it shows a message indicating prices are temporarily unavailable.

**Is the widget mobile-responsive?**
Yes. In compact mode (`data-compact="true"`), it works well on screens as narrow as 320px. The default mode requires approximately 500px minimum width.

**Can I use the widget commercially?**
Yes. The widget and the underlying public prices API are free to use. Attribution is appreciated but not required.

## Заключение

Adding live TRON energy prices to your website takes two lines of HTML and zero backend work. The MERX price widget handles data fetching, sorting, styling, and auto-refresh out of the box. For custom implementations, the public prices API is available without authentication.

Whether you run a TRON wallet, a blockchain blog, or an energy management tool, showing real-time provider prices gives your users actionable information they cannot easily find elsewhere.

- Платформа MERX: [merx.exchange](https://merx.exchange)
- Документация API: [merx.exchange/docs](https://merx.exchange/docs)
- Full source examples: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)

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
