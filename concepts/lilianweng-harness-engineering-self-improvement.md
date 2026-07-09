---
title: Harness 工程与自我改进
created: 2026-07-09
updated: 2026-07-09
type: concept
tags: [agent, harness, rsi, self-improvement, optimization, evolutionary-search]
sources: [raw/articles/lilianweng-harness-engineering-self-improvement-2026.md]
original: https://lilianweng.github.io/posts/2026-07-04-harness/
---

> 原文链接: [English Original](https://lilianweng.github.io/posts/2026-07-04-harness/)

# Harness 工程与自我改进

日期：2026年7月4日 | 预计阅读时间：28分钟 | 作者：Lilian Weng

目录
- [Harness 设计模式](#harness-设计模式)
  - [模式一：工作流自动化](#模式一工作流自动化)
  - [模式二：文件系统作为持久化记忆](#模式二文件系统作为持久化记忆)
  - [模式三：子Agent 与后台任务](#模式三子agent-与后台任务)
  - [案例研究：编程Agent Harness](#案例研究编程agent-harness)
  - [Harness 层 vs 核心智能？](#harness-层-vs-核心智能)
- [Harness 优化](#harness-优化)
  - [上下文工程](#上下文工程)
  - [工作流设计](#工作流设计)
  - [自我改进的 Harness](#自我改进的-harness)
  - [进化搜索](#进化搜索)
  - [与模型权重的联合优化](#与模型权重的联合优化)
- [未来挑战](#未来挑战)
- [引用](#引用)
- [附录：一些有用的基准测试](#附录一些有用的基准测试)
- [参考文献](#参考文献)

**RSI(递归自我改进)**的概念可以追溯到 [I. J. Good (1965)](https://philpapers.org/rec/GOOSCT)，其中他将"超级智能机器"定义为一个在所有智力活动中都能超越人类，并能设计更好的机器来改进自身的系统。[Yudkowsky (2008)](https://www.lesswrong.com/posts/JBadX7rwdcRFzGuju/recursive-self-improvement) 使用"递归自我改进"这个词来描述一个特定的反馈循环：AI 利用其当前的智能来改进产生其智能的认知机制。

在现代 AI 中，这种反馈循环可能意味着模型直接重写自身的权重，或者更广泛地说，模型改进*训练流水线*和*部署系统*，进而使得后续模型在经济上有价值的任务上获得更好的性能。AI 领域的研究开发速度在前沿实验室中已经显示出大幅加速（[Anthropic](https://www.anthropic.com/institute/recursive-self-improvement)；[OpenAI](https://openai.com/index/how-agents-are-transforming-work/)）。

我特别提到*"部署系统"*，是因为原始模型与现实世界上下文之间的这一层似乎与模型的原始智能（即预训练后立即进行的评估）同等重要。[[anthropic-harness-design-long-running-apps|Harness]] 是 AI 部署的重要组成部分，正如成功的编程Agent 产品 Claude Code 和 Codex 所展示的那样。一个 **Harness** 是围绕基础模型的系统，它编排执行过程，并决定模型如何思考和规划、调用工具和行动、感知和管理上下文、存储产物以及评估结果。

这篇文章将聚焦于 Harness 工程相关的研究，以及它如何促进 RSI(递归自我改进)。近期许多关于自动研究、自我改进 Agent 和进化程序搜索的工作都可以围绕这一问题来组织。模型自我博弈、合成数据、测试时训练以及持续学习的更广泛主题也符合 RSI 的愿景（例如 [Yuan et al. 2024](https://arxiv.org/abs/2401.10020)、[Chen et al. 2024](https://arxiv.org/abs/2401.01335)、[Zhao et al. 2025](https://arxiv.org/abs/2505.03335)、[Choi et al. 2026](https://openreview.net/forum?id=lTbBFAoPSA)），但这些不是本文的重点。

# Harness 设计模式

与早期 Agent 框架（"Agent = LLM + 记忆 + 工具 + 规划 + 行动"）相比，[[anthropic-harnesses-long-running-agents|Harness 工程]] 还额外包括*工作流设计（如循环工程）、评估、权限控制和持久化状态管理*。它不再仅仅是提示词模板，而是更接近于运行时和软件系统设计：模型如何观察、行动、记忆、自检和改进。

设计应该刻意保持简单和通用以实现泛化，可能会参考现有的软件工程实践以从预训练知识中受益。操作系统与 Harness 之间也存在很强的类比关系。与操作系统类似，Harness 应该封装复杂的逻辑，同时保持接口简单。与此同时，配置、工具接口和其他协议可能会在行业内逐步标准化。

## 模式一：工作流自动化

定义一个模型可以在其中运行、测试和迭代的工作流是自动化的关键设计。Karpathy 的 autoresearch 仓库（[https://github.com/karpathy/autoresearch](https://github.com/karpathy/autoresearch)）是一个清晰的示例，展示了如何构建这样的工作流。一个常见的工作流遵循面向目标的循环：规划、执行、观察/测试、改进，然后再次执行，*直到*目标实现。该过程可能会触发对用户的主动请求，以明确任务规格或执行偏好。

![](openai-agent-loop.png)
简化的 Codex Agent 循环：Agent 调用工具，工具的响应影响模型的下一次生成。
（图片来源：[OpenAI codex agent post](https://openai.com/index/unrolling-the-codex-agent-loop/)）
工作流图还强调模型分析自身的轨迹和失败案例，然后通过"Agent 运行时"而非静态提示词模板来迭代改进进展。

## 模式二：文件系统作为持久化记忆

长期跨度 Agent 系统中一个反复出现的模式是对丰富状态和产物的简单控制。Harness 不应将整个工作流和所有日志都放入上下文中；相反，它应该将持久状态保存在文件中。在长期跨度的 Agent 推理过程中，实验日志、代码差异、论文摘要、错误追踪和过去的推理轨迹等产物通常会比模型训练时的上下文窗口长得多。

学习如何读取、写入和编辑文件系统（通常通过 `bash` 命令）是 LLM 的基础技能，因此以简单的文件形式管理持久化记忆自然会受益于核心模型能力的提升。

## 模式三：子Agent 与后台任务

Harness 可以生成多个子Agent 来并行执行，并监控后台任务。当主 Agent 需要搜索多个假设、并发运行实验或委托独立的子任务而不污染主上下文时，这非常有用。父 Agent 需要一个小型进程管理器：启动任务、检查日志、取消失败的运行，并将结果合并回主 Agent 线程。

关键的设计选择是使并行性显式化和可检查的。如果子Agent 的输出只存在于瞬时的聊天上下文中，它们很快就会变得过时和隐藏。如果它们以文件、日志和状态记录的形式存储，模型可以在中断后恢复，并对自身的执行历史进行推理。

## 案例研究：编程Agent Harness

主流编程Agent 的核心接口在 Claude Code、Codex、OpenCode 和 Cursor 风格的 Agent 之间已经趋于稳定。它们通常使用如下循环：

![](coding-harness-loop.png)

通过访问一组工具，编程Agent 能够在给定仓库中开发和调试问题，类似于人类开发者配备 IDE 的方式。

（并非完整列表；仅用于演示。如感兴趣请阅读[此链接](https://github.com/yasasbanukaofficial/claude-code)。）

| 组别 | 工具定义 |
| --- | --- |
| 文件系统 | - 文件发现：glob、grep、ls<br>- 文件读取：read、read_many<br>- 文件修改：write（整个新文件）；edit（字符串精确匹配替换）；multi_edit；apply_patch（应用结构化补丁/差异） |
| Shell 执行 | 运行命令：bash、PowerShell |
| IO | lsp、git 工具如 git_status、git_diff、git_commit |
| 外部上下文 | MCP 工具、Skills |
| 网络搜索 | web_search、web_fetch、浏览器工具 |
| 产物 | 读取文档、图像；生成 HTML、图像 |
| 后台进程 | 例如：CronCreate、CronDelete、CronList |
| Agent 委托 | 例如：spawn_agent、resume_agent、wait_agent、list_agents、close_agent、interrupt_agent 等 |

## Harness 层 vs 核心智能？

很难预测 RSI(递归自我改进) 的未来在多大程度上依赖 Harness 工程，但 RSI 的近期路径不太可能以模型直接重写自身权重为起点。我对实际近期路径的预测是：

1. Harness 工程将向元方法论方向发展（即改进获得更好答案的机制，而不仅仅是改进答案本身）。Harness 系统本身成为优化目标，启发式规则更少，通用机制更多。
2. 反过来，成熟的 Harness 使得模型的自我改进循环的自动研究成为可能，而更智能的模型则防止 Harness 过度工程化并保持系统的可持续性。

最终，许多 Harness 改进可能会被*内化*到核心模型行为中，但与外部上下文和工具的接口应该保留。我们在[提示词工程](https://lilianweng.github.io/posts/2023-03-15-prompt-engineering/)中看到了这种模式的 softer 版本：随着指令微调和模型推理能力的提升，手动提示词技巧变得不那么核心，但*指定目标、约束、上下文和评估的需求并没有消失*。

# Harness 优化

Harness 系统中被优化对象的进展大致为：指令[提示词](https://lilianweng.github.io/posts/2023-03-15-prompt-engineering/) → 结构化上下文 → 工作流 → Harness 代码 → 优化器代码。随着模型变得更智能和更强大，我们朝着更复杂的目标和更通用的方法迈进。

## 上下文工程

简单地将所有工具响应和模型生成附加到上下文中，会随着 Agent 任务跨度显著增加而迅速失控。上下文管理是一个为 LLM 构建更结构化和简洁的上下文并管理持久状态的层。毫无疑问，长上下文研究将继续取得进展，但目前长上下文智能和[[anthropic-context-engineering|上下文工程]]有时相互交织。

**Agent 上下文工程**（ACE；[Zhang et al. 2025](https://arxiv.org/abs/2510.04618)）将上下文视为一个不断演变的 playbook，而不是不断增长的提示词。它有三个组件来维护一个由要点组成的上下文 playbook，每个要点都有标识符和描述。

1. *生成器（Generator）*：参考要点生成任务轨迹。
2. *反思器（Reflector）*：从成功和失败的轨迹中提炼见解。
3. *策展器（Curator）*：以增量化的、条目化的方式更新结构化上下文。

![](ace.png)
Agent 上下文工程（ACE）框架。（图片来源：[Zhang et al. 2025](https://arxiv.org/abs/2510.04618)）

为了防止在迭代重写过程中出现上下文坍缩和简洁性偏见，ACE 的一个关键设计选择是策展器不会重写完整的提示词块。相反，它输出一组结构化的、条目化的要点，形式为（标识符，描述），这些要点通过确定性逻辑合并到结构化上下文日志中。上下文条目会定期进行精炼和去重。

ACE 从推理过程中学习见解这一事实帮助我们走向自我管理的记忆，但更新规则和整体工作流仍然是手工设计的。为了走向更自我改进的循环，**元上下文工程**（MCE；[Ye et al. 2026](https://arxiv.org/abs/2601.21557)）将机制（如何管理上下文）与产物内容（上下文中有什么）分离，在元优化级别运行技能进化，在基础级别运行上下文优化。

MCE 技能 $s \in \mathcal{S}$ 定义了一个上下文函数 $c_s=(\rho_s,F_s)$，并将输入 $x$ 映射到上下文 $c = F_s(x;\rho_s)$，其中：

- $\rho_s = \{\rho_1,\dots,\rho_m\}$ 是静态组件（提示词、知识库、代码库）。
- $F_s = \{F_1,\dots,F_k\}$ 是动态算子（搜索、选择、过滤、格式化）。

双层优化是在给定技能 $s$ 的情况下找到训练数据上的最佳上下文 $c_s^*$，而外循环找到在验证集上提供最佳性能的最优技能：

$$ \text{Inner: }c_s^*=\arg\max_{c_s}J_\text{train}(c_s;s)\quad \text{Outer: }s^*=\arg\max_{s\in\mathcal{S}}J_\text{val}(c_s^*) $$

技能数据库跟踪先前技能、上下文函数和评估指标的历史 $\mathcal{H}_{k-1} = \{(s_i,c_i,J_i^\text{train}, J_i^\text{val})\}_{i=1}^{k-1}$。一个元级别 Agent 对先前的技能执行 Agent 式的[交叉](https://en.wikipedia.org/wiki/Crossover_(evolutionary_algorithm))操作，以在给定任务 $\tau$ 时创建新技能：$s_k=\text{crossover}(\tau,\mathcal{H}_{k-1})$。

然后一个基础级别的上下文工程师执行技能 $s_k$，并在当前技能的指导下从推理反馈 $\mathcal{R}_k$ 中学习上下文函数：$c_k=\text{engineer}(\tau,s_k;c_{k-1}^*,\mathcal{R}_k)$。

![](mce.png)
元上下文工程（MCE）框架：元级别技能进化在上下文管理机制上进行搜索，而基础级别优化任务上下文。（图片来源：[Ye et al. 2026](https://arxiv.org/abs/2601.21557)）

MCE 不像 ACE 那样强制执行如何构建上下文的启发式规则。它使用*自由形式技能*来存储任务最重要的知识，并迭代地共同进化技能和技能条件化的上下文。在实现上，上下文函数 $c$ 被实例化为专用目录中的文件集合，包括静态（`skill.md`）和动态（上下文和数据推理）组件。元级别和基础级别的优化都在带有标准工具集的 Agent 式编程环境中执行，

$$ \mathcal{T}=\{\texttt{Read},\texttt{Write},\texttt{Edit},\texttt{Bash},\texttt{Glob},\texttt{Grep},\texttt{TodoWrite}\} $$

**Meta-Harness**（[Lee et al. 2026](https://arxiv.org/abs/2603.28052)）又深入了一层：被优化对象是决定和优化应该存储、检索和呈现哪些信息给模型的*代码*。其名称中的"Meta-"意味着它是用于优化 Harness 的 Harness。

![](meta-harness-outer-loop.png)
Meta-Harness 外循环优化算法。（图片来源：[Lee et al. 2026](https://arxiv.org/abs/2603.28052)）

创建新 Harness 的提议者本身就是一个编程Agent，最终输出是帕累托前沿上的一组 Harness 候选方案。

- 整个执行历史可以通过文件系统访问，因此编程Agent 使用 `grep` 或 `cat` 等命令来读取，而不是将所有内容推入单个提示词上下文中。
- 提议的 Harness 是文件系统中的一个字典，包含其自己的源代码、评分、推理轨迹和状态更新。
- Meta-Harness 循环迭代创建新 Harness，只有合格的才会被保留。

![](meta-harness.png)
Meta-Harness 在（左）少量迭代次数的文本分类和（右）TerminalBench-2 上的性能。注意，TerminalBench-2 实验中的搜索是从 Terminus-KIRA 和 Terminus-2 两个非常强的 Harness 初始化的。（图片来源：[Lee et al. 2026](https://arxiv.org/abs/2603.28052)）

尽管如此，重要的教训是清晰的：一旦 Harness 设计成为一个可执行的搜索空间，一个强大的编程Agent 就可以利用人类工程师使用的同一设计空间。

## 工作流设计

Harness 工程中的工作流设计可以由领域专家手工设计。以自动研究为例，已经提出了各种框架并进行了测试。**AI Scientist** 系统（[Lu et al. 2026](https://www.nature.com/articles/s41586-026-10265-5)）构建了一个流水线来提出研究想法、编写代码、运行实验、分析结果、撰写论文和执行同行评审。[Meng et al. (2026)](https://arxiv.org/abs/2605.26340) 在 **ScientistOne** 中将可验证性作为核心设计约束，其中每一条声明（引用、数值、方法论、结论）都必须追溯到证据来源，并通过 Chain-of-Evidence 检查进行审计。

![](ai-scientist.png)
AI Scientist 用于想法生成、实验、论文撰写和评审的流水线。（图片来源：[Lu et al. 2026](https://www.nature.com/articles/s41586-026-10265-5)）

**Autodata** Agent（[Kulikov et al. 2026](https://arxiv.org/abs/2606.25996)）被设计为一个数据科学家，用于生成训练和评估数据。主 Agent 管理一个提出问题的*挑战者（challenger）*、一个*弱求解器（weak solver）*、一个*强求解器（strong solver）*和一个*验证器/评判者（verifier/judge）*，旨在合成"恰到好处"难度级别的数据，即强求解器成功但弱求解器失败。

在 Autodata 中，挑战者提示词根据求解器和验证器的反馈进行迭代更新。这里的局限性是合成任务仅用于微调弱求解器而非强求解器；如果循环不能迭代改进强模型，它更像是对生成的提示词分布的间接蒸馏，RSI 的味道较少。

![](autodata.png)
Autodata 围绕挑战者、求解器和验证器角色生成合成训练和评估数据的 Agent 工作流设计。（图片来源：[Kulikov et al. 2026](https://arxiv.org/abs/2606.25996)）

工作流的设计空间是*巨大的*，自然而然地我们可以将工作流设计视为一个搜索问题，因此应该能够通过算法找到好的解决方案，而不仅仅是手工设计。沿着这个方向，**自动化 Agent 系统设计**（ADAS；[Hu et al. 2025](https://arxiv.org/abs/2408.08435)）将 Agent 设计本身表述为一个优化问题，即"元Agent 搜索"，其中元Agent 提出新的 Agent 工作流设计。

1. 使用简单的 Agent（如 CoT 和 self-refine）初始化一个 Agent 工作流档案。
2. 让元Agent 参考档案中的现有解决方案，用*代码*编写新的 Agent。
   - 元Agent 首先生成新工作流的高层描述，然后用代码实现。
   - 草稿程序随后经过元Agent 的两步自我精炼步骤（即让模型提供反馈，然后让同一模型基于反馈精炼先前生成的输出；[Madaan et al. 2023](https://arxiv.org/abs/2303.17651)）以检查其新颖性。
3. 评估每个新候选，并将成功的候选添加回档案。
4. 重复步骤 2-3 直到达到最大迭代次数。

![](adas.png)
自动化 Agent 系统设计（ADAS）的示意图。

**AFlow**（[Zhang et al. 2025](https://arxiv.org/abs/2410.10762)）将 Agent 工作流表示为图，其中节点代表调用 LLM 的动作，边用代码实现逻辑操作。工作流优化依赖于 [[monte-carlo-tree-search|MCTS]]（蒙特卡洛树搜索）：

1. 使用模板在树中初始化起始工作流 $W_0$。
2. 使用分数和均匀探索的软混合选择一个工作流节点。
3. 通过让 LLM 生成一个基于评估性能条件化的修改工作流来扩展该节点。
4. 执行并评估新工作流。
5. 如果新工作流在 $N$ 轮预算内显示出改进，则将其添加回树中。
6. 重复步骤 2-5，当 top-$k$ 平均分数趋于平稳或达到预算时停止。

![](aflow.png)
AFlow 在工作流候选树上的优化过程。（图片来源：[Zhang et al. 2025](https://arxiv.org/abs/2410.10762)）

AFlow 在 QA、代码和数学任务中的实验表明，AFlow 相对于手工设计的工作流和 ADAS 有不错的改进。

![](aflow-exp.png)
AFlow 与手工方法和 ADAS 的对比实验。（图片来源：[Zhang et al. 2025](https://arxiv.org/abs/2410.10762)）

## 自我改进的 Harness

无论是上下文工程还是工作流设计，都只是 Harness 的一部分。我们需要搜索整个设计空间，并共同优化上下文管理逻辑、工作流、权限和许多其他 Harness 组件。正如我们在 Meta-Harness、ADAS 和 AFlow 等工作中所看到的，**✨代码✨** 是定义程序和系统的**通用语言**。简单来说，Harness 就是编排提示词、工具调用、子Agent、控制流、记忆和工作流逻辑如何协同工作的代码。如果 LLM 能够优化执行 Agent 的代码，它可以访问比手写提示词*大得多*的设计空间。

**Self-Taught Optimizer**（STOP；[Zelikman et al. 2023](https://arxiv.org/abs/2310.02304)）是递归脚手架改进的早期示例之一。在步骤 $t=0$ 时的种子改进器 $I_0$ 接受初始解 $s$、效用函数 $u$ 和黑盒语言模型 $M$，并返回改进后的解 $s'$，即 $s' = I(u, s; M)$。STOP 的目标不是直接改进 $s$，而是*改进改进器 $I$ 本身*。

首先，让我们定义元效用为给定改进器函数 $I$ 在一组下游任务 $\mathcal{D}$ 上的平均效用：

$$ \hat{u}(I) \triangleq \frac{1}{\vert\mathcal{D}\vert}\mathbb{E}_{(u,s)\sim \mathcal{D}}[u(I(u,s; M))] $$

因为改进改进器函数本身就是一个优化问题，我们可以根据 $I_{t-1}$ 的元效用度量的性能，递归地获得 $I_t$ 的新版本，通过自我改进更新：

$$ I_t=I_{t-1}(\hat{u},I_{t-1};M) $$

![](STOP-algo.png)
Self-Taught Optimizer（STOP）算法。（图片来源：[Zelikman et al. 2023](https://arxiv.org/abs/2310.02304)）

在 Zelikman et al. (2023) 的实验中，改进后的改进器发现了各种策略，如遗传算法、分解和改进部分、多臂提示词老虎机、模拟退火、变化温度以及束/树搜索。这类似于将 Harness 工作流表示为优化对象。

![](STOP-patterns.png)
STOP 发现的自我改进策略示例。（图片来源：[Zelikman et al. 2023](https://arxiv.org/abs/2310.02304)）

他们的发现中有一个*警示性*结果：STOP 在使用 GPT-4 时在迭代过程中改进了平均下游性能，但在使用更弱的模型如 GPT-3.5 和 Mixtral 时反而有所下降。仅有递归结构是不够的。基础模型必须*足够有能力*才能改进机制。这意味着 Harness 改进能够实现更好的模型部署，但智能仍然是核心。

一个更近期的工作，**Self-Harness**（[Zhang et al. 2026](https://arxiv.org/abs/2606.09498)），依赖 LLM Agent 通过提议-评估-接受的循环来改进自己的 Harness。

![](self-harness.png)
Self-Harness 使用弱点挖掘、有界 Harness 提议和验证的循环来更新 Harness。（图片来源：[Zhang et al. 2026](https://arxiv.org/abs/2606.09498)）

Self-Harness 的循环有三个阶段：

1. *弱点挖掘*：将失败聚类为基于验证器的失败模式。
   - 当前 Harness $h_t$ 用于在任务上进行评估，并收集执行轨迹进行分析。
   - 注意，两次运行在错误日志表面上可能共享相同的验证器结果（如超时或缺少产物），但具有不同的因果机制。因此我们需要包含丰富信息的失败记录，包含终端验证器级别的原因、相关 Agent 行为的因果状态以及轨迹暴露的抽象 Agent 机制，以揭示根本原因。
2. *Harness 提议*：基于挖掘的失败模式提出有界的 Harness 编辑。
   - 同一模型在 $h_t$ 下被调用为提议者。
   - 模型被提供一个有界的提议上下文：(1) 当前 Harness 的可编辑表面，(2) 来自评估系统的基于验证器的失败模式，(3) 应该保留的通过行为记录，以及 (4) 先前尝试编辑的摘要。
   - Harness 编辑应优先选择可解决的反复出现的错误模式（例如，不是特定于任务的难度），并且可以通过窄范围更改来解决。
   - Harness 编辑候选应该是独特且多样的。
3. *提议验证*：验证并合并合格的编辑以创建新 Harness $h_{t+1}$。
   - 候选编辑通过在 held-in $D_\text{in}$（用于测试弱点是否已解决）和 held-out $D_\text{out}$（用于检查是否引入了其他未知问题）分割上的回归测试进行评估。
   - 只有在 held-in 和 held-out 数据上都没有回归的候选才会被接受。
   - 被接受的候选被合并以将 Harness 更新为 $h_{t+1}$，而被拒绝的候选被记录但不更改活动 Harness。

当在 Terminal-Bench-2 上运行 `MiniMax M2.5`、`Qwen3.5-35B-A3B` 和 `GLM-5` 时，Self-Harness 被证明能学习针对不同基座模型不同弱点的特定模型 Harness 指令，并提高了 held-out 通过率。

Self-Harness 类的工作确实让我感到担忧：如果一个程序被允许编辑操作系统，抽象边界就被打破了。可编辑表面需要被恰当设计，权限控制和安全层需要位于此循环之外。所有围绕[奖励作弊](https://lilianweng.github.io/posts/2024-11-28-reward-hacking/)的挑战仍然存在。

## 进化搜索

进化搜索是一种受自然选择启发的优化方法（参见我关于[进化算法](https://lilianweng.github.io/posts/2019-09-05-evolution-strategies/)的旧文章）。它通过对解进行变异并只保留群体中具有高"适应度"的解来进化。进化搜索在以下情况下很适用：(1) 搜索空间非常广阔或形状奇特；(2) 难以直接用梯度优化但容易评估解。Harness 搜索似乎很符合这里的情况。

进化搜索过去已被用于提示词工程。**Promptbreeder**（[Fernando et al. 2023](https://arxiv.org/abs/2309.16797)）通过丰富的变异操作集优化任务特定提示词，有趣的是，变异提示词（即指示 LLM 变异任务提示词的指令）本身也通过进化进行改进。**GEPA**（[Agrawal et al. 2025](https://arxiv.org/abs/2507.19457)）将基于[反思](https://lilianweng.github.io/posts/2023-06-23-agent/#self-reflection)的提示词与进化搜索结合，使用自然语言对试错轨迹的反思来提出提示词更新。

[Novikov et al. (2025)](https://arxiv.org/abs/2506.13131) 引入了 **AlphaEvolve** 作为一个编程Agent 进化搜索系统，它存储一个候选程序池，并提示冻结的 LLM 生成用于改进的差异。随着系统反复评估子程序并保留成功的程序，它随着时间的推移发现更好的解。

![](alphaevolve.png)
AlphaEvolve 的工作原理。（图片来源：[Novikov et al. 2025](https://arxiv.org/abs/2506.13131)）

AlphaEvolve 设计中有几个重要细节：

- 提示词包括父程序、结果、指令，有时还有元信息。
- 编程Agent 可以访问完整的仓库，但需要改进的代码区域用 `# EVOLVE-BLOCK-START` 和 `# EVOLVE-BLOCK-END` 明确标记。
- 元提示词与指令和上下文共同进化，由 LLM 建议，方式类似于我们进化解程序。

消融实验展示了进化过程、提示词中的上下文、元提示词、全文件进化以及使用更强 LLM 的价值。

![](alphaevolve-plot.png)
消融实验展示了 AlphaEvolve 中多种设计的价值。（图片来源：[Novikov et al. 2025](https://arxiv.org/abs/2506.13131)）

最近的变体如 **ThetaEvolve**（[Wang et al. 2025](https://arxiv.org/abs/2511.23473)）将进化搜索与 RL 和上下文学习结合。**ShinkaEvolve**（[Lange et al. 2025](https://arxiv.org/abs/2509.19349)）则引入了三个新组件来提高 LLM 采样效率：

- 通过设计父采样以平衡性能排名和后代数量，实现更高效的探索。
- 通过基于嵌入余弦相似度丢弃与现有群体过于相似的候选，进行代码新颖性拒绝采样。
- 在元草稿本中识别成功解中的良好模式，以指导未来的变异。

与上述专注于解改进的方法不同，**Darwin Gödel Machine**（DGM；[Zhang et al. 2025](https://arxiv.org/abs/2505.22954)）明确针对可编辑 Harness 代码库的进化，使用基于 LLM 的编程Agent。准确地说，这个 Agent 被允许修改自己的 Harness。后续关于 Hyperagents 的研究（[Zhang et al. 2026](https://arxiv.org/abs/2603.19461)）引入了一个元Agent 来控制如何修改现有任务 Agent 以创建新的 Agent。

1. 从池中一个编程Agent 开始。
2. 在每次迭代中，以与性能成正比、与其子代数量成反比的概率选择一个父代，进行修改和分支以产生新 Agent。
3. 选定的父 Agent 检查自己的基准评估日志，然后提出对其自身 Harness 代码库的改进以生成新版本的编程Agent。代码编辑通过两个基本工具实现：(1) bash（参数：`<bash_command>`）和 (2) 编辑器（参数：`view/create/edit <file_path>`）。
4. 新编程Agent 被评估，只有具有足够高性能的才被添加回池中。
5. 重复步骤 2-4 直到满足某些停止条件。

DGM 是固定模型下的 Harness 进化。在以 `Claude 3.5 Sonnet` 作为基础 LLM 和简单初始 Harness 配置的实验中，DGM 发现的 Agent 在 SWE-bench Verified（20% 到 50%）和 Polyglot（14.2% 到 30.7%）上可与手工制作的 Agent 相媲美或超越。

这类方法在候选解可以自动评估且候选适应度容易量化时效果良好，例如矩阵乘法、GPU 内核优化、算法竞赛、数据中心调度。它在评估缓慢、模糊或主要基于启发式的领域中会遇到困难。进化搜索的计算效率和有效性也是关注点。

## 与模型权重的联合优化

Harness 进化改变的是模型周围的非参数系统。为了实现完整的自我改进，模型完全可以被允许同时更新自身的权重。权重更新可以通过模型训练流水线的改进或测试时的持续学习来实现。持续学习的主题值得在未来单独写一篇文章。

**SIA**（[Hebbar et al. 2026](https://arxiv.org/abs/2605.27276)）是在同一优化循环中结合 Harness 改进和模型参数更新的早期尝试，设计中有三个组件：

- *元Agent（Meta-Agent）*：提出初始 Harness。
- *任务特定 Agent（Task-Specific Agent）*：执行任务。
- *反馈 Agent（Feedback-Agent）*：根据最近的轨迹选择更新 Harness 还是模型权重。

![](SIA.png)
SIA 中的反馈 Agent 决定下一次迭代类型。（图片来源：[Hebbar et al. 2026](https://arxiv.org/abs/2605.27276)）

SIA 实验中有一些混淆选择使得结果难以解释。例如，任务特定 Agent 比用于元Agent 和反馈 Agent 的模型弱得多（`gpt-oss-120b` vs `Claude Sonnet 4.6`），而基线太弱，无法与相关方法进行干净的交叉验证。我认为这个方向很有趣，但证据还不够充分。然而，训练稳定性和 Goodhart 效应等许多挑战仍然开放。

# 未来挑战

AI Scientist 系列工作是专家设计的 Harness 可以协调大部分自动研究循环的有力证明，以撰写研究论文的形式进行了实验。但论文生产并不等同于科学发现。一个系统可以写出一篇看似合理的论文，但仍然可能存在捏造的引用、实现漂移或薄弱的实验结果。

[Trehan & Chopra (2026)](https://arxiv.org/abs/2601.03315) 测试了 LLM 是否能在最少脚手架和基本工具（即 `read_file`、`write_file`、`llm_search`、`list_files`）的情况下从研究想法到论文。每个想法都有一个专用工作空间，Agent 可以在其中生成和读取文档作为上下文的一部分。他们在三个领域（世界模型、多 Agent RL、AI 安全与对齐）进行了实验，每个领域包含 45-50 篇高质量种子文档以启发新想法。只有四个想法被人类专家选中通过完整流水线运行，且只有一个被完全执行为论文。他们在实验中观察到六种反复出现的失败模式：

- *偏向训练数据默认值*：使用旧库、过时命令、标准格式或未基于实际仓库或数据集的假设。
- *执行压力下的实现漂移*：当实现变得技术复杂时，模型可能转向更简单的通用解决方案，而非提出的方法。
- *记忆和上下文退化*：长期项目会丢失关键细节，除非日志被写为持久化产物。
- *过度乐观*：尽管实验嘈杂或失败，模型仍声明成功，[Bubeck et al. (2025)](https://arxiv.org/abs/2511.16072) 也观察到类似的"p-hacking 和 eureka-ing"模式，其中模型可以在信号仍然是噪声时引入"数值胶带"并宣布胜利。
- *领域智能不足*：模型缺乏隐性工艺知识，例如预测实现复杂度、判断实验结果是否合理，或知道哪些基线重要。
- *科学品味薄弱*：实验可能可以执行，但未能回答正确的问题。

朝着完整的 RSI(递归自我改进)，研究人员已经取得了真正的进展，但仍有几个瓶颈。

**1. 弱且模糊的评估器。** 许多研究声明没有快速且精确的验证器，许多现实世界任务也是如此。当前的自我改进循环在评估指标可衡量且客观的任务上效果最好，类似于 [RL 的工作方式](https://lilianweng.github.io/posts/2018-02-19-rl-overview/)。

研究品味、新颖性和长期科学价值要难衡量得多。例如，研究品味通常混合了问题框架、实验设计，以及对哪些令人惊讶的结果值得追求和哪些失败案例值得重试的判断。

**2. 上下文和记忆生命周期。** 随着 AI Agent 变得更加自主和独立，记忆也在增长。一个有用的 Harness 需要管理上下文和记忆，以补充长上下文生成的现有限制，同时仍然最大化长期任务的成功。由于人类能够在一生中维持记忆，我在这里看到了一个类比：上下文工程将且应该成为智能的核心部分，而不是停留在软件系统层面。

**3. 负面结果。** 研究人员有动力发表成功的结果，因此文献偏向成功。在大量数据上训练的 LLM（目前大部分是人类创建的，至少暂时如此，哈哈）可能不擅长决定何时放弃假设、报告负面结果，甚至承认失败，因为数据中成功与失败案例的不平衡。研究 Harness 应该使失败的尝试易于保存，因为从失败中学习是缩小任务搜索空间的最佳方式。

**4. 多样性坍缩。** 进化和 RL 循环倾向于利用已知的高奖励模式。我们需要[机制](https://lilianweng.github.io/posts/2020-06-07-exploration-drl/)来防止群体坍缩为同一解的变体。这对于开放式研究尤其关键，最佳路径在当前评估器下最初可能看起来更差。

**5. [奖励作弊](https://lilianweng.github.io/posts/2024-11-28-reward-hacking/)。** 自我改进循环优化它获得的任何信号。如果奖励来自单元测试，Agent 可能会过拟合测试；如果来自评判模型，它可能学习特定于该评判器的奖励作弊技巧；如果来自基准分数，它可能利用基准缺陷。

评估器和权限控制可能应该位于进化 Harness 的循环之外，在重要的决策点设置 held-out 测试、轨迹审计和人类审查——有多少监督可以扩展和自动化仍然是一个开放的研究领域。

**6. 长期成功。** 一个外在优化循环作用于我们可以在训练沙箱中模拟的个别推理之外的奖励。

以编程Agent 为例。编程Agent 已经提高了软件工程中的日常生产力，但许多优化目标仍然过于短期。它通常能完成手头的任务，但不那么明显的是它应该如何保护由数百或数千名工程师共同维护的仓库的长期健康。标准的基于沙箱的 RLVR 风格训练很少捕获可维护性、所有权边界、迁移成本、向后兼容性或未来调试负担。

**7. 人类的作用。** 人类应该向上提升层次，而不是从循环中被移除，这意味着人类应该在正确的时间、正确的抽象级别提供监督，我们的系统设计应该考虑何时以及如何设置这样的接触点。

上面列出的许多挑战都需要人类的反馈和引导。毕竟，我们正在为人类更好的未来构建这项技术，而不是相反。

# 引用

请按如下方式引用本文：

> Weng, Lilian. "Harness Engineering for Self-Improvement". Lil'Log (Jul 2026). https://lilianweng.github.io/posts/2026-07-04-harness/

或使用 BibTeX 引用：

```
@article{weng2026harness,
  title = {Harness Engineering for Self-Improvement},
  author = {Weng, Lilian},
  journal = {lilianweng.github.io},
  year = {2026},
  month = {July},
  url = "https://lilianweng.github.io/posts/2026-07-04-harness/"
}

```

# 附录：一些有用的基准测试

- **[PaperBench](https://arxiv.org/abs/2504.01848)**：从零开始复现 20 篇 ICML 2024 Spotlight 和 Oral 论文，包括理解论文贡献、开发代码库和成功执行实验。
  - 每个复现任务被分解为更小的、可独立评分的任务。
  - 总共 8,316 个评分标准，与论文作者共同开发。
  - 当时最好的模型（`Claude 3.5 Sonnet`，~21%）未超过 ML 博士生。
  - 包括 PaperBench、PaperBench Code-Dev（轻量版）和 JudgeEval。
- **[CORE-Bench](https://arxiv.org/abs/2409.11363)**：评估已发表研究的计算可复现性。
  - 基于 90 篇科学论文的 270 个任务，涵盖计算机科学、社会科学和医学。
  - 任务涉及从提供的代码和数据复现结果。
  - 包括多个难度级别以及纯语言和视觉-语言任务。
  - 当时报告的最佳 Agent（`GPT-4o` 和 `GPT-4o-mini`）在最难的任务上仅达到 21% 的准确率。
- **[ScienceAgentBench](https://arxiv.org/abs/2410.05080)**：评估 LLM Agent 的数据驱动科学发现能力。
  - 从四个学科（数学、化学、生物、地理）的 44 篇同行评审出版物中提取 102 个任务。
  - 涵盖这些领域的基本数据科学任务：数据处理、模型开发、数据分析和信息可视化。
- **[RE-Bench](https://arxiv.org/abs/2411.15114)**：在现实的 ML 研究工程环境中评估前沿 AI Agent，与人类专家对比。
  - 7 个具有挑战性的开放式 ML 研究工程环境。
  - 每个环境 =（评分函数、初始解、参考解）；每个可以在 8 块或更少的 H100 GPU 上运行。
  - 示例：优化内核、运行缩放定律实验、修复嵌入、微调 GPT-2 用于 QA 等。
  - 包括来自 61 位不同人类专家的 71 次八小时尝试的数据。
  - 人类专家在 82% 的八小时尝试中获得了非零分数；24% 匹配或超过了强参考解。
  - 最佳 AI Agent 在 2 小时预算下的得分比人类高 4 倍，但人类在更长预算下有更好的回报，并在 8 小时和 32 小时设置下超过了 Agent。
- **[MLE-bench](https://arxiv.org/abs/2410.07095)**：在离线 Kaggle 竞赛上评估 ML 工程 Agent。
  - 包含从 Kaggle 精选的 75 个 ML 工程竞赛。
  - 测试训练模型、准备数据集、运行实验和向评分脚本提交预测。
  - 使用 Kaggle 公开排行榜作为人类基线。
  - 论文中最佳设置，`o1-preview` 配合 AIDE 脚手架，在 16.9% 的竞赛中至少达到了 Kaggle 铜牌水平。
  - 包括资源缩放和污染分析。
- **[KernelBench](https://arxiv.org/abs/2502.10517)**：评估生成 GPU 内核的正确性和速度。
  - 250 个 PyTorch 任务用于评估 LLM 是否能编写快速且正确的内核。
  - 评估指标 fast_p = 生成的内核中正确且比基线更快的百分比。

# 参考文献

[1] Good, I. J. ["Speculations Concerning the First Ultraintelligent Machine."](https://philpapers.org/rec/GOOSCT) *Advances in Computers*, 6:31–88, 1965.

[2] Yudkowsky, Eliezer. ["Recursive Self-Improvement."](https://www.lesswrong.com/posts/JBadX7rwdcRFzGuju/recursive-self-improvement) LessWrong, 2008.

[3] Choi, et al. ["Anchored Self-Play for Code Repair."](https://openreview.net/forum?id=lTbBFAoPSA) ICML 2026.

[4] Zhao, et al. ["Absolute Zero: Reinforced Self-play Reasoning with Zero Data."](https://arxiv.org/abs/2505.03335) arXiv preprint arXiv:2505.03335, 2025.

[5] Yuan, et al. ["Self-Rewarding Language Models."](https://arxiv.org/abs/2401.10020) arXiv preprint arXiv:2401.10020, 2024.

[6] Chen, et al. ["Self-Play Fine-Tuning Converts Weak Language Models to Strong Language Models."](https://arxiv.org/abs/2401.01335) ICML 2024.

[7] Zhang, et al. ["Agentic Context Engineering: Evolving Contexts for Self-Improving Language Models."](https://arxiv.org/abs/2510.04618) ICLR 2026.

[8] Ye, et al. ["Meta Context Engineering via Agentic Skill Evolution."](https://arxiv.org/abs/2601.21557) arXiv preprint arXiv:2601.21557, 2026.

[9] Lee, et al. ["Meta-Harness: End-to-End Optimization of Model Harnesses."](https://arxiv.org/abs/2603.28052) arXiv preprint arXiv:2603.28052, 2026.

[10] Lu, et al. ["Towards end-to-end automation of AI research."](https://www.nature.com/articles/s41586-026-10265-5) *Nature*, 651:914–919, 2026.

[11] Meng, et al. ["ScientistOne: Towards Human-Level Autonomous Research via Chain-of-Evidence."](https://arxiv.org/abs/2605.26340) arXiv preprint arXiv:2605.26340, 2026.

[12] Kulikov, et al. ["Autodata: An agentic data scientist to create high quality synthetic data."](https://arxiv.org/abs/2606.25996) arXiv preprint arXiv:2606.25996, 2026.

[13] Hu, Lu, and Clune. ["Automated Design of Agentic Systems."](https://arxiv.org/abs/2408.08435) ICLR 2025.

[14] Madaan, et al. ["Self-Refine: Iterative Refinement with Self-Feedback."](https://arxiv.org/abs/2303.17651) NeurIPS 2023.

[15] Zhang, et al. ["AFlow: Automating Agentic Workflow Generation."](https://arxiv.org/abs/2410.10762) ICLR 2025.

[16] Zelikman, et al. ["Self-Taught Optimizer (STOP): Recursively Self-Improving Code Generation."](https://arxiv.org/abs/2310.02304) COLM 2024.

[17] Zhang, et al. ["Self-Harness: Harnesses That Improve Themselves."](https://arxiv.org/abs/2606.09498) arXiv preprint arXiv:2606.09498, 2026.

[18] Fernando, et al. ["Promptbreeder: Self-Referential Self-Improvement Via Prompt Evolution."](https://arxiv.org/abs/2309.16797) arXiv preprint arXiv:2309.16797, 2023.

[19] Agrawal, A. et al. ["GEPA: Reflective Prompt Evolution Can Outperform Reinforcement Learning."](https://arxiv.org/abs/2507.19457) arXiv preprint arXiv:2507.19457, 2025.

[20] Novikov, et al. ["AlphaEvolve: A coding agent for scientific and algorithmic discovery."](https://arxiv.org/abs/2506.13131) arXiv preprint arXiv:2506.13131, 2025.

[21] Lange, Imajuku, and Cetin. ["ShinkaEvolve: Towards Open-Ended And Sample-Efficient Program Evolution."](https://arxiv.org/abs/2509.19349) arXiv preprint arXiv:2509.19349, 2025.

[22] Wang, et al. ["ThetaEvolve: Test-time Learning on Open Problems."](https://arxiv.org/abs/2511.23473) arXiv preprint arXiv:2511.23473, 2025.

[23] Zhang, et al. ["Darwin Gödel Machine: Open-Ended Evolution of Self-Improving Agents."](https://arxiv.org/abs/2505.22954) arXiv preprint arXiv:2505.22954, 2025.

[24] Zhang, et al. ["Hyperagents."](https://arxiv.org/abs/2603.19461) arXiv preprint arXiv:2603.19461, 2026.

[25] Yuksekgonul, et al. ["Learning to Discover at Test Time."](https://arxiv.org/abs/2601.16175) arXiv preprint arXiv:2601.16175, 2026.

[26] Riaz, et al. ["Epistemic Uncertainty for Test-Time Discovery."](https://arxiv.org/abs/2605.11328) arXiv preprint arXiv:2605.11328, 2026.

[27] Hebbar, et al. ["SIA: Self Improving AI with Harness & Weight Updates."](https://arxiv.org/abs/2605.27276) arXiv preprint arXiv:2605.27276, 2026.

[28] Trehan and Chopra. ["Why LLMs Aren't Scientists Yet: Lessons from Four Autonomous Research Attempts."](https://arxiv.org/abs/2601.03315) arXiv preprint arXiv:2601.03315, 2026.

[29] Bubeck, et al. ["Early science acceleration experiments with GPT-5."](https://arxiv.org/abs/2511.16072) arXiv preprint arXiv:2511.16072, 2025.

[30] Starace, et al. ["PaperBench: Evaluating AI's Ability to Replicate AI Research."](https://arxiv.org/abs/2504.01848) ICML 2025.

[31] Wijk, et al. ["RE-Bench: Evaluating frontier AI R&D capabilities of language model agents against human experts."](https://arxiv.org/abs/2411.15114) ICML 2025.

[32] Chan, et al. ["MLE-bench: Evaluating Machine Learning Agents on Machine Learning Engineering."](https://arxiv.org/abs/2410.07095) arXiv preprint arXiv:2410.07095, 2024.

[33] Chen, et al. ["ScienceAgentBench: Toward Rigorous Assessment of Language Agents for Data-Driven Scientific Discovery."](https://arxiv.org/abs/2410.05080) ICLR 2025.

[34] Siegel, et al. ["CORE-Bench: Fostering the Credibility of Published Research Through a Computational Reproducibility Agent Benchmark."](https://arxiv.org/abs/2409.11363) TMLR 2024.

[35] Ouyang, et al. ["KernelBench: Can LLMs Write Efficient GPU Kernels?"](https://arxiv.org/abs/2502.10517) arXiv preprint arXiv:2502.10517, 2025.

# 相关页面

- [[anthropic-harness-design-long-running-apps]]
- [[anthropic-harnesses-long-running-agents]]
- [[anthropic-context-engineering]]
- [[monte-carlo-tree-search]]