---
title: The Era of Experience（体验时代）
created: 2026-05-11
updated: 2026-05-12
type: concept
tags: [rl, deepmind, alpha, reasoning, alignment]
sources:
  - raw/articles/deepmind-era-of-experience-podcast-2025.md
  - raw/articles/david-silver-era-of-experience-speech-2025-10.md
---

# The Era of Experience（体验时代）

## 概述

David Silver（Google DeepMind 首席研究科学家、UCL 教授）于 2025 年与 Richard Sutton 合著论文 **"Welcome to The Era of Experience"**，提出 AI 发展正从"人类数据时代"（Era of Human Data）迈向"体验时代"（Era of Experience）。核心论点：**AI 真正的突破不在于模仿人类，而在于通过与环境的直接交互自主生成经验数据，通过试错和自我改进超越人类水准。**

> "AI真正的能力，既不是模仿人类，也不是照本宣科，而在于发现人类尚未触及的真知。"

## 来源

- **论文**: "Welcome to The Era of Experience", David Silver & Richard Sutton, 2025
- **播客**: Google DeepMind Podcast — "Is human data enough? with David Silver", 主持人 Hannah Fry
  - YouTube: https://www.youtube.com/watch?v=zzXyPGEtseI
  - Bilibili: https://www.bilibili.com/video/BV19SdiY6EsG/
  - 中文翻译: 腾讯新闻 — https://news.qq.com/rain/a/20250422A08QQE00
- **演讲**: Algorithmic Innovation and Entrepreneurship Global Summit on Open Problems for AI, Friends House, London, 2025-10-28
- **Ineffable Intelligence**: Silver 于 2026 年 1 月离开 DeepMind 创办，种子轮 11 亿美元（估值 51 亿），Sequoia / Lightspeed 领投，NVIDIA / Google / 英国主权AI基金跟投

## 两个时代

### 人类数据时代（Era of Human Data）

- 将人类拥有的全部知识提取出来输入给机器
- 大语言模型的核心范式：吸收人类写下的所有文字，实现"全知"
- 局限：永远无法超越人类已有的知识上限，无法创造新范式
- 类比：**化石能源** — 能让 AI 赢在起跑线，但终究有限，大部分已用尽

### 体验时代（Era of Experience）

- 让机器真正与世界互动，通过自身经历获得经验
- 经验是推动下一代 AI 的"燃料"
- 类比：**可持续能源** — 自我生成、自我消化、自我成长的动态经验
- 智能体自行获取的知识终将远超互联网规模

## 体验时代的四大特征

1. **经验流沉浸** — 智能体沉浸在持续的经验流中，不停互动
2. **扎根环境** — 行动和观察深度扎根于环境，能真正改变世界
3. **世界锚定的奖励** — 奖励来自世界中的实际后果，而非人类标注者的偏好
4. **基于经验的规划** — 推理针对实际发生的互动，非抽象的脱离经验的方式

## 案例研究

### 1. AlphaZero — 零人类数据 → 更快、更强

- 完全不使用任何人类数据（"Zero"的含义），从随机权重开始
- 蒙特卡洛树搜索 + 两个神经网络（Policy / Value Function），三步循环
- 几小时内击败最好的手工编写程序，进而在国际象棋、将棋、围棋中击败世界冠军程序
- **"苦涩启示"（Bitter Lesson）**: 人类辛苦积累的知识嵌入系统反而限制了 AI 上限
- **"第37手"**: AlphaGo 对李世石第二盘，落在五路线，完全超出人类认知，象征无限创新之路

### 2. AlphaProof — 将数学当作博弈

- 用 AlphaZero 框架做数学定理证明
- 使用 Lean 形式化语言，"策略"（tactics）如同游戏中的行动
- 学习曲线与围棋一样平滑——从极少初始知识出发，纯自我博弈发现证明
- 2024 年成为史上第一个在 IMO 获得奖牌的 AI（银牌，距金牌仅差一分）
- 一道不到 2% 参赛者解出的题，AlphaProof 通过基于经验的规划发现证明
- 2025 年已有多个系统获得 IMO 金牌

### 3. DiscoRL — 发现强化学习算法

- 在元层面应用"从经验中学习"：用经验学习 RL 算法本身
- 用神经网络表示学习算法，让它在多种环境中自行摸索最优算法
- 超越人类设计的最佳 RL 算法（如 MuZero），且能迁移到从未见过的环境
- 新的 Scaling Law：接触越多训练环境，在所有环境（含未见过）上的表现都越好

## 关键论点

### RLHF 的致命弱点

- RLHF 让 AI 朝人类更喜欢的方向优化，但**无法突破人类知识的上限**
- 人类评价员无法识别新的、更好的解法 → AI 永远学不到那条路线
- 真正扎根的反馈应基于现实世界的结果，而非人类偏好标注

### 自我生成体验 vs 合成数据

- 合成数据（用大模型生成新数据）终究会遇到上限
- 自我生成体验：系统变强 → 遇到更难的问题 → 刚好匹配能力 → 永远有新的体验可"燃烧"

### 模糊领域的扩展

- 围棋/数学有明确标准，但现实领域往往没有
- 方案：将模糊目标拆解为一组可量化的指标，指标组合随反馈自适应调整
- AI 能自主判断"此刻该优化什么目标"

### 对齐与安全

- **造纸夹悖论**: AI 只追求单一指标会走向极端
- 引入人类的痛苦/快乐信号作为自适应目标调整机制
- 需要让 AI 有持续多年、不断累积自我经验的过程

### 路灯下的寓言

- 整个 AI 领域趋同到 LLM 这盏"路灯"下，另一盏路灯（从经验中学习）几乎无人探索
- 人类数据解决了"AI 的浅层问题"——把已有知识装进模型
- "AI 的深层问题"是：一个智能体如何为自己学习

## 核心金句

| 原文 | 译文/解读 |
|------|-----------|
| "The bitter lesson" | 人类知识嵌入越多，系统最终表现越差 |
| 人类数据 = 化石能源 | 能让 AI 起步，但有限 |
| RL = 可持续能源 | 自我生成、永不枯竭 |
| "第37手" | 无限创新序列中的一个节点 |
| RLHF 连孩子和洗澡水都倒掉了 | 虽然有用，但锁死了 AI 超越人类的可能 |
| 路灯寓言 | 整个领域聚集在一盏灯下，忽视另一盏 |

## 与其他概念的关系

- **强化学习（RL）**: 体验时代的核心技术路径，见 AlphaGo/AlphaZero/AlphaProof
- **LLM 的局限**: 当前大模型本质是模仿人类，难以产生超越人类的创新
- **对齐问题**: 体验时代的 AI 需要新的安全范式（自适应目标调整），而非简单的 RLHF
- **Bitter Lesson**: Richard Sutton 提出的经典论点，Silver 在此基础上发展
- **Reward is Enough**: Silver 2021 年论文，提出单靠强化学习足以实现 AGI

## 相关页面

- [[anthropic-building-effective-agents]] — 构建有效 Agent 的方法论
- [[anthropic-context-engineering]] — 上下文工程，当前 LLM 的优化方向
- [[hermes-agent-memory-architecture]] — Agent 记忆系统的另一种设计思路
