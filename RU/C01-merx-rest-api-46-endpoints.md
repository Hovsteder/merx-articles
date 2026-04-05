# MERX REST API: 46 конечных точек для торговли TRON energy

MERX REST API предоставляет 46 конечных точек, которые охватывают полный цикл торговли TRON energy и bandwidth — от открытия цен в реальном времени у восьми поставщиков до исполнения ордеров, управления аккаунтом, запросов в блокчейне и автоматизированных постоянных ордеров. Эта статья описывает архитектуру API, модель аутентификации, группы конечных точек, ограничения частоты запросов, обработку ошибок и практические примеры кода на curl, JavaScript и Python.

## Почему единый API имеет значение

Рынок TRON energy разделён между несколькими поставщиками, каждый из которых имеет собственный формат API, схему аутентификации и модель ценообразования. Разработчик, который хочет получить лучшую цену на трансфер USDT, должен интегрироваться с каждым поставщиком отдельно, обрабатывать логику отказоустойчивости и постоянно отслеживать цены.

MERX объединяет всё это в единый REST API. Один API ключ, один заголовок аутентификации, один формат ошибок, один набор SDK. Платформа опрашивает всех подключённых поставщиков каждые 30 секунд, маршрутизирует ордеры к самому дешёвому доступному источнику и проверяет делегирования в блокчейне.

API имеет версию `/api/v1/` и будет поддерживать обратную совместимость. Все конечные точки возвращают JSON в едином формате конверта.

## Аутентификация

MERX использует аутентификацию по API ключу. Каждый аутентифицированный запрос должен включать заголовок `X-API-Key`.

