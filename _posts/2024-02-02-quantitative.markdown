---
layout: post
title: '从 0 手撸一个量化交易回测平台'
subtitle:   "Create a quantitative trading backtesting platform from scratch"
date:       2024-02-02 09:00:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - Machine learning
    - Quantitative trading
    - LLM
---

## 背景介绍
程序员掌握技能最快的手段就是撸起袖子开始干，之前一阵子对大模型辅助编程与项目生成做过一些比较多的调研，但是一直是试用，没能真正体现出大模型辅助编程的威力。因此本次从新领域入手，借助大模型的力量，从 0 手撸一个量化交易回测平台，从这个过程看看现阶段程序员是否会被大模型替代，以及程序员如何利用大模型提升编程体验。

## 效果展示
最终花费 2 周的休息时间，耗时 116 个番茄钟，从 0 撸出一个量化交易回测的服务。当前支持了 Windows，Linux，Mac 平台，本地一键启动，一秒化身量化交易回测服务器。最终效果如下：

接入了实时和离线的数据源，实时数据源来自于 yahoo finance，内置了 KDJ, MACD, 突破，动量与海龟策略，支持策略参数动态调节：

![operations](/img/in-post/quantitative/operations.png)

回测的效果支持了交易回测 K 线图，账户价值变动与年化收益率曲线，最重要的还支持了详细交易记录展示与导出：

![plot](/img/in-post/quantitative/plot.png)

![plot2](/img/in-post/quantitative/plot2.png)

![details](/img/in-post/quantitative/details.png)

最后还给他配上一个亮闪闪 icon，支持一键启动：

![icon](/img/in-post/quantitative/icon.png)

当然时间精力有限，而且没有设计师帮助，页面目前还基本上工程师审美阶段，但是流程基本上完备了，自我感觉算是一个合格的 MVP 产品。

## 大模型工具
在本次实践过程中，主要尝试了下面的一些大模型工具：

- [GPT Engineer](https://gpt-engineer.readthedocs.io/en/latest/index.html)， 一款类似之前大热的 [Auto GPT](https://docs.agpt.co/) 的项目代码生成工具，可以根据需求描述直接生成完整的项目代码；
- [chatALL](https://github.com/sunner/ChatALL)，一个多 GPT 同时响应问题的工具，可以同时对接多个 GPT，提升 GPT 查询效率；
- [Github copilot](https://github.com/features/copilot)，微软出的编程协助工具，直接在编程 IDE 中帮助生成代码，提升编程效率；

## 实践过程

#### 项目生成尝试
在之前 [GPT Engineer 实践与源码解析](https://zhuanlan.zhihu.com/p/667865664) 的文章中提出过一个想法，如果可以直接根据需求描述直接生成主体的项目代码，然后基于编程辅助工具 Github copilot 进行高效的二次开发，那么程序员开发的效率应该可以得到极大的改善。所以本次先试试这个捷径。

实际测试使用的 prompt 如下所示：

```
请实现一个量化交易回测 web 服务，支持接入离线和实时数据，并支持多种量化策略，并支持可视化展示回测效果
```

最终实际生成了一个类似基于 Flask 的 Web 服务:

![gpt-engineer](/img/in-post/quantitative/gpt-engineer.png)

而且对应的模块也帮忙做了分拆，其中 `data_manager.py` 用于做数据管理，`strategy.py` 用于做策略管理，`visualization.py` 用于可视化，甚至包含相关依赖文件和启动脚本。所以一切已经就绪？

远没那么简单，在这个相对复杂的需求前就可以看到 GPT-Engineer 的不足之处，可以帮助搭建一个简单的代码框架，但是没有任何的实现细节。生成的内容对于不知道怎么搭建一个 Flask 的 Web 服务的小白可能有一些作用，但是对于重点在如何做一个量化交易回测服务没有任何的协助。

后续又补充了必要的技术选型的细节之后让 GPT-Engineer 进行生成，依旧表现不佳，依旧只能生成偏框架流程的代码，看起来 GPT-Engineer 靠单纯组织 prompt 希望实现一个完备的项目可行性不强。

#### 技术选型

依赖大模型直接生成项目不可行了，接下来就得考虑建立在那些库的基础上实现完整的项目，合适的技术选型可以尽可能减少开发工作量。接下来就需要拆分量化交易平台的需求，至少包含哪些部分：

1. 数据来源接入，考虑支持离线和实时在线数据，实时数据接入的来源；
2. 量化回测库的选择；
3. 可视化交互支持；

这个时候就可以用上 chatALL，同时对接上多个 GPT，避免单个大模型知识来源有限，胡说八道的问题。通过选择多个 GPT 答案中的交集，那么就大概率是相对靠谱的了。在 chatALL 中我一般选择的模型是 GPT3.5, Bard 和 Bing，后续 Bing 使用报错切换为 Claude，这 3 个是在免费版本中最强的。在多个 GPT 都无法解决的复杂问题，则会使用付费的 GPT 4 的 API。这个时候就体现出 chatALL 的优势所在，在大模型之间无缝切换，体验极其丝滑。

对于完全不熟悉的情况下做技术选型，我实际尝试时会将其拆分为两步：

1. 描述业务需求，让大模型给出所有的备选项，多个 GPT 的结果综合下来，根据高频结果大概就能知道合适的范围，人工初筛排除掉完全不靠谱的；
2. 描述业务需求与自己关注的点，提供备选库列表，让 GPT 描述不同备选方案的优劣点，适合什么场景，使用中可能会存在什么风险，让其推荐最适合的，结合 GPT 的推荐与适合场景选择最佳方案，同时保留备选方案；





