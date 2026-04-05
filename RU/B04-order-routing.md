# MERX Order Routing: Как ваш ордер получает лучшую цену

Когда вы отправляете ордер на энергию в MERX, в течение миллисекунд происходит последовательность решений: у какого провайдера лучшая цена, может ли он исполнить этот ордер, что произойдет, если он не сможет, как мы проверим, что делегирование прошло в блокчейне. Это движок маршрутизации ордеров — компонент, который превращает простой вызов API в оптимизированное, отказоустойчивое и верифицированное доставку энергии.

В этой статье описывается полный жизненный цикл ордера MERX, от момента вызова `createOrder` до момента появления делегированной энергии на вашем адресе TRON.

---

## Жизненный цикл ордера

Каждый ордер проходит через шесть этапов:

```
1. Валидация    -> Правильно ли сформирован ордер?
2. Ценообразование -> Какая лучшая цена прямо сейчас?
3. Маршрутизация -> Какой провайдер(ы) исполнит заказ?
4. Исполнение    -> Отправить провайдерам
5. Верификация   -> Подтвердить делегирование в блокчейне
6. Расчет        -> Списать баланс, записать в реестр
```

Рассмотрим каждый этап.

---

## Этап 1: Валидация

Прежде чем начнется логика маршрутизации, ордер проверяется:

```typescript
// Валидация входных данных с Zod
const OrderSchema = z.object({
  energy: z.number().int().min(10000).max(100000000),
  targetAddress: z.string().refine(isValidTronAddress),
  duration: z.enum(['1h', '1d', '3d', '7d', '14d', '30d']),
  maxPrice: z.number().optional(),  // SUN на единицу энергии
  idempotencyKey: z.string().optional()
});
```

Ключевые проверки:

- **Количество энергии**: должно быть положительным целым числом в поддерживаемом диапазоне.
- **Адрес получателя**: должен быть корректным активированным адресом TRON. Система проверяет формат адреса и опционально проверяет в блокчейне, что аккаунт существует.
- **Длительность**: должна быть одним из поддерживаемых периодов делегирования.
- **Максимальная цена**: опциональный потолок. Если установлена, ордер исполняется только если лучшая доступная цена не превышает этот порог.
- **Ключ идемпотентности**: если указан, повторные отправки с тем же ключом возвращают оригинальный ордер вместо создания нового.

### Идемпотентность

Ключ идемпотентности критичен для production-интеграций. Проблемы с сетью могут заставить клиента повторить запрос, потенциально создав дублирующиеся ордеры. С ключом идемпотентности второй запрос возвращает результат первого:

```typescript
const order = await client.createOrder({
  energy: 65000,
  targetAddress: 'TBuyerAddress...',
  duration: '1h',
  idempotencyKey: 'payment-123-energy'
});

// При повторном вызове с тем же ключом возвращает тот же ордер
// Нет дублирующегося делегирования, нет двойного списания
```

---

## Этап 2: Ценообразование

Исполнитель ордера читает текущие цены из Redis-кеша. Монитор цен обновляет эти данные каждые 30 секунд, поэтому информация не старше 30 секунд.

```typescript
async function getBestPrices(
  energyAmount: number,
  duration: string
): Promise<ProviderPrice[]> {

  const allPrices = await redis.keys('prices:*');
  const validPrices = [];

  for (const key of allPrices) {
    const price = JSON.parse(await redis.get(key));

    // Фильтр: должна поддерживаться запрошенная длительность
    if (!price.durations.includes(duration)) continue;

    // Фильтр: должна быть достаточная доступность
    if (price.availableEnergy < energyAmount) continue;

    // Фильтр: должна пройти проверку здоровья
    const health = await getProviderHealth(price.provider);
    if (health.fillRate < 0.90) continue;

    validPrices.push(price);
  }

  // Сортировать по эффективной цене (с учетом надежности)
  return validPrices.sort((a, b) => {
    const effectiveA = a.energyPricePerUnit / a.fillRate;
    const effectiveB = b.energyPricePerUnit / b.fillRate;
    return effectiveA - effectiveB;
  });
}
```