API ключи создаются на панели управления MERX на сайте [merx.exchange](https://merx.exchange) или программно через конечную точку `/api/v1/keys`. Каждый ключ имеет набор прав доступа, которые контролируют, какие операции он может выполнять.

```bash
curl -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/balance
```

Ключи имеют формат `sk_live_` с последующими 64 шестнадцатеричными символами. Сырой ключ отображается ровно один раз при создании. MERX хранит только хеш bcrypt, поэтому потерянные ключи невозможно восстановить — они должны быть отозваны и заменены.

### Права доступа ключей

При создании API ключа вы присваиваете одно или несколько прав доступа:

| Право доступа      | Предоставляет доступ к                  |
|--------------------|-----------------------------------------|
| `create_orders`    | POST /orders, POST /ensure              |
| `view_orders`      | GET /orders, GET /orders/:id            |
| `view_balance`     | GET /balance, GET /history              |
| `broadcast`        | POST /chain/broadcast                   |

Это позволяет вам создавать ключи только для чтения для мониторинга панелей управления и ограниченные ключи для автоматизированных торговых систем.

## Конверт ответа

Каждый ответ следует одной и той же структуре:

```json
{
  "data": { ... }
}
```

При ошибке:

```json
{
  "error": {
    "code": "INSUFFICIENT_FUNDS",
    "message": "Account balance too low for the requested operation",
    "details": { "required": 150000000, "available": 42000000 }
  }
}
```

Коды ошибок — это строки, читаемые машиной. Поле `details` опционально и предоставляет контекст для отладки.

## Группы конечных точек

46 конечных точек организованы в девять групп. Вот полная карта.

### Цены (6 конечных точек)

Эти конечные точки являются общедоступными — API ключ не требуется.

| Метод | Путь                        | Описание                                  |
|-------|-----------------------------|-------------------------------------------|
| GET   | /api/v1/prices              | Текущие цены от всех поставщиков         |
| GET   | /api/v1/prices/best         | Самый дешёвый поставщик для типа ресурса |
| GET   | /api/v1/prices/history      | Исторические данные цен                   |
| GET   | /api/v1/prices/stats        | Статистика совокупного рынка              |
| GET   | /api/v1/prices/analysis     | Анализ тренда и рекомендация покупки     |
| GET   | /api/v1/orders/preview      | Предпросмотр стоимости перед размещением ордера |

### Ордеры (3 конечные точки)

| Метод | Путь                        | Описание                                  |
|-------|-----------------------------|-------------------------------------------|
| POST  | /api/v1/orders              | Создать новый ордер energy или bandwidth  |
| GET   | /api/v1/orders              | Список ордеров с пагинацией и фильтрами  |
| GET   | /api/v1/orders/:id          | Получить детали ордера с разбором заполнений |

### Аккаунт (7 конечных точек)

| Метод | Путь                        | Описание                                  |
|-------|-----------------------------|-------------------------------------------|
| GET   | /api/v1/balance             | Текущие балансы TRX и USDT                |
| GET   | /api/v1/deposit/info        | Адрес депозита и мемо                    |
| POST  | /api/v1/deposit/prepare     | Подготовить транзакцию депозита          |
| POST  | /api/v1/deposit/submit      | Отправить доказательство депозита        |
| POST  | /api/v1/withdraw            | Вывести TRX или USDT                     |
| GET   | /api/v1/history             | История исполнения ордеров                |
| GET   | /api/v1/history/summary     | Статистика совокупного аккаунта          |

### API ключи (3 конечные точки)

| Метод | Путь                        | Описание                                  |
|-------|-----------------------------|-------------------------------------------|
| GET   | /api/v1/keys                | Список всех API ключей                    |
| POST  | /api/v1/keys                | Создать новый API ключ                    |
| DELETE| /api/v1/keys/:id            | Отозвать API ключ                         |

### Аутентификация (2 конечные точки)

| Метод | Путь                        | Описание                                  |
|-------|-----------------------------|-------------------------------------------|
| POST  | /api/v1/auth/register       | Создать новый аккаунт                     |
| POST  | /api/v1/auth/login          | Аутентифицироваться и получить JWT токен  |

### Оценка (2 конечные точки)

| Метод | Путь                        | Описание                                  |
|-------|-----------------------------|-------------------------------------------|
| POST  | /api/v1/estimate            | Оценить energy и стоимость для транзакции |
| POST  | /api/v1/ensure              | Гарантировать минимальные ресурсы на адресе |

### Вебхуки (3 конечные точки)

| Метод | Путь                        | Описание                                  |
|-------|-----------------------------|-------------------------------------------|
| POST  | /api/v1/webhooks            | Создать подписку на вебхук                |
| GET   | /api/v1/webhooks            | Список подписок на вебхуки                |
| DELETE| /api/v1/webhooks/:id        | Удалить вебхук                            |

### Постоянные ордеры и мониторы (7 конечных точек)

| Метод | Путь                             | Описание                           |
|-------|----------------------------------|-------------------------------------|
| POST  | /api/v1/standing-orders          | Создать постоянный ордер           |
| GET   | /api/v1/standing-orders          | Список постоянных ордеров          |
| GET   | /api/v1/standing-orders/:id      | Получить детали постоянного ордера |
| DELETE| /api/v1/standing-orders/:id      | Отменить постоянный ордер          |
| POST  | /api/v1/monitors                 | Создать монитор ресурсов           |
| GET   | /api/v1/monitors                 | Список активных мониторов          |
| DELETE| /api/v1/monitors/:id             | Отменить монитор                   |

### Прокси цепи (10 конечных точек)

Эти конечные точки проксируют запросы сети TRON через MERX, устраняя необходимость для клиентов вызывать TronGrid напрямую.

| Метод | Путь                             | Описание                           |
|-------|----------------------------------|-------------------------------------|
| GET   | /api/v1/chain/account/:address   | Информация аккаунта и ресурсы     |
| GET   | /api/v1/chain/balance/:address   | Баланс TRX                         |
| GET   | /api/v1/chain/resources/:address | Разбор energy и bandwidth         |
| GET   | /api/v1/chain/transaction/:txid  | Детали транзакции                  |
| GET   | /api/v1/chain/block/:number      | Блок по номеру (или последний)     |
| GET   | /api/v1/chain/parameters         | Параметры цепи                     |
| GET   | /api/v1/chain/history/:address   | История транзакций адреса          |
| POST  | /api/v1/chain/read-contract      | Вызвать постоянную функцию контракта |
| POST  | /api/v1/chain/broadcast          | Транслировать подписанную транзакцию |
| GET   | /api/v1/address/:addr/resources  | Сводка ресурсов адреса             |

### x402 Pay-Per-Use (3 конечные точки)

| Метод | Путь                        | Описание                                  |
|-------|-----------------------------|-------------------------------------------|
| POST  | /api/v1/x402/invoice        | Создать счёт платежа                      |
| GET   | /api/v1/x402/invoice/:id    | Проверить статус счёта                    |
| POST  | /api/v1/x402/verify         | Проверить платёж и исполнить ордер        |

## Ключевые конечные точки в деталях

### GET /api/v1/prices

Возвращает текущие цены от всех активных поставщиков. Аутентификация не требуется. Это конечная точка, которую вы вызываете, чтобы увидеть полный рынок с одного взгляда.

```bash
curl https://merx.exchange/api/v1/prices
```

Ответ (сокращённо):

```json
{
  "data": [
    {
      "provider": "sohu",
      "is_market": false,
      "energy_prices": [
        { "duration_sec": 3600, "price_sun": 24 },
        { "duration_sec": 86400, "price_sun": 30 }
      ],
      "bandwidth_prices": [],
      "available_energy": 5000000,
      "available_bandwidth": 0,
      "fetched_at": 1743292800
    }
  ]
}
```

Каждая запись поставщика включает ценовые уровни (по продолжительности), доступную ёмкость и временную метку последнего успешного опроса.

### POST /api/v1/orders

Создаёт ордер energy или bandwidth. Платформа сопоставляет ордер с доступными поставщиками и маршрутизирует к самому дешёвому, который может его исполнить.

```bash
curl -X POST https://merx.exchange/api/v1/orders \
  -H "X-API-Key: sk_live_your_key_here" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: unique-request-id-123" \
  -d '{
    "resource_type": "ENERGY",
    "order_type": "MARKET",
    "amount": 65000,
    "target_address": "TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    "duration_sec": 3600
  }'
```

Ответ:

```json
{
  "data": {
    "id": "ord_a1b2c3d4",
    "status": "PENDING",
    "created_at": "2026-03-30T12:00:00.000Z"
  }
}
```

Заголовок `Idempotency-Key` предотвращает дублирование ордеров, если один и тот же запрос повторяется. Если ключ был виден раньше, API возвращает исходный ордер вместо создания нового.

Типы ордеров:
- `MARKET` — исполнить немедленно по лучшей доступной цене
- `LIMIT` — исполнить только если цена равна или ниже `max_price_sun`
- `PERIODIC` — повторяющийся ордер по расписанию
- `BROADCAST` — трансляция предподписанной транзакции делегирования

### GET /api/v1/orders/:id

Возвращает ордер с его деталями заполнения — какие поставщики исполнили ордер, по какой цене и какие ID транзакций в блокчейне.

```bash
curl -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/orders/ord_a1b2c3d4
```

```json
{
  "data": {
    "id": "ord_a1b2c3d4",
    "resource_type": "ENERGY",
    "order_type": "MARKET",
    "status": "FILLED",
    "amount": 65000,
    "target_address": "TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    "duration_sec": 3600,
    "total_cost_sun": 1560000,
    "fills": [
      {
        "provider": "sohu",
        "amount": 65000,
        "price_sun": 24,
        "cost_sun": 1560000,
        "delegation_tx": "abc123def456...",
        "verified": true,
        "tronscan_url": "https://tronscan.org/#/transaction/abc123def456..."
      }
    ]
  }
}
```

### POST /api/v1/estimate

Оценивает energy и bandwidth, необходимые для операции TRON, затем сравнивает стоимость аренды со стоимостью сжигания TRX.

```bash
curl -X POST https://merx.exchange/api/v1/estimate \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "trc20_transfer",
    "to_address": "TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    "amount": "1000000"
  }'
```

```json
{
  "data": {
    "energy_required": 65000,
    "bandwidth_required": 345,
    "rental_cost": {
      "energy": {
        "best_price_sun": 24,
        "best_provider": "sohu",
        "cost_trx": "1.560"
      }
    },
    "total_rental_trx": "1.905",
    "total_burn_trx": "27.645",
    "savings_percent": 93.1
  }
}
```

Эта конечная точка полезна для того, чтобы показать пользователям ровно, сколько они экономят, арендуя energy через MERX вместо сжигания TRX.

## Ограничения частоты запросов

Ограничения частоты запросов применяются по IP адресу, используя скользящие временные окна.

| Группа конечных точек | Лимит              | Окно    |
|------------------------|--------------------|---------|
| Цены (общедоступные)   | 300 запросов       | 1 мин   |
| По умолчанию (общее)   | 100 запросов       | 1 мин   |
| Баланс                 | 60 запросов        | 1 мин   |
| История                | 60 запросов        | 1 мин   |
| Ордеры                 | 10 запросов        | 1 мин   |
| Выводы                 | 5 запросов         | 1 мин   |
| Трансляция             | 20 запросов        | 1 мин   |
| Регистрация            | 5 запросов         | 1 час   |

Когда ограничение частоты запросов превышено, API возвращает HTTP 429 в стандартном формате ошибки:

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded"
  }
}
```

Заголовки ограничения частоты запросов (`RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`) включены во все ответы в соответствии со стандартом черновика IETF.

## Коды ошибок

| Код                    | HTTP | Описание                                            |
|------------------------|------|-----------------------------------------------------|
| `UNAUTHORIZED`         | 401  | Неверный или отсутствующий API ключ               |
| `RATE_LIMITED`         | 429  | Слишком много запросов                             |
| `VALIDATION_ERROR`     | 400  | Тело запроса или параметры не прошли валидацию    |
| `INVALID_ADDRESS`      | 400  | Не является действительным адресом TRON            |
| `INSUFFICIENT_FUNDS`   | 400  | Баланс аккаунта слишком низкий                     |
| `BELOW_MINIMUM_ORDER`  | 400  | Размер ордера ниже минимума поставщика            |
| `DUPLICATE_REQUEST`    | 409  | Ключ идемпотентности уже использован              |
| `ORDER_NOT_FOUND`      | 404  | Ордер или ресурс не найдены                        |
| `PROVIDER_UNAVAILABLE` | 404  | Ни один поставщик не может исполнить запрос       |
| `INTERNAL_ERROR`       | 500  | Ошибка на стороне сервера                          |

## Быстрый старт с SDK

Хотя REST API можно вызывать напрямую с любым HTTP клиентом, MERX предоставляет официальные SDK для JavaScript и Python, которые обрабатывают аутентификацию, парсинг ошибок и безопасность типов.

### JavaScript / TypeScript

```bash
npm install merx-sdk
```

```javascript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({ apiKey: 'sk_live_your_key_here' })

