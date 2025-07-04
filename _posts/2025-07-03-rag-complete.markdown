---
layout: post
title: "跟着企业 RAG 竞赛冠军学习 RAG 最佳实践"
subtitle:   "Learn RAG best practices from the enterprise RAG competition champion"
date:       2025-07-03 08:00:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - rag
    - llm
---

## 背景介绍
我一直以来都相信，大模型竞赛是一个相对有价值的试验场所。以 RAG 的实践方案而言，相关的论文多如牛毛，但是真正实践下来有用的策略相对有限。而竞赛就是一个相对客观的场地，在合适的验证集下，方案的有效性一试便知。

在之前的文章 [RAG 最佳实践](https://zhuanlan.zhihu.com/p/8861103446) 中结合过往实践与相关研究整理过个人实际验证过的最佳策略组合，最近刚好又注意到 Ilya Rice 分享了他参加企业 RAG 竞赛获奖的方案，今天尝试拆解其中的策略组合，期望从中发现有价值策略，期望对之前的最佳策略组合有一些补充。

## 整体架构
方案的整体架构如下所示：

![arch](/img/in-post/rag-complete/arch.png)

从现有架构图来看，有几个与常规的 RAG 方案存在差异的地方：

1. 知识库的路由，根据问题进行知识库路由，这样可以收缩需要检索的文档范围，提升检索效率与准确性；
2. 模板路由，根据实际的问题路由选择对应的模板，问题分类合理的情况可以提升大模型回答的质量；
3. 父子检索，此方案可以避免检索信息不完整的问题，此方案因为使用性价比高而且效果不错，实践中使用广泛；
4. 大模型重排序，一般情况下会使用重排序模型进行重排序，但是此方案中直接使用了大模型，预期效果会更好，但是成本也会更高；

而项目开发者 Ilya Rice 认为带来良好效果的策略组合如下所示：

1. Custom PDF parsing with Docling（基于 Docling 实现的自定义 PDF 解析器）
2. Vector search with parent document retrieval（父子检索）
3. LLM reranking for improved context relevance（大模型重排序）
4. Structured output prompting with chain-of-thought reasoning（基于 COT 推理的结构化输出）
5. Query routing for multi-company comparisons（用于多公司比较的查询路由）

在下面的方案分析会重点关注这些独特之处。

## 知识库构建

#### 文件解析

**解析器选型**

因为项目需要处理大量的 PDF 文件，而 PDF 的文本解析是比较困难的事情。考虑到 PDF 文件结构的多样性，目前还没有完美的解决方案。

此项目开发者实际测试下来，表现最好的是 [Docling](https://github.com/docling-project/docling), 这是一款来自 IBM 热度比较高的开源 PDF 解析方案，我实际测试表现确实不错。

当然即使是使用 Docling，依旧不能完美地进行 PDF 解析，项目中是将 PDF 解析为 JSON 文件，并使用部分正则表达式进行文本修正。

**表格数据处理**

在进行向量检索时，表格数据检索的效果比较差。因为表格真正可以用来检索的信息主要来自表头标题与当前行第 0 列信息的组合，之后还需要跨越多个分隔的数据才能找到正确的数据元素。项目开发者给出了一个比较形象的图：

![table](/img/in-post/rag-complete/table.png)

如果 RAG 的检索中没有做出合适的表格预处理，表格数据基本无法正确检索到。针对表格有两种比较常规的做法：

1. 表格切片进行检索，如果匹配到表格片段，则获取完整的表格信息，构造表格 Agent 进行数据检索，类似的实现在 langchain 中存在 [Pandas Agent](https://python.langchain.com/api_reference/experimental/agents/langchain_experimental.agents.agent_toolkits.pandas.base.create_pandas_dataframe_agent.html) 提供了支持；
2. 将原始表格按行进行序列化，将表头信息扩充至各个数据行中，这样可以避免后续无表头的数据行没有任何有效的检索信息导致无法检索，类似的实现在 Dify 的 [Csv Parser](https://github.com/langgenius/dify/blob/main/api/core/rag/extractor/csv_extractor.py) 中可以看到相关实现；
3. 将完整的表格数据提供给大模型，利用大模型的处理能力进行数据的序列化；

在此项目中采用的是第三种方案，其使用的是 GPT-4o-mini 将原始表格信息以及表格前后的内容提供给大模型，之后利用大模型直接进行序列化，对应的实现如下所示：

```python
for table in json_report["tables"]:
    # 获取表格前后的内容

    context_before, context_after = self._get_table_context(json_report, table_index)
    # 获取表格对应的 html 格式内容

    table_info = next(table for table in json_report["tables"] if table["table_id"] == table_index)
    table_content = table_info["html"]

    # 构造大模型查询的 prompt

    query = ""
    if context_before:
        query += f'Here is additional text before the table that might be relevant (or not):\n"""{context_before}"""\n\n'
    query += f'Here is a table in HTML format:\n"""{table_content}"""'
    if context_after:
        query += f'\n\nHere is additional text after the table that might be relevant (or not):\n"""{context_after}"""'

    queries.append(query)

# 调用大模型生成序列化

results = await AsyncOpenaiProcessor().process_structured_ouputs_requests(
    model='gpt-4o-mini-2024-07-18',
    temperature=0,
    system_content=TableSerialization.system_prompt,
    queries=queries,
    response_format=TableSerialization.TableBlocksCollection,
    preserve_requests=False,
    preserve_results=False,
    logging_level=20,
    requests_filepath=requests_filepath,
    save_filepath=results_filepath,
)
```

此方案相对其他方案的优势在于实现方便，而且可以结合表格前后的数据，这些地方可能会包含表名的信息，可以有效提升表格召回率。但是劣势在于预处理的成本会更高一些。


#### 分片

为了保证向量模型可以正确召回信息，一般需要将原始的文本进行切片，在 [RAG 最佳实践](https://zhuanlan.zhihu.com/p/8861103446) 我给出过建议方案：出于工程效率与检索效率的平衡，建议采用 递归字符切片 + 特定文档格式切片的组合。建议的分片长度为 256 ~ 512 tokens。

在此项目中实际使用的是递归字符切片，选择的切片长度为 300 tokens, 重合区域为 50 tokens。

#### 向量模型与向量数据库

向量模型对检索效果的影响也是比较明显的，当前项目实际选择的模型为 OpenAI 的 [text-embedding-3-large](https://platform.openai.com/docs/models/text-embedding-3-large)。

在实际进行向量模型的选型时，可以参考 [MTEB](https://huggingface.co/spaces/mteb/leaderboard)，从最近的测评来看 Qwen3-Embedding 系列的表现还不错，可以使用自己的数据集进行验证，选择最适合自己的模型。

向量数据库在当前项目中使用的是 Faiss, 这个是因为竞赛项目不需要保证服务可靠性，在生产项目中还是建议使用 Milvus。


## 知识库问答

#### 检索
在项目的检索方案中，项目探索了向量检索与 BM25 的混合检索方案。但是按照他的最简化实现，效果表现不佳，最终放弃了这个优化方向。一般而言，混合检索可以相对有效提升向量检索的召回率，建议大家在自己的实际项目中还是建议尝试。

#### 检索增强（Passage augmenter）
在检索增强的策略的选择中，实际应用的是 [RAG 最佳实践](https://zhuanlan.zhihu.com/p/8861103446) 中重点介绍的父子检索策略。从目前而言，父子检索因为实现成本低，而且可以有效的缓解单个分片信息不完整的问题，确实是一个不错的增强策略。

前不久在 LLamaIndex 的 [Building Performant RAG Applications for Production](https://docs.llamaindex.ai/en/stable/optimizing/production_rag/) 的文章中提到了一个很有价值的洞见：解耦用于检索分片与提供给大模型作为上下文的分片。相对有道理，父子检索就是实现此想法众多方案的一个实现。

#### 重排序（Passage Rerank）
重排序一般是基于重排序模型的来实现的，但是目前因为大模型在更大范围的语料的上进行过训练，一般情况下能力会更强，因此目前重排序模型


#### 检索过滤（Passage Filter）


#### 检索压缩（Passage Compressor）


#### 结果生成（Generator）


## 总结
