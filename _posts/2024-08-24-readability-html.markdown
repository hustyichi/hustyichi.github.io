---
layout: post
title: "深入源码，洞察迭代 14 年的高质量 html 提取方案 "
subtitle:   "Go deep into the source code and gain insight into the high-quality HTML extraction solution that has been iterated for 14 years"
date:       2024-08-24 21:30:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - rag
    - html
---

## 背景介绍

在大模型时代，开源项目的生命周期被加速了，往往迭代速度很快，但是热门项目也容易突然就无疾而终了。最近看到一款历经 14 年的开源 html 内容提取项目 [python-readability](https://github.com/buriy/python-readability)，从最早建立到目前，已经迭代了 14 年。

![time](/img/in-post/readability/time.png)

本文是在实际项目中使用 python-readability 之后，发现一些异常 case，因此深入源码了解其中的技术细节，因此在本文中对这款跨越 14 年的开源项目的技术细节进行了解读。

## 项目体验与概述

在之前的 [RAG 项目结构化文件解析方案比较](https://zhuanlan.zhihu.com/p/712193089) 文章中对常见的 html 解析方案进行了比较，当时实际测试发现 html_text + python-readability 的技术方案解析效果是最好的，解析的内容组织形式是最符合人阅读形式的，并对大量无用内容进行了清理。

举例来看，使用如下所示的医学论文进行测试时，框架准确从正文部分开始内容提取，过滤了上面的作者介绍信息。

![start](/img/in-post/readability/start.png)

而在论文的结尾部分的解析，框架准确剔除了参考文献等低价值内容。

![end](/img/in-post/readability/end.png)

同时网站左侧的导航栏和与右侧的一些与内容无关的信息都被一一去除，真正所见即所得。类似的能力如果使用大模型可能相对容易理解，但是完全不借助任何人工智能的手段，如何从复杂的 html 结构中准确提取用户关注的内容，而且适配于完全不同的前端 html 结构呢。

核心的内容的提取与无关内容的清洗基本都是在 python-readability 中完成的，而 html_text 则完成了 html 元素到符合阅读形式的文本转换。本文主要就是对 python-readability 原理的详细解读，后续有空可以考虑解析下 html_text 的实现方案。

## 原理概述

此项目的提取核心内容的关键点在于：html 文件中主要正文内容是聚积在一起的，而那些与正文无关的内容往往散落在其他位置，比如上面例子中的作者介绍， 左侧的导航等。

因此思路就是找到一个最佳正文内容块（`best_candidate`），之后向四周进行扩散找到所有的正文内容块，而没有被扩散到的内容都会被丢弃。对于上面的文章，实际的结果如下所示：

![example](/img/in-post/readability/example.png)

作者简介就是因为无法被扩散到，从而被过滤掉。

## 实现细节

从上面的原理来看，主要需要关注两个核心细节：

1. 如何找到最佳正文内容块，保证扩散的起点的准确性；
2. 如何进行内容块的扩散，从而保证尽可能的覆盖完整的正文内容，同时尽可能避免覆盖无用内容；

#### 最佳正文内容块的判断

最佳正文内容块的判断是基于评分机制，对所有的正文内容块进行评分，之后找到分数最高的一块作为最佳正文内容块。

选出最佳正文内容块的过程就是一个简单的排序：

```python
sorted_candidates = sorted(
    candidates.values(), key=lambda x: x["content_score"], reverse=True
)

best_candidate = sorted_candidates[0]
```

因此核心就在于如何进行内容块的评分，其评分主要由两部分组成：

1. 内容块自身获得的得分，主要基于其 html 元素中的 `class` 和 `tag` 评分；
2. 内容块包含的子段落得分，包含的子段落的文本段落越多，长度越长，得分就越高。

在获得得分后会根据链接密度进行评分缩放，链接过多的块得分会降低。

**内容块自身得分**

目前主要基于 `class` 和 `tag` 评分，根据匹配的正则表达式确定是否应该加分还是减分：

```python
def class_weight(self, e):
    weight = 0
    # 基于 class 进行匹配，根据匹配的积极，消极正则表达式确定分数

    for feature in [e.get("class", None), e.get("id", None)]:
        if feature:
            if REGEXES["negativeRe"].search(feature):
                weight -= 25

            if REGEXES["positiveRe"].search(feature):
                weight += 25

            if self.positive_keywords and self.positive_keywords.search(feature):
                weight += 25

            if self.negative_keywords and self.negative_keywords.search(feature):
                weight -= 25

    # 基于 tag 进行匹配，根据积极，消极关键词进行加分，减分

    if self.positive_keywords and self.positive_keywords.match("tag-" + e.tag):
        weight += 25

    if self.negative_keywords and self.negative_keywords.match("tag-" + e.tag):
        weight -= 25

    return weight
```

可以查看对应的积极，消极正则表达式，具体如下所示：

```python
# 积极正则表达式

"positiveRe": re.compile(
    r"article|body|content|entry|hentry|main|page|pagination|post|text|blog|story",
    re.I,
),

# 消极正则表达式

"negativeRe": re.compile(
    r"combx|comment|com-|contact|foot|footer|footnote|masthead|media|meta|outbrain|promo|related|scroll|shoutbox|sidebar|sponsor|shopping|tags|tool|widget",
    re.I,
),
```

可以看到会将可能包含文字内容的字段加分，将可能与内容无关的内容进行减分，从而保证内容块的得分高，并过滤掉不相关的内容。

**子段落得分**

子段落会根据段落长度以及字数计算得分，子段落的得分会传递给父节点，并折半传递给祖父节点，这种设计可以避免更高级别的节点的得分过高。具体实现如下所示：

```python
for elem in self.tags(self._html(), "p", "pre", "td"):
    parent_node = elem.getparent()
    grand_parent_node = parent_node.getparent()

    inner_text = clean(elem.text_content() or "")
    inner_text_len = len(inner_text)

    # 根据父节点和祖父节点自身的 class 和 tag 评分

    if parent_node not in candidates:
        candidates[parent_node] = self.score_node(parent_node)
        ordered.append(parent_node)

    if grand_parent_node is not None and grand_parent_node not in candidates:
        candidates[grand_parent_node] = self.score_node(grand_parent_node)
        ordered.append(grand_parent_node)

    # 子段落的分数计算，根据 `，` 划分得到的段落 + 字数 / 100 得到的得分叠加

    content_score = 1
    content_score += len(inner_text.split(","))
    content_score += min((inner_text_len / 100), 3)

    # 子段落的得分传递给父节点，折半传递给祖父节点

    candidates[parent_node]["content_score"] += content_score
    if grand_parent_node is not None:
        candidates[grand_parent_node]["content_score"] += content_score / 2.0
```

可以看到子段落会按照 `,` 划分的段落数与字数得到合计得分，并将子段落的得分提供给父节点和祖父节点。这样就可以客观评价内容块是否包含大量的文字内容，保证最终选出的都是信息密度高的内容块。

**链接密度缩放**

如果文字中都是跳转链接，那么说明这个内容块的信息可能教少，而且更可能是外部广告信息，因此需要进行必要的缩放。因此实际会根据链接密度进行得分缩放：

```python
for elem in ordered:
    candidate = candidates[elem]
    ld = self.get_link_density(elem)
    # 根据链接密度缩放得分

    candidate["content_score"] *= 1 - ld
```

可以看到链接密度为 0.2， 最终的得分则为 `原始得分 * 0.8`。链接密度为 0.9, 则最终的得分为 `原始得分 * 0.1`。

#### 内容块的扩散

最佳正文内容块是信息密度最高的块，一般情况同级的块也会包含大量有价值正文内容，因此需要进行扩散，从而保证尽可能的覆盖完整的正文内容。

目前的实现方案就是遍历同级节点，将内容块符合要求的加入到最终的输出中：

```python
# 得分阈值，为最佳正文内容块的 20%

sibling_score_threshold = max([10, best_candidate["content_score"] * 0.2])

best_elem = best_candidate["elem"]
parent = best_elem.getparent()
siblings = parent.getchildren() if parent is not None else [best_elem]

# 遍历所有同级节点

for sibling in siblings:
    append = False
    if sibling is best_elem:
        append = True

    sibling_key = sibling
    # 根据阈值进行过滤, 符合阈值的不过滤

    if (
        sibling_key in candidates
        and candidates[sibling_key]["content_score"] >= sibling_score_threshold
    ):
        append = True

    if sibling.tag == "p":
        link_density = self.get_link_density(sibling)
        node_content = sibling.text or ""
        node_length = len(node_content)

        # 文字内容较多，且链接密度较低，则加入

        if node_length > 80 and link_density < 0.25:
            append = True
        # 文字内容较少，但是无链接，则加入

        elif (
            node_length <= 80
            and link_density == 0
            and re.search(r"\.( |$)", node_content)
        ):
            append = True

    if append:
        if html_partial:
            output.append(sibling)
        else:
            output.getchildren()[0].getchildren()[0].append(sibling)
```

可以看到扩散时符合要求的内容块为：

1. 得分超过最佳正文内容块的 20%；
2. 节点为 p 标签，且文字内容较多，且链接密度较低；

## 总结
python-readability 是一个高质量的 html 内容提取库，可以支持大部分的 html 页面的内容提取，并过滤掉与核心正文无关的干扰内容，从而实现高质量的内容提取。

从实际测试情况来看，虽然不能保证 100% 的准确提取，但是适用于大部分 html 内容提取的场景，如果有类似需求可以进行尝试。
