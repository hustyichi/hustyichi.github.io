---
layout: post
title: "RAG 远远不够，外部数据增强的不同姿势探索"
subtitle:   "RAG is far from enough, exploring different solutions for external data augmentation"
date:       2025-04-10 22:30:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - rag
    - llm
---

## 背景介绍

在过去一两年中，实际进行了不同的 RAG 方案的实践和调优，从早期的 langchain Demo 搭建到实际记录过的 [langchain chatchat](https://zhuanlan.zhihu.com/p/689947142), 后续又记录过 [QAnything](https://zhuanlan.zhihu.com/p/699339963) 和 [RagFlow](https://zhuanlan.zhihu.com/p/697902937)。检索手段也从向量检索发展至[知识图谱](https://zhuanlan.zhihu.com/p/1891156928022939096)和 Agentic RAG 方案，甚至偶尔还需要针对结构化数据尝试 NL2SQL 方案。

但是在实践过程中，依旧会感受到 RAG 方案的局限性，现实中的场景的复杂程度往往超出 RAG 可用的范畴。如果更好地用好外部数据是一个极有挑战的事情，刚好最近微软亚洲研究院的论文 [Retrieval Augmented Generation (RAG) and Beyond](https://arxiv.org/pdf/2409.14924) ,系统介绍了外部数据增强的不同手段，感觉颇受收获，因此记录在这边记录相关内容。

## 数据检索层级

![levels](/img/in-post/external-data/levels.png)

## Level-1 Explicit Facts

## Level-2 Implicit Facts

## Level-3 Interpretable Rationales

## Level-4 Hidden Rationales

