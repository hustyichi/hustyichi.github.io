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

RAG 的第一步是对文件进行解析，由于 Embedding 和 LLM 模型的长度限制，往往需要将解析后的文件进行切片。原始的 RAG 就是直接按照固定长度对文件进行切分，导致最终检索到的内容都是碎片化的，效果往往不佳。因此后续的改进期望能按照文件的结构进行切分，保证分块信息的完整性，这就是所谓的 `"structure-aware" chunker`。

但是并非所有的文件都容易获取到结构信息，比如 pdf 文件获取结构化的信息就比较困难，一个可选的方案就是将 pdf 等难以处理的文档转换为相对容易获取结构的格式，基于转换后的文档进行结构化解析和切分。目前一般会选择转换为 html 或 markdown 格式。

本文就以相对基础的 html 文件为例，比较目前热门 RAG 项目中的结构化解析文件的能力，看看目前 RAG 项目处理文件的基本功如何。

## 技术方案比较

在梳理了现有的 RAG 开源项目之后，发现使用的技术存在不少相似之处，因此先总结使用的技术方案，后面在与开源项目意义对应。

实际测试时使用的是如下所示的一个 html 片段：

![test](/img/in-post/html-parser/test.png)

#### 基于 unstructured 解析方案

[unstructured](https://unstructured.io/) 是一个目前热门的开源非结构化文件解析方案，专门为 RAG 场景进行设计，支持了文件的解析，切片等多种场景。

目前基于 unstructured 的方案存在两种用法：

1. 使用 unstructured 提取出完整的文本内容，之后将完整的内容提供给 RAG 的 Splitter 环节进行切片，这种方案没办法做任何结构化的优化，因为结构化信息在解析环节已经全部丢弃，后续只能根据符号和文本内容进行分片；
2. 使用 unstructured 拆分出 html 元素，提取各个元素的内容并进行必要的拼接，之后再提交给 Splitter 环节进行处理，这种方式可以保留部分文件结构化信息；

下面简单实现 unstructed 的文档解析拆分如下所示：

```python
from langchain_community.document_loaders import UnstructuredHTMLLoader

loader = UnstructuredHTMLLoader(
    "./xxx.html",
    mode="elements",
    strategy="fast",
)
docs = loader.load()
for doc in docs:
    print("-------------->")
    print(doc.page_content)

```

查看 unstructured 拆分的结果如下所示：

![unstructed](/img/in-post/html-parser/unstructed.png)

可以看到 unstructed 的切分是按照细粒度的元素切分的，导致大量人工看起来不太合适的内容也被切分开了，后续使用的效果可能不是特别理想。但是可以保证基础元素内部不会出现断句的情况。

下面是基于 unstructed 提供的 `chunk_by_title()` 方法进行合并后看到的分片效果：

![unstructed](/img/in-post/html-parser/unstructed_combine.png)

可以看到合并后分片的新的分片可能在在视觉上单行中间，效果表现一般，但是预期比直接忽略结构分片可能会略好一些。


#### 基于 html_text 解析方案

[html_text](https://github.com/zytedata/html-text) 是一个相对小众的 html 解析开源项目，同样用于 html 内容提取。html_text 提取了内容后，会在接近可视化内容的部分增加换行符，后续可以基于换行符进行切分。

目前开源项目是基于 html_text + readability 实现的，简化后如下所示：

```python
import chardet
import html_text
import readability

# 获取文件编码

def get_encoding(file):
    with open(file, "rb") as f:
        tmp = chardet.detect(f.read())
        return tmp["encoding"]


def get_data(file_path):
    with open(file_path, "r", encoding=get_encoding(file_path)) as f:
        txt = f.read()
        html_doc = readability.Document(txt)
        # 基于 html_text 提取内容

        content = html_text.extract_text(html_doc.summary(html_partial=True))
        sections = content.split("\n")
        return sections

ret = get_data(file_path)
for d in ret:
    print("---------->")
    print(d)

```

实际基于 html_text 解析得到的内容如下所示：

![html_text](/img/in-post/html-parser/html_text.png)

可以看到实际的分片位置更符合可视化页面的效果，不会将最原始的元素拆分开。实际测试下来，`<span>` 这种不会产生换行的元素不会被切分，`<p>` 这种产生换行的元素会切分为不同块。相对 unstructed 而言更符合视觉效果一些

#### 基于 BeautifulSoup 解析方案

BeautifulSoup 是 python 生态中比较常用的 html 解析方案，所以部分 RAG 项目中会基于 BeautifulSoup 实现，这种情况一般会基于 `get_text()` 将文本内容全部获取出来，放弃文件的结构信息。实现如下所示：

```python
from bs4 import BeautifulSoup

with open(file_path, "rb") as fp:
    soup = BeautifulSoup(fp, "html.parser")
    text = soup.get_text()
    text = text.strip() if text else ""
    print(text)
```

#### 基于 trafilatura 解析方案

[Trafilatura](https://github.com/adbar/trafilatura) 是一个类似 BeautifulSoup 的 html 解析方案，从官方文档描述来看，Trafilatura 解析的速度和质量都比较高。

使用 Trafilatura 解析后，直接提取了文档的内容，放弃了文件的结构，其实现也比较简单：

```python
import trafilatura

with open(file_path, "rb") as file:
    html_content = file.read()

    # 使用 Trafilatura 提取内容

    result = trafilatura.extract(html_content)
```

## 开源项目方案比较

之前在文章 [来自工业界的开源知识库 RAG 项目最全细节对比](https://zhuanlan.zhihu.com/p/707842657) 中对常规的开源项目进行了详细对比，本文就对其中的一些热门开源项目的 html 解析方案进行比较：

| 项目 | html 技术方案 |
| --- | --- |
| [RAGFlow](https://github.com/infiniflow/ragflow) | html_text 解析 |
| [Langchain-Chatchat](https://github.com/chatchat-space/Langchain-Chatchat) | unstructured 解析 |
| [dify](https://github.com/langgenius/dify) | BeautifulSoup 解析 |
| [GoMate](https://github.com/gomate-community/GoMate) | html_text 解析 |
| [haystack](https://github.com/deepset-ai/haystack) | trafilatura 解析 |
| [QAnything](https://github.com/netease-youdao/QAnything) | 暂未支持，但是很多格式都是基于 unstructured 解析 |
| [langchain](https://github.com/langchain-ai/langchain) | 支持 unstructured 和 BeautifulSoup 解析，同时也支持 [按照指定元素进行切分](https://python.langchain.com/v0.2/docs/how_to/HTML_header_metadata_splitter/) |

从目前来看，基于 unstructured 的方案是最多的，原因是 unstructured 作为开源非结构化解析库，对不同的格式都能提供一个还不错的支持。但是从上面的测试来看，html_text 在 html 的分片支持上，看起来可以提供一个更符合人类可视化效果的切分。

## 总结

本文对现有开源项目的结构化文件解析的方案进行了比较，从目前来看，主要分为两种方案：

1. 放弃文档结构化信息，直接提取内容，后续依赖 splitter 提供一个语义上合适的分片；
2. 基于文档结构化信息进行文档拆分，接下来根据长度要求进行必要的组合，尽可能保证在元素交界处切分；

预期方案 2 可以一定程度上减少 RAG 中段落中间被切断导致的效果不佳的问题，但是如果特定段落确实过长，超过 Embedding 模型的合适长度，依旧需要在段落中选择一个合适的位置进行分片。结构化解析分片无法完全解决段落分片问题，但是可以提供一个还不错的分片选项，一定程度缓解分片不合适的问题。

