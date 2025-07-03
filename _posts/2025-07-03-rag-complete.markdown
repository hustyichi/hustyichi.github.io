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

而 Ilya Rice 认为最终取得良好效果的策略组合如下所示：

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

此项目中实际测试下来，表现最好的是 [docling](https://github.com/docling-project/docling), 我实际



#### 分片


#### 向量模型


#### 向量数据库



## 知识库问答

#### 查询扩展（query expansion）


#### 检索


#### 检索增强（Passage augmenter）


#### 重排序（Passage Rerank）


#### 检索过滤（Passage Filter）


#### 检索压缩（Passage Compressor）


#### 结果生成（Generator）


## 总结
