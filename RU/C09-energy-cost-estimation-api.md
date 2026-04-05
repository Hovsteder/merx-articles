# API оценки стоимости энергии MERX: узнайте стоимость перед покупкой

Каждая транзакция TRON потребляет ресурсы. Передача USDT требует примерно 65 000 единиц энергии. Одобрение смарт-контракта сжигает около 15 000. Сложное DeFi взаимодействие может потребовать 200 000 или больше. Точные числа зависят от контракта, операции и текущих параметров сети.

Если вы разрабатываете продукт, который обрабатывает транзакции TRON от имени пользователей — кошелек, платежный процессор, торговый бот — вам нужно знать стоимость перед тем, как взять на себя обязательство. Сколько энергии требуется? Какова будет стоимость аренды в сравнении со сжиганием? Сколько экономит пользователь?

MERX предоставляет два endpoint'а, которые точно ответят на эти вопросы: `POST /api/v1/estimate` для общей оценки стоимости и `GET /api/v1/orders/preview` для предпросмотра стоимости конкретного заказа. Вместе они позволяют показать пользователям точные затраты и экономию до того, как изменится хотя бы один SUN.

## Endpoint оценки

`POST /api/v1/estimate` вычисляет требования по энергии и bandwidth для определённого типа транзакции и возвращает сравнение стоимости между арендой энергии через MERX и сжиганием TRX на уровне протокола.

### Базовый запрос

```bash
curl -X POST https://merx.exchange/api/v1/estimate \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "trc20_transfer",
    "target_address": "TTargetAddressHere"
  }'
```

### Формат ответа

```json
{
  "operation": "trc20_transfer",
  "target_address": "TTargetAddressHere",
  "energy_required": 64895,
  "bandwidth_required": 345,
  "costs": {
    "burn": {
      "trx_cost": 27370000,
      "trx_cost_readable": "27.37 TRX",
      "usd_equivalent": 2.19
    },
    "rental": {
      "best_provider": "sohu",
      "price_per_unit_sun": 22,
      "total_cost_sun": 1427690,
      "total_cost_trx": "1.43 TRX",
      "usd_equivalent": 0.11,
      "duration_hours": 1
    },
    "savings": {
      "trx_saved": "25.94 TRX",
      "percent": 94.8,
      "usd_saved": 2.08
    }
  },
  "address_resources": {
    "current_energy": 0,
    "current_bandwidth": 1200,
    "energy_deficit": 64895,
    "bandwidth_deficit": 0
  },
  "timestamp": "2026-03-30T10:00:00Z"
}
```

Ответ содержит всё необходимое для принятия решения о затратах на транзакцию:

- **energy_required** — сколько энергии требуется для операции. Для стандартной передачи TRC-20 это примерно 65 000 единиц, хотя точное число зависит от контракта и состояния целевого адреса.
- **bandwidth_required** — сколько bandwidth использует транзакция. Большинство простых передач требуют 300–400 пунктов bandwidth. Новые аккаунты (активация адреса в первый раз) требуют больше.
- **costs.burn** — какова стоимость, если пользователь ничего не делает и позволяет протоколу сжечь TRX. Это «стандартная» стоимость.
- **costs.rental** — самый дешёвый доступный вариант аренды через MERX. Включает провайдера, цену за единицу, общую стоимость и длительность аренды.
- **costs.savings** — разница между сжиганием и арендой, выраженная как абсолютный сэкономленный TRX, сэкономленный процент и эквивалент в USD.
- **address_resources** — текущая энергия и bandwidth на целевом адресе. Если адрес уже имеет энергию от стейкинга или предыдущего делегирования, дефицит уменьшается соответственно.

### Поддерживаемые операции

Поле `operation` принимает несколько предопределённых типов, охватывающих наиболее распространённые транзакции TRON:

#### trc20_transfer

Стандартная передача токена TRC-20. Это наиболее распространённая операция — отправка USDT, USDC или любого другого токена TRC-20 с одного адреса на другой.

```json
{
  "operation": "trc20_transfer",
  "target_address": "TSenderAddressHere"
}
```

Требуемая энергия: примерно 64 895 единиц для USDT при стандартной передаче (адрес уже активирован с балансом USDT). Первые передачи на адрес, который никогда не владел токеном, требуют больше — до 100 000 энергии — потому что контракт должен создать новый слот хранилища.

#### trc20_approve

Транзакция одобрения TRC-20, используется для разрешения смарт-контракту тратить токены от вашего имени. Требуется перед взаимодействием с контрактами DEX, протоколами кредитования и большинством DeFi приложений.

