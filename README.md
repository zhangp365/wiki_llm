# LLM Wiki — Knowledge Base

> 基于 [Karpathy's LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 模式构建的个人知识库。

## 📌 说明

这是一个持久化的、互相链接的 Markdown 知识库。不同于 RAG 每次重新发现知识，Wiki 一次性编译知识并保持更新。

**分工：** 人类负责筛选来源和引导分析，Agent 负责总结、交叉引用、归档和维护一致性。

## 📁 结构

```
wiki/
├── docs/               # Obsidian vault + MkDocs 内容源
│   ├── SCHEMA.md       # 规范、结构规则、标签体系
│   ├── index.md        # 内容目录（按类型分段）
│   ├── concepts/       # Layer 2: 概念/主题页面
│   ├── comparisons/    # Layer 2: 对比分析
│   └── assets/         # 站点图片、图表
├── log.md              # 操作日志（只追加）
├── raw/                # Layer 1: 原始来源（不可修改）
│   ├── articles/       # 文章、网页
│   ├── papers/         # 论文、PDF
│   ├── transcripts/    # 会议记录、访谈
│   └── assets/         # 图片、图表
└── mkdocs.yml          # GitHub Pages 站点配置
```

## 🚀 使用方式

- **Obsidian**: 直接打开 `docs/` 为 vault，支持 `[[wikilinks]]` 和 Graph View
- **VS Code**: 配合 Markdown 预览使用
- **GitHub Pages**: 通过 MkDocs Material 部署为静态站点，`[[wikilinks]]` 由插件转换为网页链接
- **Hermes Agent**: 通过 llm-wiki skill 自动维护

## 🔗 在线访问

- **GitHub 仓库**: https://github.com/zhangp365/wiki_llm
- **MkDocs 站点**: https://zhangp365.github.io/wiki_llm/

## 📋 标签体系

详见 [SCHEMA.md](docs/SCHEMA.md)，涵盖：
- **AI/ML 技术**: model, architecture, agent, training, inference, alignment, benchmark, multimodal
- **协议与框架**: protocol, framework, tool-use
- **行业与生态**: company, person, open-source, product, trend
- **个人知识**: note, reflection, how-to, comparison
- **Meta**: timeline, controversy, prediction
