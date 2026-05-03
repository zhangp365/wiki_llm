---
title: A2A Task 状态机
created: 2026-05-02
updated: 2026-05-02
type: concept
tags: [protocol, agent, architecture]
sources: [raw/articles/a2a-official-spec.md]
---

# A2A Task 状态机

> Task 是 A2A 协议的核心工作单元。理解状态机是正确实现 A2A 的基础。

## 状态列表

### 非终态（Non-Terminal）

| 状态 | 枚举值 | 说明 |
|------|--------|------|
| 已提交 | `TASK_STATE_SUBMITTED` | 任务已创建，等待处理 |
| 处理中 | `TASK_STATE_WORKING` | Agent 正在执行任务 |
| 需要输入 | `TASK_STATE_INPUT_REQUIRED` | Agent 需要更多信息才能继续 |
| 需要授权 | `TASK_STATE_AUTH_REQUIRED` | Agent 需要授权才能继续（v1.0 新增） |

### 终态（Terminal）

| 状态 | 枚举值 | 说明 |
|------|--------|------|
| 已完成 | `TASK_STATE_COMPLETED` | 任务成功完成 |
| 已失败 | `TASK_STATE_FAILED` | 任务执行失败 |
| 已取消 | `TASK_STATE_CANCELLED` | 任务被取消 |
| 未知 | `TASK_STATE_UNKNOWN` | 遗留/错误回退状态 |

## 状态转换图

```
                    ┌──────────────────┐
                    │ TASK_STATE_      │
                    │   SUBMITTED      │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
           ┌──────►│ TASK_STATE_      │◄──────┐
           │        │   WORKING        │       │
           │        └──┬──┬──┬──┬─────┘       │
           │           │  │  │  │              │
           │     ┌─────┘  │  │  └──────┐      │
           │     │        │  │         │      │
           │     ▼        │  ▼         ▼      │
           │  ┌────────┐  │ ┌─────────────┐   │
           │  │COMPLETED│  │ │INPUT_REQUIRED│  │
           │  └────────┘  │ └──────┬──────┘   │
           │              │        │           │
           │     ┌────────┐  ┌─────▼──────┐   │
           │     │ FAILED │  │AUTH_REQUIRED│   │
           │     └────────┘  │(v1.0 only) │   │
           │                 └─────┬──────┘   │
           │                       │           │
           │  ┌────────────┐       │           │
           └──│ CANCELLED  │◄──────┴───────────┘
              └────────────┘
              (任何非终态 → cancelled)
```

## 状态转换规则

### 正向流转

| 起始状态 | 目标状态 | 触发条件 |
|----------|----------|----------|
| `submitted` | `working` | Agent 开始处理 |
| `working` | `completed` | 任务成功完成 |
| `working` | `failed` | 执行出错 |
| `working` | `input_required` | Agent 需要用户澄清 |
| `working` | `auth_required` | Agent 需要授权（v1.0） |
| `input_required` | `working` | 客户端提供所需信息 |
| `auth_required` | `working` | 客户端完成授权 |

### 取消

任何非终态都可以被取消：
- `submitted` → `cancelled`
- `working` → `cancelled`
- `input_required` → `cancelled`
- `auth_required` → `cancelled`

### v0.3 → v1.0 变更

| v0.3 | v1.0 |
|------|------|
| `submitted` | `TASK_STATE_SUBMITTED` |
| `working` | `TASK_STATE_WORKING` |
| `completed` | `TASK_STATE_COMPLETED` |
| `failed` | `TASK_STATE_FAILED` |
| `cancelled` | `TASK_STATE_CANCELLED` |
| `unknown` | `TASK_STATE_UNKNOWN` |
| `input-required` | `TASK_STATE_INPUT_REQUIRED` |
| _(不存在)_ | `TASK_STATE_AUTH_REQUIRED` |

## 实现注意点

1. **终态不可逆** — 一旦进入 `completed`/`failed`/`cancelled`，不能转换到其他状态
2. **多轮 input_required** — `input_required → working → input_required` 循环是合法的
3. **取消不一定即时** — Agent 可以在 `working` 状态下完成当前操作后再进入 `cancelled`
4. **unknown 是兜底** — 客户端遇到无法识别的状态值时应映射为 `unknown`
5. **状态更新必须投递** — 在流式模式中，每次状态变更都必须通过 SSE 事件通知客户端

## 相关页面

- [a2a-protocol](a2a-protocol.md) — A2A 协议完整详解
- [a2a-security-analysis](a2a-security-analysis.md) — 安全问题深度分析
