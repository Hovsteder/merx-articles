# Виджет цен MERX: встройте живые цены на энергию TRON на любой сайт

Цены на энергию TRON постоянно меняются. Поставщики корректируют тарифы в зависимости от спроса, состояния сети и доступных мощностей. Если ваш сайт обслуживает пользователей TRON — будь то кошелек, dApp, обозреватель блокчейна или справочник по ресурсам — отображение живых цен на энергию добавляет для посетителей немедленную практическую ценность.

MERX предоставляет встраиваемый виджет цен, который отображает цены на энергию в реальном времени от всех крупных поставщиков, отсортированные по стоимости. Требует всего две строки HTML, автоматически обновляется каждые 60 секунд и использует профессиональную темную тему, которая без изменений подходит для большинства сайтов, связанных с блокчейном.

В этой статье рассказывается, как встроить виджет, как он работает внутри и как его настроить под вашу задачу.

## Две строки HTML

Максимально простая интеграция. Добавьте эти две строки в любое место HTML-кода:

```html
<div id="merx-prices"></div>
<script src="https://merx.exchange/widget/prices.js"></script>
```

Вот и всё. Скрипт инициализируется автоматически, получает текущие цены из открытого API MERX и отображает стилизованную таблицу внутри `div`. Никакой API ключ не требуется. Никакой процесс сборки. Никакой зависимости от фреймворка.

Виджет работает на любой HTML-странице — статических сайтах, WordPress, Webflow, Squarespace (через блоки пользовательского кода) или любом фреймворке, который отображает содержимое в браузере. Он загружается асинхронно и не блокирует отображение страницы.

## Что отображает виджет

Виджет отображает компактную таблицу со всеми активными поставщиками энергии TRON и их текущими ценами. Каждая строка включает:

- **Имя поставщика** — поставщик энергии (Sohu, CatFee, NETTs, TronSave, Feee, iTRX, PowerSun).
- **Цена за единицу** — текущая цена энергии в SUN. Это стоимость за единицу энергии при стандартной делегации.
- **Минимум/максимум заказа** — минимальное и максимальное количество энергии, которое поставщик в настоящий момент принимает.
- **Длительность** — доступные сроки аренды (1 час, 3 часа, 1 день и т. д.).
- **Статус** — находится ли поставщик в сети и принимает ли заказы.

Поставщики отсортированы по цене, дешевле всех в начале. Сортировка обновляется при каждом обновлении, поэтому если поставщик снизит цену, он автоматически переместится вверх.

### Внешний вид по умолчанию

Виджет использует тёмную тему по умолчанию:

- Фон: `#0a0a0a` (почти чёрный)
- Текст: `#e0e0e0` (светло-серый)
- Границы таблицы: `#1a1a1a` (тонкие тёмные границы)
- Цвет акцента: `#00d4aa` (зелень MERX, используется для выделения самой низкой цены)
- Шрифт: IBM Plex Mono (загружается из Google Fonts, если ещё не доступен)

Визуальный стиль специально минималистичен. Никаких скруглённых углов, градиентов, теней. Он гармонирует с эстетикой большинства криптовалютных и блокчейн-интерфейсов.

## Как это работает внутри

Понимание внутреннего устройства виджета помогает с настройкой и устранением неполадок.

### Источник данных

Виджет получает данные из открытой точки API цен MERX:

```
GET https://merx.exchange/api/v1/prices
```

Эта точка открыта — не требуется аутентификация, не нужен API ключ. Она возвращает текущие цены от всех подключённых поставщиков:

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

### Цикл обновления

Виджет опрашивает точку цен каждые 60 секунд. Каждое обновление бесшумно — никакого спиннера загрузки, никакого мерцания пустого содержимого. Таблица обновляется на месте. Если загрузка не удаётся (проблема с сетью, тайм-аут сервера), виджет сохраняет последние успешно полученные данные и повторяет попытку в следующем цикле.

Небольшая отметка времени в нижней части виджета показывает, когда данные в последний раз обновлялись, поэтому пользователи могут сразу увидеть, актуальны ли цены.

### Загрузка скрипта

Скрипт `prices.js` доставляется из CDN MERX с агрессивным кешированием (1 час) и сжатием gzip. Типичное время загрузки — менее 50 мс на широкополосных соединениях. Скрипт занимает приблизительно 8 КБ в сжатом и минифицированном виде.

При загрузке скрипт:

1. Находит элемент `#merx-prices` (или пользовательскую цель, если она настроена).
2. Вводит стили CSS в области видимости (с префиксами, чтобы избежать конфликтов со стилями вашей страницы).
3. Делает первый вызов API.
4. Отображает таблицу.
5. Устанавливает интервал обновления в 60 секунд.

Все стили находятся в области видимости под `.merx-widget`, чтобы предотвратить конфликты CSS с существующими стилями.

## Параметры настройки

Виджет принимает конфигурацию через data-атрибуты на контейнере `div` или через объект конфигурации JavaScript.

### Конфигурация через data-атрибуты

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

Доступные data-атрибуты:

| Атрибут | По умолчанию | Описание |
|----------|---------|-------------|
| `data-refresh` | `60` | Интервал обновления в секундах (минимум 15) |
| `data-providers` | все | Разделённый запятыми список поставщиков для отображения |
| `data-duration` | все | Фильтр по определённой длительности (1, 3, 24 часа) |
| `data-theme` | `dark` | `dark` или `light` |
| `data-max-rows` | все | Максимальное количество поставщиков для отображения |
| `data-show-header` | `true` | Показать или скрыть заголовок "MERX Energy Prices" |
| `data-show-footer` | `true` | Показать или скрыть нижний колонтитул с отметкой времени |
| `data-compact` | `false` | Компактный режим — меньше колонок, меньше текста |

