---
title: "使用 Claude Agent SDK 构建 Agent"
created: 2026-05-05
updated: 2026-05-05
type: concept
tags: [agent, tool-use, anthropic]
sources: [raw/articles/anthropic-2-building-agents-claude-agent-sdk.md]
original: https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk
---

# 使用 Claude Agent SDK 构建 Agent

> 原文链接: [English Original](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)

Claude Agent SDK 是一组帮助开发者在 Claude Code 之上构建强大 Agent 的工具集。在本文中，我们将介绍如何入门并分享最佳实践。

- 分类：Claude Code、Agent
- 产品：Claude Code、Claude Platform
- 日期：2025年9月29日
- 阅读时间：5分钟

去年，我们与客户分享了构建高效 Agent 的经验教训。从那以后，我们发布了 Claude Code——一个 Agent 编码解决方案，最初是为了支持 Anthropic 的开发者生产力而构建的。

过去几个月里，Claude Code 已经远不止是一个编码工具。在 Anthropic，我们一直在使用它进行深度研究、视频创建和笔记记录，以及无数其他非编码应用。事实上，它已经开始驱动几乎所有我们主要的 Agent 循环。

换句话说，驱动 Claude Code 的 Agent 框架（Claude Code SDK）也可以驱动许多其他类型的 Agent。为了反映这一更广泛的愿景，我们将 Claude Code SDK 更名为 Claude Agent SDK。

在本文中，我们将重点介绍为什么我们构建了 Claude Agent SDK、如何用它构建你自己的 Agent，并分享从我们团队自身部署中涌现的最佳实践。

## 给 Claude 一台计算机

Claude Code 背后的关键设计原则是 Claude 需要程序员每天使用的相同工具。它需要能够在代码库中找到适当的文件、编写和编辑文件、lint 代码、运行代码、调试、编辑，有时还需要迭代执行这些操作直到代码成功。

我们发现，通过给 Claude 访问用户计算机的权限（通过终端），它拥有了像程序员一样编写代码所需的一切。

但这使得 Claude Code 中的 Claude 在非编码任务上也变得有效。通过给它运行 bash 命令、编辑文件、创建文件和搜索文件的工具，Claude 可以读取 CSV 文件、搜索网络、构建可视化、解读指标，以及做各种其他数字工作——简而言之，创建具有计算机能力的通用 Agent。Claude Agent SDK 背后的关键设计原则是给你的 Agent 一台计算机，让它们像人类一样工作。

## 创建新型 Agent

我们相信给 Claude 一台计算机解锁了构建比以前更有效的 Agent 的能力。例如，使用我们的 SDK，开发者可以构建：

- **金融 Agent：** 构建能理解你的投资组合和目标的 Agent，并通过访问外部 API、存储数据和运行代码进行计算来帮助你评估投资。
- **个人助理 Agent：** 构建能帮助你预订旅行和管理日历的 Agent，以及安排约会、整理简报等，通过连接你的内部数据源和跨应用程序跟踪上下文。
- **客户支持 Agent：** 构建能处理高模糊性用户请求（如客户服务工单）的 Agent，通过收集和审查用户数据、连接外部 API、回复用户以及在需要时升级给人工。
- **深度研究 Agent：** 构建能在大文档集合中进行全面研究的 Agent，通过搜索文件系统、分析和综合来自多个来源的信息、跨文件交叉引用数据并生成详细报告。

以及更多。在其核心，SDK 为你提供了构建 Agent 的原语，用于自动化你试图实现的任何工作流。

## 构建 Agent 循环

在 Claude Code 中，Claude 通常在一个特定的反馈循环中操作：收集上下文 → 执行操作 → 验证工作 → 重复。Agent 通常在一个特定的反馈循环中操作：收集上下文 → 执行操作 → 验证工作 → 重复。

这为思考其他 Agent 以及它们应该具备的能力提供了一种有用的方式。为了说明这一点，我们将通过在 Claude Agent SDK 中构建邮件 Agent 的示例来进行讲解。

## 收集上下文

在开发 Agent 时，你希望给它的不仅仅是一个 prompt：它需要能够获取和更新自己的上下文。以下是 SDK 中的功能如何帮助实现这一点的。

