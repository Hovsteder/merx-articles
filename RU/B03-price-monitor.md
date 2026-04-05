# Монитор цен MERX: как мы отслеживаем каждого поставщика каждые 30 секунд

Монитор цен — это сердце MERX. Каждые 30 секунд он обращается к каждому интегрированному поставщику энергии, получает их текущие цены, нормализует данные и публикует их в остальную систему. Без него маршрутизация по лучшей цене была бы угадыванием. С ним каждый заказ маршрутизируется к самому дешевому доступному поставщику на основе данных, не старше 30 секунд.

Эта статья — глубокое техническое погружение в архитектуру монитора цен: как он опрашивает поставщиков, как паттерн адаптера обеспечивает расширяемость системы, как Redis pub/sub распределяет данные о ценах в реальном времени и как история цен питает аналитику и принятие решений.

---

## Почему 30 секунд

Интервал опроса — это сознательный выбор проектирования. Цены на энергию в TRON не меняются каждую секунду — это не спот-форекс или крипто-биржи. Цены поставщиков обычно меняются несколько раз в час, а иногда и реже. Интервал в 30 секунд захватывает все значимые изменения цен, избегая при этом нескольких проблем:

- **Ограничения по частоте API поставщиков**: большинство поставщиков разрешают 1-2 запроса в секунду. При интервалах в 30 секунд мы остаемся значительно в рамках лимитов, даже с повторными попытками.
- **Сетевые затраты**: опрос 7+ поставщиков создает HTTP-трафик. При 30 секундах это незначительно. При 1 секунде это было бы существенно.
- **Свежесть данных vs шум**: изменения цен на энергию быстрее чем за 30 секунд почти всегда являются шумом, а не сигналом. 30 секунд отфильтровывают шум, захватывая настоящие движения.
- **Использование ресурсов системы**: монитор цен работает наряду с другими сервисами. Агрессивный опрос конкурировал бы за CPU и память без добавления ценности.

TTL для кэшированных цен установлен на 60 секунд — в два раза больше интервала опроса. Если опрос не удается, предыдущая цена остается действительной еще один цикл, прежде чем истечь. Это предотвращает удаление поставщика из книги ордеров из-за одного неудачного опроса.

---

## Паттерн адаптера

Каждый поставщик энергии имеет другой API. Разные конечные точки, разные методы аутентификации, разные форматы ответов, разные коды ошибок. Монитор цен использует паттерн адаптера, чтобы изолировать эти различия от основной логики опроса.

### Интерфейс поставщика

Каждый адаптер поставщика реализует общий интерфейс:

```typescript
interface IEnergyProvider {
  name: string;

  // Fetch current pricing
  getPrices(): Promise<ProviderPriceResponse>;

  // Check if provider is operational
  healthCheck(): Promise<boolean>;

  // Get available inventory
  getAvailability(): Promise<AvailabilityResponse>;
}

interface ProviderPriceResponse {
  energyPricePerUnit: number;   // SUN per energy unit
  bandwidthPricePerUnit: number;
  minOrder: number;
  maxOrder: number;
  durations: Duration[];
  timestamp: number;
}
```

### Адаптер поставщика

Каждый поставщик получает свой файл адаптера. Вот упрощенный пример того, как выглядит адаптер поставщика:

```typescript
// providers/tronsave/index.ts
import { IEnergyProvider, ProviderPriceResponse } from '../base';

export class TronSaveProvider implements IEnergyProvider {
  name = 'tronsave';
  private apiUrl: string;
  private apiKey: string;

  constructor(config: ProviderConfig) {
    this.apiUrl = config.apiUrl;
    this.apiKey = config.apiKey;
  }

  async getPrices(): Promise<ProviderPriceResponse> {
    const response = await fetch(`${this.apiUrl}/v1/prices`, {
      headers: { 'Authorization': `Bearer ${this.apiKey}` }
    });

    const data = await response.json();

    // Normalize provider-specific format to standard format
    return {
      energyPricePerUnit: this.normalizePrice(data.energy_price),
      bandwidthPricePerUnit: this.normalizePrice(data.bandwidth_price),
      minOrder: data.min_energy || 10000,
      maxOrder: data.max_energy || 10000000,
      durations: this.normalizeDurations(data.available_durations),
      timestamp: Date.now()
    };
  }

  async healthCheck(): Promise<boolean> {
    try {
      const response = await fetch(`${this.apiUrl}/health`);
      return response.ok;
    } catch {
      return false;
    }
  }

  // ... provider-specific normalization methods
}
```

### Добавление нового поставщика

Одно из ключевых преимуществ этой архитектуры заключается в том, что добавление нового поставщика требует ровно один новый файл:

1. Создайте `providers/newprovider/index.ts`, реализующий `IEnergyProvider`.
2. Зарегистрируйте поставщика в конфигурации.
3. Монитор цен автоматически начинает его опрашивать.
4. Исполнитель ордеров автоматически включает его в решения маршрутизации.

Никаких изменений в мониторе цен, никаких изменений в исполнителе ордеров, никаких изменений в API. Паттерн адаптера гарантирует, что сложность, специфичная для поставщика, инкапсулирована.

