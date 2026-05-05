---
title: "面向长时间运行 Agent 的有效 Harness"
created: 2026-05-05
updated: 2026-05-05
type: concept
tags: [agent, tool-use, anthropic, long-running, harness]
sources: [raw/articles/anthropic-9-harnesses-long-running-agents.md]
original: https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
---

# 面向长时间运行 Agent 的有效 Harness

> 原文链接: [English Original](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)

Agent 在跨多个 context window 工作时仍面临挑战。我们从人类工程师那里汲取灵感，为长时间运行的 Agent 创建了更有效的 harness。

随着 AI Agent 变得更有能力，开发者越来越多地要求它们承担需要数小时甚至数天工作的复杂任务。然而，让 Agent 在多个 context window 中保持一致的进展仍然是一个开放性问题。

长时间运行 Agent 的核心挑战在于，它们必须在离散的会话中工作，而每个新会话开始时都没有之前的记忆。想象一个由轮班工程师组成的软件项目，每个新工程师到来时都没有上一班发生了什么的记忆。由于 context window 是有限的，且大多数复杂项目无法在单个 window 中完成，Agent 需要一种方法来弥合编码会话之间的差距。

我们开发了一个两方面的解决方案，使 Claude Agent SDK 能够在多个 context window 中有效工作：一个在首次运行时设置环境的 **initializer agent**，以及一个在每个会话中承担增量进展任务同时为下一个会话留下清晰记录的 **coding agent**。你可以在配套的 quickstart 中找到代码示例。

## 长时间运行 Agent 的问题

Claude Agent SDK 是一个强大的通用 Agent harness，擅长编码以及其他需要模型使用工具收集 context、规划和执行的任务。它具有 context 管理功能，如 compaction，使 Agent 可以在不耗尽 context window 的情况下工作。理论上，在这种设置下，Agent 应该能够无限期地继续做有用的工作。

然而，compaction 并不足够。开箱即用，即使是像 Opus 4.5 这样的前沿编码模型在 Claude Agent SDK 上跨多个 context window 循环运行，如果只给定一个高层级的 prompt（如"构建一个 claude.ai 的克隆"），也无法构建一个生产质量的 Web 应用。

Claude 的失败表现为两种模式。首先，Agent 倾向于一次做太多——本质上是尝试一次性完成整个应用。通常，这导致模型在实现过程中 context 耗尽，留给下一个会话的是一个功能半实现且无文档记录的状态。Agent 然后不得不猜测发生了什么，并花费大量时间试图让基本应用重新工作。即使使用 compaction 也会发生这种情况，因为 compaction 并不总是向下一个 Agent 传递完全清晰的指令。

第二种失败模式通常在项目后期出现。一些功能已经构建后，后面的 Agent 实例会环顾四周，看到已经取得了进展，然后宣布工作完成。

这将问题分解为两部分。首先，我们需要设置一个初始环境，为给定 prompt 所需的所有功能奠定基础，这将 Agent 设置为逐步、逐功能地工作。其次，我们应该 prompt 每个 Agent 朝着其目标取得增量进展，同时在会话结束时将环境保持在干净状态。所谓"干净状态"是指适合合并到主分支的那种代码：已提交到 git、没有未完成的半成品功能、有清晰的 commit message。

关键洞察在于找到一种方法让 Agent 在使用全新 context window 开始工作时快速理解工作状态，这通过 claude-progress.txt 文件与 git 历史配合实现。这些实践的灵感来自于了解高效软件工程师每天做的事情。

## 环境管理

在更新的 Claude 4 prompting guide 中，我们分享了一些多 context window 工作流的最佳实践，包括使用"不同 prompt 用于第一个 context window"的 harness 结构。这个"不同 prompt"要求 initializer agent 用所有必要的 context 设置环境，以便未来的 coding agent 能够有效工作。在这里，我们深入探讨这种环境的一些关键组件。

## 功能列表

为了解决 Agent 一次性完成应用或过早认为项目完成的问题，我们 prompt initializer agent 编写一个全面的功能需求文件，扩展用户的初始 prompt。在 claude.ai 克隆示例中，这意味着超过 200 个功能，如"用户可以打开新聊天、输入查询、按回车并看到 AI 响应"。这些功能最初都标记为"failing"，以便后来的 coding agent 对完整功能的样子有清晰的概览。

```json
{
  "category": "functional",
  "description": "New chat button creates a fresh conversation",
  "steps": [
    "Navigate to main interface",
    "Click the 'New Chat' button",
    "Verify a new conversation is created",
    "Check that chat area shows welcome state",
    "Verify conversation appears in sidebar"
  ],
  "passes": false
}
```

我们 prompt coding agent 只通过更改 passes 字段的状态来编辑此文件，并使用措辞强烈的指令，如"删除或编辑测试是不可接受的，因为这可能导致功能缺失或有 bug。"经过一些实验，我们决定使用 JSON 格式，因为与 Markdown 文件相比，模型不太可能不当地更改或覆盖 JSON 文件。

## 增量进展

有了这个初始环境脚手架，coding agent 的下一次迭代被要求一次只处理一个功能。这种增量方法被证明是解决 Agent 一次做太多倾向的关键。

一旦以增量方式工作，模型在更改代码后仍然需要将环境保持在干净状态。在我们的实验中，我们发现引发这种行为的最佳方式是要求模型通过描述性的 commit message 将其进展提交到 git，并在进度文件中写入进展总结。这使得模型能够使用 git 恢复错误的代码更改并恢复代码库的工作状态。

