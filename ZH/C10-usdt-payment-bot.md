# 使用MERX SDK构建USDT支付机器人

在TRON上发送USDT应该很简单。您有收款地址、金额和有资金的钱包。但如果不使用能量发送USDT,TRON协议会从您的钱包燃烧大约27 TRX来覆盖交易成本。在规模化场景下 - 每天数百或数千笔转账 - 这一燃烧成本会成为一笔重大支出。

本教程构建一个完整的USDT支付机器人来消除这一成本。该机器人接收支付请求,通过MERX以远低于燃烧价格的成本租赁能量,使用租赁的能量发送USDT,并报告节省金额。它处理错误、支持重试,并集成Webhook通知以确保生产可靠性。

完成后,您将拥有一个工作中的支付机器人,每笔USDT转账节省超过90%。

## 架构概览

机器人遵循简单的流水线:

1. 接收支付请求(收款地址 + 金额)。
2. 检查您的MERX账户余额。
3. 估算转账所需的能量。
4. 通过MERX创建能量订单。
5. 等待能量委托完成。
6. 使用委托的能量发送USDT转账。
7. 报告交易和节省金额。

每个步骤都是隔离的且可重试的。如果能量订单失败,USDT不会被发送。如果USDT转账失败,您仍然拥有能量(它在租赁期结束后过期,但不会损失资金)。

## 前提条件

开始构建前,您需要:

