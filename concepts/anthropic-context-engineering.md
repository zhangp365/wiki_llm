---
title: Anthropic 上下文工程
created: 2026-05-05
updated: 2026-05-05
type: concept
tags: [agent, tool-use, anthropic, context-engineering]
sources: [raw/articles/anthropic-7-context-engineering.md]
original: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
---

# Anthropic 面向 AI Agent 的有效 Context Engineering

> 原文链接: [English Original](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

Context 是 AI Agent 的一项关键但有限的资源。在本文中，我们将探讨有效策划和管理驱动 Agent 的 context 的策略。

在 prompt engineering 成为应用 AI 领域关注焦点的几年之后，一个新术语开始脱颖而出：**context engineering**。基于语言模型的构建正变得越来越少地关注如何为 prompt 找到合适的词语和短语，而更多地转向回答一个更广泛的问题："什么样的 context 配置最有可能产生模型期望的行为？"

Context 是指从大型语言模型（LLM）采样时包含的 token 集合。当前的工程问题是在 LLM 固有约束下优化这些 token 的效用，以持续达成期望的结果。有效地驾驭 LLM 通常需要以 context 的方式思考——换句话说：考虑 LLM 在任何给定时刻可用的整体状态，以及该状态可能产生哪些潜在行为。

在本文中，我们将探讨 context engineering 这一新兴艺术，并为构建可操控、高效的 Agent 提供一个精炼的心智模型。

## Context Engineering 与 Prompt Engineering 的区别

在 Anthropic，我们将 context engineering 视为 prompt engineering 的自然演进。Prompt engineering 是指编写和组织 LLM 指令以获得最优结果的方法（参见我们的文档了解概述和有用的 prompt engineering 策略）。Context engineering 是指在 LLM 推理期间策划和维护最优 token（信息）集的策略，包括 prompt 之外可能进入 context 的所有其他信息。

在使用 LLM 进行工程的早期，prompt 是 AI 工程工作的最大组成部分，因为日常聊天交互之外的大多数用例都需要针对一次性分类或文本生成任务优化的 prompt。顾名思义，prompt engineering 的主要焦点是如何编写有效的 prompt，特别是系统 prompt。然而，随着我们走向工程化更强大的、在多轮推理和更长时间跨度上运行的 Agent，我们需要管理整个 context 状态（系统指令、工具、Model Context Protocol (MCP)、外部数据、消息历史等）的策略。

在循环中运行的 Agent 会生成越来越多的可能与下一轮推理相关的数据，这些信息必须被循环优化。Context engineering 是一种艺术和科学，它从不断演变的可能信息宇宙中策划出有限 context window 中应包含的内容。

与编写 prompt 的离散任务不同，context engineering 是迭代的，策划阶段在我们每次决定传递给模型什么内容时都会发生。

## 为什么 Context Engineering 对构建强大的 Agent 至关重要

尽管 LLM 速度越来越快、管理数据量的能力越来越大，但我们观察到，LLM 和人类一样，在某个点会失去焦点或经历困惑。对"大海捞针"式基准测试的研究揭示了 **context rot**（上下文腐化）的概念：随着 context window 中 token 数量的增加，模型从该 context 中准确回忆信息的能力会下降。

虽然一些模型表现出比其他模型更温和的退化，但这一特征在所有模型中都会出现。因此，context 必须被视为一种边际收益递减的有限资源。就像具有有限工作记忆容量的人类一样，LLM 在解析大量 context 时也有一个"注意力预算"。引入的每一个新 token 都会消耗这个预算的一部分，增加了仔细策划可用 token 的需求。

这种注意力稀缺性源于 LLM 的架构约束。LLM 基于 transformer 架构，该架构使得每个 token 都能关注整个 context 中的所有其他 token。这对 $n$ 个 token 产生了 $n^2$ 的成对关系。

随着 context 长度的增加，模型捕捉这些成对关系的能力会被摊薄，在 context 大小和注意力焦点之间产生自然的张力。此外，模型从训练数据分布中发展其注意力模式，其中较短序列通常比较长序列更常见。这意味着模型对 context 范围内的依赖关系的经验更少，专门的参数也更少。

Position encoding interpolation 等技术允许模型通过将其适应最初训练的较小 context 来处理更长的序列，尽管 token 位置理解会有一些退化。这些因素创造了一个性能梯度而非硬性悬崖：模型在更长 context 下仍然高度有能，但在信息检索和长程推理方面可能表现出降低的精度。

## 有效 Context 的解剖结构

鉴于 LLM 受限于有限的注意力预算，良好的 context engineering 意味着找到尽可能最小的、最大化某些期望结果可能性的高信号 token 集。实施这一实践说起来容易做起来难，但在以下部分，我们概述了这一指导原则在 context 的不同组件中的实际含义。

**系统 prompt** 应该非常清晰，使用简单直接的语言，以适合 Agent 的合适高度呈现概念。合适的高度是两种常见失败模式之间的 Goldilocks 区间。在一个极端，我们看到工程师在 prompt 中硬编码复杂、脆弱的逻辑以引发精确的 Agent 行为。这种方法随着时间的推移会制造脆弱性并增加维护复杂性。在另一个极端，工程师有时提供模糊的高层指导，未能给 LLM 提供期望输出的具体信号，或错误地假设共享的 context。最优高度在两者之间取得平衡：足够具体以有效引导行为，又足够灵活以为模型提供强有力的启发式指导来引导行为。

频谱的一端是脆弱的 if-else 硬编码 prompt，另一端是过于笼统或错误假设共享 context 的 prompt。

我们建议将 prompt 组织成不同的部分（如 `<role>`、`<tool_guidance>`、`## Output description` 等），并使用 XML 标签或 Markdown 标题等技术来划分这些部分，尽管随着模型变得更加强大，prompt 的确切格式可能变得不那么重要。

无论你决定如何构建系统 prompt，都应该努力提供完全概述期望行为的最小信息集。（请注意，最小不一定意味着简短；你仍然需要提前给 Agent 足够的信息以确保它遵循期望的行为。）最好从用最好的模型测试一个最小 prompt 开始，然后仅在有证据表明 Agent 偏离预期行为时才添加更多指令。

**工具定义** 本质上也是 context，并且同样受益于精心策划。每个工具定义所用的 token 都会从可用的注意力预算中消耗一部分，因此工具定义的 token 效率对于 Agent 性能至关重要。我们在之前关于构建高效 Agent 的帖子中讨论了 token 效率工具的概念——工具的输入和输出模式经过精心设计，使用最少的 token 来传递最大量的信息。

**检索到的 context** 是 Agent 被赋予的任务相关信息。理想情况下，这是通过 RAG 或 agentic search 获取的，我们将在下面讨论。

## Context 检索与 Agentic Search

在 [[anthropic-building-effective-agents|构建高效 AI Agent]] 一文中，我们强调了基于 LLM 的工作流和 Agent 之间的区别。自那篇文章发表以来，我们倾向于对 Agent 采用一个简单的定义：LLM 在循环中自主使用工具。

与客户合作，我们看到业界正在向这个简单的范式靠拢。随着底层模型变得更有能力，Agent 的自主水平可以扩展：更聪明的模型允许 Agent 独立导航细微差别的问题空间并从错误中恢复。

我们现在看到工程师在设计 Agent 的 context 方式上的转变。如今，许多 AI 原生应用程序采用某种形式的基于嵌入的预推理时检索来提供重要的 context 供 Agent 推理。随着领域转向更 Agent 化的方法，我们越来越多地看到团队用"just in time"（即时）context 策略来增强这些检索系统。

"just in time"方法不是预先处理所有相关数据，而是让 Agent 维护轻量级标识符（文件路径、存储的查询、Web 链接等），并使用这些引用在运行时通过工具动态加载数据到 context 中。Anthropic 的 Agent 编码解决方案 Claude Code 使用这种方法对大型数据库执行复杂的数据分析。模型可以编写针对性的查询、存储结果，并利用 Bash 命令如 `head` 和 `tail` 分析大量数据，而无需将完整数据对象加载到 context 中。这种方法反映了人类的认知方式：我们通常不会记住整个信息库，而是引入外部组织和索引系统（如文件系统、收件箱和书签）来按需检索相关信息。

除了存储效率之外，这些引用的元数据还提供了一种有效优化行为的机制，无论是否显式提供或是直观的。对于一个在文件系统中操作的 Agent 来说，tests 文件夹中名为 `test_utils.py` 的文件的存在意味着与 `utils.py` 文件不同的功能角色。这种元数据使 Agent 能够做出更明智的决策，而不需要先读取这些文件的全部内容。

在 RAG 之外，我们还看到了 Agent 自主确定何时检索信息的价值——通过使用搜索工具、读取文档或执行代码来探索其环境。这赋予了 Agent 自主探索的灵活性，类似于人类工程师在项目之间导航或阅读文档以解决具体问题时的方式。

## 长时间跨度任务的 Context Engineering

长时间跨度任务要求 Agent 在 token 数超过 LLM context window 的动作序列中保持连贯性、context 和目标导向行为。对于跨越数十分钟到数小时持续工作的任务，如大型代码库迁移或综合研究项目，Agent 需要专门的技术来绕过 context window 大小的限制。

等待更大的 context window 似乎是一个明显的策略。但可以预见的是，至少在想要最强 Agent 性能的情况下，所有大小的 context window 都会受到 context 污染和信息相关性问题的影响。为了使 Agent 能够在扩展的时间跨度上有效工作，我们开发了几种直接解决 context 污染约束的技术：**compaction**、**structured note-taking** 和 **multi-agent architectures**。

### Compaction

Compaction 是将近乎 context window 限制的对话进行总结，然后以总结内容重新启动新 context window 的做法。Compaction 通常作为 context engineering 中驱动更好长期连贯性的第一个杠杆。其核心是以高保真方式蒸馏 context window 的内容，使 Agent 能够以最小的性能退化继续工作。

例如在 Claude Code 中，我们通过将消息历史传递给模型来总结和压缩最关键的细节来实现这一点。模型保留架构决策、未解决的 bug 和实现细节，同时丢弃冗余的工具输出或消息。Agent 然后可以继续使用这个压缩的 context 加上最近访问的五个文件。用户获得连续性而无需担心 context window 的限制。

Compaction 的艺术在于选择保留什么与丢弃什么，因为过于激进的 compaction 可能导致丢失微妙但关键的 context。

### 结构化笔记

结构化笔记是一种 practice，即要求 Agent 在每轮结束时明确写下一份简明的状态报告。这在 Agent 之间传递信息方面比仅依赖 compacted context 更有效。

这种方法为下一个 Agent 提供了一个清晰、可操作的起点，大大减少了"猜测之前发生了什么"的时间。关键在于为笔记创建一个结构化的格式——在 Claude Code 中，我们使用 claude-progress.txt 文件，Agent 在其中记录其工作状态和计划。

### Multi-Agent Architectures

Multi-agent architectures 将 long-horizon 任务分解为专门的子任务，每个子任务由一个专门的 Agent 处理。通过给每个 Agent 一个更窄的 focus，可以减少每个 Agent 需要跟踪的 context 量，并使整体行为更加可靠。一个 Agent 可以专注于研究，另一个专注于写作，还有一个专注于审查——每个 Agent 都从之前 Agent 的输出中受益，而无需承担整个上下文的负担。

## 结语

Context engineering 代表了我们使用 LLM 构建方式的一个根本性转变。随着模型变得更有能力，挑战不仅仅是编写完美的 prompt——而是深思熟虑地策划每一步进入模型有限注意力预算的信息。无论你是在为长时间跨度任务实现 compaction、设计 token 高效的工具，还是使 Agent 能够即时探索其环境，指导原则保持不变：找到最大化期望结果可能性的最小高信号 token 集。

我们概述的技术将继续随着模型的改进而发展。我们已经看到更聪明的模型需要更少的规范性工程，允许 Agent 以更多自主权运行。但即使能力扩展，将 context 视为珍贵有限的资源仍将是构建可靠、高效 Agent 的核心。

立即在 Claude Developer Platform 中开始使用 context engineering，并通过我们的 memory 和 context management cookbook 获取有用的技巧和最佳实践。

## 致谢

本文由 Anthropic 的 Applied AI 团队撰写：Prithvi Rajasekaran、Ethan Dixon、Carly Ryan 和 Jeremy Hadfield，Rafi Ayub、Hannah Moran、Cal Rueb 和 Connor Jennings 也有贡献。特别感谢 Molly Vorwerck、Stuart Ritchie 和 Maggie Vo 的支持。

## 相关链接
- [[anthropic-contextual-retrieval]] — Contextual Retrieval 上下文感知检索
- [[anthropic-think-tool]] — 复杂任务中的结构化思考工具
- [[anthropic-harnesses-long-running-agents]] — 长时间运行 Agent 的 Harness（context 管理实践）
