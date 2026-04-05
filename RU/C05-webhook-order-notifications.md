# Интеграция вебхуков: получайте уведомления о заполнении заказов energy

Вебхуки MERX доставляют уведомления в реальном времени через HTTP когда происходят события на вашем аккаунте — заполнение заказов, отказы в заказах, поступления депозитов, завершение снятий средств. Эта статья охватывает все четыре типа событий, формат payload, проверку подписи HMAC-SHA256 используя заголовок `X-Merx-Signature`, политику повторных попыток с экспоненциальной задержкой, автоматическую деактивацию после повторных отказов и полные реализации серверов на Express.js и Flask с проверкой подписи.

## Почему вебхуки лучше, чем polling

Альтернатива вебхукам — polling эндпоинта `/api/v1/orders/:id` в цикле, ожидание изменения статуса с `PENDING` на `FILLED`. Это работает для простых случаев, но имеет явные недостатки:

- Потраченные запросы. Большинство polling запросов возвращают неизменный статус.
- Задержка. Ваше приложение обнаруживает изменение состояния только на следующем интервале polling.
- Rate limits. С лимитом 10 запросов в минуту на эндпоинт заказов, агрессивный polling быстро достигает потолка.
- Сложность. Логика polling требует обработку повторных попыток, управление timeout и отслеживание состояния.

Вебхуки инвертируют модель. Вместо вопроса к MERX "что-нибудь изменилось?", MERX сообщает вам сразу же когда что-нибудь произойдёт. Ваш сервер получает HTTP POST с полным payload события, обрабатывает его и идёт дальше. Нет polling циклов, нет потраченных запросов, нет искусственных задержек.

## Создание вебхука

Вы можете создавать вебхуки через REST API или SDK.

### REST API

```bash
curl -X POST https://merx.exchange/api/v1/webhooks \
  -H "X-API-Key: sk_live_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-server.com/webhooks/merx",
    "events": ["order.filled", "order.failed", "deposit.received", "withdrawal.completed"]
  }'
```

Ответ:

```json
{
  "data": {
    "id": "wh_abc123",
    "url": "https://your-server.com/webhooks/merx",
    "events": ["order.filled", "order.failed", "deposit.received", "withdrawal.completed"],
    "secret": "a1b2c3d4e5f6...64-hex-characters...9876543210",
    "is_active": true,
    "created_at": "2026-03-30T12:00:00.000Z"
  }
}
```

Поле `secret` — это 64-символная шестнадцатеричная строка, сгенерированная из 32 случайных байтов. Оно возвращается только в ответе на создание. Сохраняйте его в безопасности — он вам понадобится для проверки подписей входящих вебхуков.

### JavaScript SDK

```javascript
import { MerxClient } from 'merx-sdk'

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY })

const webhook = await merx.webhooks.create({
  url: 'https://your-server.com/webhooks/merx',
  events: ['order.filled', 'order.failed', 'deposit.received'],
})

console.log(`Webhook ID: ${webhook.id}`)
console.log(`Secret: ${webhook.secret}`)  // Store this securely
```

### Python SDK

```python
from merx import MerxClient

client = MerxClient(api_key="sk_live_your_key_here")

webhook = client.webhooks.create(
    url="https://your-server.com/webhooks/merx",
    events=["order.filled", "order.failed", "deposit.received"],
)

print(f"Webhook ID: {webhook.id}")
print(f"Secret: {webhook.secret}")  # Store this securely
```

## Типы событий

MERX поддерживает четыре типа событий вебхука. При создании вебхука вы выбираете, на какие события подписываться. Вы можете подписаться на все четыре или только на нужные вам.

### order.filled

Отправляется когда заказ полностью исполнен. Все делегирования поставщиков подтверждены на цепи.