### Конфигурация через JavaScript

Для большего контроля инициализируйте виджет программно:

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

Обратные вызовы `onUpdate` и `onError` позволяют реагировать на события виджета в собственном коде. Обратный вызов `onUpdate` получает разобранный массив цен при каждом успешном обновлении.

### Светлая тема

Для сайтов со светлым фоном:

```html
<div id="merx-prices" data-theme="light"></div>
<script src="https://merx.exchange/widget/prices.js"></script>
```

Цвета светлой темы:

- Фон: `#ffffff`
- Текст: `#1a1a1a`
- Границы таблицы: `#e0e0e0`
- Акцент: `#00a88a` (более тёмная зелень для светлых фонов)

### Пользовательские стили

CSS-классы виджета стабильны и документированы. Переопределите их в собственной таблице стилей:

```css
/* Сделать виджет во всю ширину */
.merx-widget {
  width: 100%;
  max-width: none;
}

/* Пользовательский шрифт */
.merx-widget table {
  font-family: 'JetBrains Mono', monospace;
  font-size: 13px;
}

/* Пользовательский цвет акцента */
.merx-widget .merx-best-price {
  color: #ff6b00;
}

/* Скрыть определённые колонки */
.merx-widget .merx-col-duration {
  display: none;
}
```

Значение `max-width` виджета по умолчанию составляет 640 пикселей. Установка его на `100%` позволяет заполнить контейнер.

## Пример полной HTML-страницы

Вот полная, самодостаточная HTML-страница с встроенным виджетом. Скопируйте её, откройте в браузере, и у вас будет живой трекер цен на энергию TRON:

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

## Создание собственного виджета с открытым API

Если готовый виджет не соответствует вашим требованиям, вы можете создать свой собственный, используя точку открытого API цен. Это даёт вам полный контроль над интерфейсом.

### Пример Vanilla JavaScript

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

### Пример компонента React

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

## Ограничения скорости для открытой точки

Точка `GET /api/v1/prices` открыта и не требует аутентификации, но ограничена 300 запросами в минуту на IP-адрес. Для виджета, обновляющегося каждые 60 секунд, вы хорошо в пределах этого ограничения.

Если вы создадите пользовательское решение, которое опрашивает более активно — например, бэкенд, который агрегирует цены каждые 5 секунд — рассмотрите возможность кеширования результатов и их отправки на фронтенд с вашего собственного сервера. Это снижает использование API MERX и улучшает время отклика для ваших пользователей.

```javascript
// Пример кеширования на сервере (Node.js/Express)
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

Это кеширует ответы MERX на вашем сервере на 15 секунд, позволяя фронтенду опрашивать так часто, как нужно, без увеличения нагрузки на API MERX.

## Соображения о SEO

Виджет отображается клиентской стороной через JavaScript, что означает, что поисковые системы, которые не выполняют JavaScript, не будут индексировать данные цен. Если SEO важна для вашей страницы цен, рассмотрите рендеринг начальных данных о ценах на сервере и последующую гидратацию виджетом для живых обновлений.

Как вариант, структурируйте страницу со статическим содержимым о ценах на энергию TRON и используйте виджет как дополнительный элемент в реальном времени. Окружающий текст обеспечивает ценность для SEO, а виджет обеспечивает полезность в реальном времени.

## Часто задаваемые вопросы

**Замедляет ли виджет мою страницу?**
Нет. Скрипт занимает приблизительно 8 КБ в сжатом виде и загружается асинхронно. Он не блокирует отображение. Первый вызов API добавляет один сетевой запрос, но цены обычно отвечают в течение 100 мс.

**Можно ли показывать только конкретных поставщиков?**
Да. Используйте `data-providers="sohu,catfee"`, чтобы отфильтровать отображение по определённым поставщикам.

**Что происходит, если API MERX выходит из строя?**
Виджет отображает последние успешно полученные данные. Если он никогда не успешно загружался (первая загрузка страницы при отключённом API), он показывает сообщение о том, что цены временно недоступны.

**Адаптивен ли виджет для мобильных устройств?**
Да. В компактном режиме (`data-compact="true"`) он хорошо работает на экранах шириной всего 320 пикселей. Режим по умолчанию требует минимальную ширину примерно 500 пикселей.

**Могу ли я использовать виджет в коммерческих целях?**
Да. Виджет и лежащий в его основе открытый API цен бесплатны в использовании. Указание авторства приветствуется, но не требуется.

## Заключение

Добавление живых цен на энергию TRON на ваш сайт занимает две строки HTML и не требует никакой работы с бэкенда. Виджет цен MERX обрабатывает загрузку данных, сортировку, стилизацию и автоматическое обновление из коробки. Для пользовательских реализаций открытый API цен доступен без аутентификации.

Независимо от того, управляете ли вы кошельком TRON, блогом о блокчейне или инструментом управления энергией, отображение цен поставщиков в реальном времени предоставляет вашим пользователям полезную информацию, которую они не могут легко найти в других местах.

- Платформа MERX: [merx.exchange](https://merx.exchange)
- Документация API: [merx.exchange/docs](https://merx.exchange/docs)
- Полные примеры исходного кода: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)


## Попробуйте прямо сейчас с AI

Добавьте MERX в Claude Desktop или любой MCP-совместимый клиент — без установки, без API ключа для инструментов только для чтения:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Спросите у вашего AI-агента: "What is the cheapest TRON energy right now?" и получите живые цены от всех подключённых поставщиков.

Полная документация MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)