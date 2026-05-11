---
title: Hermes Agent vs Aider 提示词缓存设计对比
created: 2026-05-12
updated: 2026-05-12
type: comparison
tags: [inference, agent, comparison, open-source]
sources:
  - https://github.com/nicepkg/hermes-agent
  - https://github.com/Aider-AI/aider
---

# Hermes Agent vs Aider 提示词缓存设计对比

## 概述

对比 Hermes Agent 和 Aider 两个开源 AI Agent 的 Prompt 构建策略与缓存优化设计。两者都高度重视 Prompt Caching 对成本和延迟的影响，但设计哲学不同：**Aider 追求 Anthropic 单一 Provider 上的极致多层递进缓存**，**Hermes 追求系统提示词的 byte 级冻结 + 全 Provider 覆盖**。

一句话结论：Aider 在 Anthropic 上缓存命中率更高（3 层递进断点 + cache warming），Hermes 的缓存设计更通用（同时适配 Anthropic、OpenAI、OpenRouter、本地模型），但缺少 cache warming 是最大短板。

## Hermes Agent 的 Prompt 结构与缓存设计

### 提示词组装顺序

```
[Agent Identity (SOUL.md / DEFAULT_AGENT_IDENTITY)]     ← 固定，几乎不变
[Tool-Use 行为指导 (条件注入：Memory/Search/Skills)]     ← 工具集确定后固定
[Model-Specific Enforcement (GPT/Codex/Gemini 约束)]    ← 模型确定后固定
[用户/网关 system_message]                              ← 可选，会话级固定
[MEMORY.md 快照 + USER.md 快照 + 外部记忆 Provider]     ← 会话开始时冻结
[Skills 索引 (两层缓存：LRU + 磁盘快照)]               ← 工具集确定后固定
[上下文文件 (.hermes.md > AGENTS.md > .cursorrules)]    ← 会话级固定
[时间戳 + Session ID + 模型名 + Provider]               ← 会话开始时冻结
[环境检测 (WSL/Termux)]                                 ← 系统级固定
[平台格式提示 (Telegram/Discord/WhatsApp/CLI)]           ← 平台级固定
  ↓ 组装成消息序列
[System Message (上面全部)]    ← cache breakpoint #1
[Tool Definitions (字母排序)]  ← 独立参数传递
[对话历史]                     ← cache breakpoint #2-4 (最后 3 条消息)
[当前用户消息]                 ← 每轮都变
```

### 缓存优化措施

**✅ 做对的：**

1. **系统提示词会话级冻结** — `_build_system_prompt()` 结果缓存在 `self._cached_system_prompt`，整个会话期间不重建。只有在上下文压缩（compression）时才通过 `_invalidate_system_prompt()` 强制重建
2. **SQLite 持久化精确文本** — 系统提示词的每个字节都存入 `session_db`，后续轮次直接读取，保证 **byte 级一致的前缀匹配**
3. **Anthropic `cache_control: {"type": "ephemeral"}` 标记** — `prompt_caching.py` 实现 "system_and_3" 策略：系统消息 + 最后 3 条非系统消息，共 4 个断点（Anthropic 允许的最大值）
4. **Tool 定义字母排序** — `registry.get_definitions()` 用 `sorted(tool_names)` 保证工具 schema 跨 API 调用完全一致
5. **动态内容隔离** — `ephemeral_system_prompt` 不放进缓存前缀，在 API 调用时才注入到 `effective_system`。Plugin 上下文注入到用户消息而非系统提示词
6. **多 Provider 适配** — 原生 Anthropic 用内层 content block 标记；OpenRouter 用消息 envelope 标记；OpenAI 用 `prompt_cache_key=session_id`；Qwen Portal 单独适配
7. **JSON 序列化规范化** — `json.dumps(args, separators=(",",":"), sort_keys=True)` 保证 tool call 参数的 bit 级一致性，对本地模型（llama.cpp、vLLM、Ollama）的 KV cache 复用至关重要
8. **Deep copy 防污染** — 注入 `cache_control` 标记时做 `copy.deepcopy(api_messages)`，不修改原始消息列表

**⚠️ 不够理想的：**

1. **无 Cache Warming** — 没有周期性 heartbeat 请求保活 Anthropic 5 分钟 TTL。如果用户思考超过 5 分钟，系统提示词的缓存全部失效，这是最大的短板
2. **环境/平台信息在系统提示词中** — WSL/Termux 检测、平台格式提示虽固定但增加了前缀长度，对非 Anthropic Provider 无法利用显式缓存
3. **Memory 写入有延迟** — 中途写入的 memory 要到下次 compression 重建才会反映到系统提示词中

### 缓存标记实现核心逻辑

