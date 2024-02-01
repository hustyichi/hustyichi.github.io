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