### Применение максимальной цены

Если покупатель указал `maxPrice`, ордер отклоняется, если никакой провайдер не может это обеспечить:

```typescript
if (maxPrice && bestPrice.energyPricePerUnit > maxPrice) {
  throw new OrderError({
    code: 'PRICE_EXCEEDED',
    message: `Best available price (${bestPrice.energyPricePerUnit} SUN) exceeds your maximum (${maxPrice} SUN)`,
    details: { bestAvailable: bestPrice.energyPricePerUnit, maxPrice }
  });
}
```

Это предотвращает неожиданные расходы во время скачков цен.

---

## Этап 3: Маршрутизация

Движок маршрутизации решает, какой провайдер(ы) исполнит ордер. Это основной интеллект системы.

### Простой случай: Исполнение у одного провайдера

Для ордеров в пределах мощности одного провайдера:

```
Ордер: 65,000 энергии
Лучший провайдер: itrx по 85 SUN/единица, 500,000 доступно

Маршрут: 100% к itrx
```

### Раздельный случай: Исполнение у нескольких провайдеров

Для больших ордеров или когда самый дешевый провайдер имеет ограниченный запас:

```
Ордер: 500,000 энергии

Провайдер A: 200,000 доступно по 85 SUN
Провайдер B: 180,000 доступно по 87 SUN
Провайдер C: 300,000 доступно по 92 SUN

План маршрутизации:
  Часть 1: Провайдер A -> 200,000 энергии по 85 SUN
  Часть 2: Провайдер B -> 180,000 энергии по 87 SUN
  Часть 3: Провайдер C -> 120,000 энергии по 92 SUN

Смешанная цена: (200K*85 + 180K*87 + 120K*92) / 500K = 87.28 SUN
```

Маршрутизатор заполняет заказ от дешевле к дороже, беря как можно больше у каждого провайдера перед переходом к следующему.

### Цепь отказа

Каждый план маршрутизации включает цепь отказа — упорядоченный список альтернативных провайдеров для использования если основной не справится:

```
Основной:    Провайдер A (85 SUN)
Отказ 1:     Провайдер B (87 SUN)
Отказ 2:     Провайдер C (92 SUN)
Отказ 3:     Провайдер D (95 SUN)
```

Если Провайдер A не может исполнить (ошибка API, timeout, недостаточно средств), исполнитель автоматически переходит к Провайдеру B без любого действия со стороны покупателя.

---

## Этап 4: Исполнение

Исполнитель отправляет ордер выбранному провайдеру(ам) и отслеживает завершение.

### Поток исполнения

```typescript
async function executeOrder(
  order: Order,
  routingPlan: RoutingPlan
): Promise<ExecutionResult> {

  const results: LegResult[] = [];

  for (const leg of routingPlan.legs) {
    try {
      const result = await executeLeg(leg, order);
      results.push(result);
    } catch (error) {
      // Основной провайдер не справился - пробуем отказ
      const failoverResult = await executeWithFailover(
        leg,
        order,
        routingPlan.failoverChain
      );
      results.push(failoverResult);
    }
  }

  return {
    orderId: order.id,
    legs: results,
    totalEnergy: results.reduce((sum, r) => sum + r.energy, 0),
    totalCostSun: results.reduce((sum, r) => sum + r.costSun, 0)
  };
}

async function executeWithFailover(
  failedLeg: RoutingLeg,
  order: Order,
  failoverChain: Provider[]
): Promise<LegResult> {

  for (const provider of failoverChain) {
    try {
      const result = await executeLeg(
        { ...failedLeg, provider: provider.name },
        order
      );
      return result;
    } catch (error) {
      // Логируем и переходим к следующему отказу
      continue;
    }
  }

  throw new OrderError({
    code: 'ALL_PROVIDERS_FAILED',
    message: 'Order could not be filled by any available provider'
  });
}
```

