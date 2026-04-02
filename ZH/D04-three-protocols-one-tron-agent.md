# 三种协议，一个 TRON 代理：MERX 上的 MCP、A2A 和 ACP

AI 代理生态系统正在跨协议分裂。Anthropic 有 MCP。Google 推出了 A2A。BeeAI 构建了 ACP。每种协议以不同方式解决代理间通信问题，每种都有自己的框架和编排器生态系统。对于构建 TRON 应用的开发者来说，选择一种协议也意味着选择哪些框架能够访问 TRON 资源。MERX 通过在单一平台上支持所有三种协议消除了这一选择困境——这是第一个也是唯一一个做到这一点的 TRON 代理。

## 2026 年的协议格局

三种协议现在主导着 AI 代理基础设施：

**MCP (Model Context Protocol)** -- 由 Anthropic 创建。一种基于工具的协议，代理可以发现和调用函数。54 个工具、30 个 prompts、21 个资源。被 Claude、Cursor、Windsurf 以及数百个兼容 MCP 的客户端使用。这是代理与工具直接交互最成熟的协议。

**A2A (Agent-to-Agent Protocol)** -- 由 Google 创建，现归 Linux Foundation 管理。一种基于任务的协议，编排器将任务提交给专业代理并异步接收结果。被 LangChain、CrewAI、Vertex AI Agent Builder、AutoGen 和 Mastra 使用。专为一个代理将工作委托给另一个代理的多代理系统设计。

**ACP (Agent Communication Protocol)** -- 由 BeeAI (IBM) 创建。一种基于运行的企业编排器协议。ACP 正在 Linux Foundation 下与 A2A 合并，但协议端点对现有 ACP 客户端仍然有用。

每种协议都有不同的发现机制、不同的执行模型和不同的兼容框架集。一个只支持 MCP 的 TRON 代理对 LangChain 来说是不可见的。一个只支持 A2A 的代理对 Claude 来说是不可见的。在此之前，没有任何 TRON 项目支持一种以上的协议。

## MERX 现在支持什么

截至 2026 年 4 月，MERX 通过单一部署支持所有三种协议：

| 协议 | 发现 | 执行 | 兼容框架 |
|----------|-----------|-----------|----------------------|
| MCP | `merx.exchange/mcp/sse` | 工具调用（请求-响应） | Claude、Cursor、Windsurf、任何 MCP 客户端 |
| A2A | `merx.exchange/.well-known/agent.json` | 任务（异步，SSE 流式传输） | LangChain、CrewAI、Vertex AI、AutoGen、Mastra |
| ACP | `merx.exchange/.well-known/agent-manifest.json` | 运行（异步，long-polling） | BeeAI、IBM watsonx、ACP 框架 |

三种协议共享相同的后端。当一个 A2A 任务调用 `buy_energy` 时，它执行的订单路由逻辑与 MCP `create_order` 工具调用完全相同。这些协议是进入同一个 MERX 聚合引擎的不同入口。

## MCP：53 个用于直接集成的工具

MCP 是最深层的集成点。MERX MCP 服务器提供 54 个工具，分为 15 个类别：

- **价格情报**（5 个工具）：来自全部 7 个提供商的实时价格、最优价格路由、历史数据、市场分析
- **资源交易**（4 个工具）：创建订单、列出订单、获取订单详情、确保资源
- **代币操作**（4 个工具）：发送 TRX、发送 TRC-20 代币、批准授权额度、查询代币信息
- **DEX 兑换**（3 个工具）：获取 SunSwap 报价、执行兑换、查询代币价格
- **长期订单**（4 个工具）：带有价格触发器、调度计划、余额警报的服务端自动化
- **链上查询**（5 个工具）：账户信息、余额、交易、区块数据
- 另外还有 28 个工具涵盖估算、便捷操作、合约、网络、入驻、支付、意图执行和会话管理

MCP 服务器还提供 30 个用于引导工作流的 prompts 和 21 个用于结构化数据访问的资源。

### 一行代码即可连接

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

无需安装。只读工具无需 API keys。22 个工具立即可用。

## A2A：6 个面向编排框架的技能

A2A 将 MERX 作为专业代理暴露出来，编排器可以将任务委托给它。位于 `/.well-known/agent.json` 的 Agent Card 公布了 7 个技能：

| 技能 | 描述 | 需要认证 |
|-------|-------------|---------------|
| `buy_energy` | 从聚合市场购买委托能量 | 是 |
| `get_prices` | 来自全部 7 个提供商的当前能量价格 | 否 |
| `analyze_prices` | 包含趋势和建议的市场分析 | 否 |
| `check_balance` | 账户余额和链上资源分配 | 可选 |
| `ensure_resources` | 声明式资源配置（仅购买差额） | 是 |
| `create_standing_order` | 服务端自动化规则 | 是 |

### A2A 如何工作

A2A 协议使用基于任务的模型。编排器提交一个任务，MERX 异步处理它，编排器获取结果。

**步骤 1：发现代理**

```bash
curl https://merx.exchange/.well-known/agent.json
```

这将返回 Agent Card，包含所有 7 个技能、它们的输入模式、支持的模式和认证要求。

**步骤 2：提交任务**

```bash
curl -X POST https://merx.exchange/a2a/tasks/send \
  -H "Content-Type: application/json" \
  -d '{
    "id": "task-001",
    "message": {
      "role": "user",
      "parts": [{
        "type": "data",
        "data": { "action": "get_prices" }
      }]
    }
  }'
```

响应立即返回，状态为 `submitted`。任务在后台处理。

**步骤 3：获取结果**

