---
layout: post
title: "来自学术界的知识库 RAG 调优方案实践（一）"
subtitle:   "RAG tuning solution practice from academia (1)"
date:       2024-05-27 21:50:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - rag
    - langchain
---

## 背景介绍

在之前的文章整理过来自工业界的 RAG 方案 [QAnything](https://zhuanlan.zhihu.com/p/697031773) 和 [RagFlow](https://zhuanlan.zhihu.com/p/697902937) 的源码解析，这次主要整理下来自学术界的一系列 RAG 优化方案，通过从梳理方案的原理与对应的实现，帮助大家优化知识库 RAG 的效果。

## 基础介绍
在综述论文 [Retrieval-Augmented Generation for Large Language Models: A Survey](https://arxiv.org/pdf/2312.10997) 介绍了三种不同的 RAG 方案：

![rag_types](/img/in-post/advanced-rag/rag_types.png)

1. Native RAG: 原始 RAG，对应最原始的 RAG 流程，和之前 [搭建离线私有大模型知识库](https://zhuanlan.zhihu.com/p/689947142) 介绍的流程基本一致；
2. Advanced RAG：高级 RAG，在原始 RAG 上增加了一些优化手段，之前实践过的 [RAG Rerank](https://zhuanlan.zhihu.com/p/699339963) 优化手段就属于高级 RAG 中的 Post-Retrieval (检索后的优化)；
3. Modular RAG：模块化 RAG，通过模块化的架构设计，可以提供灵活的功能组合，方便实现功能强大的 RAG 服务。

本篇文章主要实践的还是高级 RAG，采取的优化手段主要是之前较少涉及的 Pre-Retrieval（检索前的优化），目前有大量相关论文的研究，目前主要选择其中最有代表性的方案进行实践，主要的实现都是基于 langchain 来完成。

## 优化方案

#### HyDE
HyDE 的优化手段来自于论文 [Precise Zero-Shot Dense Retrieval without Relevance Labels](https://arxiv.org/pdf/2212.10496)，主要流程如下所示：

![hyde](/img/in-post/advanced-rag/hyde.png)

用户原始的问题与需要检索文档的向量相似度上不接近，因此向量检索效果不佳。因此 HyDE 的设计思想如下：

1. 根据原始问题使用大模型生成假设文档，可以理解为使用大模型先给出的答案，此答案没有额外知识，可能存在幻觉；
2. 基于生成的假设文档进行向量检索；

为什么假设文档检索的效果会好于通过问题检索呢？我直观理解下来就是与答案语义上接近的更有可能是所需的答案，而且大模型是通过大量原始文档学习出来，因此生成的文档与原始文档上更接近，易于检索。

**实现方案**

HyDE 的实现在 langchain 已经支持了，可以通过 `from langchain.chains import hyde` 进行使用，提供是一个向量化查询的转换支持，可以看到其中最核心的方法如下所示：

```python
def embed_query(self, text: str) -> List[float]:
    """Generate a hypothetical document and embedded it."""
    # 通过大模型生成文本对应的响应

    var_name = self.llm_chain.input_keys[0]
    result = self.llm_chain.generate([{var_name: text}])
    documents = [generation.text for generation in result.generations[0]]
    # 文档向量化

    embeddings = self.embed_documents(documents)
    return self.combine_embeddings(embeddings)
```

熟悉 langchain 的研发同学应该都了解这个方法的用途，主要是用于将原始文本向量化。常规情况下需要调用下面 `self.embed_documents()` 将原始查询 `text` 向量化。

HyDE 中会增加一个大模型生成回答的流程 `self.llm_chain.generate([{var_name: text}])`，与前面的描述的基本一致。

接下来可以看看 HyDE 中使用 prompt，这部分可以在 `from langchain.chains.hyde import prompts` 中看到，看起来是根据不同场景设计了不同的 prompt，我们以 web_search 为例查看，对应如下所示：

```python
from langchain_core.prompts.prompt import PromptTemplate

web_search_template = """Please write a passage to answer the question
Question: {QUESTION}
Passage:"""
web_search = PromptTemplate(template=web_search_template, input_variables=["QUESTION"])
```

可以看到 prompt 也相对简单容易理解。

**实践效果**

理想很丰满，实践下来发现 HyDE 实际效果不佳，实际测试中大模型给出的响应与原始文档表达形式并不接近，导致最终测试时原始 query 还可以检索到部分相关文档，使用大模型给出的回答完全检索不到任何内容。

目前来看，HyDE 在大模型可以给出与文档类似的表达形式的内容时可能会有一些效果，因此对大模型的选型和使用场景上都有一些要求。

#### Rewrite-Retrieve-Read

Rewrite-Retrieve-Read 的想法来自于 [Query Rewriting for Retrieval-Augmented Large Language Models
](https://arxiv.org/pdf/2305.14283)，对应的流程如下所示：

![rewrite](/img/in-post/advanced-rag/rewrite.png)

此方法的主要思想是用户的原始问题检索效果不佳，通过大模型进行重写提升问题对应的检索能力。这个有实际上线 RAG 服务的应该有类似的遭遇，用户的问题都是千奇百怪的，确实存在原始问题检索效果不佳的情况。

**实现方案**

Rewrite-Retrieve-Read 的实现方案相对明确，使用大模型直接重写问题即可，对应的实现可以参考 [langchain template](https://github.com/langchain-ai/langchain/blob/master/templates/rewrite-retrieve-read/rewrite_retrieve_read/chain.py)

核心的功能如下所示：

```python
template = """Provide a better search query for \
web search engine to answer the given question, end \
the queries with ’**’. Question: \
{x} Answer:"""
rewrite_prompt = ChatPromptTemplate.from_template(template)

def _parse(text):
    return text.strip("**")

rewriter = rewrite_prompt | ChatOpenAI(temperature=0) | StrOutputParser() | _parse
```

可以看到 langchain 中设计了一个特殊的 prompt 进行了原始问题的转换，思路相对简单。

**实践效果**

最终实际测试下来，效果不是特别问题，部分问题转换后的效果更好，部分效果更差，从目前的理解来看，与大模型本身的能力存在较大关系。

#### Query2Doc

Query2Doc 的想法来自于 [Query2doc: Query Expansion with Large Language Models](https://arxiv.org/pdf/2303.07678)，是在 HyDE 的想法上进行了一些提升。

原始的 HyDE 是使用大模型生成的答案（Hypothetical Document）进行检索，而 Query2Doc 则会将生成的答案与原始问题进行拼接，之后使用拼接得到的内容进行检索。

针对不同的检索方案拼接方案有所差异，其中稀疏检索的中的拼接方案如下所示：

![sparse](/img/in-post/advanced-rag/sparse.png)

其中的 `q` 为原始问题，`d` 为生成的回答，可以看到 `q` 会被重复 `n` 次，并与 `d` 拼接起来。

而密集检索的拼接方案如下所示：

![dense](/img/in-post/advanced-rag/dense.png)

可以看到是直接将原始问题 `q` 与大模型生成的回答 `d` 进行了拼接，中间使用分隔符 `[SEP]` 进行了分隔。

**实现方案**

可以参考 HyDE 进行简单调整即可实现 Query2Doc，简单的示例如下所示：

```python
def embed_query(self, text: str) -> List[float]:
    """Generate a hypothetical document and embedded it."""
    # 通过大模型生成文本对应的响应

    var_name = self.llm_chain.input_keys[0]
    result = self.llm_chain.generate([{var_name: text}])
    documents = [generation.text for generation in result.generations[0]]
    # 拼接原始查询与生成的响应，使用空格作为分隔符 SEP

    documents = [f"{text} {doc}" for doc in documents]
    # 文档向量化

    embeddings = self.embed_documents(documents)
    return self.combine_embeddings(embeddings)
```

**实践效果**

实际测试下来，相对原始的 HyDE 方案效果更好，但是实际效果改善不明显。


## 总结

本次测试了目前比较常规的几种 Pre-Retrieval 优化手段，从目前来看，优化思想都相对容易理解，利用大模型提供的能力优化了原始 query 检索效果不佳的问题。

但是实际测试下来，改善效果相对有限，不如之前测试过的 [Rerank](https://zhuanlan.zhihu.com/p/699339963) 机制那么立竿见影。后续会进一步调研其他可能的优化方案，欢迎关注。

