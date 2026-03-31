# MERX价格组件: 在任何网站嵌入实时TRON能量价格

TRON能量价格不断变化。供应商根据需求、网络状况和可用容量调整费率。如果您的网站服务于TRON用户 - 无论是钱包、dApp、区块链浏览器还是资源指南 - 展示实时能量价格能为您的访客提供即时、实用的价值。

MERX提供一个可嵌入的价格组件,按成本排序展示所有主要供应商的实时能量价格。只需两行HTML,每60秒自动刷新,并自带专业的深色主题,无需修改即可适配大多数区块链相关网站。

本文介绍如何嵌入该组件、底层工作原理,以及如何根据您的需求进行自定义。

## 两行HTML

最简单的集成方式。在HTML中任意位置添加这两行:

```html
<div id="merx-prices"></div>
<script src="https://merx.exchange/widget/prices.js"></script>
```

就是这样。脚本自动初始化,从MERX公开API获取当前价格,并在 `div` 中渲染一个带样式的表格。无需API密钥。无需构建步骤。无需框架依赖。

该组件可在任何HTML页面中工作 - 静态网站、WordPress、Webflow、Squarespace(通过自定义代码块),或任何渲染到浏览器的框架。它异步加载,不会阻塞页面渲染。

## 组件展示内容

组件渲染一个紧凑的表格,展示所有活跃的TRON能量供应商及其当前定价。每行包含:

- **供应商名称** - 能量供应商(Sohu、CatFee、NETTs、TronSave、Feee、iTRX、PowerSun)。
- **单位价格** - 当前能量价格(SUN)。这是标准委托中每单位能量的成本。
- **最小/最大订单量** - 供应商当前接受的最小和最大能量数量。
- **时长** - 可用的租赁时长(1小时、3小时、1天等)。
- **状态** - 供应商当前是否在线并接受订单。

供应商按价格排序,最便宜的排在前面。排序随每次刷新更新,因此如果某个供应商降价,它会自动上移。

### 默认外观

组件默认使用深色主题:

- 背景: `#0a0a0a`(近黑色)
- 文字: `#e0e0e0`(浅灰色)
- 表格边框: `#1a1a1a`(细微的深色边框)
- 强调色: `#00d4aa`(MERX绿色,用于最低价高亮)
- 字体: IBM Plex Mono(如果尚未加载,则从Google Fonts加载)

视觉风格刻意保持极简。没有圆角、没有渐变、没有阴影。适合大多数加密货币和区块链界面的美学。

## 底层工作原理

了解组件的内部工作有助于自定义和故障排查。

### 数据来源

组件从MERX公开价格接口获取数据:

```
GET https://merx.exchange/api/v1/prices
```

此接口为公开接口 - 无需认证,无需API密钥。它返回所有已连接供应商的当前价格:

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

### 刷新周期

组件每60秒轮询价格接口。每次刷新是静默的 - 没有加载动画,没有空白闪烁。表格原地更新。如果获取失败(网络问题、服务器超时),组件保留上次成功获取的数据,并在下一个周期重试。

组件底部的小时间戳显示数据最后更新的时间,用户可以一目了然地判断价格是否是最新的。

### 脚本加载

`prices.js` 脚本从MERX CDN提供,带有积极缓存(1小时)和gzip压缩。宽带连接下的典型加载时间低于50ms。脚本经过压缩和gzip后大约8 KB。

加载时,脚本:

1. 找到 `#merx-prices` 元素(或配置的自定义目标)。
2. 注入作用域CSS样式(添加前缀以避免与您的页面样式冲突)。
3. 发起首次API调用。
4. 渲染表格。
5. 设置60秒的刷新间隔。

所有样式都在 `.merx-widget` 下作用域化,防止与您现有样式发生CSS冲突。

## 自定义选项

组件通过容器 `div` 上的数据属性或JavaScript配置对象接受配置。

### 数据属性配置

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

可用数据属性:

| 属性 | 默认值 | 描述 |
|------|--------|------|
| `data-refresh` | `60` | 刷新间隔(秒,最小15) |
| `data-providers` | 全部 | 逗号分隔的供应商列表 |
| `data-duration` | 全部 | 过滤特定时长(1、3、24小时) |
| `data-theme` | `dark` | `dark` 或 `light` |
| `data-max-rows` | 全部 | 最大显示供应商数量 |
| `data-show-header` | `true` | 显示或隐藏"MERX Energy Prices"标题 |
| `data-show-footer` | `true` | 显示或隐藏时间戳底栏 |
| `data-compact` | `false` | 紧凑模式 - 更少列、更小字体 |

