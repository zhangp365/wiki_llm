---
title: The Era of Experience（体验时代）
created: 2026-05-11
updated: 2026-05-11
type: concept
tags: [rl, deepmind, alpha, reasoning, alignment]
sources:
  - raw/articles/deepmind-era-of-experience-podcast-2025.md
---

# The Era of Experience（体验时代）

## 概述

David Silver（Google DeepMind 首席研究科学家、UCL 教授）于 2025 年发表论文 **"Welcome to The Era of Experience"**，提出 AI 发展正从"人类数据时代"（Era of Human Data）迈向"体验时代"（Era of Experience）。核心论点：**AI 真正的突破不在于模仿人类，而在于通过与环境的直接交互自主生成经验数据，通过试错和自我改进超越人类水准。**

> "AI真正的能力，既不是模仿人类，也不是照本宣科，而在于发现人类尚未触及的真知。"
> — David Silver

## 来源

- **论文**: "Welcome to The Era of Experience", David Silver & Richard Sutton, 2025
- **播客**: Google DeepMind Podcast — "Is human data enough? With David Silver", 主持人 Hannah Fry, ~49 分钟
  - YouTube: https://www.youtube.com/watch?v=zzXyPGEtseI
  - Bilibili: https://www.bilibili.com/video/BV19SdiY6EsG/
  - X/Twitter 公告: https://x.com/GoogleDeepMind/status/1910363683215008227
- **演讲**: Algorithmic Innovation and Entrepreneurship Global Summit on Open Problems for AI, Friends House, London, 2025-10-28
- **中文翻译**: 腾讯新闻 — https://news.qq.com/rain/a/20250422A08QQE00

## 两个时代

### 人类数据时代（Era of Human Data）

- 将人类拥有的全部知识提取出来输入给机器
- 大语言模型的核心范式：吸收人类写下的所有文字，实现"全知"
- 局限：永远无法超越人类已有的知识上限
- 类比：**化石能源** — 能让 AI 赢在起跑线，但终究有限

### 体验时代（Era of Experience）

- 让机器真正与世界互动，通过自身经历获得经验
- 经验是推动下一代 AI 的"燃料"
- 类比：**可持续能源** — 自我生成、自我消化、自我成长的动态经验
- 总有一天要突破人类知识的界限，AI 需要自己探索、发现人类尚不知晓的领域

## 关键论点

### 1. AlphaZero 证明：零人类数据 → 更快、更强

- AlphaZero 完全不使用任何人类数据（"Zero"的含义）
- 方法：自己和自己对弈几百万盘，通过试错学习
- 结果：不仅学得更快，表现还比用人类棋谱启动的版本更强
- **"苦涩启示"（Bitter Lesson）**: 人类辛苦积累的知识嵌入系统反而限制了 AI 上限；放弃人类数据、让系统自学，AI 反而能无限进步

### 2. "第37手" — AI 创新的标志

- AlphaGo 对李世石第二盘第 37 手：落在五路线，完全超出人类认知
- 人类高手万分之一才会做此选择，但它是胜负手
- 意义：象征一条无限延伸的创新之路，不仅仅是一个孤立突破
- 大语言模型目前缺乏类似"第37手"的创新，因为太注重模仿人类

### 3. RLHF 的致命弱点

- RLHF（基于人类反馈的强化学习）让 AI 朝人类更喜欢的方向优化
- 但它**无法突破人类知识的上限**：如果人类评价员无法识别新的、更好的解法，AI 永远学不到那条路线
- Silver 的反转论证：人类反馈反而是**不扎根的** — 人只看答案、不验证结果（如推荐蛋糕食谱但没人真的去烤）
- 真正扎根的反馈应基于现实世界的结果

### 4. 自我生成体验 vs 合成数据

- 合成数据（用大模型生成新数据）终究会遇到上限
- 自我生成体验的独特优势：当系统变强 → 遇到更难的问题 → 刚好匹配能力 → 永远有新的体验可"燃烧"
- 没有极限，可以不停进化

### 5. AlphaProof — 数学领域的 AlphaZero

- 用 AlphaZero 框架做数学定理证明
- 使用 Lean 编程语言（形式化数学语言），证明可自动验证
- 训练方式：给 100 万个人类定理题目（不给答案），自动扩展到 1 亿道
- 成果：在**国际数学奥林匹克（IMO）达到银牌水平**
- 一道只有不到 1% 参赛者解出的题，AlphaProof 做出了完美证明
- Tim Gowers（菲尔兹奖得主）担任裁判，评价为"远超以往 AI 数学系统的飞跃"
- 目标：全面超越人类数学家，攻克黎曼猜想等世纪难题

### 6. 模糊领域的扩展

- 围棋/数学有明确标准（输赢/对错），但现实领域往往没有
- Silver 的方案：将模糊目标拆解为一组可量化的指标
  - 例："变健康" → 静息心率、BMI、焦虑水平等综合指标
  - 指标组合随反馈自适应调整
- AI 能自主判断"此刻该优化什么目标"

### 7. 对齐与安全

- **造纸夹悖论**: AI 只追求单一指标会走向极端
- 解决思路：引入人类的痛苦/快乐信号作为自适应目标调整机制
- 当 AI 的行为导致人类痛苦 → 系统自动调整目标组合
- 现有 AI 缺乏"生命史"——没有长期学习和目标调整机制
- 需要让 AI 有持续多年、不断累积自我经验的过程

## 核心金句

| 原文 | 译文/解读 |
|------|-----------|
| "The bitter lesson" | 人类知识嵌入越多，系统最终表现越差；AI 的表现完全可能超越人类 |
| 人类数据 = 化石能源 | 能让 AI 起步，但有限 |
| RL = 可持续能源 | 自我生成、永不枯竭 |
| "第37手" | 无限创新序列中的一个节点 |
| RLHF 连孩子和洗澡水都倒掉了 | 虽然有用，但锁死了 AI 超越人类的可能 |

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
