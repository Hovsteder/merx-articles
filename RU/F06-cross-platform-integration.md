# Кроссплатформенная интеграция: MERX в вашем существующем стеке

Каждый технологический стек уникален. Ваш бэкенд может быть на Node.js, Go, Python, Ruby или Java. Архитектура может быть монолитной или микросервисной. Паттерны коммуникации могут быть синхронными, событийно-ориентированными или комбинированными. Вопрос не в том, может ли MERX встроиться в ваш стек -- вопрос в том, какая точка интеграции даст вам максимальную ценность при минимальном трении.

MERX предлагает пять способов интеграции: REST API, WebSocket, webhooks, языковые SDK и MCP-сервер для AI-агентов. В этой статье объясняется, когда использовать каждый способ, как они взаимодействуют и как выбрать правильный подход для вашей архитектуры.

## Обзор способов интеграции

| Способ | Лучше всего для | Направление | Задержка | Язык |
|---|---|---|---|---|
| REST API | Операции запрос-ответ | Клиент -> MERX | ~200ms | Любой |
| WebSocket | Потоки цен в реальном времени | MERX -> Клиент | Реальное время | Любой |
| Webhooks | Асинхронные уведомления | MERX -> Клиент | Событийная | Любой |
| JS SDK | Приложения Node.js / браузер | Двусторонний | ~200ms | JavaScript/TypeScript |
| Python SDK | Python бэкенд, скрипты | Двусторонний | ~200ms | Python |
| MCP Server | Интеграция с AI-агентами | Двусторонний | ~500ms | Любой (через MCP протокол) |

## REST API: Универсальная интеграция

REST API работает с любым языком программирования, поддерживающим HTTP. Это интеграция наименьшего общего знаменателя -- если ваш язык может делать HTTP-запросы, он может общаться с MERX.

### Когда использовать REST

- Ваш бэкенд написан на языке без SDK (Go, Java, Ruby, Rust, PHP)
- Вам нужны редкие покупки energy, а не постоянное взаимодействие
- Архитектура предпочитает явные HTTP-вызовы постоянным соединениям
- Вы прототипируете и хотите максимально простую интеграцию

### Реализация

API следует стандартным REST-соглашениям с JSON-полезными нагрузками:

```bash
# Проверить цены
curl -X POST https://merx.exchange/api/v1/prices \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"energy_amount": 65000, "duration": "1h"}'

# Разместить заказ
curl -X POST https://merx.exchange/api/v1/orders \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: unique-request-id-123" \
  -d '{
    "energy_amount": 65000,
    "duration": "1h",
    "target_address": "TYourAddress..."
  }'

# Проверить статус заказа
curl https://merx.exchange/api/v1/orders/ORDER_ID \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Пример на Go

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
)

type PriceRequest struct {
    EnergyAmount int    `json:"energy_amount"`
    Duration     string `json:"duration"`
}

type PriceResponse struct {
    Best struct {
        Provider string `json:"provider"`
        PriceSun int    `json:"price_sun"`
    } `json:"best"`
    Providers []struct {
        Provider string `json:"provider"`
        PriceSun int    `json:"price_sun"`
    } `json:"providers"`
}

func getBestPrice(amount int, duration string) (*PriceResponse, error) {
    body, _ := json.Marshal(PriceRequest{
        EnergyAmount: amount,
        Duration:     duration,
    })

    req, _ := http.NewRequest(
        "POST",
        "https://merx.exchange/api/v1/prices",
        bytes.NewBuffer(body),
    )
    req.Header.Set("Authorization", "Bearer "+apiKey)
    req.Header.Set("Content-Type", "application/json")

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var result PriceResponse
    json.NewDecoder(resp.Body).Decode(&result)
    return &result, nil
}
```

### Обработка ошибок

Все ошибки API следуют единообразному формату:

```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Account balance is insufficient for this order",
    "details": {
      "required": 1820000,
      "available": 500000
    }
  }
}
```