### JavaScript配置

如需更多控制,可以以编程方式初始化组件:

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

`onUpdate` 和 `onError` 回调让您可以在自己的代码中响应组件事件。`onUpdate` 回调在每次成功刷新时接收解析后的价格数组。

### 浅色主题

对于浅色背景的网站:

```html
<div id="merx-prices" data-theme="light"></div>
<script src="https://merx.exchange/widget/prices.js"></script>
```

浅色主题颜色:

- 背景: `#ffffff`
- 文字: `#1a1a1a`
- 表格边框: `#e0e0e0`
- 强调色: `#00a88a`(适合浅色背景的深绿色)

### 自定义样式

组件的CSS类名是稳定且有文档记录的。在您自己的样式表中覆盖它们:

```css
/* 使组件全宽 */
.merx-widget {
  width: 100%;
  max-width: none;
}

/* 自定义字体 */
.merx-widget table {
  font-family: 'JetBrains Mono', monospace;
  font-size: 13px;
}

/* 自定义强调色 */
.merx-widget .merx-best-price {
  color: #ff6b00;
}

/* 隐藏特定列 */
.merx-widget .merx-col-duration {
  display: none;
}
```

组件默认 `max-width` 为640px。设置为 `100%` 让它填满容器。

## 完整HTML页面示例

以下是一个完整的、自包含的HTML页面,嵌入了该组件。复制它,在浏览器中打开,您就有了一个实时TRON能量价格跟踪器:

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

## 使用公开API构建自定义组件

如果预置组件不能满足您的需求,您可以直接使用公开价格接口构建自己的组件。这给予您对UI的完全控制。

### 原生JavaScript示例

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

// 初始加载和自动刷新
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

### React组件示例

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

## 公开接口的速率限制

`GET /api/v1/prices` 接口为公开接口,无需认证,但每个IP地址每分钟限制300次请求。对于每60秒刷新一次的组件,您完全在此限制范围内。

如果您构建了一个更频繁轮询的自定义方案 - 例如每5秒聚合价格的后端 - 请考虑缓存结果并从您自己的服务器向前端提供。这可以降低您的MERX API使用量,并改善用户的响应时间。

```javascript
// 服务端缓存示例 (Node.js/Express)
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

这在您的服务器上缓存MERX响应15秒,允许您的前端以任意频率轮询而不增加MERX API的负载。

## SEO注意事项

组件通过JavaScript在客户端渲染,这意味着不执行JavaScript的搜索引擎不会索引价格数据。如果SEO对您的价格页面很重要,请考虑服务端渲染初始价格数据,然后用组件进行实时更新的水合。

或者,在页面中使用关于TRON能量定价的静态内容,将组件作为补充的实时元素。周围的文本提供SEO价值,而组件提供实时实用功能。

## 常见问题

**组件会拖慢我的页面吗?**
不会。脚本经gzip压缩后大约8 KB,异步加载。它不会阻塞渲染。初始API调用增加一个网络请求,但价格接口通常在100ms内响应。

**我能只显示特定供应商吗?**
可以。使用 `data-providers="sohu,catfee"` 过滤显示特定供应商。

**如果MERX API宕机了会怎样?**
组件显示最后一次成功获取的数据。如果从未成功加载过(API宕机时的首次页面加载),它会显示一条消息表明价格暂时不可用。

**组件是移动端响应式的吗?**
是的。在紧凑模式(`data-compact="true"`)下,它可以在窄至320px的屏幕上正常工作。默认模式需要大约500px的最小宽度。

**我可以商业使用组件吗?**
可以。组件和底层的公开价格API都可以免费使用。感谢署名但不强制要求。

## 总结

在您的网站上添加实时TRON能量价格只需两行HTML,零后端工作。MERX价格组件开箱即用地处理数据获取、排序、样式和自动刷新。对于自定义实现,公开价格API无需认证即可使用。

无论您运营TRON钱包、区块链博客还是能量管理工具,展示实时供应商价格都能为您的用户提供他们在其他地方难以找到的可操作信息。

- MERX平台: [merx.exchange](https://merx.exchange)
- API文档: [merx.exchange/docs](https://merx.exchange/docs)
- 完整源码示例: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
