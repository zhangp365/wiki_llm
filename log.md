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