// Получить все цены
const prices = await merx.prices.list()

// Создать ордер
const order = await merx.orders.create({
  resource_type: 'ENERGY',
  amount: 65000,
  target_address: 'TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb',
  duration_sec: 3600,
})

// Проверить статус ордера
const details = await merx.orders.get(order.id)
```

### Python

```bash
pip install merx-sdk
```

```python
from merx import MerxClient

client = MerxClient(api_key="sk_live_your_key_here")

# Получить все цены
prices = client.prices.list()

# Создать ордер
order = client.orders.create(
    resource_type="ENERGY",
    amount=65000,
    target_address="TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    duration_sec=3600,
)

# Проверить статус ордера
details = client.orders.get(order.id)
```

## WebSocket для данных в реальном времени

В дополнение к REST API, MERX предоставляет конечную точку WebSocket на `wss://merx.exchange/ws` для обновлений цен в реальном времени. Изменения цен транслируются подключённым клиентам по мере их возникновения, с обновлениями, приходящими каждые 30 секунд от каждого поставщика.

Соединение WebSocket поддерживает фильтрацию поставщиков — подписывайтесь только на интересующих вас поставщиков и игнорируйте остальных.

## Постоянные ордеры

Постоянные ордеры автоматизируют покупку energy на основе триггеров. Вы можете установить порог цены, расписание или условие баланса, и платформа автоматически исполняет ордеры в пределах вашего указанного бюджета.

