# 从构想到上线：30 天构建 MERX

MERX 从概念到上线的生产系统用了 30 天。不是一个落地页。不是一个原型。而是一个完整运营的区块链资源交易所，集成了七个供应商、实时价格聚合、链上订单执行、复式记账、完整文档、两种语言的 SDK，以及一个拥有 52 个工具的 MCP 服务器用于 AI 代理集成。

本文是这一切如何发生的技术故事——架构决策、我们解决的问题、我们刻意没有走的捷径，以及在不牺牲重要事项的前提下快速构建金融平台的经验教训。

---

## 第 0 天：问题陈述

TRON 能量市场是碎片化的。七个或更多供应商提供能量委托服务，每个都有自己的 API、定价和可靠性特征。如果你想获得最优价格，你需要与所有供应商集成。如果你想要故障转移，你需要构建路由逻辑。如果你想要透明度，你需要构建监控。

每个在 TRON 上发送 USDT 的企业都面临这个集成税。解决方案是一个聚合层——一个处理多供应商路由、最优价格选择和自动故障转移的单一 API。

还没有人构建过。我们决定来做。

---

## 第 1 周：基础

### 先做架构

在编写一行代码之前，我们花了两天时间做架构设计。成果是一份涵盖 40 个章节的架构文档，从数据库模式到 API 错误格式再到颜色色值，应有尽有。这份文档成为每个实现决策的唯一真相来源。

在那两天做出的关键架构决策：

**决策 1：从第一天起就用微服务。**

不是因为微服务时髦，而是因为金融系统需要隔离。资金库签名器不能被 API 服务访问。价格监控不能对用户余额有写入权限。Docker 容器天然提供这种隔离。

```
services/
  api/              HTTP/WebSocket API
  price-monitor/    Provider price polling
  order-executor/   Order routing and execution
  ledger/           Double-entry accounting
  deposit-monitor/  Incoming payment detection
  treasury-signer/  Transaction signing (isolated)
```

**决策 2：PostgreSQL + Redis，不用特殊数据库。**

PostgreSQL 用于所有需要 ACID 保证的内容（余额、订单、账本记录）。Redis 用于所有需要速度的内容（价格缓存、pub/sub、限频）。两者都久经考验、文档完善、运维简单。

**决策 3：所有金额使用 SUN。**

每个金融值以 SUN 整数存储（1 TRX = 1,000,000 SUN）。金融路径中没有任何浮点数。这在我们编写第一个函数之前就消除了一整类错误。

**决策 4：服务使用 Node.js + TypeScript，匹配引擎使用 Go。**

TypeScript 用于系统的大部分——快速开发、强类型、出色的异步 I/O 适合 API 和监控工作负载。Go 保留给原始性能重要的匹配引擎。

### 数据库模式

数据库迁移在第 3 天编写。每个表都以金融完整性为核心设计：

```sql
-- Core principle: every balance mutation creates a ledger entry
CREATE TABLE ledger (
  id BIGSERIAL PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(id),
  type VARCHAR(50) NOT NULL,
  amount_sun BIGINT NOT NULL,
  direction VARCHAR(6) NOT NULL CHECK (direction IN ('DEBIT', 'CREDIT')),
  reference_type VARCHAR(50),
  reference_id UUID,
  balance_before BIGINT NOT NULL,
  balance_after BIGINT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- No UPDATE or DELETE triggers - ledger is append-only
```

每个迁移一个文件，命名为 `YYYYMMDD_description.sql`。到 30 天结束时，有 14 个迁移文件，每个都是增量的，没有破坏性的。

### 供应商接口

`IEnergyProvider` 接口在第 4 天定义。这是每个供应商适配器将实现的约定：

```typescript
interface IEnergyProvider {
  name: string;
  getPrices(): Promise<ProviderPriceResponse>;
  createOrder(params: OrderParams): Promise<OrderResult>;
  getOrderStatus(orderId: string): Promise<OrderStatus>;
  healthCheck(): Promise<boolean>;
}
```

这个接口从未变过。七个供应商在随后几周内基于它完成了集成，每个都在自己的文件中，没有一个需要更改核心系统。

---

## 第 2 周：核心服务

### 价格监控

