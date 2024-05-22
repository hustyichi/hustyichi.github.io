---
layout: post
title: "以《红楼梦》的视角看大模型知识库 RAG 服务的 Rerank 调优"
subtitle:   "Rerank tuning of RAG service in large model knowledge base from the perspective of "Dream of Red Mansions""
date:       2024-05-22 21:00:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - rag
---

## 背景介绍
在之前的文章 [有道 QAnything 源码解读](https://zhuanlan.zhihu.com/p/697031773) 中介绍了有道 RAG 的一个主要亮点在于对 Rerank 机制的重视。

从目前来看，Rerank 确实慢慢演化为 RAG 的一个重要模块，在这篇文章中就希望能讲清楚为什么 RAG 服务需要 Rerank 机制，以及如何选择最合适的 Rerank 模型。最终以完整的《红楼梦》知识库为例进行实践，看看 Rerank 带来的效果差异。

## Rerank 是什么
以有道 QAnything 的架构来看 Rerank 在 RAG 服务中所处的环节

![qanything_arch](/img/in-post/rerank/qanything_arch.png)

可以看到有道 QAnything 中的 Rerank 被称为为 2nd Retrieval, 主要的作用是将向量检索的内容进行重新排序，此时更精准的文章会出现在前面，通过提供更精准的文档，从而提升 RAG 的效果。

在目前的 2 阶段检索设计下，Embedding 模型只需要保证召回率，从海量的文档中获取到备选文档列表，不需要保证顺序的绝对准确，而 Rerank 模型负责对少量的备选文档列表的顺序进行精调，优中选优确定最终传递给大模型的文档。

简单打个比方，目前的 Embedding 检索类似学校教育筛选中的实验班，从大量的学生中捞出有潜质的优秀学生。而 Rerank 则从实验班中进一步精心排序，选出少数能上清北的重点培养。

## Embedding 相似分为什么不够用

Embedding 检索时会获得问题与文本之间的相似分，以往的 RAG 服务直接基于相似分进行排序，但是事实上向量检索的相似分是不够准确的。

原因是因为 Embedding 过程是将文档的所有可能含义压缩到一个向量中，方便使用向量进行检索。但是文本压缩为向量必然会损失信息，从而导致最终 Embedding 检索的相似分不够准确。参考 [Pipecone](https://www.pinecone.io/learn/series/rag/rerankers/) 对应的图示如下所示:

![embedding](/img/in-post/rerank/embedding.webp)

而 Rerank 阶段不会向量化，而是将查询与匹配的单个文档 1 对 1 的计算相似分，此时必然会得到更好的效果，对应的过程如下所示：

![rerank_diff](/img/in-post/rerank/rerank_diff.webp)

除了这个原因以外，拆分 Rerank 阶段也提供了更加灵活的选择文档的能力，比如之前介绍过的 [Ragflow](https://zhuanlan.zhihu.com/p/697902937) 就是在检索中使用 `0.3 * 文本匹配得分 + 0.7 * 向量匹配得分` 加权计算出综合得分进行排序，Rerank 阶段可以提供类似这种灵活的选择手段。

## 支持超大上下文的模型是否可以避免 Rerank
我们之所以需要提供准确的文档，是因为大模型提供的窗口有限，无法塞入过多的文档。但是目前有模型提供了更大的上下文窗口，重排是否就失去了意义。

Pipecone 提供的曲线图可以看到 GPT3.5 中存在如下所示现象：

![more_issue](/img/in-post/rerank/more_issue.webp)

随着塞入上下文中的文档增加时，大模型从文档中召回信息的准确性会下降。

以我实际碰到的一个例子来看，两次检索与重排得到的都是同样顺序的文本内容，而且答案都出现在排名第一的文档中，当我选择


## Rerank 模型的选择