```json
{
  "event": "order.filled",
  "timestamp": "2026-03-30T12:05:00.000Z",
  "data": {
    "order_id": "ord_abc123",
    "resource_type": "ENERGY",
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

Это наиболее важное событие для автоматизированных систем. Когда вы его получите, energy делегирован и целевой адрес может перейти к своим TRON транзакциям.

### order.failed

Отправляется когда заказ не может быть исполнен. Это может произойти когда все поставщики недоступны, мощность исчерпана или произойдёт ошибка на стороне поставщика.

```json
{
  "event": "order.failed",
  "timestamp": "2026-03-30T12:05:00.000Z",
  "data": {
    "order_id": "ord_def456",
    "resource_type": "ENERGY",
    "amount": 500000,
    "target_address": "TJYpFDq5cVnRJey8Xt8HfaRtNkqFTZwBb",
    "reason": "No provider could fulfill the order within the specified parameters",
    "refunded": true,
    "refund_amount_sun": 0
  }
}
```

Когда заказ отклоняется, любой зарезервированный баланс возвращается. Поле `refunded` подтверждает это, а `refund_amount_sun` показывает сумму возврата, если платёж уже был вычтен.

### deposit.received

Отправляется когда обнаружен депозит на ваш аккаунт MERX и он зачислен.

```json
{
  "event": "deposit.received",
  "timestamp": "2026-03-30T12:10:00.000Z",
  "data": {
    "deposit_id": "dep_ghi789",
    "amount_sun": 100000000,
    "amount_trx": "100.000000",
    "currency": "TRX",
    "tx_id": "789abc...",
    "new_balance_trx": 250.5
  }
}
```

### withdrawal.completed

Отправляется когда запрос на снятие обработан и транзакция на цепи подтверждена.

```json
{
  "event": "withdrawal.completed",
  "timestamp": "2026-03-30T12:15:00.000Z",
  "data": {
    "withdrawal_id": "wdr_jkl012",
    "amount": 50,
    "currency": "TRX",
    "address": "TExternalAddress...",
    "tx_id": "012def...",
    "tronscan_url": "https://tronscan.org/#/transaction/012def..."
  }
}
```

## Проверка подписи

Каждая доставка вебхука включает заголовок `X-Merx-Signature` содержащий подпись HMAC-SHA256 тела запроса, вычисленную используя ваш секрет вебхука как ключ.

Процесс проверки:

1. Прочитайте сырое тело запроса (до парсинга JSON).
2. Вычислите HMAC-SHA256 сырого тела используя ваш сохранённый секрет вебхука.
3. Сравните вычисленную подпись со значением заголовка `X-Merx-Signature`.
4. Если они совпадают, запрос аутентичен. Если нет, отклоните его.

Это защищает от:
- Поддельных запросов от третьих сторон которые не знают ваш секрет.
- Повреждённых payload где тело было изменено при передаче.
- Атак повторного воспроизведения (в сочетании с проверкой временной метки).

Всегда используйте функцию сравнения с постоянным временем при проверке подписей. Стандартное сравнение строк (`===` или `==`) уязвимо для timing атак.

## Express.js обработчик вебхука

Вот полный сервер Express.js который получает и проверяет вебхуки MERX:

```javascript
import express from 'express'
import crypto from 'node:crypto'

const app = express()
const WEBHOOK_SECRET = process.env.MERX_WEBHOOK_SECRET

// Use raw body for signature verification
app.post('/webhooks/merx', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-merx-signature']
  if (!signature || !WEBHOOK_SECRET) {
    res.status(401).json({ error: 'Missing signature or secret' })
    return
  }

  // Compute expected signature
  const expected = crypto
    .createHmac('sha256', WEBHOOK_SECRET)
    .update(req.body)
    .digest('hex')

  // Constant-time comparison
  if (!crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected))) {
    console.warn('Webhook signature mismatch')
    res.status(401).json({ error: 'Invalid signature' })
    return
  }

  // Signature verified - parse and handle the event
  const event = JSON.parse(req.body.toString())
  console.log(`Received event: ${event.event}`)

  switch (event.event) {
    case 'order.filled':
      handleOrderFilled(event.data)
      break
    case 'order.failed':
      handleOrderFailed(event.data)
      break
    case 'deposit.received':
      handleDepositReceived(event.data)
      break
    case 'withdrawal.completed':
      handleWithdrawalCompleted(event.data)
      break
    default:
      console.warn(`Unknown event type: ${event.event}`)
  }

  // Always respond with 200 quickly to acknowledge receipt
  res.status(200).json({ received: true })
})

function handleOrderFilled(data) {
  console.log(`Order ${data.order_id} filled`)
  console.log(`  Amount: ${data.amount} ${data.resource_type}`)
  console.log(`  Cost: ${(data.total_cost_sun / 1_000_000).toFixed(3)} TRX`)
  console.log(`  Fills: ${data.fills.length}`)

  for (const fill of data.fills) {
    console.log(`  ${fill.provider}: ${fill.amount} at ${fill.price_sun} SUN`)
    if (fill.tronscan_url) {
      console.log(`    TX: ${fill.tronscan_url}`)
    }
  }

  // Your business logic here:
  // - Mark the transaction as ready to proceed
  // - Trigger the USDT transfer
  // - Update your database
}

