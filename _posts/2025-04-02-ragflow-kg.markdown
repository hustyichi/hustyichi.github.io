---
layout: post
title: "来自工业界的知识库 RAG 服务(七)，RagFlow 知识图谱实践与优化方案探索"
subtitle:   "RAG service from the industry (7), RagFlow knowledge graph practice and optimization solution exploration"
date:       2025-04-02 12:30:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - agent
    - llm
    - ragflow
---

## 背景介绍
在2024 年撰写过 [RagFlow 源码全流程深度解析](https://zhuanlan.zhihu.com/p/697902937) 对 RagFlow 的完整 RAG 流程进行了深度解析，在之后 RagFlow 经历了多轮的迭代，后续补充了大量的功能，包括 Agent 能力的扩展，知识图谱的支持等。

最近看到了 RagFlow 工作号的一篇文章 [RAGFlow 0.16.0 特性总览](https://mp.weixin.qq.com/s/bm0Bwfo6y55TmdX6VlhN5A) 介绍了 RagFlow 对现有知识图谱机制的不少优化，因此花费了一些时间深入了解了 RagFlow 的知识图谱的实现流程与机制，希望帮助大家也能快速了解实现细节，从而更好地用上 RagFlow 的相关机制。

## RagFlow 产品页面

我们先来关注下现有的产品页面，方便我们更好地理解实现细节。

截止目前(2025-4)，在 RagFlow 的 Web 页面中，涉及知识图谱的配置页面如下所示：

![product](/img/in-post/ragflow-kg/product.png)

可以看到 RagFlow 的知识图谱支持进行如下所示维度的配置：

- 实体类型：知识图谱最重要的就是实体与对应关系的提取，哪些类型的实体需要提取需要被提取出来是很重要的，需要选择实际业务真正有价值的实体，这样知识图谱才能更精准;
- 方法：目前支持 Light 和 General, Light 采用了 LightRAG 的实体抽取 prompt，General 则采用了微软 GraphRAG 的 prompt，后者更长，耗费的 token 更多;
- 实体归一化开关：用于将相同的实体合并在一起，从而提升知识图谱的准确性。这是从 RagFlow 0.9 中就引入了，但是在 RagFlow 0.16 中支持了手动关闭，因为实体重复的判断是基于大模型的判断的，需要消耗大量的 Token;
- 社区报告生成：这个是 GraphRAG 中引入的，基于社区的实体和关系生成对应的摘要，可以用于准确召回对应的社区，但是社区摘要的生成同样是依赖大模型生成的，需要消耗大量的 Token，从 RagFlow 0.16 中支持了可选关闭;

从产品端可以看到 RagFlow 事实上做了不少减少大模型 Token 消耗的权衡，GraphRAG 被诟病很多情况下也是因为这个原因。下面就来具体介绍下具体的实现方案。

## GraphRAG 方案

因为 RagFlow 的知识图谱使用的就是 GraphRAG 的方案，之后在 GraphRAG 的基础上做二次优化，因此为了理解 RagFlow 的实现方案，需要先了解 GraghRAG 的方案，这部分内容可以查看微软对应的论文 [From Local to Global: A GraphRAG Approach to Query-Focused Summarization](https://arxiv.org/pdf/2404.16130)。此方案最核心的流程就是这张图：

![pipeline](/img/in-post/ragflow-kg/pipeline.png)

左侧部分的环节是从原始文档到知识图谱的构建，右侧对应的是基于知识图谱的社区构建与摘要生成。其中一些核心环节如下所示：

- 实体和关系抽取(Text Chunks -> Entities and Relations)：此环节是从原始的文档中提取实体和关系，目前是利用大模型直接抽取完成的，上面提到的 Light 和 General 的区别影响的就是这部分抽取的 prompt;
- 知识图谱构建(Entities and Relations -> Knowledge Graph)：此环节是直接将实际和关系构建为知识图谱，属于标准操作;
- 社区构建(Knowledge Graph -> Graph Communities)：此环节是使用社区检测算法将图划分为强连接节点的社区，是数据的聚类操作，GraphRAG 使用的是 Leiden 社区检测算法;
- 社区摘要生成(Graph Communities -> Community Summaries)：此环节是基于社区的实体和关系生成对应的摘要，这样才能更精准地召回对应的社区，上面提到的社区报告生成的可选开关就是关闭这部分摘要的生成;

## RagFlow 知识图谱构建实现方案

因为 RagFlow 整体采用的还是 GraphRAG 的方案，因此整体的流程与 GraphRAG 类似，在此基础上补充了一些额外的机制，下面来看具体的实现细节

#### 构建整体流程

RagFlow 的知识图谱构建的完整流程在 `graphrag/general/index.py` 中的 `run_graphrag()` 方法中实现，其中包含如下所示的环节：

1. 基于当前需要处理的文档分片构建知识图谱，其中就包括实体和关系的抽取，以及知识图谱的构建;
2. 知识图谱的合并，考虑到此知识库原有文档可能已经构建了知识图谱，因此需要将新构建的知识图谱与原有知识图谱进行合并;
3. 知识图谱的实体合并，这样可以减少实体重复，提升知识图谱的准确性，这部分是 RagFlow 额外补充的；
4. 基于知识图谱构建社区，其中就包含了社区构建与摘要生成；

对应的代码实现如下所示：

```python
async def run_graphrag(
    row: dict,
    language,
    with_resolution: bool,
    with_community: bool,
    chat_model,
    embedding_model,
    callback,
):
    # 获取文档分片

    tenant_id, kb_id, doc_id = row["tenant_id"], str(row["kb_id"]), row["doc_id"]
    chunks = []
    for d in settings.retrievaler.chunk_list(
        doc_id, tenant_id, [kb_id], fields=["content_with_weight", "doc_id"]
    ):
        chunks.append(d["content_with_weight"])

    # 构建知识图谱

    subgraph = await generate_subgraph(
        LightKGExt
        if row["parser_config"]["graphrag"]["method"] != "general"
        else GeneralKGExt,
        tenant_id,
        kb_id,
        doc_id,
        chunks,
        language,
        row["parser_config"]["graphrag"]["entity_types"],
        chat_model,
        embedding_model,
        callback,
    )
    if not subgraph:
        return

    # 合并知识图谱

    subgraph_nodes = set(subgraph.nodes())
    new_graph = await merge_subgraph(
        tenant_id,
        kb_id,
        doc_id,
        subgraph,
        embedding_model,
        callback,
    )
    assert new_graph is not None

    if not with_resolution or not with_community:
        return

    # 可关闭的实体合并机制

    if with_resolution:
        await resolve_entities(
            new_graph,
            subgraph_nodes,
            tenant_id,
            kb_id,
            doc_id,
            chat_model,
            embedding_model,
            callback,
        )

    # 可关闭的社区发现机制

    if with_community:
        await extract_community(
            new_graph,
            tenant_id,
            kb_id,
            doc_id,
            chat_model,
            embedding_model,
            callback,
        )
    return
```

#### 实体与关系抽取

知识图谱的构建的基础是实体与对应的关系，这部分也是之前大模型时代之前耗大量人力成本的部分，目前 GraphRAG 的方案中，这部分就是直接构造一个合适 prompt ，然后让大模型去抽取实体和关系。

这部分对应的实现在 `graphrag/general/graph_extractor.py` 中的 `_process_single_content()` 方法中完成，GraphRAG 为实体与关系提取设计了一个精巧的 prompt, 并通过 Few Shot 的机制进行了必要的优化，从 Few Shot 可以大致了解提取的实体与关系的格式：

![example](/img/in-post/ragflow-kg/example.png)