Эта согласованность означает, что логика обработки ошибок работает одинаково независимо от того, какую конечную точку вы вызываете или какой поставщик работает за кулисами.

## WebSocket: Потоки цен в реальном времени

WebSocket-соединения предоставляют обновления цен в реальном времени без опроса. Цены поступают в ваше приложение по мере их изменения у всех семи поставщиков.

### Когда использовать WebSocket

- Вам нужны живые отображения цен (панели мониторинга, интерфейсы торговли)
- Ваше приложение принимает решения о покупке на основе движения цен
- Вы хотите запускать действия, когда цены пересекают пороговые значения
- Вы создаете инструменты мониторинга в реальном времени

### Реализация

```typescript
import WebSocket from 'ws';

const ws = new WebSocket(
  'wss://merx.exchange/ws',
  { headers: { 'Authorization': `Bearer ${API_KEY}` } }
);

ws.on('open', () => {
  // Подписаться на обновления цен для конкретных параметров
  ws.send(JSON.stringify({
    type: 'subscribe',
    channel: 'prices',
    params: {
      energy_amount: 65000,
      duration: '1h'
    }
  }));
});

ws.on('message', (data) => {
  const event = JSON.parse(data.toString());

  switch (event.type) {
    case 'price_update':
      console.log(
        `${event.provider}: ${event.price_sun} SUN`
      );
      break;

    case 'order_update':
      console.log(
        `Order ${event.order_id}: ${event.status}`
      );
      break;
  }
});
```

### Комбинирование WebSocket с REST

Распространённый паттерн: использовать WebSocket для мониторинга цен и REST для размещения заказов:

```typescript
// WebSocket отслеживает цены
ws.on('message', async (data) => {
  const event = JSON.parse(data.toString());

  if (event.type === 'price_update' &&
      event.price_sun <= targetPrice) {
    // REST API размещает заказ
    const response = await fetch(
      'https://merx.exchange/api/v1/orders',
      {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${API_KEY}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          energy_amount: 65000,
          duration: '1h',
          target_address: walletAddress
        })
      }
    );
  }
});
```

## Webhooks: Асинхронные уведомления о событиях

Webhooks отправляют уведомления на ваш сервер при возникновении событий. В отличие от WebSocket (которые требуют постоянного соединения), webhooks работают с любым сервером, способным получать POST-запросы HTTP.

### Когда использовать Webhooks

- Ваша архитектура событийно-ориентирована (очереди сообщений, serverless-функции)
- Вы не можете поддерживать постоянные WebSocket-соединения
- Вам нужна надёжная доставка с повторными попытками
- Вы хотите отделить закупку energy от выполнения транзакций

### Реализация

```typescript
import express from 'express';

const app = express();
app.use(express.json());

app.post('/webhooks/merx', (req, res) => {
  const event = req.body;

  switch (event.type) {
    case 'order.filled':
      handleOrderFilled(event.data);
      break;

    case 'order.failed':
      handleOrderFailed(event.data);
      break;

    case 'standing_order.triggered':
      handleStandingOrderTriggered(event.data);
      break;

    case 'auto_energy.purchased':
      handleAutoEnergyPurchased(event.data);
      break;
  }

  // Всегда ответить 200 для подтверждения получения
  res.status(200).json({ received: true });
});

async function handleOrderFilled(
  data: OrderFilledEvent
): Promise<void> {
  // Energy доступна -- перейти к выполнению транзакции
  const pendingTx = await db.getPendingTransaction(
    data.order_id
  );
  if (pendingTx) {
    await executeTransaction(pendingTx);
  }
}
```

### Надёжность Webhooks

MERX повторяет неудачные доставки webhooks с экспоненциальной задержкой. Ваша конечная точка должна:

1. Ответить со статусом 200 быстро (в течение 5 секунд)
2. Обработать событие асинхронно, если обработка занимает время
3. Обрабатывать дублирующиеся доставки идемпотентно (используйте ID события для дедупликации)

