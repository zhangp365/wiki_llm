# ACP stdio 授权完整流程详解

> Agent Client Protocol (ACP) — 基于 JSON-RPC 2.0 over stdin/stdout 的 UI ↔ Agent 授权机制

---

## 核心架构前提

本地 Agent 作为 Editor 的**子进程**运行，通过 JSON-RPC over stdio 双向通信。

- 传输格式：**newline-delimited JSON**，每条消息是完整 JSON 对象，以 `\n` 结尾
- stdout：专用于 JSON-RPC 消息流（双向）
- stderr：专用于日志输出，不污染消息流
- 消息分两种类型：
  - **Request**（有 `id`）：需要对方回 Response，发起方**阻塞等待**
  - **Notification**（无 `id`）：单向推送，发起方不等待

---

## 完整流程总览

```
Editor UI (Client)          stdin / stdout            Agent 子进程
      |                           |                        |
      |── ① spawn 子进程 ─────────────────────────────────>|  进程启动，监听 stdin
      |                           |                        |
      |── initialize (Request) ──>|──────────────────────>|
      |<── agentCapabilities ─────|<─────────────────────|
      |                           |                        |
      |── session/new (Request) ──|──────────────────────>|
      |<── sessionId ─────────────|<─────────────────────|
      |                           |                        |
      |── session/prompt (Req) ───|──────────────────────>|  ③ 调用 LLM
      |                           |                        |  LLM 返回 tool_call
      |<── session/update (Notif) |<─────────────────────|  status: pending
      |  [UI 显示待授权]           |                        |
      |                           |                        |
      |<── request_permission ────|<─────────────────────|  ⑤ Agent 阻塞等待
      |  [弹出授权对话框]          |                        |  ⏸ 挂起
      |  用户点击 "允许一次"        |                        |
      |── Response (allow-once) ──|──────────────────────>|  Agent 解除阻塞
      |                           |                        |
      |<── tool_call_update ──────|<─────────────────────|  status: in_progress
      |<── (fs 反向调用) ─────────|<─────────────────────|  可选
      |<── tool_call_update ──────|<─────────────────────|  status: completed
      |                           |                        |
      |<── agent_message_chunk ───|<─────────────────────|  流式推送总结
      |<── session/prompt (Resp) ─|<─────────────────────|  stopReason: end_turn
      |  [Turn 完成]              |                        |
```

---

## 分阶段详解

### ① 进程启动（Spawn）

Editor 将 Agent 作为子进程启动，建立 stdio 管道：

```javascript
// Client 侧示例（TypeScript SDK）
import * as acp from "@agentclientprotocol/sdk";
import { Writable, Readable } from "node:stream";

const agentProcess = spawn("claude-agent-acp", [], { stdio: "pipe" });

const input  = Writable.toWeb(agentProcess.stdin);
const output = Readable.toWeb(agentProcess.stdout);

const conn = new acp.ClientSideConnection(
  (_agent) => new MyClient(),
  input,
  output
);
```

- Agent 进程启动后开始监听 stdin
- stderr 用于 Agent 内部日志，Client 可选择性读取展示

---

### ② initialize 握手

连接建立后，Client **必须**首先发送 `initialize`，双方交换能力表。

**Client → Agent（Request, id: 0）：**

```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "initialize",
  "params": {
    "protocolVersion": 1,
    "clientCapabilities": {
      "fs": {
        "readTextFile": true,
        "writeTextFile": true
      },
      "terminal": true
    },
    "clientInfo": {
      "name": "my-editor",
      "title": "My Editor",
      "version": "1.0.0"
    }
  }
}
```

**Agent → Client（Response, id: 0）：**

```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "result": {
    "protocolVersion": 1,
    "agentCapabilities": {
      "loadSession": true,
      "promptCapabilities": {
        "image": true,
        "embeddedContext": true
      },
      "mcpCapabilities": {
        "http": true
      }
    },
    "agentInfo": {
      "name": "my-agent",
      "version": "1.0.0"
    },
    "authMethods": []
  }
}
```

**关键字段说明：**

