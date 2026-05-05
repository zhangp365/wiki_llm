---
title: "超越权限提示：让 Claude Code 更安全、更自主"
created: 2026-05-05
updated: 2026-05-05
type: concept
tags: [agent, tool-use, anthropic, security, sandboxing]
sources: [raw/articles/anthropic-13-claude-code-sandboxing.md]
original: https://www.anthropic.com/engineering/claude-code-sandboxing
---

# 超越权限提示：让 Claude Code 更安全、更自主

> 原文链接: [English Original](https://www.anthropic.com/engineering/claude-code-sandboxing)

## 让 Claude Code 中的用户更安全

在 Claude Code 中，Claude 与你一起编写、测试和调试代码，浏览你的代码库、编辑多个文件并运行命令来验证其工作。给予 Claude 对代码库和文件如此多的访问权限可能会引入风险，特别是在 prompt injection（提示注入）的情况下。

为了帮助解决这个问题，我们在 Claude Code 中引入了两个基于沙箱技术构建的新功能，这两个功能旨在为开发者提供一个更安全的工作环境，同时允许 Claude 更自主地运行，并减少权限提示。在我们的内部使用中，我们发现沙箱技术安全地将权限提示减少了 84%。通过定义 Claude 可以在其中自由工作的固定边界，它们提高了安全性和自主性。

Claude Code 基于权限模型运行：默认情况下，它是只读的，这意味着它在进行修改或运行任何命令之前都会请求权限。有一些例外情况：我们自动允许 `echo` 或 `cat` 等安全命令，但大多数操作仍然需要明确的批准。

不断点击"批准"会减慢开发周期，并可能导致"审批疲劳"——用户可能不再仔细关注他们批准的内容，从而反而降低了开发的安全性。

为了解决这个问题，我们为 Claude Code 推出了沙箱功能。

## 沙箱：一种更安全、更自主的方法

沙箱创建了预定义的边界，Claude 可以在其中更自由地工作，而不需要为每个操作请求权限。启用沙箱后，权限提示大幅减少，安全性也得到提高。

我们的沙箱方法建立在操作系统级别的特性之上，实现了两个边界：

- **文件系统隔离**：确保 Claude 只能访问或修改特定目录。这在防止被 prompt injection 的 Claude 修改敏感系统文件方面尤为重要。
- **网络隔离**：确保 Claude 只能连接到已批准的服务器。这防止了被 prompt injection 的 Claude 泄露敏感信息或下载恶意软件。

值得注意的是，有效的沙箱需要同时具备文件系统和网络隔离。没有网络隔离，被攻破的 agent 可以窃取敏感文件（如 SSH 密钥）；没有文件系统隔离，被攻破的 agent 可以轻松逃出沙箱并获得网络访问权限。只有同时使用这两种技术，我们才能为 Claude Code 用户提供更安全、更快速的 agentic 体验。

## Claude Code 中的两个新沙箱功能

### 沙箱化 Bash 工具：无需权限提示的安全 Bash 执行

我们正在引入一个新的沙箱运行时（目前作为研究预览版提供 beta），它允许你精确地定义你的 agent 可以访问哪些目录和网络主机，而无需启动和管理容器的开销。它可以用于对任意进程、agent 和 MCP 服务器进行沙箱化。该功能也作为开源研究预览版提供。

在 Claude Code 中，我们使用这个运行时对 bash 工具进行沙箱化，允许 Claude 在你设定的限制范围内运行命令。在安全的沙箱中，Claude 可以更自主地运行，安全地执行命令而无需权限提示。如果 Claude 尝试访问沙箱外的内容，你会立即收到通知，并可以选择是否允许。

我们基于操作系统级别的原语构建了这一功能，例如 Linux 的 bubblewrap 和 MacOS 的 seatbelt，在操作系统级别执行这些限制。它们不仅覆盖 Claude Code 的直接交互，还包括命令产生的任何脚本、程序或子进程。

如上所述，该沙箱强制执行以下两项：

- **文件系统隔离**：允许对当前工作目录进行读写访问，但阻止修改其外的任何文件。
- **网络隔离**：仅允许通过连接到沙箱外部运行的代理服务器的 unix domain socket 进行互联网访问。该代理服务器对进程可以连接的域执行限制，并处理新请求域的用户确认。如果你希望进一步提高安全性，我们还支持自定义此代理以对出站流量执行任意规则。

两个组件都是可配置的：你可以轻松地选择允许或禁止特定的文件路径或域。Claude Code 的沙箱架构通过文件系统和网络控制来隔离代码执行，自动允许安全操作，阻止恶意操作，仅在需要时请求权限。

沙箱确保即使成功的 prompt injection 也被完全隔离，不会影响用户的整体安全。这样，被攻破的 Claude Code 无法窃取你的 SSH 密钥，也无法连接到攻击者的服务器。

要开始使用此功能，请在 Claude Code 中运行 `/sandbox`，并查看有关我们安全模型的更多技术细节。

为了让其他团队更轻松地构建更安全的 agent，我们已将此功能开源。我们认为其他人应该考虑为他们的 agent 采用这项技术，以增强其 agent 的安全态势。

### Claude Code on the Web：在云端安全运行 Claude Code

今天，我们还发布了 Claude Code on the Web，使用户能够在云端的隔离沙箱中运行 Claude Code。Claude Code on the Web 在隔离的沙箱中执行每个 Claude Code 会话，使其能够以安全可靠的方式完全访问其服务器。我们设计了此沙箱，确保敏感凭据（如 git 凭据或签名密钥）永远不会与 Claude Code 一起存在于沙箱内。这样，即使沙箱中运行的代码被攻破，用户也能免受进一步的损害。

Claude Code on the Web 使用自定义代理服务来透明地处理所有 git 交互。在沙箱内，git 客户端使用自定义的范围凭据向该服务进行身份验证。代理验证此凭据和 git 交互的内容（例如确保它只推送到配置的分支），然后在将请求发送到 GitHub 之前附加正确的身份验证令牌。Claude Code 的 Git 集成通过一个安全代理路由命令，该代理验证身份验证令牌、分支名称和仓库目标——允许安全的版本控制工作流，同时防止未经授权的推送。

## 开始使用

我们新的沙箱化 bash 工具和 Claude Code on the Web 为使用 Claude 进行工程工作的开发者在安全性和生产力方面提供了显著的改进。

要开始使用这些工具：
- 在 Claude 中运行 `/sandbox`，并查看有关如何配置此沙箱的文档。
- 前往 claude.com/code 试用 Claude Code on the Web。

或者，如果你正在构建自己的 agent，请查看我们开源的沙箱代码，并考虑将其集成到你的工作中。我们期待看到你的作品。

要了解更多关于 Claude Code on the Web 的信息，请查看我们的发布公告。

## 致谢

本文由 David Dworken 和 Oliver Weller-Davies 撰写，Meaghan Choi、Catherine Wu、Molly Vorwerck、Alex Isken、Kier Bradwell 和 Kevin Garcia 参与贡献。