价格监控是第一个上线的服务。它每 30 秒轮询每个供应商，规范化价格，发布到 Redis，并将历史存储到 PostgreSQL。实现大约是三个文件共 180 行 TypeScript。

最困难的部分不是轮询逻辑——而是规范化。每个供应商返回的价格格式略有不同：

- 供应商 A：每能量单位 SUN
- 供应商 B：固定能量数量的总 TRX
- 供应商 C：每能量单位 SUN，但最低订单量不同
- 供应商 D：基于交易量的阶梯定价

每个适配器将其供应商的格式转换为标准的 `ProviderPriceResponse`。价格监控不关心供应商的差异；它只看到规范化的数据。

### 订单执行器

订单执行器是最复杂的服务。它从 Redis 读取价格，确定最优路由，向供应商提交订单，监控链上确认，并发布结算事件。

故障转移链是关键设计元素。如果供应商 A 失败，尝试供应商 B。如果 B 失败，尝试 C。只要任何供应商可用，买家的 API 调用就会成功。

```
Order received -> Read prices -> Select cheapest
  -> Execute at Provider A
    -> Success? Verify on-chain -> Settle
    -> Failure? Try Provider B
      -> Success? Verify on-chain -> Settle
      -> Failure? Try Provider C
        -> ... and so on
```

### 账本服务

账本服务强制执行复式记账约束。每次余额变更创建配对记录。服务每小时运行一次对账检查：

```sql
SELECT SUM(CASE direction
  WHEN 'DEBIT' THEN amount_sun
  WHEN 'CREDIT' THEN -amount_sun
END) FROM ledger;
-- Must be 0. If not: alert immediately.
```

在 30 天的开发和测试中，这个检查从未触发。约束从未被违反，因为架构使违反在结构上不可能，而不仅仅是不太可能。

---

## 第 3 周：API、前端和链上验证

### API 设计

API 遵循 REST 规范，严格版本化（`/api/v1/...`）。每个端点在实现前先设计：

```
GET    /api/v1/prices          Current prices from all providers
GET    /api/v1/prices/best     Best current price
POST   /api/v1/orders          Create a new order
GET    /api/v1/orders/:id      Get order status
GET    /api/v1/balance          Get account balance
POST   /api/v1/deposit         Get deposit address
POST   /api/v1/withdraw        Request withdrawal
```

错误响应使用一致的格式：

```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Account balance (5.2 TRX) is insufficient for this order (8.1 TRX)",
    "details": {
      "balance": 5200000,
      "required": 8100000
    }
  }
}
```

没有端点在所有输入都经过 Zod 验证之前发布。

### 前端

前端是 Next.js 应用，具有严格的设计系统：仅深色主题、圆角不超过 2px、无渐变、无阴影、标题使用 Cormorant Garamond 字体、其他所有内容使用 IBM Plex Mono 字体。视觉身份在架构文档中定义并忠实实现。

### 链上验证

每笔订单都在 TRON 区块链上验证。验证服务监视委托交易并确认能量到达目标地址。这是最具挑战性的集成，因为区块链确认时间是可变的，供应商交易格式各不相同。

在测试阶段验证了八笔主网交易，确认了从 API 调用到链上委托的端到端流程与真实 TRX 和真实供应商正确运作。

---

## 第 4 周：SDK、MCP 服务器和文档

### JavaScript SDK

JavaScript SDK 为 Node.js 和浏览器环境构建：

```typescript
import { MerxClient } from '@merx/sdk';

const client = new MerxClient({ apiKey: 'your-key' });
const prices = await client.getPrices({ energy: 65000 });
const order = await client.createOrder({
  energy: 65000,
  targetAddress: 'TAddress...',
  duration: '1h'
});
```

源代码：[https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)

### Python SDK

Python SDK 与 JavaScript SDK 的 API 接口保持一致：

```python
from merx import MerxClient

client = MerxClient(api_key='your-key')
prices = client.get_prices(energy=65000)
order = client.create_order(
    energy=65000,
    target_address='TAddress...',
    duration='1h'
)
```

