---
title: Anthropic 扩展托管 Agent
created: 2026-04-08
updated: 2026-04-08
type: concept
tags: [agent, anthropic, managed-agents]
sources: [raw/articles/anthropic-scaling-managed-agents.md]
original: https://www.anthropic.com/engineering/managed-agents
---

# Anthropic 扩展托管Agent：将大脑与双手解耦

> 原文链接: [English Original](https://www.anthropic.com/engineering/managed-agents)

*Engineering at Anthropic*

发布于 2026年4月8日

Harness编码的假设会随着模型改进而过时。Managed Agents——我们用于长期Agent工作的托管服务——围绕随着Harness变化而保持稳定的接口构建。

[通过我们的文档开始使用Claude Managed Agents。]

Engineering Blog上的一个持续主题是如何构建有效的Agent和为长期工作设计Harness。仅仅作为一个例子，在早期工作中我们发现Claude Sonnet 4.5会在感觉到其上下文限制临近时过早结束任务——这种行为有时被称为"上下文焦虑"。我们通过在Harness中添加上下文重置来解决这个问题。

Managed Agents通过将长期工作分解为可管理的会话来解决这些问题，而无需特定于Harness的假设。服务保持Agent状态，在会话之间管理上下文窗口，并通过稳定的接口处理工作进程的执行。这使得在模型改进或Harness设计演变时可以迭代Agent实现，而无需更改调用代码。

本文介绍了Managed Agents，如何在其上构建自定义Agent实现，以及我们学到的部署大规模长期Agent工作的经验。

## 什么是Managed Agent？

托管Agent是一种长期运行的Agent工作，其中托管服务（而不是调用代码）管理Agent的会话、上下文和执行过程。

调用代码向托管Agent提交工作，并等待结果。服务负责：
- 管理会话生命周期（启动、暂停、恢复、完成）
- 在会话之间管理上下文窗口
- 处理执行错误和重试
- 将进度流式传输回调用者

托管Agent是围绕工作进程的概念构建的，工作进程是长期任务的可恢复单元。工作进程包含：
- Agent执行的会话历史
- 文件系统状态
- 来自先前会话的待处理操作

这托管服务接口可以在工作进程上执行操作：
- `run(work_process_id)`：恢复工作进程并执行一个会话
- `pause(work_process_id)`：暂停工作进程
- `get_status(work_process_id)`：获取工作进程的当前状态
- `list_work_processes()`：列出所有工作进程

托管服务通过REST API公开这些操作，并通过Webhooks发送进度更新。

## 为什么托管Agent？

在长期Agent工作中出现三个主要问题，托管Agent通过将管理从调用代码中分离出来来解决这些问题：

### 1. 上下文窗口管理

长期工作通常会耗尽上下文窗口。当模型达到其上下文限制时，它需要：
- 识别当前状态
- 保存必要信息以供恢复
- 从干净状态重新开始

托管服务自动处理这种上下文重置。它保留完整会话历史以供调试，但仅将相关状态传递给模型以进行每个会话。

### 2. 执行可靠性

长期工作会遇到执行错误：
- API故障
- 网络超时
- 模型输出错误
- 资源限制

托管服务实现自动重试和恢复。它跟踪每个操作的执行状态，并从失败点恢复。

### 3. 会话管理

长期工作跨越多个会话。托管服务管理：
- 何时开始新会话
- 何时暂停等待输入
- 何时完成工作进程
- 如何在会话之间传递状态

调用代码不需要处理这些细节——它只需提交工作并等待结果。

## 架构

托管Agent由四个主要组件组成：

### 工作进程存储

工作进程存储保留Agent执行的状态。对于每个工作进程，它存储：
- 会话历史（所有Agent调用的完整历史）
- 当前状态（下一个会话的相关状态）
- 文件系统快照（工作目录的状态）
- 待处理操作（尚未执行的队列操作）

工作进程存储是持久化的，并可在会话之间访问。

### 调度器

调度器管理工作进程的生命周期。它接收操作请求（`run`、`pause`、`get_status`）并将它们路由到适当的工作进程。

调度器还管理并发限制——它确保不超过最大并发工作进程数，并按优先级对工作进程进行排序。

### 执行器

执行器运行Agent会话。它接收工作进程和要执行的Harness，然后：
- 从工作进程存储加载状态
- 为新会话准备上下文窗口
- 调用Agent API
- 处理工具调用
- 更新工作进程存储
- 流式传输进度更新

执行器实现重试和恢复逻辑。如果Agent调用失败，执行器会：
- 捕获错误
- 回滚部分状态更新
- 记录失败
- 重试操作（使用指数退避）

### Harness接口

Harness是定义Agent如何在特定任务上工作的代码。它包括：
- 系统提示
- 工具定义
- 工作流程逻辑（例如，如何处理工具结果）
- 评估标准（如何确定工作完成）

托管服务与Harness解耦。它调用Harness的钩子来执行会话：
- `prepare_context(work_process_state)`：为下一个会话准备上下文窗口
- `execute(session_input)`：执行一个会话
- `evaluate(state)`：评估工作是否完成

这种解耦使得Harness可以随时间变化，而无需更改托管服务或调用代码。

## 使用托管服务

以下是如何使用托管服务来运行长期Agent工作：

### 1. 创建工作进程

向`POST /work_processes`提交请求以创建新工作进程：
```json
{
  "harness_id": "my_custom_harness",
  "input": {
    "task": "Fix the bug in the authentication module"
  },
  "options": {
    "max_sessions": 100,
    "timeout": 3600
  }
}
```

响应包含`work_process_id`和初始状态。

### 2. 运行工作进程

向`POST /work_processes/{work_process_id}/run`提交请求以启动工作进程。

托管服务将运行会话直到：
- Harness报告工作完成
- 达到`max_sessions`限制
- 发生不可恢复的错误
- 调用者调用`pause`操作

### 3. 监控进度

通过Webhook接收进度更新。对于每个会话，托管服务发送包含以下内容的消息：
- 会话ID
- 会话状态（运行中、完成、失败）
- 模型输出
- 工具调用
- 错误（如果有）

调用者可以实时跟踪Agent的进度。

### 4. 检索结果

当工作进程完成时，调用`GET /work_processes/{work_process_id}`以检索最终结果。

响应包含：
- 最终状态（成功、失败、超时）
- 输出（Agent生成的任何结果）
- 会话历史（完整的执行历史）

## 构建自定义Harness

托管服务设计为支持自定义Harness。要构建自己的Harness：

### 1. 定义Harness接口

实现三个钩子：
- `prepare_context`：准备下一个会话的上下文窗口
- `execute`：执行一个会话
- `evaluate`：评估工作是否完成

### 2. 上传Harness

将Harness代码上传到托管服务。Harness可以使用任何语言——服务只是执行它。

### 3. 注册Harness

调用`POST /harnesses`以注册Harness。请求包括：
- Harness ID
- 代码位置（例如，Docker镜像、S3路径）
- 钩子规范（如何调用每个钩子）

托管服务验证Harness并使其可用于工作进程。

## 经验教训

在构建和部署托管Agent时，我们学到了几个经验教训：

### 1. 解耦Harness和托管服务

通过将Harness和托管服务解耦，我们可以在不破坏调用者代码的情况下迭代它们。这使我们能够在模型改进时快速更新Harness。

### 2. 自动上下文管理

手动上下文管理容易出错。托管服务自动处理上下文重置，确保Agent不会丢失状态或重复工作。

### 3. 执行可靠性

长期工作需要强大的重试和恢复。托管服务实现自动重试，并在每个操作后检查状态，以便从故障中恢复。

### 4. 可观测性

调试长期工作很难。托管服务保留完整的会话历史，并提供详细的进度更新，以便你可以理解Agent在做什么。

### 5. 可扩展性

托管服务支持数千个并发工作进程。调度器管理资源并确保高优先级工作首先完成。

## 未来方向

我们正在积极开发托管Agent的几个方向：

### 1. 更智能的上下文管理

我们正在探索更好的算法来压缩和重置上下文窗口。目标是在将最相关的信息传递给模型的同时，最小化令牌使用。

### 2. 更强的错误恢复

我们正在增强错误恢复逻辑，以处理更复杂的失败场景。这包括：
- 检测何时重试无用（例如，由于模型中的逻辑错误）
- 自动切换到备用模型
- 实现人工介入机制

### 3. 更多的Harness集成

我们正在构建与更多Harness的集成，包括：
- Claude Code的编码Harness
- 自定义研究Harness
- 数据分析Harness

### 4. 改进的调试工具

我们正在构建工具来可视化和调试长期工作。这包括：
- 会话历史的时间线视图
- 状态变化的可视化
- 工具调用的跟踪

## 开始使用

托管服务现已普遍可用。要开始使用：
1. 阅读[文档](https://platform.claude.com/docs/en/managed-agents/overview)
2. 探索[示例Harness](https://github.com/anthropics/managed-agents-examples)
3. 在你自己的工作中部署托管Agent

我们很高兴看到你构建了什么。如果你有任何问题或反馈，请在[GitHub](https://github.com/anthropics/managed-agents)上告诉我们。

---

*脚注：*

1. *上下文焦虑是指模型在感知到上下文限制临近时过早结束任务的行为。这通常会导致任务未完成，因为模型试图在耗尽令牌之前"收尾"。*

2. *上下文重置是一种技术，用于在会话之间清除上下文窗口，同时保留相关信息。托管服务自动实现上下文重置。*

3. *工作进程是长期任务的可恢复单元。它包含所有必要的状态，以便Agent可以从任何先前的会话恢复工作。*

4. *Harness是定义Agent如何在特定任务上工作的代码。托管服务与Harness解耦，以便它们可以独立进化。*

5. *托管服务管理会话、上下文和执行，以便调用代码只需提交工作并等待结果。这简化了调用者代码，并使Agent工作更加可靠。*

---

**相关文章：**

*   [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
*   [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)
*   [Building a C compiler with a team of parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler)

---

*关于作者：本文由Anthropic的Managed Agents团队撰写。*

*想要更多吗？[订阅开发者通讯](https://www.anthropic.com/newsletter)以获取每月产品更新。*

## 相关链接
- [[anthropic-harnesses-long-running-agents]] — 长时间运行 Agent 的 Harness 设计
- [[anthropic-harness-design-long-running-apps]] — Harness 设计的三个模式
- [[anthropic-building-effective-agents]] — Agent 设计模式