### Agentic 搜索和文件系统

文件系统代表可以拉入模型上下文的信息。

当 Claude 遇到大文件（如日志或用户上传的文件）时，它会决定使用 bash 脚本（如 `grep` 和 `tail`）以何种方式将这些内容加载到其上下文中。本质上，Agent 的文件夹和文件结构成为了一种上下文工程的形式。

我们的邮件 Agent 可能会将之前的对话存储在名为"Conversations"的文件夹中。这使它能够在被问及时搜索这些历史对话以获取上下文。

### 语义搜索

语义搜索通常比 agentic 搜索更快，但准确性较低，更难维护，且透明度较低。它涉及将相关上下文"分块"（chunking），将这些块嵌入为向量，然后通过查询这些向量来搜索概念。鉴于其局限性，我们建议从 agentic 搜索开始，只有在需要更快结果或更多变体时才添加语义搜索。

### Subagent

Claude Agent SDK 默认支持 subagent。Subagent 有两个主要用途。首先，它们实现了并行化：你可以启动多个 subagent 同时处理不同任务。其次，它们帮助管理上下文：subagent 使用自己隔离的 context window，只将相关信息发送回编排器，而不是完整上下文。这使它们非常适合需要筛选大量信息且大部分信息不会有用的情况。

在设计我们的邮件 Agent 时，我们可能会给它一个"搜索 subagent"功能。邮件 Agent 可以并行启动多个搜索 subagent——每个对邮件历史运行不同查询——并让它们只返回相关摘录而非完整邮件线程。

### Compaction

当 Agent 长时间运行时，上下文维护变得至关重要。Claude Agent SDK 的 compact 功能在接近上下文限制时自动总结之前的消息，使你的 Agent 不会耗尽上下文。这建立在 Claude Code 的 compact slash command 之上。

## 执行操作

收集上下文后，你希望给 Agent 灵活的执行操作方式。

### 工具（Tools）

工具是 Agent 执行的主要构建模块。工具在 Claude 的 context window 中占据显著位置，使它们成为 Claude 在决定如何完成任务时考虑的主要操作。这意味着你应该有意识地设计工具以最大化上下文效率。你可以在我们的博客文章 "Writing effective tools for agents – with agents" 中看到更多最佳实践。

因此，你的工具应该是你希望 Agent 执行的主要操作。了解如何在 Claude Agent SDK 中制作自定义工具。

对于我们的邮件 Agent，我们可能会定义如"fetchInbox"或"searchEmails"之类的工具作为 Agent 的主要、最频繁的操作。

### Bash 和脚本

Bash 作为通用工具很有用，允许 Agent 使用计算机进行灵活的工作。

在我们的邮件 Agent 中，用户可能在附件中存储了重要信息。Claude 可以编写代码下载 PDF，将其转换为文本，并搜索其中以找到有用的信息，如下所示：

### 代码生成

Claude Agent SDK 擅长代码生成——这是有充分理由的。代码精确、可组合且可无限复用，使其成为需要可靠执行复杂操作的 Agent 的理想输出。

在构建 Agent 时，考虑：哪些任务受益于以代码形式表达？通常，答案能解锁重要能力。

例如，我们最近在 Claude.AI 中推出的文件创建功能完全依赖于代码生成。Claude 编写 Python 脚本来创建 Excel 电子表格、PowerPoint 演示文稿和 Word 文档，确保一致的格式和复杂的、难以通过其他方式实现的功能。

在我们的邮件 Agent 中，我们可能希望允许用户为入站邮件创建规则。为了实现这一点，我们可以编写在该事件发生时运行的代码：

### MCP

Model Context Protocol（MCP）提供了与外部服务的标准化集成，自动处理认证和 API 调用。这意味着你可以将 Agent 连接到 Slack、GitHub、Google Drive 或 Asana 等工具，而无需编写自定义集成代码或自己管理 OAuth 流程。

对于我们的邮件 Agent，我们可能想要搜索 Slack 消息以了解团队上下文，或检查 Asana 任务以查看是否已有人被分配处理客户请求。通过 MCP servers，这些集成开箱即用——你的 Agent 只需调用如 `search_slack_messages` 或 `get_asana_tasks` 之类的工具，MCP 处理其余的一切。

