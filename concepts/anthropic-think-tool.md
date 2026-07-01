---
title: Anthropic Think 工具
created: 2026-05-05
updated: 2026-05-05
type: concept
tags: [agent, tool-use, anthropic]
sources: [raw/articles/anthropic-5-think-tool.md]
original: https://www.anthropic.com/engineering/claude-think-tool
---

# Anthropic Think 工具

> 原文链接: [English Original](https://www.anthropic.com/engineering/claude-think-tool)

*Engineering at Anthropic*

发布于 2025年3月20日

一种提升 Claude 复杂问题解决性能的新工具。

### Extended Thinking 更新（2025年12月15日）

Extended thinking 能力自首次发布以来已有所改进，因此我们建议在大多数情况下使用该功能而不是专用的 think 工具。Extended thinking 提供了类似的好处——为 Claude 提供推理复杂问题的空间——同时具有更好的集成和性能。有关实现细节，请参阅我们的 extended thinking 文档。

---

随着我们不断提升 Claude 的复杂问题解决能力，我们发现了一种特别有效的方法：一个"think"工具，它在复杂任务期间创建专门的空间进行结构化思考。

这种简单却强大的技术——正如我们将在下面解释的，它不同于 Claude 新的"extended thinking"能力（extended thinking 实现细节请参见此处）——已经在 Claude 的 agentic 工具使用能力方面取得了显著的改进。这包括遵循策略、做出一致决策和处理多步骤问题，所有这些都只需最少的实现开销。

在本文中，我们将探讨如何在不同应用上实现"think"工具，并基于经验证的基准结果分享面向开发者的实用指导。

## 什么是"think"工具？

通过"think"工具，我们赋予 Claude 在得出最终答案的过程中包含一个额外思考步骤的能力——配有自己指定的空间。

虽然它听起来与 extended thinking 类似，但这是一个不同的概念。Extended thinking 关乎的是 Claude 在开始生成响应之前所做的事情。使用 extended thinking，Claude 在采取行动之前深入考虑并迭代其计划。"think"工具则是让 Claude 在开始生成响应后，添加一个停下来思考它是否拥有继续前进所需所有信息的步骤。这在执行长链工具调用或与用户进行长时间多步骤对话时特别有帮助。

这使得"think"工具更适合 Claude 无法仅从用户查询中获取 formulate 响应所需的全部信息的情况，以及需要处理外部信息（例如工具调用结果中的信息）的情况。Claude 使用"think"工具执行的推理不如 extended thinking 获得的推理全面，而是更专注于模型发现的新信息。

我们推荐在简单的工具使用场景（如非顺序工具调用或简单指令遵循）中使用 extended thinking。Extended thinking 也适用于不需要 Claude 调用工具的场景，如编程、数学和物理。"think"工具更适合 Claude 需要调用复杂工具、在长链工具调用中仔细分析工具输出、在具有详细指导原则的策略密集型环境中导航，或做出每个步骤都建立在之前步骤之上且错误代价高昂的顺序决策的场景。

以下是一个使用来自 τ-Bench 的标准工具规格格式的示例实现：

```json
{
  "name": "think",
  "description": "Use the tool to think about something. It will not obtain new information or change the database, but just append the thought to the log. Use it when complex reasoning or some cache memory is needed.",
  "input_schema": {
    "type": "object",
    "properties": {
      "thought": {
        "type": "string",
        "description": "A thought to think about."
      }
    },
    "required": ["thought"]
  }
}
```

## τ-Bench 上的性能

我们使用 τ-bench（tau-bench）评估了"think"工具，这是一个旨在测试模型在真实客户服务场景中使用工具能力的综合基准，其中"think"工具是评估标准环境的一部分。

τ-bench 评估 Claude 的以下能力：
- 与模拟用户进行真实对话
- 一致地遵循复杂的客服 agent 策略指导原则
- 使用各种工具访问和操作环境数据库

τ-bench 中使用的主要评估指标是 pass^k，它衡量给定任务的所有 k 次独立任务试验都成功的概率，在所有任务上取平均值。与 LLM 评估中常见的 pass@k 指标（衡量 k 次试验中是否至少有一次成功）不同，pass^k 评估一致性和可靠性——这对于需要一致遵循策略的客户服务应用来说是关键品质。

## 性能分析

我们的评估比较了几种不同的配置：
- 基线（无"think"工具，无 extended thinking 模式）
- 仅 extended thinking 模式
- 仅"think"工具
- "think"工具 + 优化 prompt（针对航空领域）

结果显示，当 Claude 3.7 在基准的"航空"和"零售"客户服务领域中有效使用"think"工具时，取得了显著的改进：
- **航空领域**："think"工具配合优化 prompt 在 pass^1 指标上达到 0.570，相比之下基线仅为 0.370——相对改进达 54%。
- **零售领域**：仅"think"工具就达到 0.812，相比之下基线为 0.783。

Claude 3.7 Sonnet 在 Tau-Bench 评估"航空"领域四种不同配置下的性能：

| 配置 | k=1 | k=2 | k=3 | k=4 | k=5 |
|------|-----|-----|-----|-----|-----|
| "Think" + Prompt | 0.584 | 0.444 | 0.384 | 0.356 | 0.340 |
| "Think" | 0.404 | 0.254 | 0.186 | 0.140 | 0.100 |
| Extended thinking | 0.412 | 0.290 | 0.232 | 0.192 | 0.160 |
| 基线 | 0.332 | 0.206 | 0.148 | 0.116 | 0.100 |

四种不同配置下的评估结果。分数为比例值。

航空领域的最佳性能是通过将"think"工具与优化 prompt 配对实现的，该 prompt 给出了在分析客户请求时应使用的推理方法类型的示例。以下是优化 prompt 的示例：

## 使用 think 工具

在采取任何行动或在收到工具结果后响应用户之前，使用 think 工具作为草稿板来：

- 列出适用于当前请求的具体规则
- 检查是否已收集所有必要信息
- 验证计划的操作是否符合所有策略
- 迭代检查工具结果的正确性

以下是在 think 工具中进行迭代检查的一些示例：

```
用户想取消航班 ABC123
- 需要验证：用户 ID、预订 ID、原因
- 检查取消规则：
  * 是否在预订后 24 小时内？
  * 如果不是，检查机票等级和保险
- 验证没有已飞行或已过去的航段
- 计划：收集缺失信息、验证规则、获取确认

用户想预订 3 张到 NYC 的机票，每人 2 件托运行李
- 需要用户 ID 来检查：
  * 会员等级对应的行李额度
  * 个人资料中存在哪些支付方式
- 行李计算：
  * 经济舱 × 3 位乘客
  * 普通会员：每人 1 件免费行李 → 3 件额外行李 = $150
  * 银卡会员：每人 2 件免费行李 → 0 件额外行李 = $0
  * 金卡会员：每人 3 件免费行李 → 0 件额外行李 = $0
- 需验证的支付规则：
  * 最多 1 张旅行券、1 张信用卡、3 张礼品卡
  * 所有支付方式必须在个人资料中
  * 旅行券余额将作废
- 计划：
  1. 获取用户 ID
  2. 验证会员等级以确定行李费用
  3. 检查个人资料中的支付方式及其组合是否允许
  4. 计算总价：机票价格 + 任何行李费用
  5. 获取明确的预订确认
```

特别有趣的是不同方法的比较方式。使用"think"工具配合优化 prompt 取得了显著优于 extended thinking 模式的结果（后者与无提示的"think"工具表现相似）。仅使用"think"工具（无提示）改进了基线性能，但仍不及优化方法。

"think"工具与优化 prompting 的组合以显著优势交付了最强性能，这可能归因于基准中航空策略部分的高度复杂性，模型在给出"思考"示例时获益最多。

在零售领域，我们还测试了各种配置以了解每种方法的具体影响。

Claude 3.7 Sonnet 在 Tau-Bench 评估"零售"领域三种不同配置下的性能：

| 配置 | k=1 | k=2 | k=3 | k=4 | k=5 |
|------|-----|-----|-----|-----|-----|
| "Think"（无 prompt） | 0.812 | 0.735 | 0.685 | 0.650 | 0.626 |
| Extended thinking | 0.770 | 0.681 | 0.623 | 0.581 | 0.548 |
| 基线 | 0.783 | 0.695 | 0.643 | 0.607 | 0.583 |

三种不同配置下的评估结果。分数为比例值。

"think"工具即使在没有额外提示的情况下也达到了 0.812 的最高 pass^1 分数。与航空领域相比，零售策略明显更容易导航，Claude 仅通过拥有思考空间就能改进，无需进一步指导。

## τ-Bench 分析的关键洞察

我们的详细分析揭示了几个可以帮助你有效实现"think"工具的模式：

- **在困难领域中 prompting 非常重要**。仅仅让"think"工具可用可能会在一定程度上提高性能，但在困难领域中将它与优化 prompting 配对会产生戏剧性的更好结果。然而，较容易的领域可能仅从拥有"think"访问权限中受益。
- **跨试验的一致性改进**。使用"think"的改进在 pass^k 上一直保持到 k=5，表明该工具帮助 Claude 更有效地处理边缘案例和不寻常场景。

## SWE-Bench 上的性能

在评估 Claude 3.7 Sonnet 时，类似的"think"工具也被添加到我们的 SWE-bench 设置中，为达到 0.623 的最先进分数做出了贡献。适配后的"think"工具定义如下：

```json
{
  "name": "think",
  "description": "Use the tool to think about something. It will not obtain new information or make any changes to the repository, but just log the thought. Use it when complex reasoning or brainstorming is needed. For example, if you explore the repo and discover the source of a bug, call this tool to brainstorm several unique ways of fixing the bug, and assess which change(s) are likely to be simplest and most effective. Alternatively, if you receive some test results, call this tool to brainstorm ways to fix the failing tests.",
  "input_schema": {
    "type": "object",
    "properties": {
      "thought": {
        "type": "string",
        "description": "Your thoughts."
      }
    },
    "required": ["thought"]
  }
}
```

我们的实验（n=30 个使用"think"工具的样本，n=144 个不使用的样本）显示，仅包含此工具就将性能平均提高了 1.6%（Welch's t-test: t(38.89) = 6.71, p < 0.001, Cohen's d = 1.47）。

## 何时使用"think"工具

基于这些评估结果，我们确定了 Claude 最能从"think"工具中受益的具体场景：

- **工具输出分析**。当 Claude 需要在行动之前仔细处理之前工具调用的输出时，可能需要回溯其方法。
- **策略密集型环境**。当 Claude 需要遵循详细指导原则并验证合规性时。
- **顺序决策**。当每个行动都建立在之前行动之上且错误代价高昂时（通常出现在多步骤领域）。

## 实现最佳实践

为了从 Claude 的"think"工具中获得最大收益，我们基于 τ-bench 实验推荐以下实现实践。

## 1. 使用特定领域示例进行策略性 prompting

最有效的方法是提供关于何时以及如何使用"think"工具的清晰指令，例如用于 τ-bench 航空领域的指令。提供针对你的特定用例量身定制的示例，可以显著改善模型使用"think"工具的效果：
- 推理过程中期望的详细程度
- 如何将复杂指令分解为可操作步骤
- 处理常见场景的决策树
- 如何检查是否已收集所有必要信息

## 2. 将复杂指导放在系统提示中

我们发现，当它们很长和/或很复杂时，将关于"think"工具的指令包含在系统提示中比放在工具描述本身中更有效。这种方法提供了更广泛的上下文，并帮助模型更好地将思考过程整合到其整体行为中。

## 何时不应使用"think"工具

虽然"think"工具可以提供显著的改进，但它并不适用于所有工具使用场景，并且确实需要以增加 prompt 长度和输出 token 为代价。具体来说，我们发现"think"工具在以下用例中不提供任何改进：

- **非顺序工具调用**。如果 Claude 只需进行单次工具调用或多次并行调用来完成任务，添加"think"不太可能产生任何改进。
- **简单指令遵循**。当 Claude 需要遵守的约束不多，且其默认行为已经足够好时，额外的"think"不太可能带来收益。

## 入门指南

"think"工具是对你的 Claude 实现的一个简单添加，只需几个步骤就能产生有意义的改进：

- **在 agentic 工具使用场景中测试**。从具有挑战性的用例开始——即 Claude 目前在策略合规性或长链工具调用中的复杂推理方面遇到困难的用例。
- **添加工具定义**。实现一个针对你的领域定制的"think"工具。它需要最少的代码，但能使推理更加结构化。还要考虑将关于何时以及如何使用工具的指令，以及与你的领域相关的示例添加到系统提示中。
- **监控和改进**。观察 Claude 在实践中如何使用该工具，并调整你的 prompt 以鼓励更有效的思考模式。

最好的部分是添加此工具在性能结果方面的负面影响极小。除非 Claude 决定使用它，否则它不会改变外部行为，也不会干扰你现有的工具或工作流。

## 结论

我们的研究表明，"think"工具可以显著增强 Claude 3.7 Sonnet 在需要策略合规性和长链工具调用中推理的复杂任务上的性能。"Think"不是一刀切的解决方案，但它在正确的用例中提供了实质性的好处，所有这些只需最少的实现复杂度。

我们期待看到你将如何使用"think"工具与 Claude 构建更有能力、更可靠、更透明的 AI 系统。

> 虽然我们的 τ-Bench 结果集中在 Claude 3.7 Sonnet 与"think"工具的改进上，但我们的实验表明 Claude 3.5 Sonnet (New) 也能在与 3.7 Sonnet 相同的配置下实现性能提升，表明这种改进可以推广到其他 Claude 模型。

## 相关链接
- [[anthropic-advanced-tool-use]] — 高级工具使用（Tool Search、Programmatic Tool Calling）
- [[anthropic-building-effective-agents]] — Agent 设计模式
- [[anthropic-context-engineering]] — 上下文工程（思考工具属于 context 策略的一部分）