- **Node.js 18或更高版本** - 机器人使用ES模块和现代JavaScript特性。
- **有余额的MERX账户** - 在 [merx.exchange](https://merx.exchange) 注册并充值TRX。
- **MERX API密钥** - 创建一个具有 `create_orders`、`view_orders` 和 `view_balance` 权限的密钥。
- **TRON钱包** - 有USDT余额用于支付,以及少量TRX用于带宽。
- **TronWeb** - 用于签名和广播USDT转账交易。

### 项目初始化

```bash
mkdir usdt-payment-bot
cd usdt-payment-bot
npm init -y
npm install merx-sdk tronweb dotenv uuid
```

创建 `.env` 文件(永远不要提交到版本控制):

```bash
MERX_API_KEY=merx_live_your_key_here
TRON_PRIVATE_KEY=your_tron_wallet_private_key
TRON_FULL_HOST=https://api.trongrid.io
USDT_CONTRACT=TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t
MIN_SAVINGS_PERCENT=50
```

## 第一步: 初始化客户端

创建MERX客户端和TronWeb实例:

```javascript
// src/clients.js
import { MerxClient } from 'merx-sdk';
import TronWeb from 'tronweb';
import dotenv from 'dotenv';

dotenv.config();

export const merx = new MerxClient({
  apiKey: process.env.MERX_API_KEY,
  baseUrl: 'https://merx.exchange/api/v1',
});

export const tronWeb = new TronWeb({
  fullHost: process.env.TRON_FULL_HOST,
  privateKey: process.env.TRON_PRIVATE_KEY,
});

export const USDT_CONTRACT = process.env.USDT_CONTRACT;
export const SENDER_ADDRESS = tronWeb.defaultAddress.base58;
```

## 第二步: 检查MERX余额

在执行任何操作之前,验证您有足够的余额租赁能量:

```javascript
// src/balance.js
import { merx } from './clients.js';

export async function checkMerxBalance(requiredSun) {
  const balance = await merx.account.getBalance();

  const availableSun = balance.available_sun;

  if (availableSun < requiredSun) {
    throw new Error(
      `Insufficient MERX balance. ` +
        `Available: ${(availableSun / 1_000_000).toFixed(2)} TRX, ` +
        `Required: ${(requiredSun / 1_000_000).toFixed(2)} TRX`
    );
  }

  return {
    available: availableSun,
    availableTRX: (availableSun / 1_000_000).toFixed(2),
  };
}
```

## 第三步: 估算能量成本

使用MERX估算API确定转账需要多少能量以及成本:

```javascript
// src/estimate.js
import { merx, SENDER_ADDRESS } from './clients.js';

export async function estimateTransferCost() {
  const estimate = await merx.estimate({
    operation: 'trc20_transfer',
    target_address: SENDER_ADDRESS,
  });

  const savings = estimate.costs.savings;
  const rental = estimate.costs.rental;
  const burn = estimate.costs.burn;

  return {
    energyRequired: estimate.energy_required,
    bandwidthRequired: estimate.bandwidth_required,
    burnCostTRX: burn.trx_cost_readable,
    burnCostSun: burn.trx_cost,
    rentalCostTRX: rental.total_cost_trx,
    rentalCostSun: rental.total_cost_sun,
    bestProvider: rental.best_provider,
    savingsPercent: savings.percent,
    savingsTRX: savings.trx_saved,
    durationHours: rental.duration_hours,
  };
}
```

## 第四步: 创建能量订单

通过MERX下达能量订单,使用幂等性密钥确保安全重试:

```javascript
// src/order.js
import { merx } from './clients.js';
import { randomUUID } from 'crypto';

export async function createEnergyOrder(energyAmount, targetAddress, durationHours = 1) {
  const idempotencyKey = randomUUID();

  const order = await merx.orders.create(
    {
      energy_amount: energyAmount,
      target_address: targetAddress,
      duration_hours: durationHours,
    },
    {
      idempotencyKey,
    }
  );

  return {
    orderId: order.id,
    status: order.status,
    provider: order.provider,
    totalCostSun: order.total_cost_sun,
    idempotencyKey,
  };
}
```

## 第五步: 等待能量委托

创建订单后,等待能量在链上完成委托。机器人轮询订单状态,带超时机制:

```javascript
// src/wait.js
import { merx } from './clients.js';

export async function waitForFill(orderId, timeoutMs = 60000) {
  const startTime = Date.now();
  const pollInterval = 2000; // 2秒

  while (Date.now() - startTime < timeoutMs) {
    const order = await merx.orders.get(orderId);

    if (order.status === 'filled') {
      return {
        status: 'filled',
        provider: order.provider,
        txHash: order.delegation_tx_hash,
        filledAt: order.filled_at,
      };
    }

    if (order.status === 'failed' || order.status === 'cancelled') {
      throw new Error(
        `Order ${orderId} ${order.status}: ${order.failure_reason || 'unknown reason'}`
      );
    }

    // 仍在等待或处理中 - 等待后再次轮询
    await new Promise((resolve) => setTimeout(resolve, pollInterval));
  }

  throw new Error(`Order ${orderId} timed out after ${timeoutMs / 1000}s`);
}
```

对于生产系统,请考虑使用Webhook替代轮询(本文后面有介绍)。

## 第六步: 发送USDT转账

能量已委托到您的地址后,发送USDT转账。能量被消耗,而非燃烧TRX:

```javascript
// src/transfer.js
import { tronWeb, USDT_CONTRACT, SENDER_ADDRESS } from './clients.js';

export async function sendUSDT(recipientAddress, amountUSDT) {
  // USDT在TRON上有6位小数
  const amountSun = Math.floor(amountUSDT * 1_000_000);

  // 验证收款地址
  if (!tronWeb.isAddress(recipientAddress)) {
    throw new Error(`Invalid TRON address: ${recipientAddress}`);
  }

  // 构建TRC-20转账交易
  const contract = await tronWeb.contract().at(USDT_CONTRACT);

  const tx = await contract.methods
    .transfer(recipientAddress, amountSun)
    .send({
      from: SENDER_ADDRESS,
      feeLimit: 100_000_000, // 100 TRX手续费上限(安全上限)
    });

  return {
    txHash: tx,
    recipient: recipientAddress,
    amount: amountUSDT,
    amountSun,
  };
}
```

## 第七步: 完整流程整合

主函数编排所有步骤:

```javascript
// src/bot.js
import { checkMerxBalance } from './balance.js';
import { estimateTransferCost } from './estimate.js';
import { createEnergyOrder } from './order.js';
import { waitForFill } from './wait.js';
import { sendUSDT } from './transfer.js';
import { SENDER_ADDRESS } from './clients.js';

const MIN_SAVINGS_PERCENT = parseFloat(process.env.MIN_SAVINGS_PERCENT || '50');

export async function processPayment(recipientAddress, amountUSDT) {
  const startTime = Date.now();

  console.log(`--- Payment Request ---`);
  console.log(`Recipient: ${recipientAddress}`);
  console.log(`Amount: ${amountUSDT} USDT`);

  // 第1步: 估算成本
  console.log('\n[1/5] Estimating energy cost...');
  const estimate = await estimateTransferCost();
  console.log(`  Energy required: ${estimate.energyRequired}`);
  console.log(`  Burn cost: ${estimate.burnCostTRX}`);
  console.log(`  Rental cost: ${estimate.rentalCostTRX}`);
  console.log(`  Savings: ${estimate.savingsPercent}%`);

  // 第2步: 判断租赁是否值得
  if (estimate.savingsPercent < MIN_SAVINGS_PERCENT) {
    console.log(`  Savings below threshold (${MIN_SAVINGS_PERCENT}%). Skipping energy rental.`);
    const tx = await sendUSDT(recipientAddress, amountUSDT);
    return { ...tx, energyRented: false, savings: null };
  }

  // 第3步: 检查MERX余额
  console.log('\n[2/5] Checking MERX balance...');
  const balance = await checkMerxBalance(estimate.rentalCostSun);
  console.log(`  Available: ${balance.availableTRX} TRX`);

  // 第4步: 创建能量订单
  console.log('\n[3/5] Creating energy order...');
  const order = await createEnergyOrder(
    estimate.energyRequired,
    SENDER_ADDRESS,
    estimate.durationHours
  );
  console.log(`  Order ID: ${order.orderId}`);
  console.log(`  Provider: ${order.provider}`);
  console.log(`  Status: ${order.status}`);

  // 第5步: 等待能量委托
  console.log('\n[4/5] Waiting for energy delegation...');
  const fill = await waitForFill(order.orderId);
  console.log(`  Filled by: ${fill.provider}`);
  console.log(`  Delegation TX: ${fill.txHash}`);

  // 第6步: 使用委托能量发送USDT
  console.log('\n[5/5] Sending USDT...');
  const tx = await sendUSDT(recipientAddress, amountUSDT);
  console.log(`  TX Hash: ${tx.txHash}`);

  const elapsed = ((Date.now() - startTime) / 1000).toFixed(1);

  console.log(`\n--- Payment Complete ---`);
  console.log(`  Total time: ${elapsed}s`);
  console.log(`  Energy cost: ${estimate.rentalCostTRX}`);
  console.log(`  Saved: ${estimate.savingsTRX} (${estimate.savingsPercent}%)`);

  return {
    txHash: tx.txHash,
    recipient: recipientAddress,
    amount: amountUSDT,
    energyRented: true,
    energyOrderId: order.orderId,
    rentalCost: estimate.rentalCostTRX,
    burnCostAvoided: estimate.burnCostTRX,
    savingsPercent: estimate.savingsPercent,
    savingsTRX: estimate.savingsTRX,
    elapsedSeconds: parseFloat(elapsed),
  };
}
```

### 运行机器人

```javascript
// index.js
import { processPayment } from './src/bot.js';

const recipient = process.argv[2];
const amount = parseFloat(process.argv[3]);

if (!recipient || !amount) {
  console.error('Usage: node index.js <recipient_address> <amount_usdt>');
  process.exit(1);
}

try {
  const result = await processPayment(recipient, amount);
  console.log('\nResult:', JSON.stringify(result, null, 2));
} catch (err) {
  console.error('\nPayment failed:', err.message);
  process.exit(1);
}
```

```bash
node index.js TRecipientAddressHere 100
```

示例输出:

```
--- Payment Request ---
Recipient: TRecipientAddressHere
Amount: 100 USDT

[1/5] Estimating energy cost...
  Energy required: 64895
  Burn cost: 27.37 TRX
  Rental cost: 1.43 TRX
  Savings: 94.8%

[2/5] Checking MERX balance...
  Available: 50.00 TRX

[3/5] Creating energy order...
  Order ID: ord_7k3m9x2p
  Provider: sohu
  Status: pending

[4/5] Waiting for energy delegation...
  Filled by: sohu
  Delegation TX: 5a8f2c1e...

[5/5] Sending USDT...
  TX Hash: 3d7b4e9a...

--- Payment Complete ---
  Total time: 8.2s
  Energy cost: 1.43 TRX
  Saved: 25.94 TRX (94.8%)
```

## 错误处理

生产级支付系统需要健壮的错误处理。以下是故障模式及其处理方式。

### MERX余额不足

如果MERX账户余额不足,机器人应该发出警报,并可选地回退到燃烧TRX:

```javascript
try {
  await checkMerxBalance(estimate.rentalCostSun);
} catch (err) {
  if (err.message.includes('Insufficient MERX balance')) {
    console.warn('MERX balance low. Sending without energy rental.');
    // 可选: 向运维团队发送告警
    // await alertOps('MERX balance low', err.message);
    const tx = await sendUSDT(recipientAddress, amountUSDT);
    return { ...tx, energyRented: false, fallbackReason: 'low_balance' };
  }
  throw err;
}
```

### 订单成交超时

如果能量供应商委托耗时过长,可以取消并使用其他供应商重试,或回退:

```javascript
try {
  const fill = await waitForFill(order.orderId, 30000); // 30秒超时
} catch (err) {
  if (err.message.includes('timed out')) {
    console.warn('Energy delegation timed out. Sending without energy.');
    const tx = await sendUSDT(recipientAddress, amountUSDT);
    return { ...tx, energyRented: false, fallbackReason: 'delegation_timeout' };
  }
  throw err;
}
```

### USDT转账失败

如果USDT转账本身失败(余额不足、合约错误),能量不会浪费 - 它在租赁期内保持委托状态。您可以重试转账:

```javascript
async function sendUSDTWithRetry(recipientAddress, amountUSDT, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await sendUSDT(recipientAddress, amountUSDT);
    } catch (err) {
      if (attempt === maxRetries) throw err;
      console.warn(`Transfer attempt ${attempt} failed: ${err.message}`);
      await new Promise((r) => setTimeout(r, 2000 * attempt));
    }
  }
}
```

## Webhook通知

轮询订单状态对简单机器人有效,但生产系统应使用Webhook。当订单状态变化时,MERX会向您的Webhook URL发送HTTP POST通知。

### 设置Webhook

在MERX控制面板或通过API配置Webhook端点:

```bash
curl -X POST https://merx.exchange/api/v1/webhooks \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-server.com/webhooks/merx",
    "events": ["order.filled", "order.failed"]
  }'
```

### 处理Webhook事件

```javascript
// webhook-handler.js
import express from 'express';

const app = express();
app.use(express.json());

// 存储待处理支付的回调
const pendingPayments = new Map();

app.post('/webhooks/merx', (req, res) => {
  const event = req.body;

  // 验证Webhook签名(生产环境必须)
  // const isValid = verifyWebhookSignature(req);
  // if (!isValid) return res.status(401).json({ error: 'Invalid signature' });

  if (event.type === 'order.filled') {
    const callback = pendingPayments.get(event.data.order_id);
    if (callback) {
      callback.resolve(event.data);
      pendingPayments.delete(event.data.order_id);
    }
  }

  if (event.type === 'order.failed') {
    const callback = pendingPayments.get(event.data.order_id);
    if (callback) {
      callback.reject(new Error(`Order failed: ${event.data.reason}`));
      pendingPayments.delete(event.data.order_id);
    }
  }

  res.status(200).json({ received: true });
});

// 用基于Webhook的等待替代轮询
export function waitForFillWebhook(orderId, timeoutMs = 60000) {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => {
      pendingPayments.delete(orderId);
      reject(new Error(`Order ${orderId} webhook timed out`));
    }, timeoutMs);

    pendingPayments.set(orderId, {
      resolve: (data) => {
        clearTimeout(timer);
        resolve(data);
      },
      reject: (err) => {
        clearTimeout(timer);
        reject(err);
      },
    });
  });
}
```

Webhook消除了轮询循环,减少了API调用,并能更快地响应订单成交(亚秒级通知,而轮询间隔为2秒)。

## 生产环境注意事项

### 并发

如果机器人同时处理多笔支付,每笔支付应有自己的幂等性密钥并独立运行。使用队列(Bull、BullMQ或类似工具)管理并发支付:

```javascript
import Queue from 'bull';

const paymentQueue = new Queue('payments', {
  redis: { host: '127.0.0.1', port: 6379 },
});

paymentQueue.process(5, async (job) => {
  // 最多同时处理5笔支付
  const { recipient, amount } = job.data;
  return await processPayment(recipient, amount);
});

// 将支付添加到队列
await paymentQueue.add({
  recipient: 'TRecipientAddressHere',
  amount: 100,
});
```

### 监控和告警

跟踪以下关键指标以保持运营可见性:

- **每小时支付笔数** - 吞吐量追踪。
- **平均节省百分比** - 如果节省降至80%以下,调查供应商定价。
- **MERX余额** - 当余额降至阈值以下(例如足够100笔转账)时告警。
- **成交时间** - 如果能量委托超过30秒,供应商可能拥堵。
- **失败率** - 在任何步骤失败的支付尝试百分比。

使用您偏好的监控工具(Prometheus、Datadog,或简单的JSON日志)跟踪这些指标。

### 速率限制

MERX API限制订单创建为每分钟10次请求。如果您的机器人需要每分钟发送超过10笔支付,有两种选择:

1. **排队支付** 并按限制内的速率处理。
2. **批量购买能量** - 在一个订单中购买足够多笔转账的能量,然后在能量有效期间依次发送转账。

选项2对高吞吐量场景更高效:

```javascript
async function processBatch(payments) {
  // 计算所有支付所需的总能量
  const totalEnergy = payments.length * 65000; // 每笔USDT转账约65K

  // 一个能量订单覆盖整个批次
  const order = await createEnergyOrder(totalEnergy, SENDER_ADDRESS, 1);
  await waitForFill(order.orderId);

  // 在能量有效期间发送所有USDT转账
  const results = [];
  for (const payment of payments) {
    try {
      const tx = await sendUSDT(payment.recipient, payment.amount);
      results.push({ success: true, ...tx });
    } catch (err) {
      results.push({ success: false, error: err.message, ...payment });
    }
  }

  return results;
}
```

### 安全检查清单

部署到生产环境前:

- 将TRON私钥存储在密钥管理器中,而非磁盘上的 `.env` 文件中。
- 在受限环境中运行机器人,除Webhook端点外不接受入站网络访问。
- 为机器人使用专用的TRON钱包,只保留近期支付所需的USDT。不要将整个资金库放在机器人钱包中。
- 实施提现限制 - 机器人不应能在单次运行中清空钱包。
- 记录每笔支付的完整详情以供审计跟踪。
- 对异常活动设置告警(支付金额超过阈值、快速连续失败)。

## 完整项目结构

```
usdt-payment-bot/
  .env                  # 永远不要提交
  .gitignore
  package.json
  index.js              # 入口点
  src/
    clients.js          # MERX + TronWeb 初始化
    balance.js          # 余额检查
    estimate.js         # 成本估算
    order.js            # 能量订单创建
    wait.js             # 订单成交轮询
    transfer.js         # USDT转账执行
    bot.js              # 主编排逻辑
    webhook-handler.js  # Webhook接收器(可选)
```

每个文件单一职责。代码总量不到300行。通过模拟MERX客户端进行单元测试,使用Shasta测试网进行集成测试。

## 总结

这个支付机器人展示了核心的MERX集成模式: 估算、下单、等待、执行交易。无论您是在构建支付处理器、钱包、交易所提现系统,还是任何发送TRC-20代币的应用,相同的模式都适用。

关键结论是节省的数学。以每笔转账94%的节省率,一个每天处理1,000笔USDT转账的机器人每天节省大约26,000 TRX - 每月超过750,000 TRX。这不是优化。这是根本性的成本结构变革。

先在Shasta测试网(`TRON_FULL_HOST=https://api.shasta.trongrid.io`)上验证流程,无需承担真实资金风险,确认一切正常后再切换到主网。

- MERX平台: [merx.exchange](https://merx.exchange)
- API文档: [merx.exchange/docs](https://merx.exchange/docs)
- JavaScript SDK: [github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js)
- Python SDK: [github.com/Hovsteder/merx-sdk-python](https://github.com/Hovsteder/merx-sdk-python)
- MCP Server(AI代理专用): [github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)
