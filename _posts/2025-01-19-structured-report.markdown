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
2. 调研生成阶段，独立对各个环节进行调研和内容填充，最终生成最终的报告。

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

## 调研生成阶段

调研生成阶段进行简单的总结，主要分成下面三个部分：
1. 基于规划阶段生成的报告结构，并行进行各个 Section 的调研和内容填充；
2. 基于各个 Section 的调研内容，使用大模型进行内容的组织与优化；
3. 将各个 Section 的内容进行汇总，生成最终的报告。

#### Section 调研与内容填充

进行各个 Section 的内容填充，和规划阶段类型，内容主要基于大模型生成，但是考虑到大模型的滞后性问题，因此需要使用 tavily 进行信息补充。因此流程就包含了信息检索与内容填充两部分。

与规划阶段类似，同样需要进行查询的扩充，使用的 prompt 如下所示：

```python
query_writer_instructions = """Your goal is to generate targeted web search queries that will gather comprehensive information for writing a technical report section.
    Topic for this section:
    {section_topic}

    When generating {number_of_queries} search queries, ensure they:
    1. Cover different aspects of the topic (e.g., core features, real-world applications, technical architecture)
    2. Include specific technical terms related to the topic
    3. Target recent information by including year markers where relevant (e.g., "2024")
    4. Look for comparisons or differentiators from similar technologies/approaches
    5. Search for both official documentation and practical implementation examples

    Your queries should be:
    - Specific enough to avoid generic results
    - Technical enough to capture detailed implementation information
    - Diverse enough to cover all aspects of the section plan
    - Focused on authoritative sources (documentation, technical blogs, academic papers)"""
```

之后基于大模型生成的多个查询，调用 tavily 进行信息检索，并基于检索到的信息进行调用大模型进行 Section 的内容生成。Section 内容生成的 prompt 如下所示：

```python
section_writer_instructions = """You are an expert technical writer crafting one section of a technical report.
    Topic for this section:
    {section_topic}

    Guidelines for writing:

    1. Technical Accuracy:
    - Include specific version numbers
    - Reference concrete metrics/benchmarks
    - Cite official documentation
    - Use technical terminology precisely

    2. Length and Style:
    - Strict 150-200 word limit
    - No marketing language
    - Technical focus
    - Write in simple, clear language
    - Start with your most important insight in **bold**
    - Use short paragraphs (2-3 sentences max)

    3. Structure:
    - Use ## for section title (Markdown format)
    - Only use ONE structural element IF it helps clarify your point:
    * Either a focused table comparing 2-3 key items (using Markdown table syntax)
    * Or a short list (3-5 items) using proper Markdown list syntax:
        - Use `*` or `-` for unordered lists
        - Use `1.` for ordered lists
        - Ensure proper indentation and spacing
    - End with ### Sources that references the below source material formatted as:
    * List each source with title, date, and URL
    * Format: `- Title : URL`

    3. Writing Approach:
    - Include at least one specific example or case study
    - Use concrete details over general statements
    - Make every word count
    - No preamble prior to creating the section content
    - Focus on your single most important point

    4. Use this source material to help write the section:
    {context}

    5. Quality Checks:
    - Exactly 150-200 words (excluding title and sources)
    - Careful use of only ONE structural element (table or list) and only if it helps clarify your point
    - One specific example / case study
    - Starts with bold insight
    - No preamble prior to creating the section content
    - Sources cited at end"""
```

可以看到上面的 prompt 引导大模型使用 Markdown 的语法进行内容的输出，在每个 Section 的最后，需要使用 `### Sources` 进行源信息的引用，并使用 `- Title : URL` 的格式进行源信息的引用。

#### Section 内容组织与优化

对于单个 Section 生成的内容，会执行格式化转换，并使用大模型进行内容的组织与优化，实际就是就是基于单次大模型调用实现，使用的 prompt 如下所示：

```python
final_section_writer_instructions = """You are an expert technical writer crafting a section that synthesizes information from the rest of the report.
    Section to write:
    {section_topic}

    Available report content:
    {context}

    1. Section-Specific Approach:

    For Introduction:
    - Use # for report title (Markdown format)
    - 50-100 word limit
    - Write in simple and clear language
    - Focus on the core motivation for the report in 1-2 paragraphs
    - Use a clear narrative arc to introduce the report
    - Include NO structural elements (no lists or tables)
    - No sources section needed

    For Conclusion/Summary:
    - Use ## for section title (Markdown format)
    - 100-150 word limit
    - For comparative reports:
        * Must include a focused comparison table using Markdown table syntax
        * Table should distill insights from the report
        * Keep table entries clear and concise
    - For non-comparative reports:
        * Only use ONE structural element IF it helps distill the points made in the report:
        * Either a focused table comparing items present in the report (using Markdown table syntax)
        * Or a short list using proper Markdown list syntax:
        - Use `*` or `-` for unordered lists
        - Use `1.` for ordered lists
        - Ensure proper indentation and spacing
    - End with specific next steps or implications
    - No sources section needed

    3. Writing Approach:
    - Use concrete details over general statements
    - Make every word count
    - Focus on your single most important point

    4. Quality Checks:
    - For introduction: 50-100 word limit, # for report title, no structural elements, no sources section
    - For conclusion: 100-150 word limit, ## for section title, only ONE structural element at most, no sources section
    - Markdown format
    - Do not include word count or any preamble in your response"""
```

#### 报告汇总

将各个 Section 的内容进行汇总，生成最终的报告。这部分就是简单字符串拼接，没有太多额外的操作。


## 总结
本文整体介绍了 Nvidia 的结构化报告生成方案，整体方案比较简单，比较值得关注的是整体的 Agent 流程与相关的 prompt 设计。此方案给出了一个相对简洁而且完整的结构化报告生成方案，大家都可以在此基础上进行二次改造，实现自己的论文调研助手。

从目前来看，在 langchain graph 的支持下，Agent 应用的流程构造变得相对简单。只要能清晰梳理出核心业务环节，就可以将原始业务需求拆分为不同的节点，并借助 langchain graph 即可将节点的执行逻辑串联起来，实现完整的 Agent 流程。借助现有开源框架，确定能大大降低 Agent 应用的开发成本。
