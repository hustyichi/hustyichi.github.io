---
layout: post
title: "NVIDIA 结构化报告生成方案详解"
subtitle:   "Detailed explanation of NVIDIA's structured report generation solution"
date:       2025-01-19 20:45:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - agent
    - llm
---

## 背景介绍
2025 年是大模型迅速落地的一年，Agent 的发展是其中重要的一环。最近看到了来自 NVIDIA 的结构化报告生成方案，刚好在 2024 年底被大量 RAG 总结的论文列表轰炸了一番，密集阅读大量论文时，真切感受到深入了解一个领域论文对精力的消耗是巨大的。因此深入研究了 NVIDIA 的方案，希望后续在此基础上进行二次改良，实现自己的论文阅读助手，真正减少阅读论文的时间。


## 整体方案
整体的技术方案如下所示:

![flow](/img/in-post/structured-report/flow.png)

整体的方案主要包含两部分：
1. 规划阶段，根据用户的输入，生成整体的报告的骨架，确定报告的结构；
2. 调研阶段，独立对各个环节进行调研和内容填充，最终生成最终的报告。

整体的方案存在如下所示的基础组件依赖：
1. [Tavily](https://tavily.com/) 进行内容获取，Travily 是一个搜索引擎 API 服务，可以基于用户输入的内容进行检索。
2. 基于 NVIDIA NIM 进行大模型调用，实际可以使用其他的 LLM 服务，但是需要有结构化输出的能力；
3. 基于 LangChain 进行整体的串联，实际主要基于 [Langchain Graph](https://www.langchain.com/langgraph) 作为工作流的组织方案；


## 规划阶段

规划阶段主要目的是生成整体的报告的骨架，确定报告的结构。理论上直接基于大模型就可以直接生成，但是考虑到科研环节调研往往需要具备前沿性，而大模型的信息往往往往具备滞后性，因此需要使用 Travily 额外获取信息，作为补充信息提供给大模型。

#### 信息检索

原始的用户输入直接进行检索可能会因为输入的模糊性导致检索不充分的问题，因此方案中首先进行了查询的扩充，此方案在之前的 RAG 中也比较常规了，在此方案中使用的扩充查询的 prompt 如下所示：

```python
report_planner_query_writer_instructions = """You are an expert technical writer, helping to plan a report.
    The report will be focused on the following topic:

    {topic}

    The report structure will follow these guidelines:

    {report_organization}

    Your goal is to generate {number_of_queries} search queries that will help gather comprehensive information for planning the report sections.

    The query should:

    1. Be related to the topic
    2. Help satisfy the requirements specified in the report organization

    Make the query specific enough to find high-quality, relevant sources while covering the breadth needed for the report structure.
    """
```

扩充后需要调用大模型生成多个查询，因此大模型需要具备结构化输出的能力，方便进行多个查询的提取。之后直接调用 tavily 客户端获取所需的信息。

在检索到所需的信息后，需要基于检索的 URL 地址进行去重，并将检索到的内容组织为格式化字符串，方便大模型可以正常理解并进行正确的文章结构的生成。

#### 报告骨架生成

基于检索的信息，直接调用大模型生成最终的文章的各个部分，生成各个环节的 prompt 如下所示：

```python
report_planner_instructions = """You are an expert technical writer, helping to plan a report.
    Your goal is to generate the outline of the sections of the report.

    The overall topic of the report is:

    {topic}

    The report should follow this organization:

    {report_organization}

    You should reflect on this information to plan the sections of the report:

    {context}

    Now, generate the sections of the report. Each section should have the following fields:

    - Name - Name for this section of the report.
    - Description - Brief overview of the main topics and concepts to be covered in this section.
    - Research - Whether to perform web research for this section of the report.
    - Content - The content of the section, which you will leave blank for now.

    Consider which sections require web research. For example, introduction and conclusion will not require research because they will distill information from other parts of the report."""
```

最终生成的报告结构包含多个 Section, 每个 Section 包含如下所示的信息：

1. Name: 报告的名称
2. Description: 报告的描述
3. Research: 是否需要进一步进行调研
4. Content: 报告的具体内容