```python
# agent/prompt_caching.py — "system_and_3" 策略
def apply_anthropic_cache_control(messages, native_anthropic=True, cache_ttl="5m"):
    marker = {"type": "ephemeral"}
    if cache_ttl == "1h":
        marker["ttl"] = "1h"
    
    # 断点 1: 系统消息 (第一条 role=="system")
    # 断点 2-4: 最后 3 条非系统消息 (滚动窗口)
    # native_anthropic=True → 标记在 content block 内层
    # native_anthropic=False → 标记在 message envelope 外层 (OpenRouter)
```

### 多 Provider 缓存策略

| Provider | 缓存方式 | 缓存策略 |
|----------|---------|---------|
| Anthropic 原生 | `cache_control` 内层标记 | system + last 3 messages |
| OpenRouter (Claude) | `cache_control` envelope 标记 | system + last 3 messages |
| OpenAI (GPT/Codex) | `prompt_cache_key=session_id` | 自动前缀匹配 |
| Qwen Portal | `cache_control` system only | 仅系统消息 |
| 本地模型 (vLLM/Ollama) | 无显式标记，靠 JSON 规范化 | 自然前缀命中 |

## Aider 的 Prompt 结构与缓存设计

### 分层缓存架构

```
[SYSTEM + EXAMPLES]     ← cache breakpoint #1 (极少变化)
[REPO MAP + 只读文件]    ← cache breakpoint #2 (切换文件时才变)
[CHAT FILES]            ← cache breakpoint #3 (编辑时才变)
[DONE MESSAGES]         ← 对话历史 (累积增长)
[CUR MESSAGES]          ← 当前轮 (每轮都变)
[REMINDER]              ← 提醒
```

### 缓存优化措施

**✅ 做对的：**

1. **3 层递进缓存断点** — 从稳定到不稳定逐层递进，每层有独立的 `cache_control: "ephemeral"` 标记。代码变化只破坏第 3 层缓存，前两层仍命中。这是 Aider 最大的设计亮点
2. **Cache Warming 守护线程** — 每 5 分钟发 `max_tokens=1` 请求保活 Anthropic 5 分钟 TTL（`AIDER_CACHE_KEEPALIVE_DELAY` 可调）。这是 Hermes 没有的关键能力
3. **`cacheable_messages()` 方法** — 只返回到最后一个断点的消息，最小化 cache warming 的 token 开销
4. **绝不修改前缀** — 所有变化都 append 到末尾，不 prepend 不重排
5. **按模型配置缓存** — `model-settings.yml` 里 Anthropic 模型设 `cache_control: true`，发送 `anthropic-beta: prompt-caching-2024-07-31` header

**⚠️ 局限性：**

1. **仅适配 Anthropic** — `cache_control: true` 只在 Anthropic 模型上启用，不覆盖 OpenAI、OpenRouter 等其他 Provider
2. **无记忆系统** — 没有跨会话的记忆/知识库系统，不存在 memory 写入影响缓存的问题（但也没有这个能力）
3. **无 JSON 规范化** — 没有对 tool call 参数做 Hermes 那样的 `sort_keys` 规范化

## 核心对比表

| 维度 | Hermes Agent | Aider |
|------|-------------|-------|
| **缓存设计哲学** | 系统提示词 byte 级冻结 + 全 Provider | 多层递进断点 + Anthropic 深度优化 |
| **缓存断点数** | 4 个（系统 + 最后 3 条消息） | 3 层递进（SYSTEM → FILES → CHAT） |
| **Cache Warming** | ❌ 无 | ✅ 每 5 分钟 heartbeat |
| **Provider 覆盖** | Anthropic + OpenAI + OpenRouter + Qwen + 本地模型 | 仅 Anthropic |
| **Tool 定义排序** | ✅ 字母排序 (`sorted()`) | 未特别排序 |
| **系统提示词稳定性** | 会话级冻结 + SQLite 持久化 + byte 级一致 | 按层分离，最底层极少变化 |
| **动态内容处理** | 隔离到 ephemeral prompt / 用户消息 | 从不注入系统层 |
| **JSON 规范化** | ✅ `sort_keys + 去空格` | ❌ 无 |
| **时间戳** | 冻结在首次构建时 | 不注入 prompt |
| **记忆系统影响** | memory 写入延迟到压缩时才重建 | 无记忆系统 |
| **本地模型优化** | ✅ JSON 规范化 + 前缀一致 | 无显式优化 |

## 缓存命中率场景分析

### 场景 1: 连续多轮对话

