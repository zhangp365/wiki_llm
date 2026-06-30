# Wiki Schema

## Domain
AI/ML 技术研究 + 个人知识管理。覆盖 AI 技术架构、Agent 协议、模型训练推理、行业动态，以及读书笔记、技术随笔等个人学习内容。

## Conventions
- File names: lowercase, hyphens, no spaces (e.g., `a2a-protocol.md`)
- Every wiki page starts with YAML frontmatter (see below)
- Use Obsidian wikilinks to link between pages (minimum 2 outbound links per page)
- When updating a page, always bump the `updated` date
- Every new page must be added to `index.md` under the correct section
- Every action must be appended to `log.md`
- Language: 中文为主，技术术语保留英文

## Frontmatter
```yaml
---
title: Page Title
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: entity | concept | comparison | query | note
tags: [from taxonomy below]
sources: [raw/articles/source-name.md]
---
```

## Tag Taxonomy

### AI/ML 技术
- `model` — 模型相关（GPT, Claude, GLM 等）
- `architecture` — 架构设计（Transformer, MoE 等）
- `agent` — AI Agent 框架与协议
- `training` — 训练方法（RLHF, DPO, SFT 等）
- `inference` — 推理与部署
- `alignment` — 对齐与安全
- `benchmark` — 评测基准
- `multimodal` — 多模态

### 协议与框架
- `protocol` — 通信协议（A2A, MCP, OpenAPI 等）
- `framework` — 开发框架（LangChain, CrewAI 等）
- `tool-use` — 工具使用

### 行业与生态
- `company` — 公司（OpenAI, Anthropic, Google 等）
- `person` — 人物
- `open-source` — 开源项目
- `product` — 产品与服务
- `trend` — 行业趋势

### 个人知识
- `note` — 读书/文章笔记
- `reflection` — 思考与总结
- `how-to` — 操作指南
- `comparison` — 对比分析

### Meta
- `timeline` — 时间线
- `controversy` — 争议话题
- `prediction` — 预测

Rule: every tag on a page must appear in this taxonomy. If a new tag is needed, add it here first, then use it.

## Page Thresholds
- **Create a page** when an entity/concept appears in 2+ sources OR is central to one source
- **Add to existing page** when a source mentions something already covered
- **DON'T create a page** for passing mentions, minor details, or things outside the domain
- **Split a page** when it exceeds ~200 lines — break into sub-topics with cross-links
- **Archive a page** when its content is fully superseded — move to `_archive/`, remove from index

## Entity Pages
One page per notable entity. Include:
- Overview / what it is
- Key facts and dates
- Relationships to other entities (Obsidian wikilinks)
- Source references

## Concept Pages
One page per concept or topic. Include:
- Definition / explanation
- Current state of knowledge
- Open questions or debates
- Related concepts (Obsidian wikilinks)

## Comparison Pages
Side-by-side analyses. Include:
- What is being compared and why
- Dimensions of comparison (table format preferred)
- Verdict or synthesis
- Sources

## Note Pages
Personal reading notes, reflections, learning logs. Include:
- Source reference (book, article, video)
- Key takeaways
- Personal thoughts and connections
- Related wiki pages (Obsidian wikilinks)

## Update Policy
When new information conflicts with existing content:
1. Check the dates — newer sources generally supersede older ones
2. If genuinely contradictory, note both positions with dates and sources
3. Mark the contradiction in frontmatter: `contradictions: [page-name]`
4. Flag for user review in the lint report
