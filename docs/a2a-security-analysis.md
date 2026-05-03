---
title: A2A 安全问题分析
created: 2026-05-02
updated: 2026-05-02
type: concept
tags: [protocol, agent, controversy, trend]
sources: [raw/articles/a2a-community-research.md]
---

# A2A 安全问题分析

> 基于 grith.ai 安全审计（2026.03）、Palo Alto Unit 42、Semgrep、Trustwave SpiderLabs、Solo.io 的研究。

## 核心结论

> "A2A 是一个通信协议，不是安全执行层。" — Solo.io

A2A v1.0 在协议层面**零防御** OWASP LLM Top #1（Prompt 注入）。安全完全依赖实现层，而协议没有提供标准化的安全模型。

## 10 个已知安全缺口

### 高危

| # | 漏洞 | 说明 | 来源 |
|---|------|------|------|
| 1 | **无 Prompt 注入防御** | 协议层零机制检测/防止跨 Agent 注入 | grith.ai |
| 2 | **AgentCard 签名可选** | `MAY` 而非 `MUST`，可被伪造 | Semgrep |
| 3 | **执行不透明** | 调用方无法查看远程 Agent 内部操作 | 设计缺陷 |
| 4 | **工具调用无预审** | 无协议级策略门控 | grith.ai |

### 中危

| # | 漏洞 | 说明 | 来源 |
|---|------|------|------|
| 5 | **无用户同意状态** | 只有 `input_required`，对敏感操作不够精细 | grith.ai |
| 6 | **授权模型不统一** | 每个部署自行实现，无标准可审计 | grith.ai |
| 7 | **无 Token 生命周期** | 窃取的 OAuth Token 可永久有效 | grith.ai |

### 已验证攻击

| # | 攻击方式 | 详情 | 来源 |
|---|----------|------|------|
| 8 | **多轮会话走私** | Unit 42 构建 PoC：跨轮次的隐藏指令导致未经授权的股票购买，对终端用户不可见 | Unit 42 |
| 9 | **AgentCard 投毒** | Trustwave 展示伪造 AgentCard 通过夸大描述赢得 LLM-as-judge 路由，然后篡改结果（如汇率转换） | Trustwave |
| 10 | **路线图未解决** | 聚焦治理/SDK，未提及授权标准化、同意机制、Prompt 注入防御 | grith.ai |

## 攻击场景详解

### 场景 1: Agent in the Middle

```
用户 → 编排 Agent → [恶意 AgentCard 投毒] → 路由到恶意 Agent
                                     ↓
                          返回被篡改的结果
                          (如错误的汇率数据)
```

**工作原理**: 恶意 Agent 在 AgentCard 中夸大自己的能力描述，通过 LLM-as-judge 路由机制骗取任务委托。

### 场景 2: 跨轮次 Prompt 注入

```
Turn 1: 用户 → Agent A → Agent B（正常请求）
Turn 2: Agent B 返回含隐藏指令的结果
Turn 3: 用户 → Agent A → Agent A 执行了隐藏指令（如转账）
```

**工作原理**: 在多轮对话中，恶意 Agent 在正常响应中嵌入隐藏指令，后续轮次被利用。

### 场景 3: 授权链滥用

```
用户 → 编排 Agent (已授权) → 子 Agent A → 子 Agent B → 子 Agent C
                                                         ↓
                                                   未授权操作
```

**工作原理**: 委托链中的授权边界模糊，v1.0 的 `auth_required` 状态没有标准化的授权粒度。

## 缓解建议

1. **实现层加密 AgentCard** — 即使协议不要求，也要签名验证
2. **工具调用白名单** — 在 Agent 实现中限制可执行的操作
3. **人类在环** — 对敏感操作（支付、删除）强制人工确认
4. **会话审计日志** — 记录所有跨 Agent 消息用于事后审查
5. **Token 生命周期管理** — 设置合理的过期时间和刷新机制
6. **Prompt 注入检测** — 在 Agent 输入层添加检测过滤

## 学术研究

- **ArXiv 2505.12490**: 提出在 A2A 中添加 consent orchestration（同意编排）
- 方向：在协议层引入细粒度的用户同意状态机

## 相关页面

- [a2a-protocol](a2a-protocol.md) — A2A 协议完整详解
- [a2a-task-state-machine](a2a-task-state-machine.md) — Task 状态机
