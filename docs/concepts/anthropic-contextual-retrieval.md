---
title: "Contextual Retrieval：上下文感知检索"
created: 2026-05-05
updated: 2026-05-05
type: concept
tags: [agent, tool-use, anthropic, RAG, retrieval]
sources: [raw/articles/anthropic-8-contextual-retrieval.md]
original: https://www.anthropic.com/engineering/contextual-retrieval
---

# Contextual Retrieval：上下文感知检索

> 原文链接: [English Original](https://www.anthropic.com/engineering/contextual-retrieval)

AI 模型要在特定场景中发挥作用，通常需要获取背景知识。例如，客户支持聊天机器人需要了解其服务的特定业务知识，法律分析机器人需要了解大量过去的案例。

开发者通常使用 **Retrieval-Augmented Generation (RAG)** 来增强 AI 模型的知识。RAG 是一种从知识库中检索相关信息并将其附加到用户 prompt 的方法，能显著增强模型的响应。问题在于，传统的 RAG 解决方案在编码信息时会丢失 context，这经常导致系统无法从知识库中检索到相关信息。

在本文中，我们概述了一种大幅改进 RAG 中检索步骤的方法。该方法称为 **"Contextual Retrieval"**（上下文感知检索），使用两个子技术：Contextual Embeddings 和 Contextual BM25。该方法可以将检索失败率降低 49%，当结合 reranking 时可降低 67%。这些代表了检索准确性的显著改进，直接转化为下游任务更好的性能。你可以使用我们的 cookbook 轻松部署自己的 Contextual Retrieval 解决方案。

## 关于直接使用更长 Prompt 的说明

有时最简单的解决方案就是最好的。如果你的知识库小于 200,000 个 token（约 500 页材料），你可以直接将整个知识库包含在给模型的 prompt 中，无需 RAG 或类似方法。

几周前，我们为 Claude 发布了 prompt caching 功能，这使得这种方法显著更快且更具成本效益。开发者现在可以在 API 调用之间缓存频繁使用的 prompt，将延迟降低 2 倍以上，成本降低高达 90%（你可以阅读我们的 prompt caching cookbook 了解其工作原理）。

然而，随着知识库的增长，你需要一个更具可扩展性的解决方案。这就是 Contextual Retrieval 发挥作用的地方。

## RAG 入门：扩展到更大的知识库

对于不适合 context window 的较大知识库，RAG 是典型的解决方案。RAG 通过以下步骤预处理知识库：

- 将知识库（文档的"语料库"）分解为较小的文本块，通常不超过几百个 token
- 使用嵌入模型将这些块转换为编码语义的向量嵌入
- 将这些嵌入存储在允许按语义相似性搜索的向量数据库中

在运行时，当用户向模型输入查询时，向量数据库被用来根据与查询的语义相似性找到最相关的文本块。然后，最相关的文本块被添加到发送给生成模型的 prompt 中。

虽然嵌入模型擅长捕捉语义关系，但它们可能会遗漏关键的精确匹配。幸运的是，有一种较旧的技术可以帮助处理这些情况。**BM25（Best Matching 25）** 是一种使用词法匹配来查找精确词语或短语匹配的排名函数。它对于包含唯一标识符或技术术语的查询特别有效。

BM25 基于 TF-IDF（词频-逆文档频率）概念构建。TF-IDF 衡量一个词在文档集合中对某个文档的重要性。BM25 通过考虑文档长度并对词频应用饱和函数来改进这一点，有助于防止常见词主导结果。

以下是 BM25 在语义嵌入失败之处成功的示例：假设用户在技术支持数据库中查询"Error code TS-999"。嵌入模型可能会找到关于错误代码的一般内容，但可能错过精确的"TS-999"匹配。BM25 则会查找这个特定的文本字符串来识别相关文档。

RAG 解决方案可以通过结合嵌入和 BM25 技术更准确地检索最适用的文本块，步骤如下：

- 将知识库（文档的"语料库"）分解为较小的文本块，通常不超过几百个 token
- 为这些块创建 TF-IDF 编码和语义嵌入
- 使用 BM25 基于精确匹配找到 top 文本块
- 使用嵌入基于语义相似性找到 top 文本块
- 使用 rank fusion 技术合并和去重（3）和（4）的结果
- 将 top-K 文本块添加到 prompt 以生成响应

通过利用 BM25 和嵌入模型两者，传统 RAG 系统可以提供更全面和准确的结果，在精确词语匹配与更广泛的语义理解之间取得平衡。

这种方法使你可以经济高效地扩展到巨大的知识库，远超单个 prompt 所能容纳的范围。但这些传统 RAG 系统有一个显著限制：它们经常破坏 context。

## 传统 RAG 中的 Context 困境

在传统 RAG 中，文档通常被分割成较小的文本块以便高效检索。虽然这种方法对许多应用程序效果良好，但当单个文本块缺乏足够的 context 时可能会导致问题。

例如，假设你在知识库中嵌入了一组财务信息（比如美国 SEC 文件），并收到以下问题："ACME Corp 在 2023 年第二季度的收入增长是多少？"

一个相关的文本块可能包含文本："该公司的收入比上一季度增长了 3%。"然而，这个文本块本身并未指定它指的是哪家公司或相关时间段，使得检索正确信息或有效使用信息变得困难。

## 介绍 Contextual Retrieval

Contextual Retrieval 通过在每个文本块嵌入前（"Contextual Embeddings"）和创建 BM25 索引前（"Contextual BM25"）在文本块前添加特定于该块的解释性 context 来解决这一问题。

让我们回到 SEC 文件集合的示例。以下是一个文本块可能如何被转换的示例：

```
original_chunk = "The company's revenue grew by 3% over the previous quarter."

contextualized_chunk = "This chunk is from an SEC filing on ACME corp's performance in Q2 2023; the previous quarter's revenue was $314 million. The company's revenue grew by 3% over the previous quarter."
```

值得注意的是，过去也提出过其他使用 context 改进检索的方法。其他提案包括：将通用文档摘要添加到文本块中（我们实验后发现收益非常有限）、hypothetical document embedding 以及基于摘要的索引（我们评估后发现性能较低）。这些方法与本文提出的方法不同。

## 实现 Contextual Retrieval

当然，手动注释知识库中数千甚至数百万个文本块工作量太大了。为了实现 Contextual Retrieval，我们求助于 Claude。我们编写了一个 prompt 指示模型使用整体文档的 context 提供简洁的、特定于文本块的 context 来解释该文本块。我们使用以下 Claude 3 Haiku prompt 为每个文本块生成 context：

```
<document>
{{WHOLE_DOCUMENT}}
</document>

Here is the chunk we want to situate within the whole document

<chunk>
{{CHUNK_CONTENT}}
</chunk>

Please give a short succinct context to situate this chunk within the overall document for the purposes of improving search retrieval of the chunk. Answer only with the succinct context and nothing else.
```

生成的 context 文本通常为 50-100 个 token，在嵌入和创建 BM25 索引之前被添加到文本块前面。

如果你有兴趣使用 Contextual Retrieval，可以从我们的 cookbook 开始。

## 使用 Prompt Caching 降低 Contextual Retrieval 的成本

得益于我们上面提到的特殊 prompt caching 功能，在 Claude 上以低成本实现 Contextual Retrieval 是独特可行的。使用 prompt caching，你不需要为每个文本块都传入参考文档。你只需将文档加载到缓存中一次，然后引用之前缓存的内容。假设 800 token 的文本块、8k token 的文档、50 token 的 context 指令和每个文本块 100 token 的 context，生成 contextualized 文本块的一次性成本为每百万文档 token $1.02。

## 方法论

我们在各种知识领域（代码库、小说、ArXiv 论文、科学论文）、嵌入模型、检索策略和评估指标上进行了实验。我们在附录 II 中包含了每个领域使用的问答示例。

下面的图表显示了在所有知识领域中，使用表现最佳的嵌入配置（Gemini Text 004）和检索 top-20 文本块时的平均性能。我们使用 1 减去 recall@20 作为评估指标，衡量在 top-20 文本块中未能检索到的相关文档的百分比。你可以在附录中查看完整结果——contextualization 在我们评估的每种嵌入-来源组合中都改善了性能。

## 性能改进

我们的实验表明：

- Contextual Embeddings 将 top-20 文本块检索失败率降低了 35%（5.7% → 3.7%）
- 结合 Contextual Embeddings 和 Contextual BM25 将 top-20 文本块检索失败率降低了 49%（5.7% → 2.9%）

## 实现注意事项

在实现 Contextual Retrieval 时，有几个注意事项需要牢记：

- **文本块边界**：考虑如何将文档分割成文本块。文本块大小、边界和重叠的选择会影响检索性能。
- **嵌入模型**：虽然 Contextual Retrieval 改善了我们测试的所有嵌入模型的性能，但有些模型可能比其他模型受益更多。我们发现 Gemini 和 Voyage 嵌入特别有效。
- **自定义 context 生成 prompt**：虽然我们提供的通用 prompt 效果良好，但你可能通过针对特定领域或用例定制的 prompt 获得更好的结果（例如，包含可能只在知识库其他文档中定义的关键术语词汇表）。
- **文本块数量**：将更多文本块添加到 context window 增加了包含相关信息的几率。然而，更多信息可能会分散模型的注意力，因此这是有限制的。我们尝试传递 5、10 和 20 个文本块，发现使用 20 个是这些选项中性能最好的（见附录的比较），但值得在你的用例上进行实验。

始终运行评估：响应生成可能通过传递 contextualized 文本块并区分什么是 context 什么是文本块来改善。

## 通过 Reranking 进一步提升性能

在最后一步中，我们可以将 Contextual Retrieval 与另一种技术结合以获得更多性能改进。在传统 RAG 中，AI 系统搜索其知识库以找到潜在相关的信息文本块。对于大型知识库，这种初始检索通常返回大量文本块——有时数百个——具有不同的相关性和重要性。

**Reranking** 是一种常用的过滤技术，确保只有最相关的文本块被传递给模型。Reranking 提供更好的响应并降低成本和延迟，因为模型处理的信息更少。关键步骤是：

- 执行初始检索以获取 top 潜在相关文本块（我们使用了 top 150）
- 将 top-N 文本块与用户查询一起传递给 reranking 模型
- 使用 reranking 模型，根据每个文本块与 prompt 的相关性和重要性给出分数，然后选择 top-K 文本块（我们使用了 top 20）
- 将 top-K 文本块作为 context 传递给模型以生成最终结果

## 性能改进

市场上有几种 reranking 模型。我们使用 Cohere reranker 进行了测试。Voyage 也提供 reranker，但我们没有时间测试。我们的实验表明，在各种领域中，添加 reranking 步骤进一步优化了检索。

具体来说，我们发现 Reranked Contextual Embedding 和 Contextual BM25 将 top-20 文本块检索失败率降低了 67%（5.7% → 1.9%）。

## 成本和延迟考虑

Reranking 的一个重要考虑是对延迟和成本的影响，特别是在 reranking 大量文本块时。由于 reranking 在运行时增加了额外步骤，即使 reranker 并行处理所有文本块，也不可避免地增加了少量延迟。在 reranking 更多文本块以获得更好性能与 reranking 更少文本块以降低延迟和成本之间存在固有的权衡。我们建议在你的特定用例上尝试不同设置以找到合适的平衡。

## 结论

我们进行了大量测试，比较了上述所有技术的不同组合（嵌入模型、BM25 的使用、contextual retrieval 的使用、reranker 的使用以及检索的 top-K 结果总数），涵盖了各种不同的数据集类型。以下是我们发现的总结：

- Embeddings + BM25 优于单独的 embeddings
- Voyage 和 Gemini 在我们测试的嵌入中表现最佳
- 传递 top-20 文本块给模型比仅传递 top-10 或 top-5 更有效
- 为文本块添加 context 大幅改善检索准确性
- Reranking 优于不使用 reranking
- 所有这些好处可以叠加：为了最大化性能改进，我们可以结合 contextual embeddings（来自 Voyage 或 Gemini）与 contextual BM25，加上 reranking 步骤，并将 20 个文本块添加到 prompt 中

我们鼓励所有使用知识库的开发者使用我们的 cookbook 来实验这些方法以释放新的性能水平。

## 附录 I

以下是跨数据集、嵌入提供商、嵌入之外使用 BM25、使用 contextual retrieval 以及使用 reranking 的 Retrievals @ 20 结果细分。

见附录 II 了解 Retrievals @ 10 和 @ 5 的细分以及每个数据集的问答示例。

## 致谢

研究和撰写由 Daniel Ford 完成。感谢 Orowa Sikder、Gautam Mittal 和 Kenneth Lien 的关键反馈，Samuel Flamini 实现了 cookbook，Lauren Polansky 协调项目，以及 Alex Albert、Susan Payne、Stuart Ritchie 和 Brad Abrams 对本文的贡献。