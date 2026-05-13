     1|# Wiki Index
     2|
     3|> 内容目录。每个 Wiki 页面按类型列出，附一行摘要。
     4|> 先看这里来找到相关页面。
     5|> Last updated: 2026-05-13 | Total pages: 29
     6|
     7|## Entities
     8|<!-- 按字母排序 -->
     9|
## Concepts
- [[vllm-prefix-caching]] — vLLM KV Cache 与 Prefix Caching：PagedAttention、APC 机制、Agent 场景命中率、已知 Bug、vs SGLang 对比
- [[a2a-protocol]]
    12|- [[a2a-task-state-machine]] — Task 状态机：8 个状态、转换规则、v0.3→v1.0 变更
    13|- [[a2a-security-analysis]] — 10 个已知安全缺口：Prompt 注入、AgentCard 投毒、会话走私
    14|- [[a2a-ecosystem]] — 生态工具链：Waggle、A2Apex、EDDI、实际部署案例
    15|- [[acp-protocol]] — ACP（Agent Client Protocol）全部接口分类：生命周期、会话管理、扩展机制、权限请求
    16|- [[acp-stdio-auth-flow]] — ACP stdio 授权完整流程：spawn→initialize→prompt→permission→执行→结束
    17|- [[hermes-agent-memory-architecture]] — Hermes Agent 四层记忆系统：提示词记忆、会话搜索、技能系统、Honcho 深层建模
    18|- [[anthropic-building-effective-agents]] — Anthropic 构建高效 Agent：5 种设计模式（Prompt Chaining、Routing、并行化、Orchestrator-Workers、Evaluator-Optimizer）
- [[anthropic-building-agents-agent-sdk]] — 使用 Claude Agent SDK 构建 Agent：Agent Loop、子 Agent、Compaction、MCP 集成
- [[anthropic-advanced-tool-use]] — Claude 高级工具使用：Tool Search（按需发现）、Programmatic Tool Calling、Tool Use Examples
- [[anthropic-writing-tools-for-agents]] — 用 Agent 编写高效工具：工具原型设计、评估、6 条核心原则
- [[anthropic-think-tool]] — "Think" 工具：复杂工具调用前的专用思考步骤，提升准确率
- [[anthropic-agent-skills]] — Agent Skills：用 SKILL.md 文件和渐进式披露机制扩展 Agent 能力
- [[anthropic-context-engineering]] — 高效上下文工程：上下文衰减、注意力预算、上下文检索与压缩
- [[anthropic-contextual-retrieval]] — 上下文检索（Contextual Retrieval）：Contextual Embeddings + BM25 + 重排序
- [[anthropic-harnesses-long-running-agents]] — 长运行 Agent 的 Harness 设计：初始化器、进度追踪、环境管理
- [[anthropic-multi-agent-research-system]] — Anthropic 多 Agent 研究系统：架构设计与实践经验
- [[anthropic-code-execution-mcp]] — MCP 代码执行：通过 Model Context Protocol 运行代码
- [[anthropic-demystifying-evals]] — Agent 评估方法论：评估指标、LLM-as-Judge、真实场景测试
- [[anthropic-claude-code-sandboxing]] — Claude Code 沙箱化：安全执行 Bash 和云端运行
- [[anthropic-agentic-coding-best-practices]] — Agentic Coding 最佳实践：Claude Code 使用指南
- [[anthropic-postmortem-three-issues]] — Anthropic 三起生产事故复盘：路由错误、输出损坏、XLA 编译器 Bug
- [[chromafs-virtual-filesystem]] — Mintlify ChromaFs：虚拟文件系统让 Agent 用 UNIX 命令探索文档，替代昂贵的 sandbox
- [[monte-carlo-tree-search]] — 蒙特卡洛树搜索 (MCTS)：四步循环、UCB1 探索-利用平衡、搜索宽度与深度、神经网络角色
- [[mcts-vs-alpha-beta]] — MCTS vs Alpha-Beta 搜索对比：搜索策略、适用场景、围棋为什么需要 MCTS、AlphaGo 融合方案
    19|
## Comparisons
- [[prompt-cache-design-for-agents]] — Agent 提示词缓存设计：OpenCode/Aider/Claude Code 缓存策略对比、缓存友好 Prompt 最佳实践
- [[hermes-vs-aider-prompt-caching]] — Hermes Agent vs Aider 提示词缓存设计对比：冻结策略、多 Provider 适配、cache warming
- [[a2a-vs-mcp]]
    22|
    23|## Queries
    24|
    25|## Notes
    26|