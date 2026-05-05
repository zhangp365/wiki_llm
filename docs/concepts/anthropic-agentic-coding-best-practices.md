---
title: "Claude Code：Agentic 编程最佳实践"
created: 2026-05-05
updated: 2026-05-05
type: concept
tags: [agent, tool-use, anthropic, best-practices, claude-code]
sources: [raw/articles/anthropic-14-agentic-coding-best-practices.md]
original: https://www.anthropic.com/engineering/claude-code-best-practices
---

# Claude Code：Agentic 编程最佳实践

> 原文链接: [English Original](https://www.anthropic.com/engineering/claude-code-best-practices)

Claude Code 是一个 agentic 编程环境。与回答问题然后等待的聊天机器人不同，Claude Code 可以读取你的文件、运行命令、进行修改，在你观看、重定向或完全离开时自主地解决问题。

这改变了你的工作方式。你不再自己编写代码然后让 Claude 审查，而是描述你想要什么，Claude 会弄清楚如何构建它。Claude 会探索、规划和实现。

但这种自主性仍然有一个学习曲线。Claude 在你需要理解的某些约束条件下工作。

本指南涵盖了在 Anthropic 内部团队以及在各种代码库、语言和环境中使用 Claude Code 的工程师中已被证明有效的模式。关于 agentic loop 的工作原理，请参阅"如何使用 Claude Code"。

大多数最佳实践都基于一个约束：**Claude 的 context window 填充很快，并且性能会随着填充而下降。**

Claude 的 context window 保存你的整个对话，包括每条消息、Claude 读取的每个文件以及每个命令输出。然而，这会很快填满。一个单独的调试会话或代码库探索可能生成并消耗数万个 token。

这很重要，因为 LLM 性能会随着 context 填充而下降。当 context window 快满时，Claude 可能开始"忘记"早期的指令或犯更多错误。Context window 是最重要的需要管理的资源。

## 给 Claude 提供验证工作的方式

包含测试、截图或预期输出，以便 Claude 可以自我检查。这是你能做的杠杆率最高的一件事。

当 Claude 可以验证自己的工作时（比如运行测试、比较截图和验证输出），它的表现会显著提升。

没有明确的成功标准，它可能会产生看起来正确但实际上不起作用的东西。你成了唯一的反馈循环，每个错误都需要你的关注。

| 策略 | 之前 | 之后 |
|------|------|------|
| 提供验证标准 | "实现一个验证邮箱地址的函数" | "编写一个 validateEmail 函数。示例测试用例：user@example.com 为 true，invalid 为 false，user@.com 为 false。实现后运行测试" |
| 视觉验证 UI 更改 | "让仪表盘看起来更好" | "[粘贴截图] 实现这个设计。截取结果的截图并与原图比较。列出差异并修复它们" |
| 解决根本原因，而不是症状 | "构建失败了" | "构建失败并出现此错误：[粘贴错误]。修复它并验证构建成功。解决根本原因，不要压制错误" |

UI 更改可以使用 Claude in Chrome 扩展进行验证。它会在你的浏览器中打开新标签页，测试 UI，并迭代直到代码正常工作。

你的验证也可以是测试套件、linter 或检查输出的 Bash 命令。投入精力使你的验证坚如磐石。

## 先探索，再规划，再编码

将研究和规划与实现分开，以避免解决错误的问题。

让 Claude 直接跳到编码可能会产生解决错误问题的代码。使用 plan mode 将探索与执行分开。

推荐的工作流有四个阶段：

1. **探索**：进入 plan mode。Claude 读取文件并回答问题而不进行更改。
   ```
   claude (plan mode)
   读取 /src/auth 并理解我们如何处理会话和登录。
   同时查看我们如何管理用于 secrets 的环境变量。
   ```

2. **规划**：让 Claude 创建详细的实现计划。
   ```
   claude (plan mode)
   我想添加 Google OAuth。哪些文件需要更改？
   会话流程是什么？创建一个计划。
   ```
   按 Ctrl+G 在文本编辑器中打开计划，以便在 Claude 继续之前直接编辑。

3. **实现**：切换出 plan mode，让 Claude 编码，根据其计划进行验证。
   ```
   claude (default mode)
   按照你的计划实现 OAuth 流程。为回调处理程序编写测试，运行测试套件并修复任何失败。
   ```

4. **提交**：让 Claude 提交并附带描述性消息，然后创建 PR。
   ```
   claude (default mode)
   提交并附带描述性消息，然后打开一个 PR
   ```

Plan mode 很有用，但也会增加开销。对于范围明确且修复较小（如修复拼写错误、添加日志行或重命名变量）的任务，直接让 Claude 执行即可。当你不确定方法、更改修改多个文件或你不熟悉被修改的代码时，规划最有用。如果你能用一句话描述 diff，就跳过规划。

## 在 prompt 中提供具体的上下文

你的指令越精确，需要的纠正就越少。

Claude 可以推断意图，但它无法读心。引用特定文件，提及约束条件，并指向示例模式。

| 策略 | 之前 | 之后 |
|------|------|------|
| 限定任务范围 | "为 foo.py 添加测试" | "为 foo.py 编写一个测试，覆盖用户已注销的边缘情况。避免使用 mocks。" |
| 指向来源 | "为什么 ExecutionFactory 的 API 这么奇怪？" | "查看 ExecutionFactory 的 git 历史并总结其 API 是如何形成的" |
| 引用现有模式 | "添加一个日历组件" | "查看首页上现有组件的实现方式以理解模式。HotDogWidget.php 是一个很好的例子。按照该模式实现一个新的日历组件，让用户选择月份并前后翻页选择年份。不使用代码库中已有库以外的其他库从头构建。" |
| 描述症状 | "修复登录 bug" | "用户报告会话超时后登录失败。检查 src/auth/ 中的认证流程，特别是 token refresh。编写一个复现该问题的失败测试，然后修复它" |

模糊的 prompt 在探索时可能很有用，并且可以负担得起纠正方向。像"你会改进这个文件中的什么？"这样的 prompt 可以揭示你没想到要问的事情。

## 提供丰富的内容

使用 `@` 引用文件，粘贴截图/图像，或直接管道传输数据。

你可以通过多种方式向 Claude 提供丰富的数据：

- 使用 `@` 引用文件，而不是描述代码所在位置。Claude 在响应之前会读取文件。
- 直接粘贴图像。复制/粘贴或将图像拖放到 prompt 中。
- 提供文档和 API 参考的 URL。使用 `/permissions` 将常用域列入白名单。
- 通过运行 `cat error.log | claude` 直接管道传输文件内容。
- 让 Claude 获取它需要的内容。告诉 Claude 使用 Bash 命令、MCP 工具或读取文件来自己拉取上下文。

## 配置你的环境

一些设置步骤可以使 Claude Code 在所有会话中显著更有效。有关扩展功能和何时使用每种功能的完整概述，请参阅"扩展 Claude Code"。

## 编写有效的 CLAUDE.md

运行 `/init` 基于你当前的项目结构生成一个初始的 CLAUDE.md 文件，然后随着时间的推移进行优化。

CLAUDE.md 是一个特殊文件，Claude 在每次对话开始时读取。包含 Bash 命令、代码风格和工作流规则。这为 Claude 提供了无法仅从代码推断的持久上下文。

`/init` 命令分析你的代码库以检测构建系统、测试框架和代码模式，为你提供一个可优化的坚实基础。

CLAUDE.md 文件没有要求的格式，但保持简短且人类可读。例如：

```markdown
# Code style
- Use ES modules (import/export) syntax, not CommonJS (require)
- Destructure imports when possible (eg. import { foo } from 'bar')

# Workflow
- Be sure to typecheck when you're done making a series of code changes
- Prefer running single tests, and not the whole test suite, for performance
```

CLAUDE.md 在每次会话时加载，因此只包含广泛适用的内容。对于仅偶尔相关的领域知识或工作流，使用 skills 代替。Claude 按需加载它们，而不会使每个对话变得臃肿。

保持简洁。对于每一行，问自己："删除这个会导致 Claude 犯错吗？"如果不是，就删掉。臃肿的 CLAUDE.md 文件会导致 Claude 忽略你的实际指令！

**✅ 应包含** | **❌ 不应包含**
---|---
Claude 无法猜到的 Bash 命令 | Claude 可以通过阅读代码弄清楚的事情
与默认值不同的代码风格规则 | Claude 已知的标准语言惯例
测试指令和首选测试运行器 | 详细的 API 文档（改为链接到文档）
仓库礼仪（分支命名、PR 惯例） | 频繁更改的信息
特定于你项目的架构决策 | 冗长的解释或教程
开发者环境的怪癖（所需环境变量） | 代码库的逐文件描述
常见的陷阱或不明显的行为 | "编写干净的代码"等不言自明的实践

如果尽管有规则但 Claude 仍然一直做你不想要的事情，文件可能太长了，规则被淹没了。如果 Claude 问你在 CLAUDE.md 中已经回答的问题，措辞可能不明确。像对待代码一样对待 CLAUDE.md：当出现问题时审查它，定期修剪，并通过观察 Claude 的行为是否真的改变来测试更改。

你可以通过添加强调（例如"IMPORTANT"或"YOU MUST"）来优化指令以提高遵守度。将 CLAUDE.md 提交到 git 以便你的团队可以贡献。该文件的价值会随着时间复合增长。

CLAUDE.md 文件可以使用 `@path/to/import` 语法导入其他文件：

```markdown
See @README.md for project overview and @package.json for available npm commands.

# Additional Instructions
- Git workflow: @docs/git-instructions.md
- Personal overrides: @~/.claude/my-project-instructions.md
```

你可以在多个位置放置 CLAUDE.md 文件：

- **主文件夹** (`~/.claude/CLAUDE.md`)：适用于所有 Claude 会话
- **项目根目录** (`./CLAUDE.md`)：提交到 git 以与团队共享
- **项目根目录** (`./CLAUDE.local.md`)：个人项目特定笔记；将此文件添加到 `.gitignore` 以便不与团队共享
- **父目录**：适用于 monorepo，其中 `root/CLAUDE.md` 和 `root/foo/CLAUDE.md` 都会自动拉入
- **子目录**：Claude 在处理这些目录中的文件时按需拉入子 CLAUDE.md 文件

## 配置权限

使用 auto mode 让分类器处理批准，`/permissions` 将特定命令列入白名单，或使用 `/sandbox` 进行操作系统级别的隔离。每种方式都减少中断，同时保持你的控制。

默认情况下，Claude Code 对可能修改系统的操作请求权限：文件写入、Bash 命令、MCP 工具等。这很安全但很繁琐。在第十次批准之后，你不再真正审查了，你只是在点击通过。有三种方法可以减少这些中断：

- **Auto mode**：一个单独的分类器模型审查命令，仅阻止看起来有风险的操作：范围升级、未知基础设施或恶意内容驱动的操作。当你信任任务的大方向但不想逐步点击时最佳。
- **权限白名单**：允许你知道安全的特定工具，如 `npm run lint` 或 `git commit`。
- **沙箱**：启用操作系统级别的隔离，限制文件系统和网络访问，允许 Claude 在定义的边界内更自由地工作。

了解更多关于 permission modes、permission rules 和 [anthropic-claude-code-sandboxing](concepts/sandboxing.md) 的信息。

## 使用 CLI 工具

告诉 Claude Code 在与外部服务交互时使用 `gh`、`aws`、`gcloud` 和 `sentry-cli` 等 CLI 工具。

CLI 工具是与外部服务交互的最高效的上下文方式。如果你使用 GitHub，请安装 gh CLI。Claude 知道如何使用它创建 issues、打开 pull requests 和阅读评论。没有 `gh`，Claude 仍然可以使用 GitHub API，但未经身份验证的请求经常会遇到速率限制。

Claude 在学习它不熟悉的 CLI 工具方面也很有效。尝试类似"使用 `foo-cli-tool --help` 了解 foo 工具，然后用它解决 A、B、C"的 prompt。

## 连接 MCP 服务器

运行 `claude mcp add` 连接 Notion、Figma 或你的数据库等外部工具。

通过 MCP servers，你可以让 Claude 从 issue 跟踪器实现功能、查询数据库、分析监控数据、集成来自 Figma 的设计以及自动化工作流。

## 设置 Hooks

对于每次都必须发生且零例外的情况，使用 hooks。

Hooks 在 Claude 工作流的特定点自动运行脚本。与建议性的 CLAUDE.md 指令不同，hooks 是确定性的，保证操作会发生。

Claude 可以为你编写 hooks。尝试类似"编写一个在每次文件编辑后运行 eslint 的 hook"或"编写一个阻止写入 migrations 文件夹的 hook"的 prompt。直接编辑 `.claude/settings.json` 来手动配置 hooks，运行 `/hooks` 浏览已配置的内容。

## 创建 Skills

在 `.claude/skills/` 中创建 SKILL.md 文件，为 Claude 提供领域知识和可重用的工作流。

Skills 使用特定于你的项目、团队或领域的信息扩展 Claude 的知识。Claude 在相关时自动应用它们，或者你可以直接用 `/skill-name` 调用它们。

通过在 `.claude/skills/` 中添加包含 SKILL.md 的目录来创建 skill：

```markdown
---
name : api-conventions
description : REST API design conventions for our services
---
# API Conventions
- Use kebab-case for URL paths
- Use camelCase for JSON properties
- Always include pagination for list endpoints
- Version APIs in the URL path (/v1/, /v2/)
```

Skills 也可以定义你直接调用的可重复工作流：

```markdown
---
name : fix-issue
description : Fix a GitHub issue
disable-model-invocation : true
---
Analyze and fix the GitHub issue: $ARGUMENTS.

1. Use `gh issue view` to get the issue details
2. Understand the problem described in the issue
3. Search the codebase for relevant files
4. Implement the necessary changes to fix the issue
5. Write and run tests to verify the fix
6. Ensure code passes linting and type checking
7. Create a descriptive commit message
8. Push and create a PR
```

运行 `/fix-issue 1234` 来调用它。对于具有副作用且你想要手动触发的工作流，使用 `disable-model-invocation: true`。

## 创建自定义 Subagents

在 `.claude/agents/` 中定义专门化助手，Claude 可以将任务委托给它们进行隔离处理。

Subagents 在自己的 context 中运行，拥有自己的一组允许的工具。它们对于读取许多文件或需要专门关注而不使主对话变得混乱的任务很有用。

```markdown
---
name : security-reviewer
description : Reviews code for security vulnerabilities
tools : Read, Grep, Glob, Bash
model : opus
---
You are a senior security engineer. Review code for:
- Injection vulnerabilities (SQL, XSS, command injection)
- Authentication and authorization flaws
- Secrets or credentials in code
- Insecure data handling

Provide specific line references and suggested fixes.
```

明确告诉 Claude 使用 subagents："使用 subagent 审查此代码的安全问题。"

## 安装插件

运行 `/plugin` 浏览市场。插件无需配置即可添加 skills、工具和集成。

插件将 skills、hooks、subagents 和 MCP 服务器打包成来自社区和 Anthropic 的单个可安装单元。如果你使用类型化语言，请安装代码智能插件，为 Claude 提供精确的符号导航和编辑后的自动错误检测。

有关在 skills、subagents、hooks 和 MCP 之间选择的指导，请参阅"扩展 Claude Code"。

## 有效沟通

你与 Claude Code 的沟通方式显著影响结果质量。

## 提问代码库问题

向 Claude 提出你会问高级工程师的问题。

当入职新代码库时，使用 Claude Code 进行学习和探索。你可以向 Claude 提出与问另一位工程师相同的问题：

- 日志是如何工作的？
- 我如何创建一个新的 API endpoint？
- `foo.rs` 第 134 行的 `async move { ... }` 做什么？
- `CustomerOnboardingFlowImpl` 处理哪些边缘情况？
- 为什么第 333 行的代码调用 `foo()` 而不是 `bar()`？

以这种方式使用 Claude Code 是一个有效的入职工作流，缩短上手时间并减少对其他工程师的负担。不需要特殊的 prompt 技巧：直接提问即可。

## 让 Claude 面试你

对于较大的功能，先让 Claude 面试你。从最小的 prompt 开始，让 Claude 使用 AskUserQuestion 工具来面试你。

Claude 会询问你可能还没有考虑到的事情，包括技术实现、UI/UX、边缘情况和权衡。

```
我想构建 [简短描述]。使用 AskUserQuestion 工具详细面试我。

询问技术实现、UI/UX、边缘情况、关注点和权衡。不要问明显的问题，深入挖掘我可能没有考虑到的困难部分。

持续面试直到我们涵盖了所有内容，然后将完整的规范写入 SPEC.md。
```

一旦规范完成，开始一个新会话来执行它。新会话具有完全专注于实现的干净 context，你有一个书面规范可以引用。

## 管理你的会话

对话是持久的且可逆的。充分利用这一点！

## 尽早且频繁地纠正方向

一旦你注意到 Claude 偏离轨道，立即纠正它。

最佳结果来自紧密的反馈循环。虽然 Claude 偶尔会第一次就完美解决问题，但快速纠正通常会产生更好的解决方案。

- **Esc**：用 Esc 键中途停止 Claude。Context 被保留，你可以重定向。
- **Esc + Esc** 或 `/rewind`：按 Esc 两次或运行 `/rewind` 打开回退菜单，恢复之前的对话和代码状态，或从选定的消息进行总结。
- **"Undo that"**：让 Claude 撤销其更改。
- `/clear`：在不相关的任务之间重置 context。包含无关 context 的长会话可能会降低性能。

如果你在一个会话中就同一个问题纠正 Claude 超过两次，context 中充满了失败的方法。运行 `/clear` 并使用一个包含你所学到的内容的更具体的 prompt 重新开始。一个具有更好 prompt 的干净会话几乎总是优于一个积累了纠正的长会话。

## 积极管理 context

在不相关的任务之间运行 `/clear` 以重置 context。

Claude Code 在你接近 context 限制时自动压缩对话历史，保留重要的代码和决策，同时释放空间。

在长会话期间，Claude 的 context window 可能会填满无关的对话、文件内容和命令。这可能会降低性能，有时会分散 Claude 的注意力。

- 在任务之间频繁使用 `/clear` 以完全重置 context window
- 当自动压缩触发时，Claude 会总结最重要的内容，包括代码模式、文件状态和关键决策
- 为获得更多控制，运行 `/compact`，例如 `/compact Focus on the API changes`
- 要只压缩部分对话，使用 Esc + Esc 或 `/rewind`，选择一个消息检查点，然后选择"从此处总结"。这将从此点开始压缩消息，同时保持早期 context 完整
- 在 CLAUDE.md 中自定义压缩行为，使用如"压缩时，始终保留修改文件的完整列表和任何测试命令"之类的指令，以确保关键 context 在总结中保留
- 对于不需要保留在 context 中的快速问题，使用 `/btw`。答案出现在一个可关闭的覆盖层中，永远不会进入对话历史，因此你可以在不增加 context 的情况下检查细节

## 使用 Subagents 进行调查

使用"use subagents to investigate X"委派研究。它们在单独的 context 中探索，保持主对话专注于实现。

由于 context 是你的根本约束，subagents 是可用的最强大的工具之一。当 Claude 研究代码库时，它会读取大量文件，所有这些都会消耗你的 context。Subagents 在单独的 context window 中运行并报告摘要：

```
Use subagents to investigate how our authentication system handles token
refresh, and whether we have any existing OAuth utilities I should reuse.
```

Subagent 探索代码库，读取相关文件，并报告调查结果，所有这些都不会使你的主对话变得混乱。

你还可以在 Claude 实现某事后使用 subagents 进行验证：

```
use a subagent to review this code for edge cases
```

## 使用检查点回退

Claude 的每个操作都会创建一个检查点。你可以将对话、代码或两者恢复到任何以前的检查点。

Claude 在更改之前自动创建检查点。双击 Escape 或运行 `/rewind` 打开回退菜单。你可以仅恢复对话、仅恢复代码、同时恢复两者，或从选定的消息进行总结。

与其仔细规划每一步，你可以让 Claude 尝试有风险的操作。如果不行，回退并尝试不同的方法。检查点跨会话持久保存，因此你可以关闭终端，稍后仍然可以回退。

检查点仅跟踪 Claude 做出的更改，而不是外部进程。这不是 git 的替代品。

## 恢复对话

使用 `/rename` 命名会话，并将它们视为分支：每个工作流都有自己的持久 context。

Claude Code 在本地保存对话，因此当一个任务跨越多次时，你不必重新解释 context。运行 `claude --continue` 接续最近的会话，或运行 `claude --resume` 从列表中选择。给会话起描述性的名字（如 `oauth-migration`），以便你以后找到它们。

## 自动化和扩展

一旦你有效地使用一个 Claude，通过并行会话、非交互模式和扇出模式来倍增你的输出。

到目前为止的所有内容都假设一个人、一个 Claude 和一个对话。但 Claude Code 可以水平扩展。本节中的技术展示了你可以如何完成更多工作。

## 运行非交互模式

在 CI、pre-commit hooks 或脚本中使用 `claude -p "prompt"`。添加 `--output-format stream-json` 以获取流式 JSON 输出。

使用 `claude -p "your prompt"`，你可以非交互式地运行 Claude，无需会话。非交互模式是你将 Claude 集成到 CI 管道、pre-commit hooks 或任何自动化工作流中的方式。输出格式允许你以编程方式解析结果：纯文本、JSON 或流式 JSON。

```bash
# One-off queries
claude -p "Explain what this project does"

# Structured output for scripts
claude -p "List all API endpoints" --output-format json

# Streaming for real-time processing
claude -p "Analyze this log file" --output-format stream-json
```

## 运行多个 Claude 会话

并行运行多个 Claude 会话以加速开发、运行隔离实验或启动复杂工作流。

选择适合你想自己做多少协调的并行方法：

- **Worktrees**：在隔离的 git checkout 中运行单独的 CLI 会话，使编辑不会冲突
- **桌面应用**：可视化管理多个本地会话，每个在自己的 worktree 中
- **Claude Code on the Web**：在 Anthropic 管理的云基础设施上的隔离 VM 中运行会话
- **Agent teams**：多个会话的自动化协调，具有共享任务、消息传递和团队负责人

除了并行化工作，多个会话还启用了专注于质量的工作流。干净的 context 改善代码审查，因为 Claude 不会偏向于它刚编写的代码。

例如，使用 Writer/Reviewer 模式：

| 会话 A（Writer） | 会话 B（Reviewer） |
|---|---|
| 为我们的 API endpoint 实现一个速率限制器 | 审查 @src/middleware/rateLimiter.ts 中的速率限制器实现。查找边缘情况、竞态条件和与现有中间件模式的一致性。 |

你可以在测试中做类似的事情：让一个 Claude 编写测试，然后另一个编写代码来通过测试。

## 跨文件扇出

循环遍历任务，为每个任务调用 `claude -p`。使用 `--allowedTools` 限制批量操作的权限。

对于大型迁移或分析，你可以在许多并行的 Claude 调用中分配工作：

1. **生成任务列表**：让 Claude 列出所有需要迁移的文件（例如列出所有 2,000 个需要迁移的 Python 文件）
2. **编写脚本循环遍历列表**：

```bash
for file in $( cat files.txt ); do
claude -p "Migrate $file from React to Vue. Return OK or FAIL." \
--allowedTools "Edit,Bash(git commit *)"
done
```

3. **在几个文件上测试，然后大规模运行**：根据前 2-3 个文件出错的情况优化你的 prompt，然后在完整集合上运行。`--allowedTools` 标志限制 Claude 可以做什么，这在无人值守运行时很重要。

你还可以将 Claude 集成到现有的数据/处理管道中：

```bash
claude -p " " --output-format json | your_command
```

在开发期间使用 `--verbose` 进行调试，在生产中关闭它。

## 使用 Auto mode 自主运行

对于带有后台安全检查的不间断执行，使用 auto mode。分类器模型在命令运行前审查它们，阻止范围升级、未知基础设施和恶意内容驱动的操作，同时让例行工作无需提示即可继续。

```bash
claude --permission-mode auto -p "fix all lint errors"
```

对于使用 `-p` 标志的非交互式运行，如果分类器反复阻止操作，auto mode 会中止，因为没有用户可以回退。有关阈值，请参阅 auto mode 回退的时机。

## 避免常见的失败模式

这些是常见的错误。尽早识别它们可以节省时间：

- **大杂烩会话**：你从一个任务开始，然后问 Claude 一个无关的事情，然后回到第一个任务。Context 充满了无关信息。
  - 修复：在不相关的任务之间使用 `/clear`。

- **反复纠正**：Claude 做错了，你纠正它，它仍然错，你再纠正。Context 被失败的方法污染了。
  - 修复：两次纠正失败后，使用 `/clear` 并编写一个包含你所学内容的更好的初始 prompt。

- **过度指定的 CLAUDE.md**：如果你的 CLAUDE.md 太长，Claude 会忽略其中一半，因为重要规则被淹没在噪音中。
  - 修复：无情修剪。如果 Claude 在没有指令的情况下已经正确地做某事，删除它或将其转换为 hook。

- **信任但未验证的差距**：Claude 产生了一个看似合理的实现，但没有处理边缘情况。
  - 修复：始终提供验证（测试、脚本、截图）。如果你无法验证它，就不要发布它。

- **无限探索**：你让 Claude "调查"某事而没有限定范围。Claude 读取数百个文件，填满 context。
  - 修复：缩小调查范围或使用 subagents，这样探索就不会消耗你的主 context。

## 培养你的直觉

本指南中的模式并非一成不变。它们是在一般情况下效果很好的起点，但可能不是每种情况的最佳选择。

有时你应该让 context 累积，因为你深入处理一个复杂的问题，历史记录很有价值。有时你应该跳过规划让 Claude 自己弄清楚，因为任务是探索性的。有时一个模糊的 prompt 恰好正确，因为你想要在约束它之前看看 Claude 如何解释问题。

关注什么有效。当 Claude 产生出色输出时，注意你做了什么：prompt 结构、你提供的 context、你处于的模式。当 Claude 遇到困难时，问为什么。Context 太嘈杂了吗？Prompt 太模糊了吗？任务对于一次处理来说太大了？

随着时间的推移，你会培养出任何指南都无法捕捉的直觉。你会知道什么时候应该具体，什么时候应该开放；什么时候应该规划，什么时候应该探索；什么时候应该清除 context，什么时候应该让它累积。