|| Hermes | Aider |
|---|---|---|
| System prompt | ✅ 每轮命中（冻结） | ✅ 每轮命中（不变） |
| Tool definitions | ✅ 每轮命中 | ✅ 每轮命中 |
| 早期对话历史 | ✅ 最后 3 条标记缓存 | ✅ 断点 2 覆盖 |
| 新增内容 | ✅ 仅当前消息变 | ✅ 仅 CUR MESSAGES 变 |

**结论：** 连续多轮对话两者表现相当。

### 场景 2: 用户思考超过 5 分钟（idle gap）

|| Hermes | Aider |
|---|---|---|
| 5 分钟后缓存 | ❌ **全部失效**（TTL 到期，无 warming） | ✅ **仍然命中**（warming 保活） |
| 恢复后首轮成本 | 全额计费 | 缓存价格（90% off） |

**结论：** Aider 明显优于 Hermes。这是 Hermes 最大的缓存短板。假设系统提示词 10K tokens、对话历史 20K tokens，idle 后 Hermes 每次多付 $0.09（Claude Sonnet 价格），对频繁使用的用户累积成本显著。

### 场景 3: 代码文件变动

|| Hermes | Aider |
|---|---|---|
| 系统提示词 | ✅ 不受影响 | ✅ 不受影响（第 1 层） |
| 文件上下文 | 变化反映在工具返回中 | 仅第 2-3 层失效，第 1 层仍命中 |
| 缓存命中 | 系统级仍命中 | 系统级 + examples 仍命中 |

**结论：** Aider 的分层粒度更细，代码变化时保留更多缓存。Hermes 的文件内容通过工具调用返回（不在系统提示词中），系统级缓存不受影响。

### 场景 4: 跨 Provider

|| Hermes | Aider |
|---|---|---|
| Anthropic 原生 | ✅ 4 断点 | ✅ 3 断点 + warming |
| OpenAI (GPT/Codex) | ✅ `prompt_cache_key` | ❌ 未适配 |
| OpenRouter + Claude | ✅ envelope 标记 | ❌ |
| 本地模型 (vLLM/Ollama) | ✅ JSON 规范化 | 部分命中（无规范化） |

**结论：** Hermes 覆盖面远超 Aider，这是其核心优势。对于需要在不同 Provider 间切换的用户，Hermes 的通用缓存设计价值很大。

## 成本影响量化估算

以 Claude Sonnet 4 (2025 价格) 为例，假设系统提示词 8K tokens + 工具定义 5K tokens + 对话历史 15K tokens：

| 场景 | 无缓存成本 | Hermes 缓存后 | Aider 缓存后 |
|------|-----------|-------------|-------------|
| 连续多轮 (10 轮) | $1.50 | $0.45 (70%↓) | $0.30 (80%↓) |
| 5 分钟 idle 后 | $1.50 | **$1.50** (0%↓) | $0.35 (77%↓) |
| 跨 Provider (OpenAI) | $0.60 | $0.30 (50%↓) | $0.60 (0%↓) |

## 对 Hermes 的改进建议

1. **添加 Cache Warming（ROI 最高）** — 参考 Aider 的 `cache_warming` 线程，在 idle 超过阈值时发 `max_tokens=1` 保活请求。预计改动 ~50 行代码，但可消除 Hermes vs Aider 最大的缓存差距
2. **考虑分层缓存断点** — 将 MEMORY/USER 快照作为独立断点，与系统身份分离，memory 更新时不必放弃整个系统提示词缓存
3. **追踪缓存命中统计** — 解析 `CacheCreationInputTokens` / `CacheReadInputTokens`，给用户展示缓存命中率和节省金额

## 设计哲学总结

```
Aider:    "为 Anthropic 做到极致"     — 单 Provider 深度优化，3 层递进 + warming
Hermes:   "在所有 Provider 上都好"    — 广度优先，byte 级冻结 + JSON 规范化 + 多后端适配
```

Aider 的 3 层递进设计在 Anthropic 上确实更优，尤其有 cache warming 保底。但 Hermes 的多 Provider 适配使其在 OpenAI、OpenRouter、本地模型等场景下也能享受缓存红利，这符合 Hermes "provider-agnostic" 的核心定位。

**理想状态：** Hermes 在保持多 Provider 适配的同时，为 Anthropic Provider 增加 cache warming（~50 行代码改动），即可达到 Aider 的缓存保活能力，同时保留自己的跨平台优势。

## 相关页面

- [[prompt-cache-design-for-agents]] — Agent 提示词缓存设计模式（OpenCode/Aider/Claude Code 对比）
- [[vllm-prefix-caching]] — vLLM KV Cache 与 Prefix Caching 详解
- [[hermes-agent-memory-architecture]] — Hermes Agent 四层记忆系统架构
- [[anthropic-context-engineering]] — 高效上下文工程