| 字段 | 含义 |
|------|------|
| `protocolVersion` | 双方必须达成一致，否则应关闭连接 |
| `fs.readTextFile / writeTextFile` | 告知 Agent 可以向 Client 发起 fs 反向调用 |
| `terminal` | 告知 Agent 可以使用 terminal/* 方法 |
| `authMethods` | Agent 支持的认证方式（OAuth 等） |

---

### ③ session/new — 创建会话

`initialize` 完成后，创建对话会话：

**Client → Agent（Request, id: 1）：**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "session/new",
  "params": {
    "cwd": "/home/user/project",
    "mcpServers": []
  }
}
```

**Agent → Client（Response, id: 1）：**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "sessionId": "sess_abc123def456"
  }
}
```

后续所有消息都携带此 `sessionId`。

---

### ④ session/prompt — 发送用户任务

**Client → Agent（Request, id: 2）：**

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "session/prompt",
  "params": {
    "sessionId": "sess_abc123def456",
    "prompt": [
      {
        "type": "text",
        "text": "帮我重构 main.py，提取公共函数"
      },
      {
        "type": "resource",
        "resource": {
          "uri": "file:///home/user/project/main.py",
          "mimeType": "text/x-python",
          "text": "def foo(): ...\ndef bar(): ..."
        }
      }
    ]
  }
}
```

> ⚠️ **重要**：`session/prompt` 的 Response **不会立即返回**，整个 turn 完成才回复。
> 中间所有进展通过 `session/update` **Notification** 推送。
> 此方法**不设超时**，复杂任务可能持续数分钟。

Agent 收到后调用 LLM，LLM 决定需要执行 tool call（如写文件）：

**Agent → Client（Notification，无 id）— 上报 tool_call 待授权：**

```json
{
  "jsonrpc": "2.0",
  "method": "session/update",
  "params": {
    "sessionId": "sess_abc123def456",
    "update": {
      "sessionUpdate": "tool_call",
      "toolCallId": "call_001",
      "title": "修改 main.py",
      "kind": "edit",
      "status": "pending"
    }
  }
}
```

`kind` 枚举值：

| 值 | 含义 |
|----|------|
| `read` | 读取文件或数据 |
| `edit` | 修改文件内容 |
| `delete` | 删除文件 |
| `execute` | 执行命令或代码 |
| `fetch` | 拉取外部数据 |
| `think` | 内部推理规划 |
| `search` | 搜索信息 |

Client 收到后，UI 显示"待授权"状态。

---

### ⑤ session/request_permission — 权限请求（核心阻塞流程）

这是整个授权流程的核心，是唯一让 Agent **完全阻塞挂起**的环节。

**Agent → Client（Request, id: 5）— Agent 发出后即挂起：**

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "method": "session/request_permission",
  "params": {
    "sessionId": "sess_abc123def456",
    "toolCall": {
      "toolCallId": "call_001",
      "title": "修改 main.py",
      "kind": "edit",
      "status": "pending",
      "locations": [
        { "path": "/home/user/project/main.py", "line": 1 }
      ]
    },
    "options": [
      {
        "optionId": "allow-once",
        "name": "允许一次",
        "kind": "allow_once"
      },
      {
        "optionId": "allow-always",
        "name": "始终允许",
        "kind": "allow_always"
      },
      {
        "optionId": "reject-once",
        "name": "拒绝",
        "kind": "reject_once"
      }
    ]
  }
}
```

**此时 Agent 进程挂起，等待 Client 回复，不执行任何操作。**

Client 弹出授权对话框，展示操作详情和选项按钮。

---

#### 用户确认的三种结果

**情况 A：用户点击"允许一次"**

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "outcome": {
      "outcome": "selected",
      "optionId": "allow-once"
    }
  }
}
```

→ Agent 解除阻塞，**继续执行此 tool call**，后续同类操作仍会再次请求授权。

---

**情况 B：用户点击"始终允许"**

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "outcome": {
      "outcome": "selected",
      "optionId": "allow-always"
    }
  }
}
```

→ Agent 解除阻塞，继续执行。Client 可本地记忆此偏好，后续同类操作**自动放行**，不再弹框。

---

**情况 C：用户点击"拒绝"**

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "outcome": {
      "outcome": "selected",
      "optionId": "reject-once"
    }
  }
}
```

→ Agent 解除阻塞，**跳过此 tool call**，将拒绝结果告知 LLM，LLM 决定是否有替代方案。

---

**情况 D：用户取消整个任务（prompt turn 被取消）**

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "outcome": {
      "outcome": "cancelled"
    }
  }
}
```

→ Agent 解除阻塞，**终止整个 prompt turn**，不再继续任何操作。

---

### ⑥ 权限授予后 — Agent 继续执行 tool call

Agent 收到允许响应后，依次推送状态更新：

**Step 1 — 推送 in_progress（Notification）：**

```json
{
  "jsonrpc": "2.0",
  "method": "session/update",
  "params": {
    "sessionId": "sess_abc123def456",
    "update": {
      "sessionUpdate": "tool_call_update",
      "toolCallId": "call_001",
      "status": "in_progress"
    }
  }
}
```

**Step 2（可选）— 反向调用 Client 文件系统：**

若 Agent 需要实际写入文件，且 Client 在 `initialize` 中声明了 `fs.writeTextFile` 能力，Agent 可向 Client 发起反向调用：

```json
{
  "jsonrpc": "2.0",
  "id": 10,
  "method": "fs/write_text_file",
  "params": {
    "path": "/home/user/project/main.py",
    "text": "def foo(): ...\n\ndef common(): ..."
  }
}
```

Client 执行后回复结果，Agent 继续。

**Step 3 — 推送 completed，附带结果内容（Notification）：**

```json
{
  "jsonrpc": "2.0",
  "method": "session/update",
  "params": {
    "sessionId": "sess_abc123def456",
    "update": {
      "sessionUpdate": "tool_call_update",
      "toolCallId": "call_001",
      "status": "completed",
      "content": [
        {
          "type": "diff",
          "path": "/home/user/project/main.py",
          "oldText": "def foo(): ...\ndef bar(): ...",
          "newText": "def foo(): ...\ndef bar(): ...\n\ndef common(): ..."
        }
      ]
    }
  }
}
```

tool_call 状态流转：

```
pending  →  in_progress  →  completed
                         →  failed
```

---

### ⑦ 流式推送总结 + Turn 结束

所有 tool call 完成后，Agent 将结果送回 LLM，LLM 生成总结，逐 token 流式推送：

**流式文本（Notification，多次）：**

```json
{
  "jsonrpc": "2.0",
  "method": "session/update",
  "params": {
    "sessionId": "sess_abc123def456",
    "update": {
      "sessionUpdate": "agent_message_chunk",
      "content": {
        "type": "text",
        "text": "已完成重构，提取了公共函数 `common()`..."
      }
    }
  }
}
```

**Turn 结束（Response，对应最初 session/prompt 的 id: 2）：**

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "stopReason": "end_turn"
  }
}
```

`stopReason` 枚举：

| 值 | 含义 |
|----|------|
| `end_turn` | 正常完成 |
| `max_tokens` | 达到 token 上限 |
| `cancelled` | 被取消 |

---

## 关键设计要点汇总

| 问题 | 答案 |
|------|------|
| `session/update` 和 `request_permission` 的本质区别？ | `update` 是 Notification（无 id，单向），`request_permission` 是 Request（有 id，Agent 必须阻塞等回应） |
| Agent 挂起期间 stdin/stdout 如何？ | stdin 仍然在监听（可接收取消信号），stdout 不输出任何内容 |
| Client 可以自动放行吗？ | 可以，Client 可根据用户偏好自动回复 allow/reject，不弹对话框 |
| 多个 tool call 如何处理？ | 每个 tool call 独立走一次 request_permission 流程（除非已被 allow-always） |
| prompt turn 期间能超时吗？ | 不应设超时，复杂任务 LLM 生成可能持续 5-10 分钟甚至更长 |
| 取消 prompt turn 时 pending 的权限请求怎么办？ | Client 必须对所有 pending 的 request_permission 回复 `outcome: "cancelled"` |
| fs 反向调用是否需要授权？ | 不需要单独授权，fs 能力在 initialize 阶段整体声明 |
| stderr 的内容是什么？ | Agent 进程的内部日志，不影响 stdout 消息流 |

---

## 消息类型速查

```
Client → Agent (Request)      需要 Agent 回 Response，Client 等待
  ├── initialize               握手，必须最先发
  ├── session/new              创建会话
  ├── session/prompt           发送用户任务（不设超时）
  └── session/cancel           取消当前 turn

Agent → Client (Request)      需要 Client 回 Response，Agent 等待（阻塞）
  ├── session/request_permission  ← 授权核心，Agent 在此挂起
  └── fs/read_text_file / fs/write_text_file  ← 反向文件操作

Agent → Client (Notification) 单向推送，无需回应
  └── session/update
       ├── sessionUpdate: "plan"                计划列表
       ├── sessionUpdate: "tool_call"           新 tool call（pending）
       ├── sessionUpdate: "tool_call_update"    状态更新（in_progress/completed）
       └── sessionUpdate: "agent_message_chunk" 流式文本
```

---

*参考：[Agent Client Protocol 官方文档](https://agentclientprotocol.com) | ACP GitHub: agentclientprotocol/agent-client-protocol*