### Обработка timeout

Каждое исполнение у провайдера имеет строгий timeout. Если провайдер не подтверждает ордер в течение временного окна (обычно 30 секунд), исполнитель переходит к цепи отказов:

```
T+0:    Отправить ордер Провайдеру A
T+30s:  Нет ответа -> timeout, отказ к Провайдеру B
T+31s:  Отправить ордер Провайдеру B
T+35s:  Провайдер B подтверждает -> переходим к верификации
```

---

## Этап 5: Верификация

Подтверждение от провайдера недостаточно. MERX проверяет, что делегирование энергии действительно появилось в блокчейне TRON.

### Процесс верификации в блокчейне

```typescript
async function verifyDelegation(
  targetAddress: string,
  expectedEnergy: number,
  delegationTxHash: string
): Promise<VerificationResult> {

  // Шаг 1: Проверить что транзакция делегирования существует
  const tx = await tronWeb.trx.getTransaction(delegationTxHash);
  if (!tx || tx.ret[0].contractRet !== 'SUCCESS') {
    return { verified: false, reason: 'Transaction not found or failed' };
  }

  // Шаг 2: Проверить ресурсы адреса получателя
  const resources = await tronWeb.trx.getAccountResources(targetAddress);
  const currentEnergy = resources.EnergyLimit || 0;

  // Шаг 3: Проверить что энергия увеличилась на ожидаемый размер (с допуском)
  const tolerance = expectedEnergy * 0.02; // 2% допуска
  if (currentEnergy < expectedEnergy - tolerance) {
    return { verified: false, reason: 'Energy amount below expected' };
  }

  return {
    verified: true,
    txHash: delegationTxHash,
    energyDelivered: currentEnergy,
    verifiedAt: new Date()
  };
}
```

### Почему 2% допуска

Объемы делегирования энергии основаны на коэффициентах конверсии TRX-в-энергию, которые могут слегка измениться между размещением ордера и подтверждением делегирования. 2% допуска учитывает это без приема явно неправильных размеров.

### Время верификации

```
T+0:    Провайдер подтверждает ордер
T+3-6s: Транзакция делегирования подтверждена в TRON (1 блок)
T+10s:  MERX запрашивает ресурсы адреса получателя
T+10s:  Верификация завершена, покупатель уведомлен
```

Весь процесс от отправки ордера до верифицированной доставки обычно занимает 15-45 секунд.

---

## Этап 6: Расчет

После верификации происходит финансовый расчет:

### Списание баланса

```sql
-- Атомарная проверка и списание баланса
BEGIN;

SELECT balance_sun FROM accounts
WHERE user_id = $1
FOR UPDATE;

-- Проверить достаточность баланса
-- Если недостаточно, ROLLBACK и вернуть ошибку

UPDATE accounts
SET balance_sun = balance_sun - $2
WHERE user_id = $1;

COMMIT;
```

`SELECT FOR UPDATE` гарантирует отсутствие race condition между проверкой баланса и его списанием. Если два ордера обрабатываются одновременно, второй будет ждать завершения первого перед проверкой баланса.

### Запись в реестр

Каждый расчет создает неизменяемую запись в реестре:

```sql
INSERT INTO ledger (
  user_id, type, amount_sun,
  reference_type, reference_id,
  balance_before, balance_after,
  created_at
) VALUES (
  $1, 'ORDER_PAYMENT', $2,
  'order', $3,
  $4, $5,
  NOW()
);
```

Записи в реестре добавляются только в конец. Они никогда не обновляются и не удаляются. Это создает полную, проверяемую историю каждой финансовой операции.

---

## Отслеживание вашего ордера

API MERX предоставляет статус ордера в реальном времени через REST и WebSocket:

