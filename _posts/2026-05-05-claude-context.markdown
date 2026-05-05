---
layout: post
title: "大型代码库开发的全新解法 - claude-context"
subtitle:   "A New Solution for Large Codebases — claude-context"
date:       2026-05-05 20:30:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - llm
    - agent
    - rag
---

## 背景介绍

简单介绍自 2025 下半年以来的 AI 编程的快速发展，虽然能力得到快速发展，但是依旧在小型项目，新项目中表现更好，原有的大型项目表现会更差。

原因主要来自于大型项目需要关注的信息更多，实际开发时需要考虑的内容更多，大模型很难面面俱到。大模型更像一个很聪明，但是不了解你的个人实际情况的工程师。

在过去的解决方案中，Claude Code 等开发工具选择使用 CLI 工具，比如 Grep 等帮助快速检索相关信息，但是这种方案依赖存在明确的字段精确匹配，要不然就很难发挥作用，当然过往大模型有比较强的理解能力，做了不少的能力适配，提升了构建查询条件的能力，但是依旧主要以精确匹配为主。

项目 [claude-context](https://github.com/zilliztech/claude-context) 提供了全新的解决方案，基于 RAG 提供了更强的代码检索能力，利用多维向量打通自然语言与代码之间的连接，提供更加语义化的检索能力。在官方提供的基准测试中，可以降低 40% 的 Token 消耗，可以降低成本和时间。

![efficiency](/img/in-post/claude-context/mcp_efficiency_analysis_chart.png)

## 核心架构

当前项目的核心架构如下所示：

![arch](/img/in-post/claude-context/Architecture.png)

熟悉 RAG 的工程师应该会很容易理解这个架构，其实就是一个典型的 RAG 方案，只是提供了一个针对代码库的解决方案。

- 左侧的 `Embedding Service` 对应于的项目模型的选型，考虑到当前方案需要实现代码与自然语言的关联，因此向量模型的选型需要针对代码场景进行有针对性的训练，目前选择的模型主要是 OpenAI Embdding, VoyageAI Embdding 模型
- 右侧向量库的选择目前使用的是 Milvus, 针对这种检索场景，准确性主要依赖向量模型的正确选型，知识库只要选择主流的预期都可以正常工作；
- 中间的文本处理主要是代码分片方案的设计，优先使用 AST Parse 这种基于代码语法树的切分方案，AST Parse 分片过大的情况下则基于滑动窗口 + 必要的 Overlap 去执行切分。


## 检索方案


## 代码分片方案



## 动态更新机制



## 总结