```json
{
  "operation": "trc20_approve",
  "target_address": "TApproverAddressHere"
}
```

Требуемая энергия: примерно 15 000–18 000 единиц, значительно меньше, чем при передаче.

#### trx_transfer

Простая передача TRX. Эти операции в основном потребляют bandwidth, а не энергию, но endpoint оценки обрабатывает их для полноты.

```json
{
  "operation": "trx_transfer",
  "target_address": "TSenderAddressHere"
}
```

Требуемая энергия: 0 (передачи TRX не потребляют энергию). Требуемый bandwidth: примерно 270 байт.

#### custom

Для вызовов смарт-контрактов, которые не подходят к предопределённым типам. Предоставьте адрес контракта и селектор функции, и MERX оценит потребление энергии путём симуляции вызова.

```json
{
  "operation": "custom",
  "target_address": "TCallerAddressHere",
  "contract_address": "TContractAddressHere",
  "function_selector": "stake(uint256)",
  "parameters": [
    {
      "type": "uint256",
      "value": "1000000"
    }
  ]
}
```

Операция custom запускает симуляцию против сети TRON для определения фактического потребления энергии и bandwidth. Это наиболее точный метод для нестандартных транзакций, но занимает немного больше времени (200–500 мс в сравнении с 50 мс для предопределённых типов).

## Endpoint предпросмотра заказа

Хотя `POST /estimate` даёт вам требования к ресурсам и сравнение затрат, `GET /api/v1/orders/preview` показывает вам точный вид заказа MERX — включая то, какой провайдер будет выбран, точный расход из вашего баланса и любые применяемые сборы.

### Запрос

```bash
curl "https://merx.exchange/api/v1/orders/preview?energy_amount=65000&target_address=TTargetAddressHere&duration_hours=1" \
  -H "X-API-Key: your_api_key"
```

### Ответ

```json
{
  "preview": {
    "energy_amount": 65000,
    "target_address": "TTargetAddressHere",
    "duration_hours": 1,
    "provider": "sohu",
    "price_per_unit_sun": 22,
    "subtotal_sun": 1430000,
    "fee_sun": 14300,
    "total_sun": 1444300,
    "total_trx": "1.44 TRX",
    "your_balance_sun": 50000000,
    "balance_after_sun": 48555700,
    "estimated_fill_time_seconds": 5
  },
  "alternatives": [
    {
      "provider": "catfee",
      "price_per_unit_sun": 25,
      "total_sun": 1641250,
      "total_trx": "1.64 TRX"
    },
    {
      "provider": "netts",
      "price_per_unit_sun": 28,
      "total_sun": 1838200,
      "total_trx": "1.84 TRX"
    }
  ]
}
```

Предпросмотр включает:

- **provider** — какому провайдеру MERX маршрутизировала бы заказ при текущих ценах.
- **subtotal_sun** — чистая стоимость энергии по ставке провайдера.
- **fee_sun** — комиссия платформы MERX.
- **total_sun** — общая сумма, которая будет снята с вашего баланса.
- **balance_after_sun** — ваш баланс после заказа, чтобы вы могли проверить возможность.
- **estimated_fill_time_seconds** — как долго обычно длится делегирование у этого провайдера.
- **alternatives** — другие провайдеры, которые могли бы выполнить заказ, отсортированные по цене. Полезно для показа пользователям их вариантов.

### Разница между Estimate и Preview

| Функция | POST /estimate | GET /orders/preview |
|---------|---|---|
| Назначение | Общий анализ стоимости | Точное планирование заказа |
| Требуется авторизация | Да | Да |
| Показывает экономию vs сжигание | Да | Нет |
| Показывает ваш баланс | Нет | Да |
| Показывает комиссии платформы | Нет | Да |
| Показывает альтернативы | Нет | Да |
| Показывает ресурсы адреса | Да | Нет |
| Поддерживает пользовательские контракты | Да | Нет |

Используйте `estimate`, когда нужно рассказать пользователю: «эта транзакция будет стоить X, если вы сожжёте TRX, или Y, если вы арендуете энергию — экономя Z процентов». Используйте `preview`, когда пользователь решил купить энергию и вы хотите показать точный заказ перед подтверждением.

## Примеры из реальной жизни с числами

### Пример 1: Обработчик платежей USDT

Вы управляете обработчиком платежей, который отправляет USDT торговцам. Перед каждой выплатой вы оцениваете стоимость:

