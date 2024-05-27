---
layout: post
title: "来自学术界的知识库 RAG 调优方案实践"
subtitle:   "RAG tuning solution practice from academia"
date:       2024-05-27 21:50:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - rag
---

## 背景介绍

在之前的文章整理过来自工业界的 RAG 方案 [QAnything](https://zhuanlan.zhihu.com/p/697031773) 和 [RagFlow](https://zhuanlan.zhihu.com/p/697902937) 的源码解析，这次主要整理下来自学术界的一系列 RAG 优化方案，通过从梳理方案的原理与对应的实现，帮助大家优化知识库 RAG 的效果。

## 基础介绍
在综述论文 [Retrieval-Augmented Generation for Large Language Models: A Survey](https://arxiv.org/pdf/2312.10997) 介绍了三种不同的 RAG 方案：

![rag_types](/img/in-post/advanced-rag/rag_types.png)

1. Native RAG: 原始 RAG，对应最原始的 RAG 流程，和之前 [搭建离线私有大模型知识库](https://zhuanlan.zhihu.com/p/689947142) 介绍的流程基本一致；
2. Advanced RAG：高级 RAG，在原始 RAG 上增加了一些优化手段，之前实践过的 [RAG Rerank](https://zhuanlan.zhihu.com/p/699339963) 优化手段就属于高级 RAG 中的 Post-Retrieval (检索后的优化)；
3. Modular RAG：模块化 RAG，通过模块化的架构设计，可以提供灵活的功能组合，方便实现功能强大的 RAG 服务。

本篇文章主要实践的还是高级 RAG，采取的优化手段主要是之前较少涉及的 Pre-Retrieval（检索前的优化），目前有大量相关论文的研究，目前主要选择其中最有代表性的方案进行实践。

