---
layout: post
title: "来自工业界的开源知识库 RAG 项目结构化文件解析方案比较"
subtitle:   "Comparison of structured file parsing solutions from open source RAG projects in the industry"
date:       2024-07-31 12:00:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - rag
    - agent
    - dify
---

## 背景介绍
在过去实践 RAG 的过程中，深刻体会到 [RAGFlow](https://github.com/infiniflow/ragflow) 提出的 `"Quality in, quality out"`, 只有高质量的文件处理才能获得良好的 RAG 效果。

RAG 服务进行文件预处理时，由于 Embedding 模型的长度要求，往往需要将解析后的文件进行切片。原始的 RAG 就是直接按照固定长度对文件进行切分，导致最终检索到的内容都是碎片化的，效果往往不佳。因此后续的改进期望能按照文件的结构进行切分，保证分块文档信息的完整性，这就是所谓的 `"structure-aware" chunker`。

但是并非所有的文件都相对容易获取到结构化的信息，比如 pdf 文件获取获取到结构化的信息就比较困难，此时常规的方案就是将 pdf 等难以处理的文档转换为相对容易获取结构的文档，基于转换后的文档进行结构化切分。一般会转换为 html 或 markdown 格式。

本文就以相对基础的 html 文件为例，比较目前 RAG 项目中的结构化解析文件的能力，看看目前 RAG 项目处理文件的基本功如何。主要关注的各个开源项目的 html 解析与切片的策略。

## 开源项目比较

#### RAGFlow
首先来看以文件处理见长的 [RAGFlow](https://github.com/infiniflow/ragflow), 之前的 [RAGFlow 源码解析](https://zhuanlan.zhihu.com/p/697902937) 中深入介绍过 RAGFlow 的 pdf 文件解析，实现比较复杂。接下来查看 RAGFlow 对 html 的处理方案。

实际的 html 解析在 `deepdoc/parser/html_parser.py` 中，简化后如下所示：

```python
import html_text
import readability

html_doc = readability.Document(txt)
title = html_doc.title()
content = html_text.extract_text(html_doc.summary(html_partial=True))
txt = f'{title}\n{content}'
sections = txt.split("\n")
return sections
```

从目前来看，RAGFlow 是基于 [html-text](https://github.com/zytedata/html-text) 从原始的 html 文件中直接提取出文件内容，之后直接基于换行符进行内容切分。从实际测试来看，基本上是按照 html 的各个基础元素进行了切分。预期后续可能会根据 token 数量进行组合。考虑内容中的换行出现的频率较高，可能切分的点不能保证特别准确。


