---
layout: post
title: "从《红楼梦》的视角看大模型知识库 RAG 服务的 Rerank 调优"
subtitle:   "Rerank tuning of RAG service in large model knowledge base from the perspective of 'Dream of Red Mansions'"
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

从目前来看，Rerank 确实逐渐成为 RAG 的一个重要模块，在这篇文章中就希望能讲清楚为什么 RAG 服务需要 Rerank 机制，以及如何选择最合适的 Rerank 模型。最终以完整的《红楼梦》知识库进行实践。

如果你问为什么要用红楼梦，答案就是作为读了很多遍《红楼梦》的忠实粉丝，问题的答案是不是靠谱一眼就能判断，问题相关的片段也能快速定位，就像定位 bug 一样快。

## Rerank 是什么
以有道 QAnything 的架构来看 Rerank 在 RAG 服务中所处的环节

![qanything_arch](/img/in-post/rerank/qanything_arch.png)

可以看到有道 QAnything 中的 Rerank 被称为为 2nd Retrieval, 主要的作用是将向量检索的内容进行重新排序，此时更精准的文章会出现在前面，通过提供更精准的文档，从而提升 RAG 的效果。

在目前的 2 阶段检索设计下，Embedding 模型只需要保证召回率，从海量的文档中获取到备选文档列表，不需要保证顺序的绝对准确，而 Rerank 模型负责对少量的备选文档列表的顺序进行精调，优中选优确定最终传递给大模型的文档。

简单打个比方，目前的 Embedding 检索类似学校教育筛选中的实验班，从大量的学生中捞出有潜质的优秀学生。而 Rerank 则从实验班中进一步精心排序，选出少数能上清北的重点培养。

## Embedding 相似分为什么不够

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

随着塞入大模型上下文的文档数增加，大模型从文档中召回信息的准确性会下降。

以我实际碰到的一个例子来看，两次检索与重排得到的都是同样顺序的文本内容，而且需要召回的信息都出现在排名第一的文档中，当我选择加入上下文中的文档数量为 10 时，大模型最终的回答如下所示：

![10th](/img/in-post/rerank/10th.png)

但是减少加入上下文的数量为 5 时，大模型最终的答案如下所示：

![5th](/img/in-post/rerank/5th.png)

可以看到，加入大模型的文档过多时，确实召回信息的准确性会下降。所以超大上下文窗口并不是解决所有问题的灵丹妙药

## Rerank 模型的选择

目前开源的 Rerank 模型选择不入 Embedding 模型那么多，主要是下面几个：

- 智源提供的 BGE 系列
- 有道提供的 BCE 模型
- Cohere 提供的 CohereRerank 系列模型

由于主要考虑开源可使用的模型，因此 CohereRerank 模型就不考虑，结合搜索的信息确定备选的模型如下所示：