这些方法也提高了效率，因为它们消除了 Agent 不得不猜测发生了什么并花费时间试图让基本应用重新工作的需要。

## 测试

我们观察到的最后一个主要失败模式是 Claude 倾向于在没有适当测试的情况下将功能标记为完成。在缺少显式 prompting 的情况下，Claude 倾向于进行代码更改，甚至使用单元测试或对开发服务器的 curl 命令进行测试，但未能认识到功能端到端不工作。

在构建 Web 应用的情况下，一旦被显式 prompt 使用浏览器自动化工具并以人类用户的方式进行所有测试，Claude 在端到端验证功能方面做得很好。Claude 通过 Puppeteer MCP server 在测试 claude.ai 克隆时截取的屏幕截图。

为 Claude 提供这些测试工具极大地改善了性能，因为 Agent 能够识别和修复仅从代码中看不明显的 bug。

一些问题仍然存在，比如 Claude 视觉能力的限制和浏览器自动化工具使其难以识别每种类型的 bug。例如，Claude 无法通过 Puppeteer MCP 看到浏览器原生的 alert 模态框，因此依赖这些模态框的功能往往更有 bug。

## 快速上手

有了以上所有设置，每个 coding agent 被 prompt 执行一系列步骤来了解当前状况，其中一些很基础但仍然有帮助：

- 运行 `pwd` 查看你在哪个目录工作。你只能编辑此目录中的文件。
- 阅读 git 日志和进度文件以了解最近的工作进展。
- 阅读功能列表文件，选择尚未完成的最高优先级功能来处理。

这种方法为 Claude 在每个会话中节省了一些 token，因为它不需要弄清楚如何测试代码。让 initializer agent 编写一个可以运行开发服务器的 `init.sh` 脚本也很有帮助，然后在实现新功能之前运行基本端到端测试。

在 claude.ai 克隆的情况下，这意味着 Agent 总是启动本地开发服务器并使用 Puppeteer MCP 开始新聊天、发送消息并接收响应。这确保了 Claude 能够快速识别应用是否被留在损坏状态，并立即修复现有的 bug。如果 Agent 反而开始实现新功能，很可能会使问题变得更糟。

鉴于所有这些，一个典型的会话以以下 assistant 消息开始：

```
[Assistant] I'll start by getting my bearings and understanding the current state of the project.
[Tool Use] pwd
[Tool Use] cat claude-progress.txt
[Tool Use] cat tests.json
[Assistant] Let me check the git log to see recent work.
[Tool Use] git log --oneline -10
[Assistant] Now let me check if there's an init.sh script to restart the servers.
[Tool Use] source init.sh
[Assistant] Excellent! Now let me navigate to the application and verify that some fundamental features are still working.
[Tool Use] browser navigation and testing...
[Assistant] Based on my verification testing, I can see that the fundamental functionality is working well. The core chat features, theme switching, conversation loading, and error handling are all functioning correctly. Now let me review the tests.json file more comprehensively to understand what needs to be implemented next.
```

### Agent 失败模式与解决方案总结

| 问题 | Initializer Agent 行为 | Coding Agent 行为 |
|------|----------------------|------------------|
| Claude 过早宣布整个项目完成 | 设置功能列表文件：基于输入规范，建立一个包含端到端功能描述列表的结构化 JSON 文件 | 在会话开始时阅读功能列表文件。选择单个功能开始处理。 |
| Claude 将环境留在有 bug 或未记录进展的状态 | 编写初始 git 仓库和进度笔记文件 | 通过阅读进度笔记文件和 git commit 日志开始会话，并在开发服务器上运行基本测试以捕获任何未记录的 bug。通过写入 git commit 和进度更新结束会话。 |
| Claude 过早将功能标记为完成 | 设置功能列表文件 | 自我验证所有功能。仅在仔细测试后将功能标记为"passing"。 |
| Claude 必须花时间弄清楚如何运行应用 | 编写可以运行开发服务器的 init.sh 脚本 | 通过阅读 init.sh 开始会话。 |

## 未来工作

这项研究展示了长时间运行 Agent harness 中一种可能的解决方案集，使模型能够在多个 context window 中取得增量进展。然而，仍有开放性问题。

最值得注意的是，目前尚不清楚单个通用编码 Agent 是否在所有 context 中表现最佳，还是通过 multi-agent 架构可以实现更好的性能。可以合理推断，像测试 Agent、质量保证 Agent 或代码清理 Agent 这样的专门 Agent 可以在软件开发生命周期的子任务上做得更好。

此外，这个 demo 针对全栈 Web 应用开发进行了优化。一个未来方向是将这些发现推广到其他领域。很可能这些经验中的一些或全部可以应用于例如科学研究或金融建模中所需的长时间运行 Agent 任务类型。

## 致谢

本文由 Justin Young 撰写。特别感谢 David Hershey、Prithvi Rajasakeran、Jeremy Hadfield、Naia Bouscal、Michael Tingley、Jesse Mu、Jake Eaton、Marius Buleandara、Maggie Vo、Pedram Navid、Nadine Yasser 和 Alex Notov 的贡献。

这项工作反映了 Anthropic 多个团队的集体努力，使 Claude 能够安全地进行长时间跨度的自主软件工程，特别是 code RL 和 Claude Code 团队。有意贡献的候选人欢迎在 anthropic.com/careers 申请。

## 脚注

1. 我们在此语境中称它们为独立的 Agent，只是因为它们有不同的初始用户 prompt。系统 prompt、工具集和整体 Agent harness 在其他方面是相同的。