---

## Цикл опроса

Основной цикл монитора цен простой, но тщательно разработан для надежности:

```typescript
class PriceMonitor {
  private providers: IEnergyProvider[];
  private redis: RedisClient;
  private pollInterval = 30_000; // 30 seconds

  async start() {
    // Initial poll on startup
    await this.pollAll();

    // Schedule recurring polls
    setInterval(() => this.pollAll(), this.pollInterval);
  }

  async pollAll() {
    const results = await Promise.allSettled(
      this.providers.map(provider => this.pollProvider(provider))
    );

    // Compute best price from successful results
    const validPrices = results
      .filter(r => r.status === 'fulfilled')
      .map(r => r.value);

    if (validPrices.length > 0) {
      await this.updateBestPrice(validPrices);
    }
  }

  async pollProvider(provider: IEnergyProvider) {
    const startTime = Date.now();

    try {
      const prices = await provider.getPrices();
      const responseTime = Date.now() - startTime;

      // Store in Redis with 60s TTL
      await this.redis.setex(
        `prices:${provider.name}`,
        60,
        JSON.stringify(prices)
      );

      // Publish price update event
      await this.redis.publish(
        'price-updates',
        JSON.stringify({
          provider: provider.name,
          prices,
          responseTime
        })
      );

      // Update health metrics
      await this.updateHealthMetrics(provider.name, {
        success: true,
        responseTime,
        timestamp: Date.now()
      });

      return { provider: provider.name, prices };

    } catch (error) {
      await this.updateHealthMetrics(provider.name, {
        success: false,
        error: error.message,
        timestamp: Date.now()
      });

      throw error; // Let Promise.allSettled handle it
    }
  }
}
```

### Ключевые решения проектирования

**Promise.allSettled, а не Promise.all**: сбой одного поставщика не должен блокировать обновления от других поставщиков. `allSettled` гарантирует, что каждый поставщик опрашивается независимо.

**60-секундный TTL**: если поставщик не отвечает два цикла подряд (60 секунд), его кэшированная цена автоматически истекает. Исполнитель ордеров не будет маршрутизировать на поставщика без кэшированной цены.

**Метрики здоровья наряду с ценами**: каждый опрос записывает время ответа и успех/неудачу. Эти данные питают алгоритм маршрутизации с оценкой надежности.

---

## Распределение Redis Pub/Sub

Монитор цен не служит цены прямо потребителям API. Вместо этого он публикует в Redis, и другие сервисы подписываются на необходимые им обновления.

### Структура каналов

```
Channel: price-updates
  -> Все события обновления цен (потребляется сервисом API, трансляция WebSocket)

Channel: price-alerts
  -> Значительные изменения цен (потребляется сервисом уведомлений)

Channel: provider-health
  -> Изменения статуса здоровья (потребляется панелью администратора)
```

### Почему Pub/Sub вместо прямых вызовов

Монитор цен и сервис API — это отдельные процессы (отдельные контейнеры Docker, собственно говоря). Они общаются исключительно через Redis — никаких прямых импортов, никаких вызовов функций, никаких общей памяти. Эта изоляция означает:

- Монитор цен может быть перезагружен без влияния на API.
- API может масштабироваться горизонтально (несколько экземпляров) и все получат одинаковые обновления цен.
- Ошибка в мониторе цен не может сбить сервис API.
- Каждый сервис может быть развернут независимо.

### Трансляция WebSocket

Сервис API подписывается на канал `price-updates` и транслирует обновления подключенным клиентам WebSocket:

```
Price Monitor -> Redis pub/sub -> API Service -> WebSocket -> Client

Latency: ~5ms from provider response to client notification
```

Клиенты, подписанные на веб-сокет канал, получают обновления цен в режиме реального времени, что позволяет использовать живые панели мониторинга и отзывчивые интерфейсы торговли.

---

## Хранение истории цен

Каждая точка данных о ценах хранится в PostgreSQL для исторического анализа. Схема захватывает полный снимок цены:

```sql
CREATE TABLE price_history (
  id BIGSERIAL PRIMARY KEY,
  provider VARCHAR(50) NOT NULL,
  energy_price_sun BIGINT NOT NULL,
  bandwidth_price_sun BIGINT NOT NULL,
  min_order INTEGER,
  max_order INTEGER,
  available_energy BIGINT,
  response_time_ms INTEGER,
  recorded_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_price_history_provider_time
  ON price_history(provider, recorded_at DESC);
```

### Объем данных

При интервалах в 30 секунд в течение 7 поставщиков:

```
7 providers x 2 polls/minute x 60 minutes x 24 hours = 20,160 rows/day
Monthly: ~604,800 rows
Yearly: ~7,257,600 rows
```

Каждая строка небольшая (примерно 100 байт), поэтому годовое хранилище составляет менее 1 ГБ. PostgreSQL тривиально обрабатывает этот объем с соответствующей индексацией.

### Аналитические запросы

История цен позволяет провести несколько ценных анализов:

