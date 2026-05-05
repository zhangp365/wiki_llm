---
title: LLM Wiki
---

# LLM Wiki

> 基于 [Karpathy's LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 模式构建的个人知识库。不同于 RAG 每次重新发现知识，Wiki 一次性编译知识并保持更新。

---

## 概念

| 条目 | 摘要 | 标签 |
|------|------|------|
| [A2A 协议详解](a2a-protocol.md) | A2A 协议完整详解：数据模型、会话机制、消息流转、RPC 方法 | `protocol` `agent` `architecture` |
| [Task 状态机](a2a-task-state-machine.md) | Task 状态机：8 个状态、转换规则、v0.3→v1.0 变更 | `protocol` `agent` |
| [A2A 安全分析](a2a-security-analysis.md) | 10 个已知安全缺口：Prompt 注入、AgentCard 投毒、会话走私 | `protocol` `security` |
| [A2A 生态系统](a2a-ecosystem.md) | 生态工具链：Waggle、A2Apex、EDDI、实际部署案例 | `protocol` `open-source` |
| [ACP 接口总览](acp-protocol.md) | ACP 全部接口分类：生命周期、会话管理、扩展机制、权限请求 | `protocol` `agent` `tool-use` |
| [ACP stdio 授权流程](acp-stdio-auth-flow.md) | stdio 授权完整流程：spawn→initialize→prompt→permission→执行→结束 | `protocol` `agent` `tool-use` |
| [Hermes Agent 记忆架构](hermes-agent-memory-architecture.md) | 四层记忆系统：提示词记忆、会话搜索、技能系统、Honcho 深层建模 | `agent` `architecture` `open-source` |
| [ChromaFs 虚拟文件系统](concepts/chromafs-virtual-filesystem.md) | Mintlify ChromaFs：虚拟文件系统让 Agent 用 UNIX 命令探索文档，替代昂贵 sandbox | `agent` `tool-use` `open-source` |

## 对比

| 条目 | 摘要 | 标签 |
|------|------|------|
| [A2A vs MCP](a2a-vs-mcp.md) | A2A 与 MCP 全面对比：定位、交互模式、适用场景、采纳现状 | `protocol` `comparison` |

## 实体

暂无条目。

## 查询存档

暂无条目。

## 笔记

暂无条目。

---

*共 9 个条目 · 最后更新：2026-05-05*
