---
title: A2A 协议详解 — Agent-to-Agent Communication
created: 2026-05-02
updated: 2026-05-02
type: concept
tags: [protocol, agent, architecture, open-source]
sources: [raw/articles/a2a-official-spec.md, raw/articles/a2a-community-research.md]
contradictions: []
---

# A2A 协议详解

> Agent-to-Agent Protocol — Google 主导、Linux Foundation 托管的开放标准，定义 AI Agent 之间的通信与协作方式。

## 概述

A2A 是一个开放标准，定义了自主 AI Agent 如何以对等方式通信和协作。它标准化了 Agent 发现、任务管理、消息交换、流式传输和安全机制。

- **仓库**: `https://github.com/a2aproject/A2A`
- **当前版本**: v1.0（替代 v0.3.0）
- **规范来源**: Protocol Buffers (`a2a.proto`) — 唯一的规范性数据模型
- **Content-Type**: `application/a2a+json`
- **三种协议绑定**: JSON-RPC 2.0、gRPC、HTTP/REST（功能等价）

## 核心数据模型

### Task — 任务

Task 是 A2A 的核心工作单元，所有通信围绕 Task 展开。

| 字段 | 说明 |
|------|------|
| `id` | UUID，服务端生成 |
| `contextId` | 会话标识，将相关任务分组 |
| `status` | [TaskStatus](a2a-task-state-machine.md)，包含 state + 可选 message |
| `artifacts[]` | 输出结果（文件、数据、文本） |
| `history[]` | Message 数组（对话线程） |
| `metadata` | 可扩展键值对 |
| `extensions[]` | 协议扩展标识符 |
| `createdAt` | ISO 8601 时间戳 |
| `lastModified` | ISO 8601 时间戳 |

### Message — 消息

Task 中的单次通信单元：