function handleOrderFailed(data) {
  console.log(`Order ${data.order_id} failed: ${data.reason}`)
  if (data.refunded) {
    console.log(`  Refunded: ${data.refund_amount_sun} SUN`)
  }
  // Alert your operations team
}

function handleDepositReceived(data) {
  console.log(`Deposit received: ${data.amount_trx} ${data.currency}`)
  console.log(`  New balance: ${data.new_balance_trx} TRX`)
  // Update local balance cache
}

function handleWithdrawalCompleted(data) {
  console.log(`Withdrawal ${data.withdrawal_id} completed`)
  console.log(`  ${data.amount} ${data.currency} to ${data.address}`)
  // Update withdrawal status in your system
}

app.listen(3000, () => {
  console.log('Webhook server listening on port 3000')
})
```

Ключевые детали реализации:

- Используйте `express.raw({ type: 'application/json' })` вместо `express.json()` для маршрута вебхука. Вам нужны сырые байты для вычисления подписи.
- Используйте `crypto.timingSafeEqual()` для сравнения подписей с постоянным временем.
- Отвечайте с HTTP 200 максимально быстро. Выполняйте тяжёлую обработку асинхронно после подтверждения получения.

## Flask обработчик вебхука

Вот эквивалентная реализация на Python используя Flask:

```python
import hashlib
import hmac
import json
import os

from flask import Flask, request, jsonify

app = Flask(__name__)
WEBHOOK_SECRET = os.environ.get("MERX_WEBHOOK_SECRET", "")


def verify_signature(payload: bytes, signature: str, secret: str) -> bool:
    """Verify HMAC-SHA256 signature using constant-time comparison."""
    expected = hmac.new(
        secret.encode("utf-8"),
        payload,
        hashlib.sha256,
    ).hexdigest()
    return hmac.compare_digest(signature, expected)


@app.route("/webhooks/merx", methods=["POST"])
def handle_webhook():
    signature = request.headers.get("X-Merx-Signature", "")
    raw_body = request.get_data()

    if not signature or not WEBHOOK_SECRET:
        return jsonify({"error": "Missing signature or secret"}), 401

    if not verify_signature(raw_body, signature, WEBHOOK_SECRET):
        app.logger.warning("Webhook signature mismatch")
        return jsonify({"error": "Invalid signature"}), 401

    event = json.loads(raw_body)
    event_type = event.get("event", "")
    data = event.get("data", {})

    app.logger.info(f"Received event: {event_type}")

    if event_type == "order.filled":
        handle_order_filled(data)
    elif event_type == "order.failed":
        handle_order_failed(data)
    elif event_type == "deposit.received":
        handle_deposit_received(data)
    elif event_type == "withdrawal.completed":
        handle_withdrawal_completed(data)
    else:
        app.logger.warning(f"Unknown event type: {event_type}")

    return jsonify({"received": True}), 200


def handle_order_filled(data):
    order_id = data["order_id"]
    amount = data["amount"]
    resource = data["resource_type"]
    cost_trx = data["total_cost_sun"] / 1_000_000

    app.logger.info(f"Order {order_id} filled: {amount} {resource}, {cost_trx:.3f} TRX")

    for fill in data.get("fills", []):
        app.logger.info(
            f"  {fill['provider']}: {fill['amount']} at {fill['price_sun']} SUN"
        )

    # Your business logic here


def handle_order_failed(data):
    app.logger.warning(
        f"Order {data['order_id']} failed: {data.get('reason', 'unknown')}"
    )


def handle_deposit_received(data):
    app.logger.info(
        f"Deposit: {data['amount_trx']} {data['currency']}, "
        f"new balance: {data.get('new_balance_trx', 'N/A')} TRX"
    )


def handle_withdrawal_completed(data):
    app.logger.info(
        f"Withdrawal {data['withdrawal_id']}: "
        f"{data['amount']} {data['currency']} to {data['address']}"
    )


if __name__ == "__main__":
    app.run(port=3000)
