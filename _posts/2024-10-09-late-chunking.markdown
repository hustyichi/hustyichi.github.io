---
layout: post
title: "RAG 分块长距离信息缺失，Late Chunking 值得试试"
subtitle:   "Long distance contextual information is missing in RAG, you can try Late Chunking"
date:       2024-10-09 09:00:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - RAG
    - agent
    - llm
---

## 背景介绍
实际线上部署使用过 RAG （Retrieval Augmented Generation）服务的研发同学或多或少都会发现，按照常规的 RAG 方案进行文本切片并向量化之后，部分文本切片可能难以检索命中。这个往往是因为对应的分片缺失可供检索的信息。

以医疗领域的病例检索为例，单个病例文件中病情描述（现病史）一般在文档最上面，相关的诊断结论在文档最下面，而病例的诊断结论中一般没有任何相关的病例描述。按照常规的 RAG 方式进行分片后，病情描述与诊断结论完全分离了。如果基于病情描述去检索诊断结论，大概率会检索不到。

类似的问题很常规，反映的都是 RAG 方案的盲人摸象问题。因为文本分块向量化时，只基于当前分块的上下文信息，无法包含当前分块之外的信息。为了解决这个问题，Jina 提出了 [Late Chunking](https://arxiv.org/pdf/2409.04701) 方案，通过延迟分片来解决信分片息缺失问题。


## 常规 RAG 向量化流程

在开始介绍 Late Chunking 方案前，先简单介绍下文本向量化的流程，方便后面理解 Late Chunking 中的延迟到底是什么含义。常规 RAG 方案的完整流程可以参考之前的文章 [从开发到部署，搭建离线私有大模型知识库](https://zhuanlan.zhihu.com/p/689947142)

文本向量化一般包含下面的步骤：

1. 获取文本进行 tokenizer 切分，对每个切分后的 token 执行 embedding 操作，得到每个 token 的词向量；
2. 基于文本中包含的所有 token 的词向量进行 pooling 操作，得到文本的向量表示，一个常规的 pooling 方案就是直接将词向量的均值作为文本的向量。

基于上面的知识，常规 RAG 方案的向量化流程可以被简化为三个步骤：

1. 文本分片；
2. 分片 token 的向量化；
3. 基于 token 向量的 pooling 操作；

对应的流程如下所示：

![base](/img/in-post/late-chunking/base.png)

从这个流程也能比较容易理解盲人摸象的问题，因为向量化阶段接触到就是有限的文本片段，因此很难具备完整的上下文信息。

## Late Chunking 方案

为了解决上面的信息缺失问题，Late Chunking 方案提出了延迟分片的思路，先执行文本 token 的向量化，之后再执行分片。这样 token 向量化时就可以获取更全面的上下文信息，对应的向量化步骤如下所示：

1. token 的向量化；
2. 文本分片；
3. 基于 token 向量的 pooling 操作；

对应的流程与常规的 RAG 流程的对比如下所示：

![method](/img/in-post/late-chunking/method.png)

基本不需要引入新的功能，通过简单的流程调整就可以给文本向量提供更丰富的上下文信息。

当然熟悉向量化的研发同学可能马上就注意到了，上面的流程中 token 向量化时，需要处理超长的文本，因此需要向量化的模型输入上下文的长度较长。而向量化模型支持的输入上下文的长度往往很短，因为 Transformer 架构带来的限制，往往支持的 token 数量为 512。

因此 Late Chunking 的实现方案中，引入了一个支持长上下文的向量化模型，Jina 官方推荐的是 [jina-embeddings-v2-base-en](https://huggingface.co/jinaai/jina-embeddings-v2-base-en)，此模型支持的输入上下文的长度为 8192 。

如果你的文本超过 8192 怎么办？当然还是要先分片为 8192 之内，所以可以预期此方案只能目前长上下文的模型可以提供的上下文信息。因此只是一定层度缓解了原有的单个小分片信息缺失的问题。

## Late Chunking 方案实践

Jina 提供了 Late Chunking 的简单实现方案，具体可以参考 [Late Chunking in Long Context Embedding Models](https://colab.research.google.com/drive/15vNZb6AsU7byjYoaEtXuNu567JWNzXOz#scrollTo=1380abf7acde9517)。下面对实现方案进行简单介绍：

#### 分片方案实现

在原始的 RAG 方案中，分片时值只需要保存文本即可，在 Late Chunking 方案中，需要保存文本和对应的 token 位置信息，因为最终执行 pooling 操作时，需要确定每个分片对应的 token 向量。具体如下所示：

```python
from transformers import AutoModel
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained('jinaai/jina-embeddings-v2-base-en', trust_remote_code=True)
model = AutoModel.from_pretrained('jinaai/jina-embeddings-v2-base-en', trust_remote_code=True)

# 文本分片方法，返回分片后的文本以及分片对应的 token 位置信息

def chunk_by_sentences(input_text: str, tokenizer: callable):
    inputs = tokenizer(input_text, return_tensors='pt', return_offsets_mapping=True)
    # 基于 . 字符进行分片，获取字符对应的 token, 方便后续进行确定分片点

    punctuation_mark_id = tokenizer.convert_tokens_to_ids('.')

    sep_id = tokenizer.convert_tokens_to_ids('[SEP]')
    token_offsets = inputs['offset_mapping'][0]
    token_ids = inputs['input_ids'][0]
    # 获取分片点位置

    chunk_positions = [
        (i, int(start + 1))
        for i, (token_id, (start, end)) in enumerate(zip(token_ids, token_offsets))
        if token_id == punctuation_mark_id
        and (
            token_offsets[i + 1][0] - token_offsets[i][1] > 0
            or token_ids[i + 1] == sep_id
        )
    ]

    # 基于位置信息生成分片文本以及对应的 token 位置信息

    chunks = [
        input_text[x[1] : y[1]]
        for x, y in zip([(1, 0)] + chunk_positions[:-1], chunk_positions)
    ]
    span_annotations = [
        (x[0], y[0]) for (x, y) in zip([(1, 0)] + chunk_positions[:-1], chunk_positions)
    ]
    return chunks, span_annotations

input_text = "Berlin is the capital and largest city of Germany, both by area and by population. Its more than 3.85 million inhabitants make it the European Union's most populous city, as measured by population within city limits. The city is also one of the states of Germany, and is the third smallest state in the country in terms of area."

chunks, span_annotations = chunk_by_sentences(input_text, tokenizer)
print('Chunks:\n- "' + '"\n- "'.join(chunks) + '"')
```

#### Late Chunking 向量化

Late Chunking 向量化会对原始长文本进行 token 向量化，之后根据分片的 token 位置信息，确定每个分片包含的 token，从而选择分片对应的 token 向量执行 pooling, 实际实现的是均值 pooling，具体实现如下所示：

```python
def late_chunking(
    model_output: 'BatchEncoding', span_annotation: list, max_length=None
):
    token_embeddings = model_output[0]
    outputs = []
    for embeddings, annotations in zip(token_embeddings, span_annotation):
        if (
            max_length is not None
        ):
            annotations = [
                (start, min(end, max_length - 1))
                for (start, end) in annotations
                if start < (max_length - 1)
            ]
        # 选择分片对应的 token 向量列表，执行 mean pooling 操作

        pooled_embeddings = [
            embeddings[start:end].sum(dim=0) / (end - start)
            for start, end in annotations
            if (end - start) >= 1
        ]
        pooled_embeddings = [
            embedding.detach().cpu().numpy() for embedding in pooled_embeddings
        ]
        outputs.append(pooled_embeddings)

    return outputs

#  对原始长文本转换为 token 列表

inputs = tokenizer(input_text, return_tensors='pt')

# 对 token 进行向量化

model_output = model(**inputs)

# late chunking

embeddings = late_chunking(model_output, [span_annotations])[0]
```

可以看到 late chunking 的主要流程就是选择分片对应的 token 向量列表，执行 pooling 即可，实现相对简单。

#### 测试情况

Jina Demo 的测试情况如下所示：

![vs](/img/in-post/late-chunking/vs.png)

实际测试看起来效果还是比较明显的，应用了 Late Chunking 之后，原始分片与 `Berlin` 的相似度就有了明显提升。一定程度上实现了类似指代消解的问效果。

从目前测试与上面的实现方案来看，Late Chunking 方案在当前分片存在信息缺失，一定程度的范围（不超过长上下文向量模型支持的范围）内存在相关的情况会有明显改善。

## 总结

本文介绍了 Late Chunking 方案，主要针对常规 RAG 方案单个分片内信息缺失导致难以检索的问题进行了优化，整体的方案依赖长文本的向量化模型。预期一定程度上可以缓解信息缺失导致难以检索的问题，而且实现方案极其简单，值得尝试一下。