```typescript
import { MerxClient } from 'merx-sdk';

const client = new MerxClient({ apiKey: 'your-key' });

// REST: опрос статуса ордера
const order = await client.getOrder('ord_abc123');
console.log(order.status);
// 'pending' | 'executing' | 'verifying' | 'completed' | 'failed'

// WebSocket: обновления в реальном времени
client.onOrderUpdate('ord_abc123', (update) => {
  console.log(`Ордер ${update.orderId}: ${update.status}`);
  if (update.status === 'completed') {
    console.log(`Транзакция делегирования: ${update.delegationTxHash}`);
  }
});
```

### Поток статусов ордера

```
pending -> executing -> verifying -> completed
              |                        |
              v                        v
            failed              partially_filled
```

- **pending**: Ордер получен, ожидание исполнения.
- **executing**: Отправлен провайдеру, ожидание подтверждения.
- **verifying**: Провайдер подтвердил, ожидание подтверждения в блокчейне.
- **completed**: Энергия верифицирована на адресе получателя.
- **failed**: Все провайдеры не справились. Баланс не списан.
- **partially_filled**: Некоторые части исполнены, другие не справились. Баланс списан только за исполненную часть.

---

## Обработка ошибок

### Ошибки провайдеров

Каждый провайдер может не справиться по-своему. Исполнитель ордера нормализует все ошибки провайдера в стандартные коды ошибок:

```json
{
  "error": {
    "code": "PROVIDER_UNAVAILABLE",
    "message": "Primary provider could not fill the order. Failover to secondary provider.",
    "details": {
      "primaryProvider": "tronsave",
      "primaryError": "timeout",
      "filledBy": "feee",
      "priceImpact": "+2 SUN/unit"
    }
  }
}
```

### Частичное исполнение

Для ордеров с несколькими частями, некоторые части могут исполниться, а другие не справиться. MERX обрабатывает это:

1. Исполняет успешные части нормально.
2. Пытается отказ для неудачных частей.
3. Если отказ также не справляется, возвращает результат частичного исполнения.
4. Списывает с покупателя только энергию, которая была действительно доставлена.

---

## Производительность

Типичное время исполнения ордера:

```
Валидация:     < 5ms
Ценообразование: < 10ms (чтение Redis)
Маршрутизация:  < 5ms
Исполнение:     5-30 секунд (API провайдера + блокчейн)
Верификация:    3-10 секунд (подтверждение блокчейна)

Итого: 10-45 секунд от вызова API до верифицированной доставки
```

Узкое место всегда в блокчейне. Внутренняя обработка MERX добавляет менее 50ms накладных расходов. Остальное — ожидание провайдера и сети TRON.

---

## Заключение

Маршрутизация ордеров — это то, где ценность MERX наиболее ощутима. Один вызов API запускает каскад оптимизаций: выбор лучшей цены, раздельное исполнение у нескольких провайдеров, автоматический отказ, верификация в блокчейне и атомарный расчет. Каждый шаг разработан для обеспечения получения вами самой дешевой доступной энергии, надежной доставки с полной проверяемостью.

Сложность реальна, но это сложность MERX для управления, а не ваша. Ваша интеграция остается одним вызовом API.

Начните маршрутизировать ордеры на [https://merx.exchange](https://merx.exchange). Полная документация API на [https://merx.exchange/docs](https://merx.exchange/docs).

---

*Эта статья является частью технической серии MERX. MERX — первый блокчейн-обмен ресурсами. SDK: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js) и [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python). MCP-сервер: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp).*

## Попробуйте прямо сейчас с AI

Добавьте MERX в Claude Desktop или любого MCP-совместимого клиента — без установки, без API-ключа для инструментов только для чтения:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Попросите вашего AI-помощника: "What is the cheapest TRON energy right now?" и получите текущие цены от всех подключенных провайдеров.

Полная документация MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)