```typescript
const processedEvents = new Set<string>();

app.post('/webhooks/merx', async (req, res) => {
  const event = req.body;

  // Проверка идемпотентности
  if (processedEvents.has(event.id)) {
    res.status(200).json({ received: true });
    return;
  }

  processedEvents.add(event.id);

  // Подтвердить немедленно
  res.status(200).json({ received: true });

  // Обработать асинхронно
  processEvent(event).catch(console.error);
});
```

## JavaScript SDK

JavaScript/TypeScript SDK оборачивает REST API и WebSocket-соединения типизированными интерфейсами и удобными методами.

### Когда использовать JS SDK

- Ваш бэкенд -- Node.js или ваш фронтенд работает в браузере
- Вы хотите типы TypeScript и автодополнение IDE
- Вы предпочитаете вызовы методов вместо сырых HTTP-запросов
- Вам нужны REST и WebSocket в одном пакете

### Реализация

```typescript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY!
});

// Цены
const prices = await merx.getPrices({
  energy_amount: 65000,
  duration: '1h'
});

// Заказы
const order = await merx.createOrder({
  energy_amount: 65000,
  duration: '1h',
  target_address: 'TAddress...'
});

// Постоянные заказы
const standing = await merx.createStandingOrder({
  energy_amount: 65000,
  max_price_sun: 25,
  duration: '1h',
  repeat: true
});

// Оценка energy
const estimate = await merx.estimateEnergy({
  contract_address: 'TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t',
  function_selector: 'transfer(address,uint256)',
  parameter: [recipient, amount],
  owner_address: sender
});

// WebSocket (встроен в SDK)
const ws = merx.connectWebSocket();
ws.on('price_update', (data) => {
  console.log(`${data.provider}: ${data.price_sun} SUN`);
});
```

SDK автоматически обрабатывает аутентификацию, сериализацию запросов, парсинг ошибок и валидацию типов.

## Python SDK

Python SDK предоставляет ту же функциональность для Python-бэкендов, конвейеров анализа данных и автоматизационных скриптов.

### Когда использовать Python SDK

- Ваш бэкенд написан на Python (Django, Flask, FastAPI)
- Вы создаёте инструменты анализа данных или отчётов
- Ваши DevOps-скрипты написаны на Python
- Вы интегрируете с торговыми ботами на Python

### Реализация

```python
from merx import MerxClient

merx = MerxClient(api_key="your-api-key")

# Получить цены
prices = merx.get_prices(
    energy_amount=65000,
    duration="1h"
)
print(f"Best: {prices.best.price_sun} SUN "
      f"via {prices.best.provider}")

# Разместить заказ
order = merx.create_order(
    energy_amount=65000,
    duration="1h",
    target_address="TAddress..."
)

# Оценить energy для вызова контракта
estimate = merx.estimate_energy(
    contract_address="TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
    function_selector="transfer(address,uint256)",
    parameter=[recipient, amount],
    owner_address=sender
)
print(f"Energy needed: {estimate.energy_required}")
```

### Пример анализа данных

```python
import pandas as pd
from merx import MerxClient

merx = MerxClient(api_key="your-api-key")

# Получить историю цен для анализа
history = merx.get_price_history(
    energy_amount=65000,
    duration="1h",
    period="30d"
)

df = pd.DataFrame(history.prices)
print(f"Average price: {df['price_sun'].mean():.1f} SUN")
print(f"Min price: {df['price_sun'].min()} SUN")
print(f"Max price: {df['price_sun'].max()} SUN")
print(f"Std dev: {df['price_sun'].std():.1f} SUN")
```

## MCP Server: Интеграция с AI-агентами

MERX MCP (Model Context Protocol) сервер позволяет AI-агентам взаимодействовать непосредственно с рынком TRON energy. Это новая точка интеграции, которая позволяет принципиально другую модель взаимодействия.

### Когда использовать MCP