```bash
curl https://merx.exchange/a2a/tasks/task-001
```

响应包含任务状态（`completed`、`failed` 等）和结果工件，其中包含来自全部 7 个提供商的价格数据。

**步骤 4：流式传输事件（可选）**

```bash
curl -N https://merx.exchange/a2a/tasks/task-001/events
```

SSE 流实时传递状态转换：`submitted` 到 `working` 到 `completed`。

### 技能路由

A2A 任务可以使用结构化数据或自然语言。任务处理器自动路由：

- **结构化**：`{ "action": "buy_energy", "energy_amount": 65000, "target_address": "T..." }` 直接路由到 buy_energy 技能
- **自然语言**："What is the current energy price?" 匹配关键词模式并路由到 get_prices

## ACP：面向企业的基于运行的执行

ACP 使用与 A2A 类似但具有不同 API 表面的基于运行的模型。位于 `/.well-known/agent-manifest.json` 的清单声明了相同的 7 个能力。

```bash
# 创建一次运行
curl -X POST https://merx.exchange/acp/v1/agents/merx-tron-agent/runs \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "merx-tron-agent",
    "input": [{
      "role": "user",
      "parts": [{
        "contentType": "application/json",
        "content": "{\"action\":\"get_prices\"}"
      }]
    }]
  }'

# 查询结果（使用 long-polling）
curl "https://merx.exchange/acp/v1/runs/{runId}?wait=true"
```

`?wait=true` 参数启用 long-polling：请求最多阻塞 30 秒等待运行完成，减少重复轮询的需要。

注意：ACP 正在 Linux Foundation 下与 A2A 合并。端点将继续为现有客户端运行，但新集成应使用 A2A。

## 架构：一个后端，三个入口

三种协议共享相同的执行路径：

```
MCP Tool Call ─┐
               ├──► MERX API ──► Provider Router ──► 7 Energy Providers
A2A Task ──────┤                                     (Netts, CatFee, TEM,
               ├──► MERX API                          ITRX, TronSave, Feee,
ACP Run ───────┘                                      PowerSun)
```

A2A 和 ACP 处理程序在现有的 API 服务内部运行（`services/api/src/agent-protocols/`）。它们对 MCP 服务器和 Web 控制面板使用的相同 REST 端点发起内部 HTTP 调用。这意味着：

- **相同的价格**：所有协议看到相同的实时提供商数据
- **相同的路由**：订单经过相同的最低价提供商逻辑
- **相同的认证**：X-API-Key 在所有协议中通用
- **相同的可靠性**：故障转移和重试逻辑同等适用

任务和运行状态存储在 Redis 中，TTL 为 24 小时。协议操作不需要数据库写入。

## 为什么多协议支持很重要

### 对于开发者

你正在构建一个 TRON 集成。你的编排框架使用 LangChain (A2A)。你队友的机器人使用 Claude (MCP)。你的企业客户要求 BeeAI (ACP)。使用单协议代理，你需要三种不同的集成来连接同一个底层服务。

使用 MERX，三者都连接到同一个平台。一个 API key。一套文档。一个支持渠道。

### 对于代理构建者

多代理系统正在成为标准。一个规划代理与一个交易代理、一个监控代理和一个报告代理协调。这些代理可能运行在不同的框架上。一个 CrewAI crew 可能通过 A2A 将能量购买委托给 MERX，同时一个 Claude 代理通过 MCP 监控价格。

MERX 处理两者，而代理无需知道彼此的协议选择。

### 对于 TRON 生态系统

更广泛的协议覆盖意味着更多的潜在集成。每个支持 A2A 的 AI 框架现在都可以访问 TRON 能量市场。每个 MCP 客户端都可以优化 TRON 交易成本。TRON 能量服务的总可寻址市场随着每个支持的协议而扩大。

## MERX 的上架位置

MERX 是唯一在 MCP 和 A2A 目录中都有存在的 TRON 项目：

**MCP 注册中心：**
- [Glama](https://glama.ai/mcp/servers/Hovsteder/merx-mcp)
- [Smithery](https://smithery.ai/servers/powersun/merx)
- [MCP 官方注册中心](https://registry.modelcontextprotocol.io/servers/exchange.merx/mcp)
- mcp.so
- PulseMCP

**A2A 目录：**
- [awesome-a2a](https://github.com/pab1it0/awesome-a2a)（金融服务部分）
- [a2aregistry.in](https://a2aregistry.in)

## 快速开始

### MCP (Claude, Cursor, Windsurf)

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

### A2A (LangChain, CrewAI, Vertex AI, AutoGen)

发现 URL：`https://merx.exchange/.well-known/agent.json`

### ACP (BeeAI)

发现 URL：`https://merx.exchange/.well-known/agent-manifest.json`

### 文档

- [代理协议概览](https://merx.exchange/agents)
- [MCP 服务器（54 个工具）](https://merx.exchange/mcp)
- [A2A 协议文档](https://merx.exchange/docs/tools/a2a)
- [ACP 协议文档](https://merx.exchange/docs/tools/acp)
- [GitHub](https://github.com/Hovsteder/merx-mcp)

---

*Tags: tron mcp server, a2a protocol, acp protocol, tron ai agent, ai agent tron energy, langchain tron, crewai tron, multi-protocol agent, merx exchange*

---

**立即通过 AI 试用**

添加到你的 MCP 客户端：

```json
{ "merx": { "url": "https://merx.exchange/mcp/sse" } }
```

或通过 A2A 发现：

```bash
curl https://merx.exchange/.well-known/agent.json
```

然后询问："What are the current TRON energy prices across all providers?"