- [bce-reranker-base_v1](https://huggingface.co/maidalun1020/bce-reranker-base_v1) 有道 BCE 系列只提供了这一款 Rerank 模型，上个月 huggingface 下载量 8.7k
- [bge-reranker-large](https://huggingface.co/BAAI/bge-reranker-large) 这一款为 BGE 系列的经典 Rerank 模型，评价比较高，上个月 huggingface 下载量 33.2k
- [bge-reranker-v2-m3](https://huggingface.co/BAAI/bge-reranker-v2-m3) 这一款为智源目前推荐的多种场景的建议模型，上个月 huggingface 下载量 69.4k

哪个模型表现更好呢？

有道在 [网站](https://github.com/netease-youdao/BCEmbedding/blob/master/README_zh.md) 上放出来的测评结果如下所示：

![rag_eval_multiple_domains_summary](/img/in-post/rerank/rag_eval_multiple_domains_summary.jpg)

可以看到测评得到的结果是 bce-reranker-base_v1 是略胜一筹的，当然大家都知道各家官方上给出的都是对自己有利的测评结果，但是其中对友商的测评结果大概率还是可以相信的，可以看到 bge-reranker-v2-m3 确实小幅领先 bge-reranker-large

另外一个测评是来自于 [智源网站](https://github.com/FlagOpen/FlagEmbedding/tree/master/FlagEmbedding/llm_reranker) 提供的

![llama-index](/img/in-post/rerank/llama-index.png)

可以看到表现最好是 bge-rerank-v2-minicpm-28, 但是看了下 hugggingface 上模型大小为 10G 左右，相对 2G 左右的 bge-reranker-large 大了太多了。

另外看了下网络上大家的实际体验，反而有人反馈 bce-reranker-base_v1 和 bge-reranker-large 实际应用表现更好。初步估计这三个应该都还算不错，至少应该在大部分场景下都是可用的。

## 动手实践

具体的实现可以直接参考 [langchain-chatchat](https://github.com/chatchat-space/Langchain-Chatchat/blob/master/server/reranker/reranker.py) 即可。

主要的功能就是加载 Rerank 模型，然后计算文档块与问题的相似分，之后根据此相似分进行重排。我实际测试时额外补充了一个 Rerank 后的阈值过滤，将 Rerank 相似分低于 0.1 的过滤掉了。

接下来的实践过程相对简单，将完整的《红楼梦》导入知识库，分片向量化，为了方便在本地的普通 CPU 笔记本上运行，我接入了百度提供的在线大模型 API 进行测试。而 Embedding 模型我使用的是 [stella-base-zh-v3-1792d](https://huggingface.co/infgrad/stella-base-zh-v3-1792d)，这个只是从 huggingface 的 Embedding 榜单中选择一个相对靠前而且比较轻量。

在本次测试中，我主要验证了两种场景：

1. 多 Rerank 模型的效果比较，主要关注同样的文档使用 Rerank 模型排序后的结果差异，看哪个 Rerank 排序的结果最符合人的预期；
2. 选择 Rerank 模型的优胜者和无 Rerank 情况进行比较，确认是否吊打无 Rerank 的情况；

#### 多 Rerank 模型比较

实际基于 Kimi 生成了 15 个问题，具体的问题如下所示：

1. 红楼梦中，贾宝玉和林黛玉第一次见面是在哪一年？
2. 请列举《红楼梦》中贾府的四位主要女性角色。
3. 《红楼梦》中，王熙凤为什么被称为“凤姐”？
4. 贾宝玉的通灵宝玉有什么特殊之处？
5. 林黛玉的诗作中，有一首描写春天的诗，请找出并描述其内容。
6. 在《红楼梦》中，贾母为何偏爱贾宝玉？
7. 描述《红楼梦》中薛宝钗的性格特点。
8. 请找出《红楼梦》中关于“大观园”的描述，并说明其在故事中的意义。
9. 《红楼梦》中，贾宝玉和林黛玉的关系是如何发展的？
10. 请列出《红楼梦》中提到的几种不同的茶。
11. 红楼梦中，林黛玉的死因是什么？
12. 请列举《红楼梦》中贾府的几位仆人，并描述他们的角色。
13. 《红楼梦》中，贾宝玉和薛宝钗的婚姻是如何安排的？
14. 红楼梦中，贾元春的封号是什么？
15. 请描述《红楼梦》中贾宝玉的人生观和价值观。

测试中除了 Rerank 模型外，其他部分都保持一致，最终测试下来，结论如下：

1. **bge-reranker-v2-m3 模型表现最好，bce-reranker-base_v1 和 bge-reranker-large 在不同问题中各有胜负。**
2. **bge-reranker-large 的相似分很容易出现接近 1 的高分，从人工来看并不完全相似**
3. **bce-reranker-base_v1 的相似分相对更均衡，经常在 0.4 ~ 0.7 之间，与常规的人的直觉更接近**

当然这个测试偏向简单验证，有一定的主观判断，而且问题的数量不够多，如果进行模型选型还是需要根据自己的数据集和问题进行进一步测试。

部分的问题的结果展示如下，第一个百分比为 Rerank 模型给出的相似分，第二个百分比为原始 Embedding 模型给出的相似度评分：

**1. 红楼梦中，贾宝玉和林黛玉第一次见面是在哪一年？**

bce-reranker-base_v1
![q1_bce](/img/in-post/rerank/q1_bce.png)

bge-reranker-large
![q1_bge](/img/in-post/rerank/q1_bge.png)

bge-reranker-v2-m3
![q1_bge_v2](/img/in-post/rerank/q1_bge_v2.png)

问题中大模型最后都没有正确回答，因为没有检索到正确的内容，但是 bge-reranker-v2-m3 确实给出了相当低的相似分，表明没有正确的内容，而其他模型在这种情况下都没体现出这个特征。


**13. 《红楼梦》中，贾宝玉和薛宝钗的婚姻是如何安排的？**

bce-reranker-base_v1
![q13_bce](/img/in-post/rerank/q13_bce.png)

bge-reranker-large
![q13_bge](/img/in-post/rerank/q13_bge.png)

bge-reranker-v2-m3
![q13_bge_v2](/img/in-post/rerank/q13_bge_v2.png)

这个比较明显，bce-reranker-base_v1 和 bge-reranker-large 都没能将正确的片段排序在最前面，而 bge-reranker-v2-m3 排序最靠前的片段才是符合预期的。而大模型的回答也体现出这种差异，回答明确提到了婚姻是由贾母等长辈决定的。

而且可以看到 bge-reranker-large 中由于上下文排序不合适导致大模型的幻觉更加严重了，连林黛玉意外远嫁都出来了。

#### 有无 Rerank 机制的比较

同样创建了一批问题进行测试，实际测试确实效果还是比较明显的，Rerank 后前面的回答相关性明显更强，比如：

无 Rerank 原始版本
![q3](/img/in-post/rerank/q3.png)

Rerank 版本
![q3_rerank](/img/in-post/rerank/q3_rerank.png)


## 总结
从本次《红楼梦》的实践来看，Rerank 机制的效果还是比较明显的，通过额外的精细重排，可以给大模型提供更精准的上下文，从而提升大模型知识库回答的质量。

从本次测试的情况来看，开源模型 bge-reranker-v2-m3 表现更胜一筹，当然实际的表现与数据集和问题都有很大关系，大家实际使用时可以基于自己的知识库与问题进行对应的测试。
