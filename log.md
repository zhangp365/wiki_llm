# Wiki Log

> 所有 Wiki 操作的按时间记录。只追加。
> 格式: `## [YYYY-MM-DD] action | subject`
> Actions: ingest, update, query, lint, create, archive, delete
> 超过 500 条时轮转: 重命名为 log-YYYY.md，重新开始。

## [2026-05-02] create | Wiki 初始化
- Domain: AI/ML 技术研究 + 个人知识管理
- 创建 SCHEMA.md, index.md, log.md
- 目录结构: raw/{articles,papers,transcripts,assets}, entities, concepts, comparisons, queries

## [2026-05-02] ingest | A2A 协议知识（第一批）
- 来源: GitHub a2aproject/A2A 官方规范 + 社区研究（grith.ai, HN, OpenAI Community）
- 创建原始来源: raw/articles/a2a-official-spec.md, raw/articles/a2a-community-research.md
- 创建 Wiki 页面:
  - concepts/a2a-protocol.md — 协议完整详解（数据模型、会话、消息流、RPC、注意点、安全问题）
  - concepts/a2a-task-state-machine.md — Task 状态机（8 状态、转换规则）
  - concepts/a2a-security-analysis.md — 10 个已知安全缺口
  - concepts/a2a-ecosystem.md — 生态工具链与实际案例
  - comparisons/a2a-vs-mcp.md — A2A vs MCP 全面对比
- 更新 index.md（5 个页面）
- 总计: 2 raw + 5 wiki pages

## [2026-05-03] ingest | Hermes Agent 记忆系统架构
- 来源: Manthan Gupta (@manthanguptaa) 英文原文 + @宝玉xp 中文转译
- 原文链接: https://x.com/manthanguptaa/status/2034849672985288957
- 微博链接: https://m.weibo.cn/detail/5293206420062939
- 创建原始来源:
  - raw/articles/hermes-memory-system-english-original.md — 英文原文
  - raw/articles/hermes-memory-system-chinese-translation.md — 中文翻译
- 创建 Wiki 页面:
  - concepts/hermes-agent-memory-architecture.md — 四层记忆系统完整架构
- 更新 index.md（6 个页面）
- 总计: 2 raw + 1 wiki page

## [2026-05-04] ingest | ACP（Agent Client Protocol）接口与授权流程
- 来源: 用户提供的两份 ACP 文档（acp_interfaces.md + acp_stdio_auth_flow.md）
- 创建原始来源:
  - raw/articles/acp-interfaces-summary.md — ACP 接口总览（11 个接口分类）
  - raw/articles/acp-stdio-auth-flow.md — stdio 授权完整流程（含 JSON 示例）
- 创建 Wiki 页面:
  - concepts/acp-protocol.md — ACP 全部接口分类总览（生命周期、会话管理、扩展机制、权限请求、关键设计决策）
  - concepts/acp-stdio-auth-flow.md — stdio 授权流程详解（spawn→initialize→prompt→permission→执行→结束，含消息方向速查）
- 更新 index.md（8 个页面）
- 总计: 2 raw + 2 wiki pages

## [2026-05-05] ingest | Mintlify ChromaFs 虚拟文件系统
- 来源: Mintlify Engineering Blog
- 原文链接: https://www.mintlify.com/blog/how-we-built-a-virtual-filesystem-for-our-assistant
- 作者: Dens Sumesh, 2026-03-24
- 创建原始来源:
  - raw/articles/mintlify-chromafs-virtual-filesystem.md — 原文全文存档
- 创建 Wiki 页面:
  - concepts/chromafs-virtual-filesystem.md — ChromaFs 虚拟文件系统架构（just-bash、目录树引导、RBAC、grep 两阶段优化）
- 更新 index.md（9 个页面）
- 总计: 1 raw + 1 wiki page

## [2026-05-05] ingest | Anthropic Engineering 博文合集（15 篇）
- 来源: Anthropic Engineering Blog (https://www.anthropic.com/engineering)
- 触发: 微信文章推荐的 15 篇 Agent 构建博文
- 创建原始来源: raw/articles/anthropic-{1..15}-*.md（15 篇英文原文存档）
- 创建 Wiki 页面（中文翻译，附原文链接）:
  1. concepts/anthropic-building-effective-agents.md — 构建高效 Agent（5 种设计模式）
  2. concepts/anthropic-building-agents-agent-sdk.md — Claude Agent SDK（Agent Loop、子 Agent、MCP）
  3. concepts/anthropic-advanced-tool-use.md — 高级工具使用（Tool Search、Programmatic Calling）
  4. concepts/anthropic-writing-tools-for-agents.md — 为 Agent 编写工具（6 条核心原则）
  5. concepts/anthropic-think-tool.md — Think 工具（复杂工具调用前思考步骤）
  6. concepts/anthropic-agent-skills.md — Agent Skills（SKILL.md + 渐进式披露）
  7. concepts/anthropic-context-engineering.md — 上下文工程（注意力预算、上下文衰减）
  8. concepts/anthropic-contextual-retrieval.md — 上下文检索（Contextual Embeddings + BM25）
  9. concepts/anthropic-harnesses-long-running-agents.md — 长运行 Agent Harness 设计
  10. concepts/anthropic-multi-agent-research-system.md — 多 Agent 研究系统
  11. concepts/anthropic-code-execution-mcp.md — MCP 代码执行
  12. concepts/anthropic-demystifying-evals.md — Agent 评估方法论
  13. concepts/anthropic-claude-code-sandboxing.md — Claude Code 沙箱化
  14. concepts/anthropic-agentic-coding-best-practices.md — Agentic Coding 最佳实践
  15. concepts/anthropic-postmortem-three-issues.md — 三起生产事故复盘
- 同步 docs/ 目录（15 个文件，wikilinks → 标准 markdown 链接）
- 更新 index.md（24 个页面）、mkdocs.yml nav、docs/index.md
- 总计: 15 raw + 15 wiki pages
