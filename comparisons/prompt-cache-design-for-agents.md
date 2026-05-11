---
title: Agent 提示词缓存设计模式与业界对比
created: 2026-05-11
updated: 2026-05-11
type: comparison
tags: [inference, agent, comparison, open-source]
sources: []
---

# Agent 提示词缓存设计模式与业界对比

## 概述

分析了 OpenCode、Aider、Claude Code、Cursor 等 AI 编程 Agent 的提示词模板设计，看它们是否按缓存最大命中思路来构建。结论：**Aider 是开源社区缓存设计最极致的标杆**，OpenCode 有基本的缓存意识但不够系统化。缓存友好的 prompt 设计能带来 70-90% 的成本降低和 75%+ 的延迟降低。

## OpenCode 的 Prompt 结构与缓存设计

### 提示词组装顺序

```
[Provider-specific Base Prompt (Anthropic版/OpenAI版)]
  ↓ 拼接
[Environment Info: 工作目录、git状态、平台、日期、目录列表]
  ↓ 拼接
[LSP 信息（如有配置）]
  ↓ 拼接
[Project Context: OpenCode.md + ContextPaths 指定的文件内容]
  ↓ 组装成消息序列
[System Message] → [Tool Definitions] → [History] → [Current User]
```

### 缓存优化措施

**✅ 做对的：**
- System message 标记 Anthropic `cache_control: "ephemeral"`
- 最后 3 条消息标记 `cache_control: "ephemeral"`
- 最后一个 tool 定义标记 `cache_control: "ephemeral"`
- 分别追踪 `CacheCreationTokens` / `CacheReadTokens`，按缓存价计费
- 按 Provider 分系统提示（Anthropic / OpenAI 各有专门 prompt）

**⚠️ 不够理想的：**
1. **环境信息直接拼进 System Prompt** — 工作目录、日期、git status 每次可能变化，破坏缓存前缀匹配
2. **ContextPaths 文件内容拼接进 Prompt** — 动态读取文件内容，每次文件变动使缓存失效
3. **OpenAI 端无主动缓存优化** — 没做 tool 定义的稳定排序等
4. **无 cache warming 机制** — 不像 Aider 有定时 keepalive 保活缓存

### 总结评价

OpenCode **有基本的缓存意识**（Anthropic 端做了 cache_control 标记），但 **不是专门为最大缓存命中率设计的**。动态内容混入 system prompt 是主要扣分项。

## Aider — 业界标杆

Aider 的 `ChatChunks` 架构是开源社区里缓存设计最完善的参考实现。

### 分层缓存架构

```
[SYSTEM + EXAMPLES]     ← cache breakpoint #1 (极少变化)
[REPO MAP + 只读文件]    ← cache breakpoint #2 (切换文件时才变)
[CHAT FILES]            ← cache breakpoint #3 (编辑时才变)
[DONE MESSAGES]         ← 对话历史
[CUR MESSAGES]          ← 当前轮 (每轮都变)
[REMINDER]              ← 提醒
```

### 核心设计

1. **3 层递进缓存断点**：从稳定到不稳定逐层递进，每层有独立的 `cache_control: "ephemeral"` 标记
2. **Cache warming 守护线程**：每 5 分钟发一个 `max_tokens=1` 的请求来保活 Anthropic 5 分钟 TTL，环境变量 `AIDER_CACHE_KEEPALIVE_DELAY` 可调
3. **`cacheable_messages()` 方法**：只返回到最后一个断点的消息，用于最小化 cache warming 的 token 开销
4. **绝对不在前缀中间插入新内容**：所有变化都 append 到末尾
5. **按模型配置缓存**：`model-settings.yml` 里 Anthropic 模型设 `cache_control: true`，发送 `anthropic-beta: prompt-caching-2024-07-31` header

## 缓存友好的 Prompt 设计原则

### 核心规则：前缀匹配

所有主流缓存实现（vLLM APC、SGLang RadixAttention、Anthropic Prompt Caching、OpenAI Automatic Caching）都是 **前缀匹配**。从序列开头逐 token 比较，一旦一个 token 不同，后续所有 token 的缓存全部失效。

### 最佳排列顺序

| 位置 | 放什么 | 原因 |
|------|--------|------|
| **最前** | System prompt | 所有请求共享 |
| **第二** | Tool definitions | 同一 tool set 的请求共享 |
| **第三** | 对话历史 | 累积增长，前缀重叠高 |
| **最后** | 当前用户消息 | 每轮都变 |

### 具体技巧

1. **Tool 定义按固定顺序序列化**（字母排序），避免 JSON 字段顺序变化破坏前缀
2. **System prompt 中不注入动态内容**（时间戳、请求 ID、随机数）
3. **只 append 新消息到末尾**，绝不 prepend 或重排
4. **做 cache warming**：周期性发 `max_tokens=1` 保活请求
5. **Tool pruning**：只包含相关工具，减少 unique 前缀长度

## 各平台缓存机制对比

| 平台 | 触发方式 | 最小 tokens | 折扣 | TTL |
|------|---------|------------|------|-----|
| **Anthropic** | `cache_control: "ephemeral"` 标记 | 1,024 | 缓存读 90% off | 5 分钟（命中续期） |
| **OpenAI** | 自动（>1,024 tokens） | 1,024 | 输入 50% off | 自动管理 |
| **vLLM APC** | `--enable-prefix-caching` flag | 16 (block size) | 无费用（自建） | 显存淘汰 |
| **SGLang** | 默认开启 | 1 (token级) | 无费用（自建） | LRU 淘汰 |

## Anthropic 官方缓存基准数据

| 场景 | 延迟降低 | 成本降低 |
|------|---------|---------|
| 100K token 缓存 | **79%** (11.5s→2.4s) | **90%** |
| 10-shot 多样本 | 31% | 86% |
| 10 轮多轮对话 | **75%** | 53% |

## Agent 工具调用缓存设计的关键洞察

Agent 场景是 prefix caching 的 **理想用例**，因为：
- System prompt + tool definitions（通常 3K-15K tokens）**每次请求完全相同**
- 多轮对话历史**累积增长，前缀重叠高**
- 单次工具调用的新增内容相对很小（~200-500 tokens）

但有一个重要衰减规律：**缓存命中率随对话轮次增加而递减**，因为每轮新增的 unique content 比例越来越大。SGLang 的 RadixAttention 因为能缓存生成输出，在多轮场景衰减更慢。

## 相关页面

- [[vllm-prefix-caching]] — vLLM 缓存命中逻辑详解
- [[anthropic-context-engineering]] — 高效上下文工程
- [[anthropic-building-effective-agents]] — 构建高效 Agent 五种模式
