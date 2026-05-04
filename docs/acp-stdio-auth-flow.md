---
title: ACP stdio 授权流程详解
created: 2026-05-04
updated: 2026-05-04
type: concept
tags: [protocol, agent, tool-use]
sources: [raw/articles/acp-stdio-auth-flow.md]
---

# ACP stdio 授权流程详解

> 基于 JSON-RPC 2.0 over stdin/stdout 的 UI ↔ Agent 授权机制完整拆解。
> 详见 [[acp-protocol]] 了解全部接口总览。

## 传输基础

```
Editor UI (Client)    stdin / stdout    Agent 子进程
      |                    |                  |
      |── spawn 子进程 ──────────────────────>|  监听 stdin
      |── initialize ─────>|─────────────────>|
      |<── capabilities ──|<─────────────────|
```

- **stdout**：专用于 JSON-RPC 消息流
- **stderr**：专用于日志输出，不污染消息流
- **消息格式**：newline-delimited JSON，每条消息以 `\n` 结尾

## 完整生命周期

### ① spawn + initialize 握手

Client 必须**最先**发送 `initialize`，双方交换能力表：

- `protocolVersion` — 双方必须一致
- `clientCapabilities.fs.readTextFile / writeTextFile` — 声明支持反向文件操作
- `clientCapabilities.terminal` — 声明支持 terminal 方法
- `authMethods` — Agent 支持的认证方式（可空）

### ② session/new — 创建会话

```json
{ "method": "session/new", "params": { "cwd": "/path/to/project", "mcpServers": [] } }
```

返回 `sessionId`，后续所有消息携带此 ID。

### ③ session/prompt — 发送任务

```json
{
  "method": "session/prompt",
  "params": {
    "sessionId": "sess_xxx",
    "prompt": [
      { "type": "text", "text": "帮我重构 main.py" },
      { "type": "resource", "resource": { "uri": "file:///...", "text": "..." } }
    ]
  }
}
```

> ⚠️ Response **不会立即返回**，整个 turn 完成才回复。中间进展通过 `session/update` Notification 推送。

### ④ 权限请求（核心阻塞流程）

当 Agent 需要执行受限制操作时：

1. Agent 先推送 `session/update`（tool_call pending）— UI 显示待授权
2. Agent 发送 `session/request_permission`（Request，有 id）— **Agent 挂起等待**
3. UI 弹出授权对话框

**三种授权结果 + 一种取消：**

| 用户操作 | outcome | 效果 |
|----------|---------|------|
| 允许一次 | `selected` + `allow-once` | 执行本次，后续仍需授权 |
| 始终允许 | `selected` + `allow-always` | 执行本次 + Client 记忆偏好 |
| 拒绝 | `selected` + `reject-once` | 跳过此 tool call，告知 LLM |
| 取消任务 | `cancelled` | 终止整个 prompt turn |

### ⑤ 权限授予后执行

Agent 收到允许后依次推送：

```
session/update: tool_call_update in_progress
  → (可选) fs/write_text_file 反向调用
session/update: tool_call_update completed (附 diff 内容)
```

### ⑥ 流式总结 + Turn 结束

```
session/update: agent_message_chunk (多次，逐 token)
session/prompt Response: { "stopReason": "end_turn" | "max_tokens" | "cancelled" }
```

## tool_call kind 枚举

| 值 | 含义 |
|----|------|
| `read` | 读取文件或数据 |
| `edit` | 修改文件内容 |
| `delete` | 删除文件 |
| `execute` | 执行命令或代码 |
| `fetch` | 拉取外部数据 |
| `think` | 内部推理规划 |
| `search` | 搜索信息 |

## 消息方向速查

```
Client → Agent (Request, Client 等待)
  ├── initialize          握手，必须最先发
  ├── session/new         创建会话
  ├── session/prompt      发送任务（不设超时）
  └── session/cancel      取消当前 turn

Agent → Client (Request, Agent 阻塞等待)
  ├── session/request_permission   ← 授权核心
  └── fs/read_text_file / fs/write_text_file  ← 反向文件操作

Agent → Client (Notification, 无需回应)
  └── session/update
       ├── plan                  计划列表
       ├── tool_call             新 tool call（pending）
       ├── tool_call_update      状态更新
       └── agent_message_chunk   流式文本
```

## 参见

- [[acp-protocol]] — ACP 全部接口分类总览
- [[a2a-protocol]] — Agent 间通信协议（与 ACP 互补）
- [[a2a-vs-mcp]] — 协议对比分析
