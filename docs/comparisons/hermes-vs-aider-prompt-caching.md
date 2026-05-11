     1|     1|---
     2|     2|title: Hermes Agent vs Aider Prompt 缓存设计对比
     3|     3|created: 2026-05-11
     4|     4|updated: 2026-05-11
     5|     5|type: comparison
     6|     6|tags: [inference, agent, comparison, open-source]
     7|     7|sources: []
     8|     8|---
     9|     9|
    10|    10|# Hermes Agent vs Aider Prompt 缓存设计对比
    11|    11|
    12|    12|## 概述
    13|    13|
    14|    14|对比 Hermes Agent 和 Aider 两个开源 AI Agent 的 Prompt 构建策略与缓存优化设计。两者都意识到了 Prompt Caching 对成本和延迟的重要性，但设计哲学不同：**Aider 追求极致的多层递进缓存**，**Hermes 追求系统提示词的冻结稳定性 + 多 Provider 适配**。
    15|    15|
    16|    16|结论：Aider 在 Anthropic 单一 Provider 上缓存命中更精细（3 层递进断点 + cache warming），Hermes 的缓存设计更通用（同时适配 Anthropic、OpenAI、OpenRouter 等多后端），但缺少 cache warming 是主要短板。
    17|    17|
    18|    18|## Hermes Agent 的 Prompt 结构与缓存设计
    19|    19|
    20|    20|### 提示词组装顺序
    21|    21|
    22|    22|```
    23|    23|[Agent Identity (SOUL.md 或默认身份)]           ← 固定，极少变
    24|    24|[Tool-Use 行为指导 (按已加载工具条件注入)]        ← 工具集确定后固定
    25|    25|[Tool-Use Enforcement (模型特定纪律约束)]         ← 模型确定后固定
    26|    26|[用户/网关系统消息]                               ← 可选，会话级固定
    27|    27|[固化 MEMORY.md 快照]                             ← 会话开始时冻结
    28|    28|[固化 USER.md 快照]                               ← 会话开始时冻结
    29|    29|[外部记忆 Provider 块]                            ← 插件注入，固定
    30|    30|[Skills 索引]                                     ← 工具集确定后固定
    31|    31|[上下文文件 (AGENTS.md, .cursorrules 等)]          ← 会话级固定
    32|    32|[时间戳 + 平台信息 + 模型/Provider]                ← 会话开始时冻结
    33|    33|[环境检测 (WSL, Termux)]                          ← 系统级固定
    34|    34|[平台格式提示 (Telegram/Discord/等)]               ← 平台级固定
    35|    35|  ↓ 组装成消息序列
    36|    36|[System Message (上面全部)]   ← cache breakpoint #1
    37|    37|[Tool Definitions (字母排序)] ← 独立参数传递
    38|    38|[对话历史]                    ← cache breakpoint #2-4 (最后 3 条消息)
    39|    39|[当前用户消息]                ← 每轮都变
    40|    40|```
    41|    41|
    42|    42|### 缓存优化措施
    43|    43|
    44|    44|**✅ 做对的：**
    45|    45|
    46|    46|1. **系统提示词会话级冻结** — `_build_system_prompt()` 结果缓存在 `self._cached_system_prompt`，整个会话期间不重建。只有在上下文压缩（compression）时才设为 `None` 强制重建
    47|    47|2. **Session DB 持久化提示词** — 精确的提示词文本存入 SQLite session 数据库，后续轮次直接读取，保证 **byte 级一致的前缀**
    48|    48|3. **Anthropic `cache_control: {"type": "ephemeral"}` 标记** — `agent/prompt_caching.py` 实现 "system_and_3" 策略：系统消息 + 最后 3 条非系统消息，共 4 个断点（Anthropic 允许的最大值）
    49|    49|4. **Tool 定义字母排序** — `tools/registry.py` 用 `sorted(tool_names)` 保证工具 schema 跨 API 调用完全一致
    50|    50|5. **动态内容隔离** — `ephemeral_system_prompt` 故意不放进 `_build_system_prompt()`，而是在 API 调用时注入到 `effective_system`，保持缓存前缀稳定。Plugin 上下文注入到用户消息而非系统提示词
    51|    51|6. **多 Provider 适配** — 原生 Anthropic API 用内层 content block 标记；OpenRouter 用消息级标记；OpenAI 用 `prompt_cache_key` 参数（session_id）
    52|    52|7. **时间戳冻结** — 日期时间在 `_build_system_prompt()` 时冻结，后续轮次不更新
    53|    53|8. **Deep copy 防污染** — 注入 cache_control 标记时做 `copy.deepcopy(api_messages)`，不修改原始消息列表
    54|    54|
    55|    55|**⚠️ 不够理想的：**
    56|    56|
    57|    57|1. **无 Cache Warming** — 没有周期性 heartbeat 请求保活 Anthropic 5 分钟 TTL。如果用户思考超过 5 分钟，缓存全部失效
    58|    58|2. **环境信息在系统提示词中** — WSL/Termux 检测、平台格式提示等虽固定但增加了前缀长度，对非 Anthropic Provider 无法利用缓存
    59|    59|3. **Memory 写入不触发重建但有延迟** — 中途写入的 memory 要到下次 compression 重建才会反映到系统提示词中，用户可能感知延迟
    60|    60|
    61|    61|### 缓存标记实现细节
    62|    62|
    63|    63|```python
    64|    64|# agent/prompt_caching.py 核心逻辑 (简化)
    65|    65|def apply_anthropic_cache_markers(messages, ttl="5m"):
    66|    66|    # 策略: system_and_3
    67|    67|    # 断点 1: 系统消息
    68|    68|    # 断点 2-4: 最后 3 条非系统消息
    69|    69|    marker = {"type": "ephemeral"}
    70|    70|    if ttl == "1h":
    71|    71|        marker["ttl"] = "1h"
    72|    72|    
    73|    73|    # 原生 Anthropic: 标记在 content block 内层
    74|    74|    # OpenRouter: 标记在 message envelope 外层
    75|    75|```
    76|    76|
    77|    77|## Aider 的 Prompt 结构与缓存设计
    78|    78|
    79|    79|### 分层缓存架构
    80|    80|
    81|    81|```
    82|    82|[SYSTEM + EXAMPLES]     ← cache breakpoint #1 (极少变化)
    83|    83|[REPO MAP + 只读文件]    ← cache breakpoint #2 (切换文件时才变)
    84|    84|[CHAT FILES]            ← cache breakpoint #3 (编辑时才变)
    85|    85|[DONE MESSAGES]         ← 对话历史 (累积增长)
    86|    86|[CUR MESSAGES]          ← 当前轮 (每轮都变)
    87|    87|[REMINDER]              ← 提醒
    88|    88|```
    89|    89|
    90|    90|### 缓存优化措施
    91|    91|
    92|    92|**✅ 做对的：**
    93|    93|
    94|    94|1. **3 层递进缓存断点** — 从稳定到不稳定逐层递进，每层独立的 `cache_control: "ephemeral"` 标记。代码变化只破坏第 3 层缓存，前两层仍命中
    95|    95|2. **Cache Warming 守护线程** — 每 5 分钟发 `max_tokens=1` 请求保活 Anthropic 5 分钟 TTL（`AIDER_CACHE_KEEPALIVE_DELAY` 可调）
    96|    96|3. **`cacheable_messages()` 方法** — 只返回到最后一个断点的消息，最小化 cache warming 的 token 开销
    97|    97|4. **绝不修改前缀** — 所有变化都 append 到末尾，不 prepend 不重排
    98|    98|5. **按模型配置缓存** — `model-settings.yml` 里 Anthropic 模型设 `cache_control: true`，发送 `anthropic-beta: prompt-caching-2024-07-31` header
    99|    99|
   100|   100|## 核心对比表
   101|   101|
   102|   102|| 维度 | Hermes Agent | Aider |
   103|   103||------|-------------|-------|
   104|   104|| **缓存设计哲学** | 系统提示词冻结 + 多 Provider 适配 | 多层递进断点 + Anthropic 深度优化 |
   105|   105|| **缓存断点数** | 4 个（系统 + 最后 3 条消息） | 3 层递进（SYSTEM → FILES → CHAT） |
   106|   106|| **Cache Warming** | ❌ 无 | ✅ 每 5 分钟 heartbeat |
   107|   107|| **Provider 覆盖** | Anthropic + OpenAI + OpenRouter + 第三方网关 | 仅 Anthropic（`cache_control: true`） |
   108|   108|| **Tool 定义排序** | 字母排序 (`sorted()`) | 未特别排序 |
   109|   109|| **系统提示词稳定性** | 会话级冻结 + Session DB 持久化 | 按层分离，最底层极少变化 |
   110|   110|| **动态内容处理** | 隔离到 ephemeral prompt / 用户消息 | 从不注入系统层 |
   111|   111|| **时间戳** | 冻结在首次构建时 | 不注入 prompt |
   112|   112|| **记忆系统影响** | memory 写入不触发重建（直到压缩） | 无记忆系统 |
   113|   113|| **代码变化粒度** | 整个 system prompt 原子重建 | 3 层分离，代码变只影响第 2-3 层 |
   114|   114|| **OpenAI 缓存** | ✅ 用 `prompt_cache_key` (session_id) | ❌ 未特别优化 |
   115|   115|
   116|   116|## 缓存命中率场景分析
   117|   117|
   118|   118|### 场景 1: 多轮对话（用户持续交互）
   119|   119|
   120|   120|| | Hermes | Aider |
   121|   121||---|---|---|
   122|   122|| System prompt | ✅ 每轮命中（冻结） | ✅ 每轮命中（不变） |
   123|   123|| Tool definitions | ✅ 每轮命中 | ✅ 每轮命中 |
   124|   124|| 早期对话历史 | ✅ 最后 3 条标记缓存 | ✅ 断点 2 覆盖 |
   125|   125|| 新增内容 | ✅ 仅当前消息变 | ✅ 仅 CUR MESSAGES 变 |
   126|   126|
   127|   127|**结论：** 两者在连续多轮对话中表现相当。
   128|   128|
   129|   129|### 场景 2: 用户思考超过 5 分钟（idle gap）
   130|   130|
   131|   131|| | Hermes | Aider |
   132|   132||---|---|---|
   133|   133|| 5 分钟后缓存 | ❌ **全部失效**（TTL 到期，无 warming） | ✅ **仍然命中**（warming 保活） |
   134|   134|| 恢复后首轮成本 | 全额计费 | 缓存价格（90% off） |
   135|   135|
   136|   136|**结论：** Aider 明显优于 Hermes。这是 Hermes 最大的缓存短板。
   137|   137|
   138|   138|### 场景 3: 代码文件变动
   139|   139|
   140|   140|| | Hermes | Aider |
   141|   141||---|---|---|
   142|   142|| 系统提示词 | ✅ 不受影响 | ✅ 不受影响（第 1 层） |
   143|   143|| 文件上下文 | 变化反映在工具返回中 | 仅第 2-3 层失效，第 1 层仍命中 |
   144|   144|| 缓存命中 | 系统级仍命中 | 系统级 + examples 仍命中 |
   145|   145|
   146|   146|**结论：** Aider 的分层粒度更细，代码变化时保留更多缓存。
   147|   147|
   148|   148|### 场景 4: 跨平台/跨 Provider
   149|   149|
   150|   150|| | Hermes | Aider |
   151|   151||---|---|---|
   152|   152|| Anthropic 原生 | ✅ | ✅ |
   153|   153|| OpenAI (GPT/Codex) | ✅ prompt_cache_key | ❌ 未适配 |
   154|   154|| OpenRouter + Claude | ✅ 消息级标记 | ❌ |
   155|   155|| 第三方兼容 API | ✅ 自动适配 | ❌ |
   156|   156|| 本地模型 (vLLM/SGLang) | ✅ 前缀自然命中 | ✅ 前缀自然命中 |
   157|   157|
   158|   158|**结论：** Hermes 覆盖面远超 Aider，这是其核心优势。
   159|   159|
   160|   160|## 对 Hermes 的改进建议
   161|   161|
   162|   162|1. **添加 Cache Warming** — 参考 Aider 的 `cache_warming` 线程，在 idle 超过阈值时发 `max_tokens=1` 保活请求。这是 ROI 最高的改进
   163|   163|2. **考虑分层缓存断点** — 将 MEMORY/USER 快照作为独立断点，与系统身份分离，memory 更新时不必放弃整个系统提示词缓存
   164|   164|3. **追踪缓存命中统计** — 像 Aider 一样解析 `CacheCreationInputTokens` / `CacheReadInputTokens`，给用户展示缓存命中率和节省金额
   165|   165|
   166|   166|## 设计哲学总结
   167|   167|
   168|   168|```
   169|   169|Aider:    "为 Anthropic 做到极致" — 单 Provider 深度优化
   170|   170|Hermes:   "在所有 Provider 上都还行" — 广度优先的缓存策略
   171|   171|```
   172|   172|
   173|   173|Aider 的 3 层递进设计在 Anthropic 上确实更优，尤其有 cache warming 保底。但 Hermes 的多 Provider 适配使其在 OpenAI、OpenRouter、本地模型等场景下也能享受缓存红利，这符合 Hermes "provider-agnostic" 的核心定位。
   174|   174|
   175|   175|**理想状态：** Hermes 应该在保持多 Provider 适配的同时，为 Anthropic Provider 增加 cache warming 和更细粒度的分层断点。
   176|   176|
   177|   177|## 相关页面
   178|   178|
   179|   179|- [prompt-cache-design-for-agents](../prompt-cache-design-for-agents.md) — Agent 提示词缓存设计模式与 OpenCode/Aider 对比
   180|   180|- [vllm-prefix-caching](../vllm-prefix-caching.md) — vLLM KV Cache 与 Prefix Caching 详解
   181|   181|- [hermes-agent-memory-architecture](../hermes-agent-memory-architecture.md) — Hermes Agent 四层记忆系统架构
   182|   182|- [anthropic-context-engineering](../concepts/anthropic-context-engineering.md) — 高效上下文工程
   183|   183|