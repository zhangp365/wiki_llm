---
title: "使用 Agent Skills 为 Agent 装备真实世界能力"
created: 2026-05-05
updated: 2026-05-05
type: concept
tags: [agent, tool-use, anthropic]
sources: [raw/articles/anthropic-6-agent-skills.md]
original: https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
---

# 使用 Agent Skills 为 Agent 装备真实世界能力

> 原文链接: [English Original](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)

*Engineering at Anthropic*

发布于 2025年10月16日

Claude 很强大，但真实工作需要程序性知识和组织上下文。推出 Agent Skills——一种使用文件和文件夹构建专业化 agent 的新方法。

> **更新：** 我们已将 Agent Skills 作为跨平台可移植性的开放标准发布。（2025年12月18日）

随着模型能力的提升，我们现在可以构建与完整计算环境交互的通用 agent。例如，Claude Code 可以使用本地代码执行和文件系统跨领域完成复杂任务。但随着这些 agent 变得更加强大，我们需要更加可组合、可扩展和可移植的方式来为它们配备领域专业知识。

这促使我们创建了 Agent Skills：有组织的指令、脚本和资源文件夹，agent 可以动态发现和加载这些内容以更好地执行特定任务。Skills 通过将你的专业知识打包成可组合的资源来扩展 Claude 的能力，将通用 agent 转变为满足你需求的专业化 agent。

为 agent 构建 skill 就像为新人准备入职指南。与其为每个用例构建碎片化的、定制设计的 agent，任何人现在都可以通过捕获和共享他们的程序性知识来用可组合的能力专业化他们的 agent。在本文中，我们将解释 Skills 是什么，展示它们如何工作，并分享构建你自己的 skill 的最佳实践。

Skill 是一个包含 `SKILL.md` 文件的目录，该文件包含有组织的指令、脚本和资源文件夹，赋予 agent 额外能力。

## Skill 的结构

要看到 Skills 的实际应用，让我们来看一个真实的例子：驱动 Claude 最近推出的文档编辑能力的 skill 之一。Claude 已经对理解 PDF 有了很多了解，但在直接操作它们方面能力有限（例如填写表格）。这个 PDF skill 让我们能够赋予 Claude 这些新能力。

最简单的情况下，skill 是一个包含 `SKILL.md` 文件的目录。该文件必须以包含一些必需元数据的 YAML frontmatter 开始：`name` 和 `description`。在启动时，agent 将每个已安装 skill 的名称和描述预加载到其系统提示中。

此元数据是渐进式信息披露的第一层：它为 Claude 提供了刚好足够的信息，使其知道何时应使用每个 skill，而无需将所有内容加载到上下文中。该文件的实际正文是第二层细节。如果 Claude 认为该 skill 与当前任务相关，它会通过将完整的 `SKILL.md` 读入上下文来加载该 skill。

`SKILL.md` 文件必须以包含文件名和描述的 YAML Frontmatter 开始，这些内容在启动时被加载到系统提示中。

随着 skill 在复杂度上的增长，它们可能包含过多的上下文而无法放入单个 `SKILL.md` 中，或者包含仅在特定场景中相关的上下文。在这些情况下，skill 可以在 skill 目录中捆绑额外文件，并从 `SKILL.md` 中按名称引用它们。这些额外的链接文件是第三层（及更多层）细节，Claude 可以根据需要选择性地导航和发现。

在下面显示的 PDF skill 中，`SKILL.md` 引用了 skill 作者选择与核心 `SKILL.md` 捆绑的两个额外文件（`reference.md` 和 `forms.md`）。通过将表格填写说明移到单独的文件（`forms.md`）中，skill 作者能够保持 skill 核心的精简，相信 Claude 只在填写表格时才会读取 `forms.md`。

你可以在 skill 中纳入更多上下文（通过额外文件），然后根据需要由 Claude 动态发现。

## Skills 和 Context Window

下图展示了当 skill 被用户消息触发时 context window 如何变化。

Skills 通过你的系统提示在 context window 中被触发。所示操作序列为：

