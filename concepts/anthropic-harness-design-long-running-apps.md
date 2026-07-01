---
title: Anthropic Harness 设计
created: 2026-03-24
updated: 2026-03-24
type: concept
tags: [agent, anthropic, harness-design]
sources: [raw/articles/anthropic-harness-design-long-running-apps.md]
original: https://www.anthropic.com/engineering/harness-design-long-running-apps
---

# Anthropic Harness 设计

> 原文链接: [English Original](https://www.anthropic.com/engineering/harness-design-long-running-apps)

*Engineering at Anthropic*

发布于 2026年3月24日

Harness设计在agentic编码的前沿性能中起着关键作用。以下是我们如何在前端设计和长时间自主软件工程方面推动Claude进一步发展。

这项工作起源于我们在前端设计技能和长时间编码Agent Harness方面的早期努力，我的同事和我能够通过提示工程和Harness设计将Claude的性能提高到远高于基线——但两者最终都达到了上限。然后我将这些技术应用于长时间自主编码，从早期Harness工作中传承了两个经验：将构建分解为可管理的块，并使用结构化工件在会话之间传递上下文。

在早期的实验中，我们使用初始化Agent将产品规范分解为任务列表，然后使用编码Agent一次实现一个功能，然后移交工件的上下文。

## Harness设计的三个模式

在构建Agent系统时，我们识别出三个重复出现的Harness设计模式。每个模式解决了长时间工作中的特定挑战：

### 模式1：规划器-生成器-评估器

对于复杂的多步骤任务，我们将Agent工作分解为三个阶段：

1. **规划器Agent**接收高层目标并生成详细的任务分解
2. **生成器Agent**实现任务，一次一个功能
3. **评估器Agent**评估结果并确定工作是否完成

这种分离使每个Agent能够专注于其特定角色，并且更容易调试问题。

### 模式2：状态快照和恢复

对于长时间工作，我们在每个会话后保存完整的状态快照。这包括：
- 文件系统状态
- 当前任务列表
- 先前会话的结果
- 待处理操作

当恢复工作时，我们从最新的快照开始，而不是从头开始。

### 模式3：上下文重置

当上下文窗口接近满时，我们执行上下文重置：
- 识别当前状态
- 保存关键信息
- 从干净状态重新开始

这防止了"上下文焦虑"，即模型在感知到限制临近时过早结束任务。

## 前端设计中的应用

我们将这些Harness设计模式应用于前端设计任务，发现Claude的性能显著提高：

- **任务分解**：规划器Agent将设计需求分解为可管理的组件
- **迭代实现**：生成器Agent一次构建一个组件，迭代优化
- **质量保证**：评估器Agent检查设计一致性和用户体验

这种方法使我们能够完成复杂的前端项目，而不会耗尽上下文窗口或失去对状态跟踪。

## 在长时间自主编码中的应用

在长时间自主编码中，我们应用了相同的Harness设计原则：

1. **初始化阶段**：设置环境、初始化Git仓库、创建项目结构
2. **开发阶段**：实现功能、运行测试、修复错误
3. **集成阶段**：集成所有组件、运行完整测试套件、部署

通过将这些阶段分解为可管理的会话，我们能够完成需要数千个Agent调用的大型编码项目。

## 经验教训

在构建这些Harness时，我们学到了几个关键经验：

### 1. 上下文焦虑是真实的

Claude Sonnet 4.5会在感知到上下文限制临近时过早结束任务。我们通过在Harness中添加显式的上下文重置来解决这个问题。

### 2. 状态跟踪至关重要

长时间工作会产生大量状态。如果不仔细跟踪状态，Agent会丢失进度或重复工作。我们使用结构化工件（例如，进度文件、Git提交）来在会话之间传递状态。

### 3. 分工提高可靠性

将Agent工作分解为专门的阶段（规划、生成、评估）使每个阶段更简单、更可靠。

### 4. 测试驱动开发适用于Agent

我们为每个Harness构建自动化测试。这使我们能够在更改Harnes时捕获回归。

## 未来方向

我们正在探索几个方向来改进Harness设计：

1. **更智能的上下文管理**：更好的算法来压缩和重置上下文窗口
2. **自适应任务分解**：根据Agent能力动态调整任务粒度
3. **更好的工件管理**：更结构化的方式在会话之间传递信息

## 结论

Harness设计是长时间Agent工作的关键因素。通过应用这些模式，我们能够将Claude的性能推向更高，并完成以前不可能的任务。

---

*脚注：*

1. *上下文焦虑是指模型在感知到上下文限制临近时过早结束任务的行为。*

2. *我们使用的三个Harness设计模式是：规划器-生成器-评估器、状态快照和恢复，以及上下文重置。*

3. *在前端设计和长时间自主编码中应用这些模式显著提高了Claude的性能。*

---

**相关文章：**

*   [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
*   [Building a C compiler with a team of parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler)

---

*关于作者：本文由Anthropic的Agent团队撰写。*

## 相关链接
- [[anthropic-harnesses-long-running-agents]] — 面向长时间运行 Agent 的有效 Harness
- [[anthropic-scaling-managed-agents]] — 托管 Agent 架构（长期任务的托管方案）
- [[anthropic-building-effective-agents]] — Agent 设计模式