Типы триггеров включают `price_below`, `price_above`, `schedule`, `balance_below` и `provider_available`. Типы действий включают `buy_resource`, `ensure_resources`, `deposit_trx` и `notify_only`.

Это делает MERX подходящей для полностью автоматизированного управления инфраструктурой — установите ваши правила один раз, и платформа обрабатывает исполнение.

## Что дальше

API MERX разработан для разработчиков и компаний, которым требуется надёжный и экономичный доступ к ресурсам сети TRON. Независимо от того, разрабатываете ли вы процессор платежей, приложение DeFi или биржу, API предоставляет строительные блоки для программного управления energy и bandwidth.

Полная документация API доступна на [merx.exchange/docs](https://merx.exchange/docs). JavaScript SDK находится на [GitHub](https://github.com/Hovsteder/merx-sdk-js) и [npm](https://www.npmjs.com/package/merx-sdk). Python SDK находится на [PyPI](https://pypi.org/project/merx-sdk/).

---

**Ссылки:**
- Платформа: [merx.exchange](https://merx.exchange)
- Документация: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- Python SDK: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- MCP сервер: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp) | [npm](https://www.npmjs.com/package/merx-mcp)


## Попробуйте сейчас с ИИ

Добавьте MERX в Claude Desktop или любой совместимый с MCP клиент — без установки, без API ключа для инструментов только для чтения:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Спросите у вашего ИИ агента: "Какой самый дешёвый TRON energy прямо сейчас?" и получите живые цены от всех подключённых поставщиков.

Полная документация MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)