- 首先，context window 包含核心系统提示和每个已安装 skill 的元数据，以及用户的初始消息。
- Claude 通过调用 Bash 工具读取 `pdf/SKILL.md` 的内容来触发 PDF skill。
- Claude 选择读取与 skill 捆绑的 `forms.md` 文件。
- 最后，Claude 在从 PDF skill 加载了相关指令后继续执行用户的任务。

## Skills 和代码执行

Skills 还可以包含让 Claude 作为工具自行决定执行的代码。

大型语言模型在许多任务上表现出色，但某些操作更适合传统代码执行。例如，通过 token 生成排序列表远比运行排序算法昂贵得多。除了效率考虑，许多应用需要只有代码才能提供的确定性可靠性。

在我们的示例中，PDF skill 包含一个预先编写的 Python 脚本，用于读取 PDF 并提取所有表格字段。Claude 可以运行此脚本而无需将脚本或 PDF 加载到上下文中。而且因为代码是确定性的，这个工作流是一致且可重复的。Skills 还可以包含让 Claude 根据任务性质自行决定作为工具执行的代码。

## 开发和评估 Skills

以下是一些关于开始编写和测试 skills 的有用指南：

- **从评估开始**：通过在代表性任务上运行 agent 并观察它们在哪些地方遇到困难或需要额外上下文来识别 agent 能力中的具体差距。然后逐步构建 skills 来解决这些不足。
- **为扩展而结构化**：当 `SKILL.md` 文件变得难以管理时，将其内容拆分为单独的文件并引用它们。如果某些上下文是互斥的或很少一起使用，保持路径分开将减少 token 使用量。最后，代码既可以作为可执行工具，也可以作为文档。应该明确 Claude 是应该直接运行脚本还是将它们作为参考读入上下文。
- **从 Claude 的角度思考**：监控 Claude 在真实场景中如何使用你的 skill，并根据观察进行迭代：注意意外的轨迹或对某些上下文的过度依赖。特别关注你的 skill 的名称和描述。Claude 在决定是否根据当前任务触发 skill 时会使用这些信息。
- **与 Claude 一起迭代**：当你与 Claude 一起处理任务时，让 Claude 将其成功的方法和常见错误捕获到 skill 中可重用的上下文和代码中。如果 Claude 在使用 skill 完成任务时偏离轨道，让它自我反思哪里出了问题。这个过程将帮助你发现 Claude 实际需要什么上下文，而不是试图提前预判。

## 使用 Skills 的安全注意事项

Skills 通过指令和代码为 Claude 提供新能力。虽然这使它们很强大，但也意味着恶意 skill 可能在使用环境中引入漏洞，或指示 Claude 窃取数据和执行非预期操作。

我们建议仅从可信来源安装 skills。当从较不可信的来源安装 skill 时，在使用前应彻底审计。首先阅读 skill 中捆绑文件的内容以了解其功能，特别关注代码依赖项和捆绑的资源（如图像或脚本）。同样，注意 skill 中指示 Claude 连接到潜在不可信外部网络源的指令或代码。

## Skills 的未来

Agent Skills 目前已在 Claude.ai、Claude Code、Claude Agent SDK 和 Claude Developer Platform 中得到支持。

在未来几周，我们将继续添加支持创建、编辑、发现、共享和使用 Skills 全生命周期的新功能。我们对 Skills 帮助组织和个人与 Claude 共享其上下文和工作流的机会感到特别兴奋。我们还将探索 Skills 如何补充 Model Context Protocol（MCP） server，通过教导 agent 涉及外部工具和软件的更复杂工作流。

展望更远的未来，我们希望让 agent 能够自主创建、编辑和评估 Skills，让它们将自己的行为模式编码为可重用的能力。

Skills 是一个简单的概念，具有相应简单的格式。这种简单性使组织、开发者和终端用户更容易构建定制化的 agent 并赋予它们新能力。

我们期待看到人们用 Skills 构建什么。今天就通过查看我们的 Skills 文档和 cookbook 开始吧。

## 致谢

由 Barry Zhang、Keith Lazuka 和 Mahesh Murag 撰写，他们都非常喜欢文件夹。特别感谢 Anthropic 内部许多倡导、支持和构建 Skills 的其他人。