```sql
-- Average price by provider over the last 24 hours
SELECT provider,
       AVG(energy_price_sun) as avg_price,
       MIN(energy_price_sun) as min_price,
       MAX(energy_price_sun) as max_price
FROM price_history
WHERE recorded_at > NOW() - INTERVAL '24 hours'
GROUP BY provider
ORDER BY avg_price;

-- Price trend for a specific provider
SELECT date_trunc('hour', recorded_at) as hour,
       AVG(energy_price_sun) as avg_price
FROM price_history
WHERE provider = 'tronsave'
  AND recorded_at > NOW() - INTERVAL '7 days'
GROUP BY hour
ORDER BY hour;
```

Эти запросы питают конечную точку API истории цен MERX и панель администратора.

---

## Обработка граничных случаев

### Поставщик возвращает неверные данные

Монитор цен проверяет каждый ответ перед кэшированием:

```typescript
function validatePrice(price: ProviderPriceResponse): boolean {
  // Price must be positive
  if (price.energyPricePerUnit <= 0) return false;

  // Price must be within reasonable bounds (10-500 SUN)
  if (price.energyPricePerUnit < 10_000_000) return false;  // < 10 SUN
  if (price.energyPricePerUnit > 500_000_000) return false;  // > 500 SUN

  // Must have valid timestamp
  if (price.timestamp > Date.now() + 60_000) return false;  // future
  if (price.timestamp < Date.now() - 300_000) return false;  // > 5min old

  return true;
}
```

Неверные данные регистрируются и отклоняются. Предыдущая действительная цена остается в кэше, пока она не истекает естественным образом.

### Изменения API поставщика

API поставщиков иногда меняются — новые поля, устаревшие конечные точки, измененные форматы ответов. Поскольку каждый поставщик имеет собственный адаптер, изменения API изолированы в одном файле. Адаптер обновляется, тестируется и развертывается без касания какой-либо другой части системы.

### Сетевые разделения

Если монитор цен теряет сетевое соединение, все опросы поставщиков одновременно терпят неудачу. 60-секундный TTL гарантирует, что кэшированные цены истекают в течение минуты, и исполнитель ордеров перестает маршрутизировать на всех поставщиков. При восстановлении соединения следующий цикл опроса автоматически повторно заполняет кэш.

### Дрейф часов

Монитор цен работает на одном сервере, поэтому дрейф часов между сервисами не является проблемой для относительного времени. Временные метки в истории цен используют `NOW()` из PostgreSQL, обеспечивая согласованность. Для абсолютных временных меток в ответах API сервер работает с NTP.

---

## Мониторинг монитора

Сам монитор цен контролируется несколькими механизмами:

- **Конечная точка здоровья**: сервис предоставляет конечную точку `/health`, которая сообщает последнее успешное время опроса для каждого поставщика.
- **Alerting**: если не было успешного обновления цен в течение 5 минут, запускается предупреждение.
- **Метрики**: количество опросов, частота успехов, среднее время ответа и частота попадания кэша отслеживаются.
- **Логи**: каждый результат опроса (успех или неудача) регистрируется со структурированным JSON для анализа.

---

## Характеристики производительности

```
Typical poll cycle (7 providers):
  Total time: 1-3 seconds (parallel HTTP requests)
  Redis writes: 8 (7 provider prices + 1 best price)
  Redis publishes: 7 (one per provider update)
  PostgreSQL inserts: 7 (one per provider)
  Memory usage: < 50 MB
  CPU usage: < 2% average
```

Монитор цен намеренно легкий. Он делает одно — получает и распределяет цены — и делает это эффективно. 30-секундный интервал означает, что сервис неактивен 90% времени, оставляя ресурсы доступными для более интенсивного по вычислениям исполнителя ордеров и сервисов API.

---

## Заключение

Монитор цен концептуально прост — опрашивает поставщиков, нормализует данные, публикует обновления. Но детали имеют значение. Паттерн адаптера делает добавление поставщиков тривиальным. Redis pub/sub разделяет сбор цен от потребления цен. 60-секундные TTL автоматически исключают устаревшие данные. Метрики здоровья питают решения маршрутизации. И история цен позволяет аналитику, которая помогает пользователям принимать лучшие решения.

Каждая цена, которую вы видите на MERX, каждая рекомендация лучшей цены, каждое решение автоматической маршрутизации восходит к 30-секундному биению монитора цен. Это основание, которое делает агрегацию возможной.

Изучите живые цены и исторические данные на [https://merx.exchange](https://merx.exchange). Для доступа к API см. документацию на [https://merx.exchange/docs](https://merx.exchange/docs).

---

*Эта статья является частью технического ряда MERX. MERX — первая блокчейн-биржа ресурсов. Исходный код для SDK: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js), [https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python).*


## Попробуйте прямо сейчас с AI

Добавьте MERX в Claude Desktop или любого MCP-совместимого клиента — никакой установки, без ключа API для инструментов только для чтения:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Спросите вашего AI-ассистента: "What is the cheapest TRON energy right now?" и получите живые цены от всех подключенных поставщиков.

Полная документация MCP: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)