```javascript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
});

async function estimatePayoutCost(senderAddress) {
  const estimate = await merx.estimate({
    operation: 'trc20_transfer',
    target_address: senderAddress,
  });

  console.log(`Energy needed: ${estimate.energy_required}`);
  console.log(`Burn cost: ${estimate.costs.burn.trx_cost_readable}`);
  console.log(`Rental cost: ${estimate.costs.rental.total_cost_trx}`);
  console.log(`Savings: ${estimate.costs.savings.percent}%`);

  // Decide whether to rent or burn based on savings threshold
  if (estimate.costs.savings.percent > 50) {
    return { method: 'rent', cost: estimate.costs.rental };
  } else {
    return { method: 'burn', cost: estimate.costs.burn };
  }
}
```

Типичный результат:

```
Energy needed: 64895
Burn cost: 27.37 TRX
Rental cost: 1.43 TRX
Savings: 94.8%
```

При текущих ценах аренда энергии почти всегда экономит более 90 процентов в сравнении со сжиганием.

### Пример 2: Отображение стоимости в кошельке

Вы разрабатываете кошелек TRON и хотите показать пользователю стоимость транзакции перед подтверждением отправки:

```python
from merx_sdk import MerxClient

client = MerxClient(api_key="your_api_key")


def get_transfer_cost(sender_address: str, token: str = "usdt") -> dict:
    """Get the cost to send a TRC-20 token, accounting for existing resources."""

    estimate = client.estimate(
        operation="trc20_transfer",
        target_address=sender_address,
    )

    resources = estimate["address_resources"]
    costs = estimate["costs"]

    # If the address already has enough energy, the transfer is free
    if resources["energy_deficit"] == 0:
        return {
            "cost": "0 TRX",
            "note": "Address has sufficient energy",
        }

    return {
        "without_energy": costs["burn"]["trx_cost_readable"],
        "with_energy": costs["rental"]["total_cost_trx"],
        "savings": f'{costs["savings"]["percent"]}%',
        "energy_deficit": resources["energy_deficit"],
    }
```

Интерфейс кошелька может отображать примерно следующее:

```
Send 100 USDT to TRecipient...

Transaction cost:
  Without energy rental:  27.37 TRX (burned)
  With MERX energy:       1.43 TRX (rented)
  You save:               94.8%

[ Rent Energy and Send ]    [ Send Without Energy ]
```

### Пример 3: Прогноз стоимости массовой передачи

Вам нужно отправить USDT на 500 адресов и вы хотите узнать общую стоимость заранее:

```javascript
async function estimateBatchCost(addresses) {
  let totalBurnCost = 0;
  let totalRentalCost = 0;
  let totalEnergy = 0;

  // Estimate for a representative sample (first 10 addresses)
  // Most TRC-20 transfers cost the same energy
  const sampleSize = Math.min(10, addresses.length);
  const sample = addresses.slice(0, sampleSize);

  for (const address of sample) {
    const estimate = await merx.estimate({
      operation: 'trc20_transfer',
      target_address: address,
    });

    totalBurnCost += estimate.costs.burn.trx_cost;
    totalRentalCost += estimate.costs.rental.total_cost_sun;
    totalEnergy += estimate.energy_required;
  }

  // Extrapolate to full batch
  const avgBurnCost = totalBurnCost / sampleSize;
  const avgRentalCost = totalRentalCost / sampleSize;
  const avgEnergy = totalEnergy / sampleSize;

  const projectedBurnTRX = (avgBurnCost * addresses.length) / 1_000_000;
  const projectedRentalTRX = (avgRentalCost * addresses.length) / 1_000_000;
  const projectedEnergy = avgEnergy * addresses.length;

  console.log(`Batch size: ${addresses.length} transfers`);
  console.log(`Total energy needed: ${projectedEnergy.toLocaleString()}`);
  console.log(`Cost without MERX: ${projectedBurnTRX.toFixed(2)} TRX`);
  console.log(`Cost with MERX: ${projectedRentalTRX.toFixed(2)} TRX`);
  console.log(
    `Total savings: ${(projectedBurnTRX - projectedRentalTRX).toFixed(2)} TRX`
  );
}
```

Для 500 передач при текущих рыночных ставках:

```
Batch size: 500 transfers
Total energy needed: 32,447,500
Cost without MERX: 13,685.00 TRX
Cost with MERX: 715.00 TRX
Total savings: 12,970.00 TRX
```

При примерно 0,08 USD за TRX это экономия более 1000 USD на одной партии.

### Пример 4: Оценка вызова пользовательского контракта

