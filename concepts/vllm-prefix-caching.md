---
title: vLLM KV Cache 与 Prefix Caching 命中机制
created: 2026-05-11
updated: 2026-05-11
type: concept
tags: [inference, open-source, architecture, comparison]
sources: []
---

# vLLM KV Cache 与 Prefix Caching 命中机制

## 概述

vLLM 的缓存体系由两层组成：底层的 PagedAttention 和上层的 Automatic Prefix Caching (APC)。在 Agent 多轮工具调用场景下，缓存命中率取决于共享前缀占总输入的比例，理论上最高可达 ~95%，但受 block 对齐粒度和调度策略限制，实际通常在 70-90%。

## 核心架构

### PagedAttention（基础层）

借鉴 OS 虚拟内存分页思想：
- KV cache 切分为固定大小的 block（默认 16 tokens）
- 每个请求维护一个 block table（类似页表）
- 避免连续内存分配造成的碎片化
- 支持跨请求共享 block（只读 COW 语义）

### Automatic Prefix Caching（APC，缓存命中层）

- 开启方式：`--enable-prefix-caching`
- 工作原理：对每个 block 计算 hash（基于 token 内容），新请求到来时逐 block 匹配已缓存 hash
- 匹配到的 block 直接复用，跳过 prefill 计算
- **粒度：block 级别（16 tokens 对齐）**，不是 token 级别

## Agent 工具调用场景的缓存命中

Agent 场景非常适合 prefix caching，每次工具调用重复发送：
- System prompt（不变）
- Tool definitions/Schema（不变，通常 3K-15K tokens）
- 之前的多轮对话历史（累积增长）

### 理论命中率公式

```
Hit Rate ≈ [2P + U×(N-1)] / [2P + U×(N+1)]
```
- P = 共享前缀大小（system prompt + tools）
- U = 每轮新增 unique tokens
- N = 轮次

### 实际命中率估算

| 场景 | 前缀占比 | vLLM APC | SGLang RadixAttention |
|------|---------|----------|----------------------|
| 大 system prompt + 多 tools (10K+ tokens) | 极高 | **90-95%** | **95-99%** |
| 中等 tools (3K-5K tokens) | 高 | **75-90%** | **85-95%** |
| 少量 tools (<1K tokens) | 中 | **50-70%** | **60-80%** |

### 多轮缓存命中率衰减

以 system + tools = 10K tokens，每轮新增 ~1K tokens 为例：

| 轮次 | 总输入 tokens | 可缓存 tokens | 命中率 |
|------|-------------|-------------|--------|
| Turn 2 | ~11K | ~10K | **~91%** |
| Turn 5 | ~14K | ~10K | **~71%** |
| Turn 10 | ~19K | ~10K | **~53%** |
| Turn 20 | ~29K | ~10K | **~34%** |

## vLLM vs SGLang 关键差距

| 特性 | vLLM APC | SGLang RadixAttention |
|------|----------|----------------------|
| 缓存粒度 | Block 级（16 tokens） | Token 级（Radix Tree） |
| 缓存生成输出 | ❌ 不支持 | ✅ 自动缓存 |
| 调度策略 | FCFS（无缓存亲和性） | Cache-aware 重排序 |
| 多轮对话 | 部分支持 | 完整支持 |
| 开启方式 | 需手动 flag | 默认开启 |

SGLang 在 Agent 工作负载上通常领先 vLLM **10-30% 吞吐量**，核心原因是 RadixAttention 会主动重排请求调度顺序来最大化缓存命中。实现基于 LMSYS 团队的 radix tree 数据结构，将 token 序列映射到 KV tensor，自动处理 prefix 匹配、复用和 LRU 淘汰。

## 已知 Bug 和问题

### 关键 Bug

1. **#42019** — `prompt_logprobs` 在 prefix caching 开启时结果不确定，依赖请求到达顺序。修复 PR #42245 待合并。

2. **#38182** — MTP（Multi-Token Prediction）speculative decoding 与 prefix caching 冲突，命中率从 ~92% 暴降到 ~71%，未解决。

3. **#39702** — CPU offload scheduler 存在 TOCTOU 竞态条件，长时间运行会导致 server crash。

### 性能问题

4. **#23444（13 条评论）** — 当 KV cache 利用率到 ~99% 时吞吐量崩塌。大 prompt 会驱逐大量小对话共享的缓存前缀（"缓存雪崩"）。

5. **#39806** — 推理模型（如 DeepSeek-R1）的 thinking tokens 产生大量死缓存分支，50 个并发对话浪费 65-80GB 显存。SGLang 已修复，vLLM PR 仍在 open。

6. **RFC #42185** — V1 调度器用 FCFS 排序，完全不感知缓存亲和性，这是 Agent 场景命中率上不去的架构根因。

## 业界观点

- **SGLang 的 RadixAttention 被公认为 Agent 场景最优方案**，vLLM GitHub issues 里多个用户明确表示因缓存问题切换到 SGLang
- **vLLM 团队 V2 架构计划**引入类似 RadixAttention 的 token 级缓存，但时间表未定
- **TensorRT-LLM** 在单请求延迟上可能更优，但 Agent 多轮缓存场景公开对比数据很少
- **社区共识**：纯 Agent/多轮工具调用场景选 SGLang，单轮高并发场景 vLLM 仍然很强

## 缓存友好的 Prompt 设计最佳实践

所有 prefix caching 方案（vLLM APC、SGLang、Anthropic、OpenAI）都基于前缀匹配。因此：

1. **System prompt 放最前面，Tool 定义放第二位，历史放第三，当前消息放最后**
2. **工具定义按固定顺序排列**（按名称字母排序），避免序列化顺序变化破坏前缀匹配
3. **不在 system prompt 中注入动态内容**（时间戳、请求 ID、随机数）
4. **只 append 新消息到末尾，绝不 prepend 或重排**
5. **考虑做 cache warming**：发 `max_tokens=1` 的保活请求防止 TTL 过期

## 相关页面

- [[prompt-cache-design-for-agents]] — Agent 提示词缓存设计模式与业界对比
- [[anthropic-context-engineering]] — 高效上下文工程
- [[anthropic-advanced-tool-use]] — Claude 高级工具使用