源代码：[https://github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)

### MCP 服务器：52 个工具

MCP（Model Context Protocol）服务器可能是最具前瞻性的组件。它将 MERX 功能作为工具暴露给 AI 代理直接使用。

MCP 服务器从初始版本的 7 个工具增长到 30 天结束时的 52 个工具：

```
Account management:    create_account, login, get_balance, get_deposit_info
Price data:            get_prices, get_best_price, compare_providers, analyze_prices
Order management:      create_order, get_order, list_orders, create_standing_order
Resource monitoring:   check_address_resources, estimate_transaction_cost
TRON utilities:        validate_address, convert_address, get_trx_balance
On-chain operations:   transfer_trx, transfer_trc20, approve_trc20
Analytics:             calculate_savings, get_price_history, suggest_duration
... and 30 more
```

源代码：[https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)

### 文档

文档从 5 页重建为 36 页，涵盖完整的 API 参考、SDK 指南、TRON 概念和集成教程。文档位于 [https://merx.exchange/docs](https://merx.exchange/docs)。

此外，发布了 4 个 SEO 指南页面和 7 个供应商对比页面，使站点地图达到 53 个 URL。

---

## 我们没有妥协的地方

速度创造了走捷径的压力。以下是我们明确没有走的捷径：

### 不对货币使用浮点数

对所有金融值使用整数（SUN）增加了显示格式化的复杂性，但完全消除了舍入误差。每个测试用例都精确匹配预期值。

### 不对 SQL 使用字符串拼接

每个数据库查询使用参数化语句。这是从第一天起的不可协商规则。SQL 注入是一个已解决的问题，我们保持了它的解决状态。

### 不硬编码密钥

从第一天起就使用环境变量。资金库密钥使用 Docker secret。`.gitignore` 在第一次提交之前就设置好了。

### 不让服务之间直接共享状态

服务通过 Redis pub/sub 或 REST API 调用通信。服务之间没有直接导入。这使独立部署成为可能并防止了级联故障。

### 不修改账本

从第一个迁移开始就是仅追加账本。账本表上没有 UPDATE 或 DELETE。修正创建新记录，而不是修改。

---

## 我们学到的

### 经验 1：架构文档物有所值

花在架构上的两天节省了数周的返工。每个开发者问题都能在文档中找到答案。每个设计分歧都通过引用规范解决。40 个章节不是官僚式的开销；它们是在问题变成错误之前强制思考问题的工具。

### 经验 2：供应商 API 不可靠

在集成的七个供应商中，至少有两个在 30 天构建期间经历了宕机。故障转移链不是理论上的优雅之举——它在测试的第一周内就被实际使用了。

### 经验 3：适配器模式值得其样板代码

编写七个都实现相同接口的适配器感觉很重复。但当供应商 C 在第 22 天更改了他们的 API 响应格式时，我们更新了一个文件，其他什么都没变。花 10 分钟更新适配器与花数天更新每个调用处相比，该模式的价值不言自明。

### 经验 4：MCP 是服务集成的未来

MCP 服务器最初是一个实验。但看着 AI 代理使用 MERX 工具自主管理能量采购令人震撼。这就是未来服务被消费的方式——不是通过人类开发者编写集成代码，而是通过 AI 代理直接调用工具 API。

### 经验 5：200 行文件限制是一个功能

我们在整个项目中强制执行了严格的 200 行/文件限制。这迫使我们不断分解。函数保持小巧。职责保持清晰。当文件接近 200 行时，就是拆分的时候，而拆分总是改善了清晰度。

---

## 数据一览

```
Architecture document:     40 sections
Services:                  9 Docker containers
Provider integrations:     7
Database migrations:       14
API endpoints:             20+
MCP tools:                 52 (from initial 7)
SDK languages:             2 (JavaScript, Python)
Documentation pages:       36 (from initial 5)
Sitemap URLs:              53
Mainnet transactions:      8 verified
Commission rate:           0%
Days to production:        30
```

---

## 下一步

平台已在 [https://merx.exchange](https://merx.exchange) 上线。当前重点是测试、优化和引入第一批生产用户。基础是坚实的——架构支持水平扩展，新供应商可以在数小时内添加，零佣金模式消除了用户采纳的摩擦。

TRON 上的能量聚合市场正在等待一个让它变得简单的平台。MERX 就是那个平台。

---

*MERX 是首个区块链资源交易所。在 [https://merx.exchange](https://merx.exchange) 探索平台。文档请访问 [https://merx.exchange/docs](https://merx.exchange/docs)。开源 SDK 和 MCP 服务器请访问 GitHub。*