Вы взаимодействуете с пользовательским контрактом стейкинга и нужно оценить стоимость вызова `stake()`:

```bash
curl -X POST https://merx.exchange/api/v1/estimate \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "custom",
    "target_address": "TCallerAddressHere",
    "contract_address": "TStakingContractHere",
    "function_selector": "stake(uint256)",
    "parameters": [
      {
        "type": "uint256",
        "value": "50000000"
      }
    ]
  }'
```

Ответ включает смоделированное потребление энергии, которое для сложного контракта стейкинга может быть 120 000–200 000 единиц — значительно больше, чем при простой передаче.

## Интеграция оценок в поток заказов

Endpoints оценки и предпросмотра естественно встраиваются в ориентированный на пользователя процесс заказов:

1. **Пользователь инициирует транзакцию** (например, отправка USDT).
2. **Вызовите POST /estimate** для получения требований по энергии и сравнения затрат.
3. **Отобразите стоимость сжигания в сравнении со стоимостью аренды** с процентом экономии.
4. **Пользователь выбирает аренду энергии.**
5. **Вызовите GET /orders/preview** для отображения точных деталей заказа с комиссиями.
6. **Пользователь подтверждает.**
7. **Вызовите POST /orders** с ключом идемпотентности для создания заказа.
8. **Опрашивайте или ожидайте вебхука**, подтверждающего делегирование энергии.
9. **Выполните исходную транзакцию** с делегированной энергией.

Шаги 2–3 информационные. Деньги не движутся. Пользователь видит прозрачное ценообразование перед тем, как взять обязательство. Это строит доверие и сокращает поддержку запросов от пользователей, удивлённых затратами.

## Обработка ошибок

Оба endpoints возвращают стандартные ошибочные ответы MERX:

```json
{
  "error": {
    "code": "INVALID_ADDRESS",
    "message": "Target address is not a valid TRON address",
    "details": {
      "address": "invalid_address_here"
    }
  }
}
```

Распространённые ошибки:

| Код | Причина | Решение |
|---|---|---|
| `INVALID_ADDRESS` | Целевой адрес не прошёл проверку валидности адреса TRON | Проверьте формат адреса (префикс T, base58) |
| `INVALID_OPERATION` | Неизвестный тип операции | Используйте один из: trc20_transfer, trc20_approve, trx_transfer, custom |
| `SIMULATION_FAILED` | Симуляция вызова пользовательского контракта не удалась | Проверьте адрес контракта и селектор функции |
| `NO_PROVIDERS` | Нет доступных провайдеров для требуемого количества энергии | Повторите попытку позже или уменьшите количество энергии |
| `INSUFFICIENT_BALANCE` | Баланс слишком низок для предпросмотренного заказа (только preview) | Пополните свой счёт MERX |

## Соображения по кэшированию

Результаты оценки действительны в течение короткого периода. Цены на энергию меняются по мере того, как провайдеры корректируют ставки, и параметры сети могут сдвинуть стоимость сжигания. Для большинства случаев:

- **Кэшируйте оценки на 30–60 секунд**, если вы отображаете затраты в интерфейсе. Цены не меняются быстрее, чем MERX опрашивает провайдеров (каждые 30 секунд).
- **Всегда получайте свежий предпросмотр** непосредственно перед созданием заказа. Предпросмотр отражает точную стоимость в этот момент.
- **Не кэшируйте симуляции пользовательских контрактов**, если состояние контракта часто меняется. Результат симуляции зависит от состояния в сети на момент выполнения.

## Заключение

Endpoints оценки и предпросмотра убирают неопределённость из покупки энергии TRON. Вместо аренды фиксированного количества энергии и надежды, что этого будет достаточно, вы знаете ровно столько, сколько вам нужно. Вместо принятия любой доступной цены вы видите каждый вариант, отранжированный по стоимости.

Для разработчиков, создающих продукты на TRON, эти endpoints преобразуют энергию из непредсказуемой стоимости в известный, оптимизируемый элемент. Проверьте стоимость, покажите экономию, позвольте пользователю решить, затем выполняйте с уверенностью.

- Платформа MERX: [merx.exchange](https://merx.exchange)
- Полная документация API: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- Python SDK: [github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)


## Попробуйте прямо сейчас с AI

Добавьте MERX в Claude Desktop или любого MCP-совместимого клиента — нулевая установка, ключ API не требуется для инструментов только для чтения:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Спросите у вашего AI агента: «Какова самая дешёвая энергия TRON прямо сейчас?» и получайте живые цены от всех подключённых провайдеров.

Полная документация MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)