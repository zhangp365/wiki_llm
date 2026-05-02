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