```

Ключевые детали специфичные для Python:

- Используйте `request.get_data()` для получения сырого тела запроса как байтов.
- Используйте `hmac.compare_digest()` для сравнения строк с постоянным временем. Оператор `==` Python не использует постоянное время.
- Используйте `hmac.new()` с `hashlib.sha256` для вычисления HMAC.

## Политика повторных попыток

MERX повторяет отказанные доставки вебхука используя график экспоненциальной задержки:

| Попытка | Задержка после отказа |
|---------|---------------------|
| 1       | Немедленно           |
| 2       | 30 секунд          |
| 3       | 5 минут           |

Доставка считается отказанной если:
- Ваш сервер не отвечает в течение 10 секунд.
- Ваш сервер возвращает статус код HTTP вне диапазона 2xx.
- Соединение не может быть установлено (сбой DNS, отказ в соединении, ошибка TLS).

После 3 неудачных попыток для одного события, событие отбрасывается. Нет дальнейших повторных попыток для этой конкретной доставки.

Ваш обработчик вебхука должен:
- Отвечать с HTTP 200 в течение нескольких секунд. Выполняйте тяжёлую обработку асинхронно.
- Быть идемпотентным. Одно и то же событие может быть доставлено более одного раза в граничных случаях.
- Логировать payload события для отладки если обработка отказана.

## Автоматическая деактивация

Если эндпоинт вебхука постоянно отказывает, MERX автоматически деактивирует его чтобы не тратить ресурсы на мёртвый эндпоинт.

Порог деактивации основан на последовательных отказах для множества событий. Если ваш эндпоинт повторно не может принять доставки, флаг `is_active` вебхука устанавливается в `false`.

Когда вебхук деактивирован:
- События на этот URL больше не отправляются.
- Вебхук все ещё появляется в вашем списке с `is_active: false`.
- Вы можете исправить проблему эндпоинта и создать новый вебхук.

Мониторьте статус вебхука периодически:

```javascript
const webhooks = await merx.webhooks.list()
for (const wh of webhooks) {
  if (!wh.is_active) {
    console.warn(`Webhook ${wh.id} (${wh.url}) is deactivated`)
  }
}
```

```python
webhooks = client.webhooks.list()
for wh in webhooks:
    if not wh.is_active:
        print(f"Webhook {wh.id} ({wh.url}) is deactivated")
```

## Управление вебхуками

### Список вебхуков

```bash
curl -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/webhooks
```

### Удаление вебхука

```bash
curl -X DELETE -H "X-API-Key: sk_live_your_key_here" \
  https://merx.exchange/api/v1/webhooks/wh_abc123
```

Заметьте что секрет вебхука не может быть получен после создания. Если вы потеряли секрет, удалите вебхук и создайте новый.

## Тестирование вебхуков локально

Во время разработки URL вебхука должен быть публично доступен. Инструменты вроде ngrok могут открыть локальный сервер:

```bash
# Terminal 1: Start your webhook server
node webhook-server.js

# Terminal 2: Expose it publicly
ngrok http 3000
```

Используйте URL ngrok (например, `https://abc123.ngrok.io/webhooks/merx`) при создании вебхука. Как только вы проверите что всё работает, замените его на URL вашего production.

## Лучшие практики

1. Всегда проверяйте подписи. Никогда не доверяйте payload вебхуков без проверки заголовка `X-Merx-Signature`.

2. Отвечайте быстро. Верните HTTP 200 в течение 2-3 секунд. Поставьте в очередь тяжёлую обработку для фоновых рабочих.

3. Будьте идемпотентны. Используйте `order_id` или `deposit_id` как ключ дедупликации. Если вы получите одно и то же событие дважды, вторая обработка должна быть холостой.

4. Сохраняйте сырой payload. Логируйте полное тело JSON до обработки. Если ваш обработчик имеет баг, вы можете воспроизвести события из логов.

5. Мониторьте здоровье вебхука. Регулярно проверяйте статус `is_active`. Установите оповещения если вебхуки деактивированы.

6. Используйте HTTPS. MERX требует чтобы URL вебхуков использовали HTTPS. Самозаверяющие сертификаты не принимаются.

7. Подписывайтесь выборочно. Подписывайтесь только на события которые вы фактически обрабатываете. Ненужные события тратят пропускную способность и время обработки.

## Ресурсы

- Платформа: [merx.exchange](https://merx.exchange)
- Документация: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) | [npm](https://www.npmjs.com/package/merx-sdk)
- Python SDK: [pypi.org/project/merx-sdk](https://pypi.org/project/merx-sdk/)
- MCP Server: [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)


## Попробуйте прямо сейчас с AI

Добавьте MERX в Claude Desktop или любой совместимый MCP клиент — без установки, без API ключей для read-only инструментов:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Попросите своего AI агента: "Какова самая дешёвая TRON energy прямо сейчас?" и получите живые цены от всех подключённых поставщиков.

Полная документация MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)