- Вы создаёте AI-агентов, управляющих операциями на TRON
- Вы хотите диалоговое управление энергией
- Вы используете Claude, ChatGPT или другие LLM с возможностью использования инструментов
- Вы хотите быстро прототипировать энергетические стратегии через естественный язык

### Как это работает

MCP-сервер на [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp) предоставляет возможности MERX как инструменты, которые могут вызывать AI-агенты:

- `get_prices` -- Проверить текущие цены на energy
- `create_order` -- Купить energy
- `analyze_prices` -- Получить статистику цен
- `estimate_energy` -- Смоделировать потребление energy при транзакции
- `check_resources` -- Проверить баланс energy в кошельке

AI-агент может использовать эти инструменты для ответа на вопросы вроде "Какова самая дешёвая energy прямо сейчас?" или выполнения команд вроде "Купи 65000 energy для моего кошелька по лучшей цене."

### Интеграция

```json
{
  "mcpServers": {
    "merx": {
      "command": "npx",
      "args": ["merx-mcp"],
      "env": {
        "MERX_API_KEY": "your-api-key"
      }
    }
  }
}
```

## Выбор правильной интеграции

### Матрица решений

| Ваша ситуация | Рекомендуемая интеграция |
|---|---|
| Быстрый прототип, любой язык | REST API |
| Node.js бэкенд | JS SDK |
| Python бэкенд | Python SDK |
| Отображение цен в реальном времени | WebSocket |
| Событийно-ориентированная архитектура | Webhooks |
| Serverless (Lambda, Cloud Functions) | REST API + Webhooks |
| Создание AI-агента | MCP Server |
| Go / Java / Ruby бэкенд | REST API |
| Полнофункциональное приложение | JS/Python SDK + Webhooks |

### Комбинирование точек интеграции

Большинство production-систем используют несколько способов интеграции:

- **SDK + Webhooks**: Используйте SDK для исходящих запросов (цены, заказы) и webhooks для входящих уведомлений (заказ выполнен, оповещение о цене)
- **WebSocket + REST**: Используйте WebSocket для мониторинга и REST для действий
- **REST + Webhooks**: Языко-агностичный полнофункциональный стек
- **MCP + SDK**: AI-агент для стратегии, SDK для выполнения

## Путь миграции

Если вы в настоящее время используете API одного поставщика, миграция на MERX прямолинейна:

1. **Добавить SDK MERX** в ваш проект
2. **Заменить проверки цен** на `merx.getPrices()` -- те же данные, больше поставщиков
3. **Заменить размещение заказов** на `merx.createOrder()` -- тот же процесс, маршрутизация по лучшей цене
4. **Добавить webhooks** для уведомлений о заказах (если архитектура ещё не событийно-ориентирована)
5. **Удалить старый код поставщика** после проверки интеграции с MERX

Миграция может быть сделана пошагово. Запустите обе системы параллельно во время тестирования, сравнивая цены и результаты заказов перед полным переключением.

## Заключение

MERX разработан так, чтобы встроиться в любой технологический стек через способ интеграции, соответствующий вашей архитектуре. REST для универсальности, WebSocket для реального времени, webhooks для событий, SDK для опыта разработчиков и MCP для AI-агентов.

Выбор не является исключающим -- комбинируйте точки интеграции в соответствии с вашими потребностями. Начните с самого простого варианта, который решает вашу непосредственную проблему, и расширяйте по мере роста ваших требований.

Документация API на [https://merx.exchange/docs](https://merx.exchange/docs). MCP-сервер на [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp). Платформа на [https://merx.exchange](https://merx.exchange).


## Попробуйте прямо сейчас с AI

Добавьте MERX в Claude Desktop или любой MCP-совместимый клиент -- без установки, без API-ключа для инструментов, доступных только для чтения:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Спросите у вашего AI-агента: "Какова самая дешёвая TRON energy прямо сейчас?" и получите живые цены от всех подключённых поставщиков.

Полная документация MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)