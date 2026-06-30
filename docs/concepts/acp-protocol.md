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
| `initialize` | Client → Agent | 握手，交换双方能力表（clientCapabilities / agentCapabilities），必须最先发送。详见 [Initialization](https://agentclientprotocol.com/protocol/v1/initialization) |

### 二、会话管理

| 接口 | 方向 | 说明 |
|------|------|------|
| `session/new` | Client → Agent (Request) | 创建新会话，返回 sessionId。详见 [Session Setup](https://agentclientprotocol.com/protocol/v1/session-setup) |
| `session/load` | Client → Agent (Request) | 加载历史会话，恢复上下文。详见 [Session Setup](https://agentclientprotocol.com/protocol/v1/session-setup) |
| `session/prompt` | Client → Agent (Request) | 发送用户任务，启动执行循环，**不设超时**，整个 turn 完成才回复。详见 [Prompt Turn](https://agentclientprotocol.com/protocol/v1/prompt-turn) |
| `session/cancel` | Client → Agent (Request) | 取消当前 prompt turn。详见 [Prompt Turn](https://agentclientprotocol.com/protocol/v1/prompt-turn) |
| `session/set_mode` | Client → Agent (Request) | 切换工作模式（ask / code / architect），详见 [Session Modes](https://agentclientprotocol.com/protocol/v1/session-modes) |
| `session/set_config_option` | Client → Agent (Request) | 设置配置选项（包括模型、推理级别等），详见 [Session Config Options](https://agentclientprotocol.com/protocol/v1/session-config-options) |

### 三、扩展机制

| 接口 | 方向 | 说明 |
|------|------|------|
| `ext/method` | 双向 (Request) | 扩展 RPC 命令，支持自定义功能（调试、插件等）。详见 [Extensibility](https://agentclientprotocol.com/protocol/v1/extensibility) |
| `ext/notification` | 双向 (Notification) | 扩展事件通知，轻量级单向同步。详见 [Extensibility](https://agentclientprotocol.com/protocol/v1/extensibility) |

### 四、Agent → Client（反向消息）

| 接口 | 类型 | 说明 |
|------|------|------|
| `session/update` | Notification | 执行过程状态推送（tool_call pending/completed、流式文本等）。详见 [Tool Calls](https://agentclientprotocol.com/protocol/v1/tool-calls) |
| `session/request_permission` | **Request** | 🔑 **权限请求核心**：Agent 发出后**阻塞挂起**，等待 UI 回复授权结果。详见 [Authentication](https://agentclientprotocol.com/protocol/v1/authentication) |
| `fs/read_text_file` / `fs/write_text_file` | Request | 反向文件系统调用，需 Client 在 initialize 时声明 fs 能力。详见 [File System](https://agentclientprotocol.com/protocol/v1/file-system) |

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

- **ACP vs MCP**：MCP 是模型与工具的协议（工具服务器），ACP 是 UI 与 Agent 的协议（客户端-代理），详见 [A2A vs MCP](../comparisons/a2a-vs-mcp.md)
- **ACP vs A2A**：[A2A 协议](a2a-protocol.md) 是 Agent 间通信协议（点对点），ACP 是 Agent 与宿主编排器的协议
- 三者互补：MCP 管工具，ACP 管编排，[A2A 协议](a2a-protocol.md) 管协作

## 参考

### 官方文档
- **主页**：[Agent Client Protocol](https://agentclientprotocol.com)
- **Initialization**：[初始化握手](https://agentclientprotocol.com/protocol/v1/initialization)
- **Session Setup**：[会话创建与加载](https://agentclientprotocol.com/protocol/v1/session-setup)
- **Prompt Turn**：[提示词轮次](https://agentclientprotocol.com/protocol/v1/prompt-turn)
- **Session Modes**：[切换工作模式](https://agentclientprotocol.com/protocol/v1/session-modes)
- **Session Config Options**：[设置配置选项（包括模型）](https://agentclientprotocol.com/protocol/v1/session-config-options)
- **Tool Calls**：[工具调用执行](https://agentclientprotocol.com/protocol/v1/tool-calls)
- **File System**：[文件系统访问](https://agentclientprotocol.com/protocol/v1/file-system)
- **Authentication**：[权限请求与授权](https://agentclientprotocol.com/protocol/v1/authentication)
- **Extensibility**：[扩展机制](https://agentclientprotocol.com/protocol/v1/extensibility)

## 相关链接
- [[acp-stdio-auth-flow]] — ACP stdio 授权流程详解
- [[a2a-protocol]] — Agent 间通信协议（A2A Protocol）
- [[a2a-vs-mcp]] — ACP 与 A2A/MCP 协议对比分析