| 字段 | 说明 |
|------|------|
| `messageId` | UUID，客户端提供 |
| `taskId` | 可选，引用已有任务用于后续交互 |
| `contextId` | 可选，用于会话连续性 |
| `role` | `ROLE_USER` 或 `ROLE_AGENT` |
| `parts[]` | [Part](#part-内容单元) 数组 |
| `metadata` | 可扩展键值对 |
| `referenceTaskIds[]` | 关联任务链接 |

### Part — 内容单元

多态内容，按字段区分类型：

- **TextPart**: `{ "text": "..." }` — 纯文本或结构化文本
- **FilePart**: `{ "files": [...] }` — 文件附件，含 `filename`、`mediaType`，以及 `raw`（base64）或 `url`（远程引用）
- **DataPart**: `{ "data": {...} }` — 结构化 JSON 数据

### Artifact — 产出物

任务的输出结果，与 Message 不同：Message 用于通信，Artifact 用于交付成果。

| 字段 | 说明 |
|------|------|
| `artifactId` | UUID |
| `name` | 人类可读名称 |
| `parts[]` | Part 数组 |
| `index` | 位置索引 |
| `append` | 是否追加到前一个 artifact |

> ⚠️ **注意**: 不要用 Message 来交付结果，区分 Message（通信）和 Artifact（产出）。

## 会话机制（Session）

A2A 使用 `contextId` 维持会话连续性，**没有独立的 Session 对象**。

### 会话流程

```
1. 首次消息
   Client → Message（无 taskId，无 contextId）
   Server → 创建 Task，生成 contextId，返回

2. 继续对话
   Client → Message（携带 contextId）
   Server → 在同一会话上下文中处理

3. 跟进已有任务
   Client → Message（携带 taskId）
   Server → 继续处理该任务

4. 同一会话中的新任务
   Client → Message（携带 contextId，不带 taskId）
   Server → 创建新 Task，继承会话上下文
```

### 关键规则

- `taskId` **始终由服务端生成**，客户端不能提供
- `contextId` / `taskId` 不匹配时，Agent **必须拒绝**
- Client 可以只发 `contextId` 不发 `taskId`，在同一会话中开启新任务
- 新任务可继承同一 `contextId` 下之前交互的上下文

> 📌 [对比 MCP](a2a-vs-mcp.md)：A2A 是有状态的、多轮的；MCP 通常是单次无状态调用。

## 消息流转

### 非流式请求/响应

```
Client → POST /message:send → Agent
Client ← { task: { id, contextId, status, artifacts } } ← Agent
```

### 流式传输（SSE）

```
Client → POST /message:stream → Agent
Client ← SSE: { "task": { "status": { "state": "TASK_STATE_WORKING" } } }
Client ← SSE: { "artifactUpdate": { "artifact": {...} } }
Client ← SSE: { "statusUpdate": { "status": { "state": "TASK_STATE_COMPLETED" } } }
```

需要 AgentCard 中 `capabilities.streaming` 为 `true`。

**StreamResponse** — 按存在哪个 JSON 成员来区分：
- `{ "task": {...} }` — 完整任务状态
- `{ "message": {...} }` — 消息响应
- `{ "statusUpdate": {...} }` — 状态更新
- `{ "artifactUpdate": {...} }` — 产出物更新

> ⚠️ 事件**必须按顺序**投递。同一 Task 允许多个并发流（广播给所有订阅者）。

### 多轮交互（需要输入）

```
1. Client 发送消息
2. Agent 返回 TASK_STATE_INPUT_REQUIRED + 澄清消息
3. Client 发送跟进消息（同一 taskId + contextId）
4. Agent 继续处理
```

## RPC 方法（v1.0）

| 方法 | HTTP/REST | JSON-RPC | gRPC |
|------|-----------|----------|------|
| **SendMessage** | `POST /message:send` | `SendMessage` | `SendMessage` |
| **SendStreamingMessage** | `POST /message:stream` | `SendStreamingMessage` | `SendStreamingMessage` |
| **GetTask** | `GET /tasks/{id}` | `GetTask` | `GetTask` |
| **ListTasks** | `GET /tasks` | `ListTasks` | `ListTasks` |
| **CancelTask** | `POST /tasks/{id}:cancel` | `CancelTask` | `CancelTask` |
| **SubscribeToTask** | `POST /tasks/{id}:subscribe` | `SubscribeToTask` | `SubscribeToTask` |
| **GetExtendedAgentCard** | `GET /extendedAgentCard` | `GetExtendedAgentCard` | `GetExtendedAgentCard` |
| **CreateTaskPushNotificationConfig** | `POST /tasks/{id}/pushNotificationConfigs` | ... | ... |

**v0.3 → v1.0 变更**: `message/send` → `SendMessage`，`message/stream` → `SendStreamingMessage`

**版本头**: 客户端必须发送 `A2A-Version: 1.0` 头。

## Agent 发现

### AgentCard — Agent 数字名片

| 字段 | 说明 |
|------|------|
| `name` | Agent 名称（必填） |
| `description` | 功能描述（必填） |
| `supportedInterfaces[]` | 服务端点（v1.0），含 url、protocolBinding、protocolVersion |
| `provider` | 组织信息 |
| `capabilities` | streaming, pushNotifications, stateTransitionHistory, extendedAgentCard |
| `securitySchemes` | API Key / HTTP Auth / OAuth2 / OpenID Connect / mTLS |
| `skills[]` | AgentSkill 数组，描述具体能力 |
| `defaultInputModes` | 输入格式 |
| `defaultOutputModes` | 输出格式 |

### 发现方式

1. **Well-Known URI**（推荐）: `GET https://{domain}/.well-known/agent-card.json`，遵循 RFC 8615
2. **注册中心**（企业/市场）: 查询 skills、tags、capabilities（无标准化 API）
3. **直接配置**（私有/开发）: 硬编码 URL、配置文件、环境变量

### AgentCard 签名

可选使用 JWS（RFC 7515）签名 + JCS 规范化（RFC 8785）进行防篡改验证。

## 关键注意点

### 1. v0.3 vs v1.0 破坏性变更

- 枚举值从 `kebab-case` → `SCREAMING_SNAKE_CASE`（如 `submitted` → `TASK_STATE_SUBMITTED`）
- 流事件移除了 `kind` 判别符，改用 JSON 成员名
- 移除了 `final` 布尔值
- 简单 UUID 替代复合 ID
- 方法名改为 CamelCase

### 2. JSON 字段命名

必须使用 **camelCase**（不是 proto 的 snake_case）

### 3. 消息持久化不保证

Agent 自行决定 `history[]` 中存储什么。客户端**不能依赖**通过流式传输收到所有状态消息。

### 4. 错误处理

标准化错误码：
- `TaskNotFoundError` → 404 / NOT_FOUND / -32001
- `TaskNotCancelableError` → 400 / FAILED_PRECONDITION / -32002
- `VersionNotSupportedError` → 400 / FAILED_PRECONDITION / -32009

### 5. 推送通知

基于 Webhook，使用 HTTP POST + `application/a2a+json`。客户端必须返回 2xx。Agent 必须尝试至少一次投递。

### 6. 扩展机制

在 AgentCard 中声明，通过 `A2A-Extensions` 头激活。必需扩展在不支持时应报错。

### 7. 认证

带外凭证获取，支持 OAuth2（Auth Code、Client Credentials、Device Code）、API Key、HTTP Auth、mTLS、OpenID Connect。

### 8. v1.0 新增：任务内授权

新增 `TASK_STATE_AUTH_REQUIRED` 状态，允许 Agent 在任务执行中途请求授权，支持委托链。

## 安全问题

基于 grith.ai 安全审计（2026.03）和社区分析，A2A v1.0 存在 **10 个已知安全缺口**：

1. **无 Prompt 注入防御** — 协议层零机制检测/防止跨 Agent 注入
2. **AgentCard 签名可选** — `MAY` 而非 `MUST`，可被伪造
3. **执行不透明** — 调用方无法查看远程 Agent 内部操作
4. **工具调用无预审** — 无协议级策略门控
5. **无用户同意状态** — 只有 `input_required`，不够精细
6. **授权模型不统一** — 每个部署自行实现
7. **无 Token 生命周期** — 窃取的 OAuth Token 可永久有效
8. **多轮会话走私** — Unit 42 构建 PoC 成功实现
9. **AgentCard 投毒** — Trustwave 展示了 "Agent in the Middle" 攻击
10. **路线图未解决** — 聚焦治理/SDK，未提及授权标准化

> 详见 [A2A 安全分析](a2a-security-analysis.md)

## 相关页面

- [a2a-task-state-machine](a2a-task-state-machine.md) — Task 状态机详解
- [a2a-vs-mcp](a2a-vs-mcp.md) — A2A 与 MCP 对比
- [a2a-security-analysis](a2a-security-analysis.md) — 安全问题深度分析
- [a2a-ecosystem](a2a-ecosystem.md) — 生态系统与工具链