不断增长的 MCP 生态系统意味着你可以随着预构建集成的可用快速为 Agent 添加新能力，让你专注于 Agent 行为。

## 验证你的工作

Claude Code SDK 通过评估其工作来完成 Agent 循环。能够检查和改进自己输出的 Agent 从根本上更可靠——它们在错误累积之前捕获错误，在偏离时自我纠正，并在迭代过程中变得更好。

关键是给 Claude 具体评估其工作的方法。以下是我们发现有效的三种方法：

### 定义规则

反馈的最佳形式是为输出提供明确定义的规则，然后解释哪些规则失败以及为什么。

代码 linting 是一种优秀的基于规则的反馈形式。反馈越深入越好。例如，生成 TypeScript 并进行 lint 通常比生成纯 JavaScript 更好，因为它为你提供了多个额外的反馈层。

在生成邮件时，你可能希望 Claude 检查电子邮件地址是否有效（如果不是，抛出错误）以及用户之前是否发送过邮件给他们（如果是，发出警告）。

### 视觉反馈

当使用 Agent 完成视觉任务（如 UI 生成或测试）时，视觉反馈（以截图或渲染的形式）会很有帮助。例如，如果发送带有 HTML 格式的邮件，你可以截取生成的邮件截图并将其提供给模型进行视觉验证和迭代改进。模型然后会检查视觉输出是否与请求匹配。

例如：
- **布局** - 元素是否正确定位？间距是否合适？
- **样式** - 颜色、字体和格式是否按预期出现？
- **内容层次** - 信息是否以正确的顺序呈现并带有适当的强调？
- **响应性** - 是否看起来破损或拥挤？（虽然单个截图的视口信息有限）

使用像 Playwright 这样的 MCP server，你可以在 Agent 的工作流中自动化这个视觉反馈循环——截取渲染 HTML 的截图、捕获不同的视口大小，甚至测试交互元素。来自大语言模型（LLM）的视觉反馈可以为你的 Agent 提供有用的指导。

### LLM 作为评判者

你还可以让另一个语言模型基于模糊规则"评判"你 Agent 的输出。这通常不是非常健壮的方法，并且可能有较大的延迟代价，但对于任何性能提升都值得成本的应用，它可以是有帮助的。

我们的邮件 Agent 可能有一个单独的 subagent 来评判其草稿的语气，看看它们是否与用户之前的消息风格匹配。

## 测试和改进你的 Agent

在完成 Agent 循环几次之后，我们建议测试你的 Agent，并确保它为任务配备了良好的工具。改进 Agent 的最佳方法是仔细查看其输出，特别是失败的情况，并站在它的角度思考：它是否有完成工作所需的正确工具？

在评估你的 Agent 是否配备良好以完成其工作时，这里有一些其他问题可以问：
- 如果你的 Agent 误解了任务，它可能缺少关键信息。你能改变搜索 API 的结构使其更容易找到它需要知道的内容吗？
- 如果你的 Agent 反复在某个任务上失败，你能在工具调用中添加正式规则来识别和修复失败吗？
- 如果你的 Agent 无法修复其错误，你能给它更有用或更有创意的工具来以不同方式解决问题吗？
- 如果你的 Agent 的性能随着添加功能而变化，请基于客户使用情况构建具有代表性的测试集用于程序化评估（或 evals）。

## 开始使用

Claude Agent SDK 通过让 Claude 访问可以编写文件、运行命令和迭代工作的计算机，使构建自治 Agent 变得更容易。

牢记 Agent 循环（收集上下文、执行操作和验证工作），你可以构建易于部署和迭代的可靠 Agent。

你今天就可以开始使用 Claude Agent SDK。对于已经在 SDK 上构建的开发者，我们建议按照此指南迁移到最新版本。

## 致谢

由 Thariq Shihipar 撰写，Molly Vorwerck、Suzanne Wang、Alex Isken、Cat Wu、Keir Bradwell、Alexander Bricken 和 Ashwin Bhat 提供笔记和编辑。
