     1|# Wiki Log
     2|
     3|> 所有 Wiki 操作的按时间记录。只追加。
     4|> 格式: `## [YYYY-MM-DD] action | subject`
     5|> Actions: ingest, update, query, lint, create, archive, delete
     6|> 超过 500 条时轮转: 重命名为 log-YYYY.md，重新开始。
     7|
     8|## [2026-05-02] create | Wiki 初始化
     9|- Domain: AI/ML 技术研究 + 个人知识管理
    10|- 创建 SCHEMA.md, index.md, log.md
    11|- 目录结构: raw/{articles,papers,transcripts,assets}, entities, concepts, comparisons, queries
    12|
    13|## [2026-05-02] ingest | A2A 协议知识（第一批）
    14|- 来源: GitHub a2aproject/A2A 官方规范 + 社区研究（grith.ai, HN, OpenAI Community）
    15|- 创建原始来源: raw/articles/a2a-official-spec.md, raw/articles/a2a-community-research.md
    16|- 创建 Wiki 页面:
    17|  - concepts/a2a-protocol.md — 协议完整详解（数据模型、会话、消息流、RPC、注意点、安全问题）
    18|  - concepts/a2a-task-state-machine.md — Task 状态机（8 状态、转换规则）
    19|  - concepts/a2a-security-analysis.md — 10 个已知安全缺口
    20|  - concepts/a2a-ecosystem.md — 生态工具链与实际案例
    21|  - comparisons/a2a-vs-mcp.md — A2A vs MCP 全面对比
    22|- 更新 index.md（5 个页面）
    23|- 总计: 2 raw + 5 wiki pages
    24|
    25|## [2026-05-03] ingest | Hermes Agent 记忆系统架构
    26|- 来源: Manthan Gupta (@manthanguptaa) 英文原文 + @宝玉xp 中文转译
    27|- 原文链接: https://x.com/manthanguptaa/status/2034849672985288957
    28|- 微博链接: https://m.weibo.cn/detail/5293206420062939
    29|- 创建原始来源:
    30|  - raw/articles/hermes-memory-system-english-original.md — 英文原文
    31|  - raw/articles/hermes-memory-system-chinese-translation.md — 中文翻译
    32|- 创建 Wiki 页面:
    33|  - concepts/hermes-agent-memory-architecture.md — 四层记忆系统完整架构
    34|- 更新 index.md（6 个页面）
    35|- 总计: 2 raw + 1 wiki page
    36|
    37|## [2026-05-04] ingest | ACP（Agent Client Protocol）接口与授权流程
    38|- 来源: 用户提供的两份 ACP 文档（acp_interfaces.md + acp_stdio_auth_flow.md）
    39|- 创建原始来源:
    40|  - raw/articles/acp-interfaces-summary.md — ACP 接口总览（11 个接口分类）
    41|  - raw/articles/acp-stdio-auth-flow.md — stdio 授权完整流程（含 JSON 示例）
    42|- 创建 Wiki 页面:
    43|  - concepts/acp-protocol.md — ACP 全部接口分类总览（生命周期、会话管理、扩展机制、权限请求、关键设计决策）
    44|  - concepts/acp-stdio-auth-flow.md — stdio 授权流程详解（spawn→initialize→prompt→permission→执行→结束，含消息方向速查）
    45|- 更新 index.md（8 个页面）
    46|- 总计: 2 raw + 2 wiki pages
    47|
    48|## [2026-05-05] ingest | Mintlify ChromaFs 虚拟文件系统
    49|- 来源: Mintlify Engineering Blog
    50|- 原文链接: https://www.mintlify.com/blog/how-we-built-a-virtual-filesystem-for-our-assistant
    51|- 作者: Dens Sumesh, 2026-03-24
    52|- 创建原始来源:
    53|  - raw/articles/mintlify-chromafs-virtual-filesystem.md — 原文全文存档
    54|- 创建 Wiki 页面:
    55|  - concepts/chromafs-virtual-filesystem.md — ChromaFs 虚拟文件系统架构（just-bash、目录树引导、RBAC、grep 两阶段优化）
    56|- 更新 index.md（9 个页面）
    57|- 总计: 1 raw + 1 wiki page
    58|
    59|## [2026-05-05] ingest | Anthropic Engineering 博文合集（15 篇）
    60|- 来源: Anthropic Engineering Blog (https://www.anthropic.com/engineering)
    61|- 触发: 微信文章推荐的 15 篇 Agent 构建博文
    62|- 创建原始来源: raw/articles/anthropic-{1..15}-*.md（15 篇英文原文存档）
    63|- 创建 Wiki 页面（中文翻译，附原文链接）:
    64|  1. concepts/anthropic-building-effective-agents.md — 构建高效 Agent（5 种设计模式）
    65|  2. concepts/anthropic-building-agents-agent-sdk.md — Claude Agent SDK（Agent Loop、子 Agent、MCP）
    66|  3. concepts/anthropic-advanced-tool-use.md — 高级工具使用（Tool Search、Programmatic Calling）
    67|  4. concepts/anthropic-writing-tools-for-agents.md — 为 Agent 编写工具（6 条核心原则）
    68|  5. concepts/anthropic-think-tool.md — Think 工具（复杂工具调用前思考步骤）
    69|  6. concepts/anthropic-agent-skills.md — Agent Skills（SKILL.md + 渐进式披露）
    70|  7. concepts/anthropic-context-engineering.md — 上下文工程（注意力预算、上下文衰减）
    71|  8. concepts/anthropic-contextual-retrieval.md — 上下文检索（Contextual Embeddings + BM25）
    72|  9. concepts/anthropic-harnesses-long-running-agents.md — 长运行 Agent Harness 设计
    73|  10. concepts/anthropic-multi-agent-research-system.md — 多 Agent 研究系统
    74|  11. concepts/anthropic-code-execution-mcp.md — MCP 代码执行
    75|  12. concepts/anthropic-demystifying-evals.md — Agent 评估方法论
    76|  13. concepts/anthropic-claude-code-sandboxing.md — Claude Code 沙箱化
    77|  14. concepts/anthropic-agentic-coding-best-practices.md — Agentic Coding 最佳实践
    78|  15. concepts/anthropic-postmortem-three-issues.md — 三起生产事故复盘
    79|- 同步 docs/ 目录（15 个文件，wikilinks → 标准 markdown 链接）
    80|- 更新 index.md（24 个页面）、mkdocs.yml nav、docs/index.md
    81|- 总计: 15 raw + 15 wiki pages
    82|
    83|## [2026-05-11] ingest | vLLM 缓存机制 + Agent 提示词缓存设计（2 篇）
    84|- 来源: vLLM GitHub issues/docs + SGLang 论文/LMSYS Blog + OpenCode 源码分析 + Aider 源码分析 + Anthropic Blog
    85|- 无原始来源文件（综合研究，非单篇文章 ingest）
    86|- 创建 Wiki 页面:
    87|  - concepts/vllm-prefix-caching.md — vLLM KV Cache 与 Prefix Caching：PagedAttention、APC 机制、Agent 场景命中率、已知 Bug、vs SGLang 对比
    88|  - comparisons/prompt-cache-design-for-agents.md — Agent 提示词缓存设计：OpenCode/Aider/Claude Code 缓存策略对比、缓存友好 Prompt 最佳实践
    89|- 同步 docs/ 目录（2 个文件，wikilinks → 标准 markdown 链接）
    90|- 更新 index.md（26 个页面）、docs/index.md、mkdocs.yml nav
    91|- 总计: 0 raw + 2 wiki pages
    92|
## [2026-05-11] ingest | Hermes vs Aider Prompt 缓存设计对比
- 来源: Hermes Agent 源码分析（~/.hermes/hermes-agent/）+ Aider 源码分析 + 现有 wiki 页面交叉参考
- 无原始来源文件（综合源码研究，非单篇文章 ingest）
- 创建 Wiki 页面:
  - comparisons/hermes-vs-aider-prompt-caching.md — Hermes Agent vs Aider 提示词缓存设计对比：冻结策略、多 Provider 适配、cache warming、场景分析
- 同步 docs/ 目录（1 个文件，wikilinks → 标准 markdown 链接）
- 更新 index.md（27 个页面）、docs/index.md、mkdocs.yml nav
- 总计: 0 raw + 1 wiki page

## [2026-05-11] ingest | The Era of Experience — David Silver
- Created concepts/era-of-experience.md
- Saved raw transcript to raw/articles/deepmind-era-of-experience-podcast-2025.md
- Updated index.md (28 entries)
- Updated mkdocs.yml nav
- Sources: Google DeepMind Podcast, 腾讯新闻翻译
