---
title: LLM Wiki
---

# LLM Wiki

> 基于 [Karpathy's LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 模式构建的个人知识库。不同于 RAG 每次重新发现知识，Wiki 一次性编译知识并保持更新。

---

## 概念

| 条目 | 摘要 | 标签 |
|------|------|------|
| [vLLM Prefix Caching](vllm-prefix-caching.md) | PagedAttention、APC 机制、Agent 场景命中率、已知 Bug、vs SGLang | `inference` `open-source` `architecture` |
| [A2A 协议详解](a2a-protocol.md) | A2A 协议完整详解：数据模型、会话机制、消息流转、RPC 方法 | `protocol` `agent` `architecture` |
| [Task 状态机](a2a-task-state-machine.md) | Task 状态机：8 个状态、转换规则、v0.3→v1.0 变更 | `protocol` `agent` |
| [A2A 安全分析](a2a-security-analysis.md) | 10 个已知安全缺口：Prompt 注入、AgentCard 投毒、会话走私 | `protocol` `security` |
| [A2A 生态系统](a2a-ecosystem.md) | 生态工具链：Waggle、A2Apex、EDDI、实际部署案例 | `protocol` `open-source` |
| [ACP 接口总览](acp-protocol.md) | ACP 全部接口分类：生命周期、会话管理、扩展机制、权限请求 | `protocol` `agent` `tool-use` |
| [ACP stdio 授权流程](acp-stdio-auth-flow.md) | stdio 授权完整流程：spawn→initialize→prompt→permission→执行→结束 | `protocol` `agent` `tool-use` |
| [Hermes Agent 记忆架构](hermes-agent-memory-architecture.md) | 四层记忆系统：提示词记忆、会话搜索、技能系统、Honcho 深层建模 | `agent` `architecture` `open-source` |
| [ChromaFs 虚拟文件系统](concepts/chromafs-virtual-filesystem.md) | Mintlify ChromaFs：虚拟文件系统让 Agent 用 UNIX 命令探索文档，替代昂贵 sandbox | `agent` `tool-use` `open-source` |
| [构建高效 Agent](concepts/anthropic-building-effective-agents.md) | Anthropic Agent 设计模式：Prompt Chaining、Routing、并行化、Orchestrator-Workers、Evaluator-Optimizer | `agent` `anthropic` `patterns` |
| [Claude Agent SDK](concepts/anthropic-building-agents-agent-sdk.md) | Agent Loop、子 Agent、Compaction、MCP 集成 | `agent` `anthropic` `sdk` |
| [高级工具使用](concepts/anthropic-advanced-tool-use.md) | Tool Search（按需发现）、Programmatic Tool Calling、Tool Use Examples | `agent` `anthropic` `tool-use` |
| [为 Agent 编写工具](concepts/anthropic-writing-tools-for-agents.md) | 工具原型设计、评估、6 条核心原则 | `agent` `anthropic` `tool-use` |
| [Think 工具](concepts/anthropic-think-tool.md) | 复杂工具调用前的专用思考步骤 | `agent` `anthropic` `reasoning` |
| [Agent Skills](concepts/anthropic-agent-skills.md) | SKILL.md 文件和渐进式披露机制 | `agent` `anthropic` `skills` |
| [上下文工程](concepts/anthropic-context-engineering.md) | 上下文衰减、注意力预算、上下文检索与压缩 | `agent` `anthropic` `context` |
| [上下文检索](concepts/anthropic-contextual-retrieval.md) | Contextual Embeddings + BM25 + 重排序 | `agent` `anthropic` `retrieval` |
| [长运行 Agent Harness](concepts/anthropic-harnesses-long-long-running-agents.md) | 初始化器、进度追踪、环境管理 | `agent` `anthropic` `harness` |
| [多 Agent 研究系统](concepts/anthropic-multi-agent-research-system.md) | 架构设计与实践经验 | `agent` `anthropic` `multi-agent` |
| [MCP 代码执行](concepts/anthropic-code-execution-mcp.md) | 通过 Model Context Protocol 运行代码 | `agent` `anthropic` `mcp` |
| [Agent 评估方法论](concepts/anthropic-demystifying-evals.md) | 评估指标、LLM-as-Judge、真实场景测试 | `agent` `anthropic` `evaluation` |
| [Claude Code 沙箱化](concepts/anthropic-claude-code-sandboxing.md) | 安全执行 Bash 和云端运行 | `agent` `anthropic` `security` |
| [Agentic Coding 最佳实践](concepts/anthropic-agentic-coding-best-practices.md) | Claude Code 使用指南 | `agent` `anthropic` `coding` |
|| [跨产品约束Claude的方法](concepts/anthropic-containing-claude.md) | Agent安全策略、沙箱、虚拟机隔离、防护机制设计 | `anthropic` `security` `agent` |
|| [扩展托管Agent：将大脑与双手解耦](concepts/anthropic-scaling-managed-agents.md) | 托管Agent架构、会话管理、上下文窗口、工作进程 | `anthropic` `agent` `architecture` |
|| [长时间应用开发的Harness设计](concepts/anthropic-harness-design-long-running-apps.md) | 规划器-生成器-评估器模式、状态快照、上下文重置 | `anthropic` `agent` `harness` |
|| [Claude Code质量报告](concepts/anthropic-april-23-postmortem.md) | 三起基础设施问题：负载均衡、批处理竞态、缓存失效 | `anthropic` `incident` `postmortem` |
|| [Claude Code自动模式](concepts/anthropic-claude-code-auto-mode.md) | 自动批准机制、分类器设计、权限管理、防护策略 | `anthropic` `agent` `security` |
||| [三起生产事故复盘](concepts/anthropic-postmortem-three-issues.md) | 路由错误、输出损坏、XLA 编译器 Bug | `anthropic` `incident` `postmortem` |
|| [The Era of Experience（体验时代）](concepts/era-of-experience.md) | David Silver：从人类数据时代到体验时代，AlphaZero/AlphaProof 的经验驱动路径 | `rl` `deepmind` `alpha` `reasoning` |
|| [蒙特卡洛树搜索 (MCTS)](concepts/monte-carlo-tree-search.md) | 四步循环、UCB1 探索-利用平衡、搜索宽度与深度、神经网络角色 | `architecture` `model` |
|| [MCTS vs Alpha-Beta 搜索对比](concepts/mcts-vs-alpha-beta.md) | 搜索策略差异、适用场景、围棋为什么需要 MCTS、AlphaGo 融合方案 | `comparison` `architecture` |

## 对比

| 条目 | 摘要 | 标签 |
|------|------|------|
| [Agent 提示词缓存设计](prompt-cache-design-for-agents.md) | OpenCode/Aider/Claude Code 缓存策略对比、缓存友好 Prompt 最佳实践 | `inference` `agent` `open-source` |
| [Hermes vs Aider 缓存对比](comparisons/hermes-vs-aider-prompt-caching.md) | Hermes Agent vs Aider 提示词缓存设计对比：冻结策略、多 Provider 适配、cache warming | `inference` `agent` `open-source` |
| [A2A vs MCP](a2a-vs-mcp.md) | A2A 与 MCP 全面对比：定位、交互模式、适用场景、采纳现状 | `protocol` `comparison` |

## 实体

暂无条目。

## 查询存档

暂无条目。

## 笔记

暂无条目。

---

*共 35 个条目 · 最后更新：2026-06-07*
