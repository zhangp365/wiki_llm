---
title: ACP（Agent Client Protocol）接口总览
created: 2026-05-04
updated: 2026-05-04
type: concept
tags: [protocol, agent, tool-use]
sources: [raw/articles/acp-interfaces-summary.md, raw/articles/acp-stdio-auth-flow.md]
---

# ACP（Agent Client Protocol）接口总览

ACP 是基于 **JSON-RPC 2.0** 的双向通信协议，用于 UI 客户端（如 TensorChat）与 Agent 子进程（如 OpenCode）之间的结构化交互。底层传输为 newline-delimited JSON，通过 stdin/stdout 管道通信，stderr 专用于日志。

## 核心设计

- **消息两类**：Request（有 `id`，需回应，发起方阻塞等待）与 Notification（无 `id`，单向推送）
- **双向对称**：Client 和 Agent 都可以发起 Request
- **进程模型**：Agent 作为 Editor 的子进程运行

## 接口分类

### 一、生命周期初始化

| 接口 | 方向 | 说明 |
|------|------|------|
| `initialize` | Client → Agent | 握手，交换双方能力表（clientCapabilities / agentCapabilities），必须最先发送 |

### 二、会话管理

| 接口 | 方向 | 说明 |
|------|------|------|
| `session/new` | Client → Agent (Request) | 创建新会话，返回 sessionId |
| `session/load` | Client → Agent (Request) | 加载历史会话，恢复上下文 |
| `session/prompt` | Client → Agent (Request) | 发送用户任务，启动执行循环，**不设超时**，整个 turn 完成才回复 |
| `session/cancel` | Client → Agent (Request) | 取消当前 prompt turn |
| `session/setMode` | Client → Agent (Request) | 切换工作模式（Chat / Agent） |
| `session/setModel` | Client → Agent (Request) | 切换当前会话绑定的 LLM 模型 |

### 三、扩展机制

| 接口 | 方向 | 说明 |
|------|------|------|
| `ext/method` | 双向 (Request) | 扩展 RPC 命令，支持自定义功能（调试、插件等） |
| `ext/notification` | 双向 (Notification) | 扩展事件通知，轻量级单向同步 |

### 四、Agent → Client（反向消息）

| 接口 | 类型 | 说明 |
|------|------|------|
| `session/update` | Notification | 执行过程状态推送（tool_call pending/completed、流式文本等） |
| `session/request_permission` | **Request** | 🔑 **权限请求核心**：Agent 发出后**阻塞挂起**，等待 UI 回复授权结果 |
| `fs/read_text_file` / `fs/write_text_file` | Request | 反向文件系统调用，需 Client 在 initialize 时声明 fs 能力 |

## session/update 的 sessionUpdate 类型

| 值 | 含义 |
|----|------|
| `plan` | 计划列表 |
| `tool_call` | 新 tool call（pending 状态） |
| `tool_call_update` | 状态更新（in_progress / completed / failed） |
| `agent_message_chunk` | 流式文本 token |

## 权限请求流程要点

`request_permission` 是唯一让 Agent 完全阻塞的环节：

- Agent 发送请求后挂起，stdin 仍在监听（可接收取消信号）
- Client 弹出授权对话框，用户可选：允许一次 / 始终允许 / 拒绝 / 取消整个任务
- "始终允许"后 Client 本地记忆偏好，同类操作后续自动放行
- 取消 prompt turn 时，必须对所有 pending 的 request_permission 回复 `outcome: "cancelled"`

## 关键设计决策

| 问题 | 方案 |
|------|------|
| fs 反向调用是否需要授权？ | 不需要，fs 能力在 initialize 阶段整体声明 |
| 多个 tool call 怎么处理？ | 每个 tool call 独立走一次 request_permission（除非已被 allow-always） |
| prompt turn 能设超时吗？ | 不应设超时，复杂任务可能持续 5-10 分钟 |
| tool_call 状态流转 | `pending → in_progress → completed / failed` |

## 与相关协议的关系

- **ACP vs MCP**：MCP 是模型与工具的协议（工具服务器），ACP 是 UI 与 Agent 的协议（客户端-代理）
- **ACP vs A2A**：[[a2a-protocol]] 是 Agent 间通信协议（点对点），ACP 是 Agent 与宿主编排器的协议
- 三者互补：MCP 管工具，ACP 管编排，[[a2a-protocol]] 管协作

## 参考实现

- **Client 侧**：TensorChat（Electron 应用）
  - `acpProcessManager.ts` — 进程生命周期
  - `acpSessionManager.ts` — 会话管理
  - `acpProvider.ts` — prompt 执行、权限处理
- **Agent 侧**：OpenCode
  - `server/routes/global.ts` — initialize
  - `server/routes/session.ts` — 会话控制
  - `server/routes/permission.ts` — 权限网关
- **SDK**：`@agentclientprotocol/sdk`
- **官方文档**：https://agentclientprotocol.com
