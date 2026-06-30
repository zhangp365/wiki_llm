---
title: A2A vs MCP 对比
created: 2026-05-02
updated: 2026-05-02
type: comparison
tags: [protocol, agent, framework, comparison]
sources: [raw/articles/a2a-official-spec.md, raw/articles/a2a-community-research.md]
---

# A2A vs MCP 对比

> 两个协议解决不同层面的问题，互补而非竞争。

## 核心区别

| 维度 | MCP (Model Context Protocol) | A2A (Agent-to-Agent) |
|------|------------------------------|----------------------|
| **定义者** | Anthropic | Google / Linux Foundation |
| **通信方向** | Agent ↔ Tool / Resource | Agent ↔ Agent |
| **交互模式** | 结构化函数调用 | 对话式协作 |
| **状态管理** | 通常无状态 | 有状态，多轮 |
| **粒度** | 单次工具调用 | 复杂任务编排 |
| **典型用例** | 查 DB、调 API、用计算器 | 委托给账单 Agent、协调差旅 |
| **发现机制** | 无标准化 | AgentCard + Well-Known URI |
| **流式传输** | 无内置支持 | SSE 原生支持 |
| **任务管理** | 无 | 完整状态机 |

## 关系：分层协作

两个协议工作在不同层，可以组合使用：

```
用户 → A2A → 商店经理 Agent → A2A → 配件供应商 Agent
         │                        │
         └── MCP → 订单数据库      └── MCP → 库存系统
```

- **A2A**: Agent 之间的伙伴关系，互相委托任务
- **MCP**: Agent 使用底层工具和数据的能力

## 什么时候用哪个

### 用 MCP 的场景

- Agent 需要调用特定 API 或数据库
- 获取结构化数据（天气、股票、文档）
- 单次请求-响应模式足够
- 不需要跨 Agent 协调

### 用 A2A 的场景

- 需要 Agent 之间协商和协作
- 任务需要多轮对话才能完成
- 需要跨组织/系统委托任务
- 任务有复杂状态（进行中、需要输入、需要授权）

### 两者都用的场景

- 一个编排 Agent 通过 A2A 调度子 Agent
- 每个 Agent 内部通过 MCP 访问各自工具

## 社区讨论

### "为什么不直接用 MCP 做一切？"

**学派 1**: 把 Agent 当工具，通过 MCP 暴露，不需要 A2A 的复杂性。

**学派 2**: MCP 只用于工具/数据/上下文访问；Agent 间通信保持独立。

> 类比：就像 CPU↔GPU 通信和进程间通信是不同层面的事情。REST、WebSocket、GraphQL 共存，各有用途。

### 采纳现状（2026.05）

| 方面 | MCP | A2A |
|------|-----|-----|
| 客户端分发 | Claude Desktop、Cursor、大量 IDE | 零主流客户端采用 |
| 公开 Agent/工具 | 大量 | ~100+，一半是 demo |
| 生态成熟度 | 较成熟 | 早期 |
| 核心优势 | 开发者立即可用 | 理论上更灵活 |

### 关键洞察

> "未来模型会原生互相通信 — 不需要将输出向量压缩为 token 列表、跨越工具调用协议、重新 tokenize。当 tokenizer 和向量维度跨模型对齐时，连接将是直接的。"

## 相关页面

- [[a2a-protocol]] — A2A 协议完整详解
- [[a2a-ecosystem]] — A2A 生态工具链
