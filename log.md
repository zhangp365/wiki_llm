     1|     1|# Wiki Log
     2|     2|
     3|     3|> 所有 Wiki 操作的按时间记录。只追加。
     4|     4|> 格式: `## [YYYY-MM-DD] action | subject`
     5|     5|> Actions: ingest, update, query, lint, create, archive, delete
     6|     6|> 超过 500 条时轮转: 重命名为 log-YYYY.md，重新开始。
     7|     7|
     8|     8|## [2026-05-02] create | Wiki 初始化
     9|     9|- Domain: AI/ML 技术研究 + 个人知识管理
    10|    10|- 创建 SCHEMA.md, index.md, log.md
    11|    11|- 目录结构: raw/{articles,papers,transcripts,assets}, entities, concepts, comparisons, queries
    12|    12|
    13|    13|## [2026-05-02] ingest | A2A 协议知识（第一批）
    14|    14|- 来源: GitHub a2aproject/A2A 官方规范 + 社区研究（grith.ai, HN, OpenAI Community）
    15|    15|- 创建原始来源: raw/articles/a2a-official-spec.md, raw/articles/a2a-community-research.md
    16|    16|- 创建 Wiki 页面:
    17|    17|  - concepts/a2a-protocol.md — 协议完整详解（数据模型、会话、消息流、RPC、注意点、安全问题）
    18|    18|  - concepts/a2a-task-state-machine.md — Task 状态机（8 状态、转换规则）
    19|    19|  - concepts/a2a-security-analysis.md — 10 个已知安全缺口
    20|    20|  - concepts/a2a-ecosystem.md — 生态工具链与实际案例
    21|    21|  - comparisons/a2a-vs-mcp.md — A2A vs MCP 全面对比
    22|    22|- 更新 index.md（5 个页面）
    23|    23|- 总计: 2 raw + 5 wiki pages
    24|    24|
    25|    25|## [2026-05-03] ingest | Hermes Agent 记忆系统架构
    26|    26|- 来源: Manthan Gupta (@manthanguptaa) 英文原文 + @宝玉xp 中文转译
    27|    27|- 原文链接: https://x.com/manthanguptaa/status/2034849672985288957
    28|    28|- 微博链接: https://m.weibo.cn/detail/5293206420062939
    29|    29|- 创建原始来源:
    30|    30|  - raw/articles/hermes-memory-system-english-original.md — 英文原文
    31|    31|  - raw/articles/hermes-memory-system-chinese-translation.md — 中文翻译
    32|    32|- 创建 Wiki 页面:
    33|    33|  - concepts/hermes-agent-memory-architecture.md — 四层记忆系统完整架构
    34|    34|- 更新 index.md（6 个页面）
    35|    35|- 总计: 2 raw + 1 wiki page
    36|    36|
    37|    37|## [2026-05-04] ingest | ACP（Agent Client Protocol）接口与授权流程
    38|    38|- 来源: 用户提供的两份 ACP 文档（acp_interfaces.md + acp_stdio_auth_flow.md）
    39|    39|- 创建原始来源:
    40|    40|  - raw/articles/acp-interfaces-summary.md — ACP 接口总览（11 个接口分类）
    41|    41|  - raw/articles/acp-stdio-auth-flow.md — stdio 授权完整流程（含 JSON 示例）
    42|    42|- 创建 Wiki 页面:
    43|    43|  - concepts/acp-protocol.md — ACP 全部接口分类总览（生命周期、会话管理、扩展机制、权限请求、关键设计决策）
    44|    44|  - concepts/acp-stdio-auth-flow.md — stdio 授权流程详解（spawn→initialize→prompt→permission→执行→结束，含消息方向速查）
    45|    45|- 更新 index.md（8 个页面）
    46|    46|- 总计: 2 raw + 2 wiki pages
    47|    47|
    48|    48|## [2026-05-05] ingest | Mintlify ChromaFs 虚拟文件系统
    49|    49|- 来源: Mintlify Engineering Blog
    50|    50|- 原文链接: https://www.mintlify.com/blog/how-we-built-a-virtual-filesystem-for-our-assistant
    51|    51|- 作者: Dens Sumesh, 2026-03-24
    52|    52|- 创建原始来源:
    53|    53|  - raw/articles/mintlify-chromafs-virtual-filesystem.md — 原文全文存档
    54|    54|- 创建 Wiki 页面:
    55|    55|  - concepts/chromafs-virtual-filesystem.md — ChromaFs 虚拟文件系统架构（just-bash、目录树引导、RBAC、grep 两阶段优化）
    56|    56|- 更新 index.md（9 个页面）
    57|    57|- 总计: 1 raw + 1 wiki page
    58|    58|
    59|    59|## [2026-05-05] ingest | Anthropic Engineering 博文合集（15 篇）
    60|    60|- 来源: Anthropic Engineering Blog (https://www.anthropic.com/engineering)
    61|    61|- 触发: 微信文章推荐的 15 篇 Agent 构建博文
    62|    62|- 创建原始来源: raw/articles/anthropic-{1..15}-*.md（15 篇英文原文存档）
    63|    63|- 创建 Wiki 页面（中文翻译，附原文链接）:
    64|    64|  1. concepts/anthropic-building-effective-agents.md — 构建高效 Agent（5 种设计模式）
    65|    65|  2. concepts/anthropic-building-agents-agent-sdk.md — Claude Agent SDK（Agent Loop、子 Agent、MCP）
    66|    66|  3. concepts/anthropic-advanced-tool-use.md — 高级工具使用（Tool Search、Programmatic Calling）
    67|    67|  4. concepts/anthropic-writing-tools-for-agents.md — 为 Agent 编写工具（6 条核心原则）
    68|    68|  5. concepts/anthropic-think-tool.md — Think 工具（复杂工具调用前思考步骤）
    69|    69|  6. concepts/anthropic-agent-skills.md — Agent Skills（SKILL.md + 渐进式披露）
    70|    70|  7. concepts/anthropic-context-engineering.md — 上下文工程（注意力预算、上下文衰减）
    71|    71|  8. concepts/anthropic-contextual-retrieval.md — 上下文检索（Contextual Embeddings + BM25）
    72|    72|  9. concepts/anthropic-harnesses-long-running-agents.md — 长运行 Agent Harness 设计
    73|    73|  10. concepts/anthropic-multi-agent-research-system.md — 多 Agent 研究系统
    74|    74|  11. concepts/anthropic-code-execution-mcp.md — MCP 代码执行
    75|    75|  12. concepts/anthropic-demystifying-evals.md — Agent 评估方法论
    76|    76|  13. concepts/anthropic-claude-code-sandboxing.md — Claude Code 沙箱化
    77|    77|  14. concepts/anthropic-agentic-coding-best-practices.md — Agentic Coding 最佳实践
    78|    78|  15. concepts/anthropic-postmortem-three-issues.md — 三起生产事故复盘
    79|    79|- 同步 docs/ 目录（15 个文件，wikilinks → 标准 markdown 链接）
    80|    80|- 更新 index.md（24 个页面）、mkdocs.yml nav、docs/index.md
    81|    81|- 总计: 15 raw + 15 wiki pages
    82|    82|
    83|    83|## [2026-05-11] ingest | vLLM 缓存机制 + Agent 提示词缓存设计（2 篇）
    84|    84|- 来源: vLLM GitHub issues/docs + SGLang 论文/LMSYS Blog + OpenCode 源码分析 + Aider 源码分析 + Anthropic Blog
    85|    85|- 无原始来源文件（综合研究，非单篇文章 ingest）
    86|    86|- 创建 Wiki 页面:
    87|    87|  - concepts/vllm-prefix-caching.md — vLLM KV Cache 与 Prefix Caching：PagedAttention、APC 机制、Agent 场景命中率、已知 Bug、vs SGLang 对比
    88|    88|  - comparisons/prompt-cache-design-for-agents.md — Agent 提示词缓存设计：OpenCode/Aider/Claude Code 缓存策略对比、缓存友好 Prompt 最佳实践
    89|    89|- 同步 docs/ 目录（2 个文件，wikilinks → 标准 markdown 链接）
    90|    90|- 更新 index.md（26 个页面）、docs/index.md、mkdocs.yml nav
    91|    91|- 总计: 0 raw + 2 wiki pages
    92|    92|
    93|## [2026-05-11] ingest | Hermes vs Aider Prompt 缓存设计对比
    94|- 来源: Hermes Agent 源码分析（~/.hermes/hermes-agent/）+ Aider 源码分析 + 现有 wiki 页面交叉参考
    95|- 无原始来源文件（综合源码研究，非单篇文章 ingest）
    96|- 创建 Wiki 页面:
    97|  - comparisons/hermes-vs-aider-prompt-caching.md — Hermes Agent vs Aider 提示词缓存设计对比：冻结策略、多 Provider 适配、cache warming、场景分析
    98|- 同步 docs/ 目录（1 个文件，wikilinks → 标准 markdown 链接）
    99|- 更新 index.md（27 个页面）、docs/index.md、mkdocs.yml nav
   100|- 总计: 0 raw + 1 wiki page
   101|
   102|## [2026-05-11] ingest | The Era of Experience — David Silver
   103|- Created concepts/era-of-experience.md
   104|- Saved raw transcript to raw/articles/deepmind-era-of-experience-podcast-2025.md
   105|- Updated index.md (28 entries)
   106|- Updated mkdocs.yml nav
   107|- Sources: Google DeepMind Podcast, 腾讯新闻翻译
   108|
   109|## [2026-05-12] update | The Era of Experience — 补充 Friends House 演讲完整翻译
   110|- Saved raw speech translation to raw/articles/david-silver-era-of-experience-speech-2025-10.md
   111|- Updated concepts/era-of-experience.md: 整合演讲三大案例（AlphaZero/AlphaProof/DiscoRL）、四大特征、路灯寓言、Ineffable Intelligence 融资信息
   112|- Source: 用户提供的 Friends House 演讲完整中文翻译（2025-10-28）
   113|
## [2026-05-13] create | MCTS 与 Alpha-Beta 搜索
- 来源: 用户对话讨论（蒙特卡洛树搜索 vs Alpha-Beta 剪枝）
- 创建 Wiki 页面:
  - concepts/monte-carlo-tree-search.md — MCTS 完整概念详解：四步循环、UCB1、搜索宽度深度、神经网络角色
  - concepts/mcts-vs-alpha-beta.md — MCTS vs Alpha-Beta 对比：搜索策略差异、适用场景、AlphaGo 融合方案
- 更新: index.md（新增 2 个页面，总计 29 页）

## [2026-07-09] ingest | Lilian Weng - Harness Engineering for Self-Improvement
- 来源: https://lilianweng.github.io/posts/2026-07-04-harness/
- 创建原始来源: raw/articles/lilianweng-harness-engineering-self-improvement-2026.md
- 创建 Wiki 页面:
  - concepts/lilianweng-harness-engineering-self-improvement.md — Harness 工程与自我改进（完整中文翻译：设计模式、上下文工程、进化搜索、自我改进 Harness、7 大未来挑战）
- 同步 docs/concepts/
- 更新 index.md（新增 1 页，总计 36 页）、mkdocs.yml nav
- 总计: 1 raw + 1 wiki page
