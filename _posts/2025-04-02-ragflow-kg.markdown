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
在2024 年我曾撰写过 [RagFlow 源码全流程深度解析](https://zhuanlan.zhihu.com/p/697902937)，对 RagFlow 的完整 RAG 流程进行了深入剖析。自那时起，RagFlow 经历了多轮迭代演进，不仅扩展了 Agent 能力，更是在知识图谱支持方面取得了显著进展。

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

1. 基于当前需要处理的文档分片构建知识图谱，其中就包括实体和关系的抽取，以及知识图谱的生成;
2. 知识图谱的合并，考虑到此知识库原有文档可能已经构建了知识图谱，因此需要将新生成的知识图谱与原有知识图谱进行合并;
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

根据上面看到的 example，大模型的结果是文本对应的实体和关系，并使用分隔符进行连接，之后在 `_entities_and_relations()` 方法中使用基于分隔符进行切分，即可得到对应的实体和关系。

#### 知识图谱生成与合并

在抽取到实体和关系之后，Ragflow 会根据本次的抽取的结果生成知识图谱，并与知识库中原有的知识图谱进行合并。

知识图谱的生成是基于 [networkx](https://github.com/networkx/networkx) 实现的，实现方案还是比较简单的，直接调用 networkx 中的 `add_node()` 和 `add_edge()` 方法即可, 感兴趣可以去 `graphrag/general/index.py` 中的 `generate_subgraph()` 方法中查看对应的实现。】

知识图谱的合并在 `graph_merge()` 中完成，遍历新生成的知识图谱的节点和关系，如果不存在则直接添加，在存在的情况下，处理方案如下所示：

- 节点存在情况下，将本次节点的描述信息叠加至原有节点的描述信息上；
- 关系存在情况下，将本次关系的描述信息，权重，关键词等信息都叠加至原有关系上；

#### 知识图谱的实体合并

知识图谱的实体合并是知识图谱构建中一个重要环节，重复的实体会降低检索的准确性。RagFlow 的实体合并在 `graphrag/entity_resolution.py` 中实现，目前采取两步来实现：

1. 初步相似度判断，利用编辑距离等工程手段将相似的实体初步筛选出来，编辑距离判断是基于 [editdistance](https://github.com/roy-ht/editdistance) 实现；
2. 利用大模型进行最终的相似度判断，从而确定最终的实体合并结果；

初步相似度主要是为了减少候选实体对判断的数量，从实际的情况来看，基于大模型进行实体相似度判断的准确性虽然更高，但是带来的开销也是比较大的，这也是为什么 RagFlow 提供了手动关闭的主要原因。

#### 社区构建

RagFlow 的社区构建是基于 Leiden 算法实现的，基于模块度优化，能够生成更高质量的社区划分，相关的实现原理可以参考 [From Louvain to Leiden: guaranteeing well-connected communities](https://arxiv.org/pdf/1810.08473)。这部分的功能是在 `graphrag/general/leiden.py` 中完成。

社区的构建是基于 [graspologic](https://github.com/graspologic-org/graspologic) 中的 `hierarchical_leiden` 实现，对应的代码如下所示：

```python
def _compute_leiden_communities(
        graph: nx.Graph | nx.DiGraph,
        max_cluster_size: int,
        use_lcc: bool,
        seed=0xDEADBEEF,
) -> dict[int, dict[str, int]]:

    results: dict[int, dict[str, int]] = {}
    if is_empty(graph):
        return results
    if use_lcc:
        graph = stable_largest_connected_component(graph)

    # 多层级社区划分

    community_mapping = hierarchical_leiden(
        graph, max_cluster_size=max_cluster_size, random_seed=seed
    )
    for partition in community_mapping:
        results[partition.level] = results.get(partition.level, {})
        results[partition.level][partition.node] = partition.cluster

    return results
```

#### 社区摘要生成

社区摘要的生成是基于社区中实体和关系的描述信息生成的，相关功能完全借助大模型实现。其主要完成的工作是一个复杂的 prompt 构造，通过大模型的总结，生成一段最能代表该社区核心内容的文本，方便进行后续的检索与召回。

这部分的 prompt 可以在 `graphrag/general/community_report_prompt.py` 中查看，内容比较长，可以看到核心的内容如下：

![summary](/img/in-post/ragflow-kg/summary.png)

输入数据源为实体和对应的关系，大模型会生成社区的摘要信息，并生成对应的关键见解，从而更精准地召回对应的社区。


## 知识图谱检索

在知识图谱构建完成后，RagFlow 中就可以利用知识图谱增强 RAG 的检索，可以从 RagFlow 中提供的检索接口中看到具体的实现方案，以 `/dify/retrieval` 接口为例，看到的检索接口实现如下所示：

```python
ranks = settings.retrievaler.retrieval(
    question,
    embd_mdl,
    kb.tenant_id,
    [kb_id],
    page=1,
    page_size=top,
    similarity_threshold=similarity_threshold,
    vector_similarity_weight=0.3,
    top=top,
    rank_feature=label_question(question, [kb])
)

# 可选开启知识图谱检索

if use_kg:
    ck = settings.kg_retrievaler.retrieval(question,
                                            [tenant_id],
                                            [kb_id],
                                            embd_mdl,
                                            LLMBundle(kb.tenant_id, LLMType.CHAT))
    # 知识图谱检索的内容插入常规检索的最前面，知识图谱检索优先展示

    if ck["content_with_weight"]:
        ranks["chunks"].insert(0, ck)
```


具体的检索流程在 `graphrag/search.py` 中实现，具体流程如下所示：

1. 查询重写: 调用 `query_rewrite` 方法，使用大模型提取问题中的实体类型关键词和实体，处理失败时则直接使用原始问题作为实体；
2. 实体与关系检索：通过关键词 (get_relevant_ents_by_keywords) 与实体类型 (get_relevant_ents_by_types)检索相关实体，通过原始问题检索相关关系(get_relevant_relations_by_txt);
3. 路径分析：从关键词检索到实体出发，获取实体相关的 N 跳邻居，实现了相似分的衰减，举例越远，相似分越低；
4. 结果融合与评分：将关键词检索结果，实体类型检索结果以及关系检索检索的结果进行融合，确定最终的得分；
5. 结果排序与截断: 按相似度和 PageRank 的乘积对实体和关系进行排序，截取前 N 个结果；
6. 社区检索：调用 `_community_retrival_()` 方法检索相关社区报告, 即通过社区摘要信息进行检索, 从前面可以看到，这部分检索内容是可选关闭的；
7. 结果组合：将实体、关系和社区报告组合成最终结果；

## 总结
本文介绍了 RagFlow 的知识图谱的构建与检索流程，整体流程与 GraphRAG 中的流程基本上能一一对应上，甚至有大量的代码是直接复用 GraphRAG 的实现。并在此基础上，RagFlow 提供了更多的可选方案，例如社区摘要的可关闭性、实体合并的可关闭性等，使得用户可以根据自己的需求在效果与性能之间进行权衡。
