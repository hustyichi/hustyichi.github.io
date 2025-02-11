---
layout: post
title: "$200 劝退，无缘 Deep Research，可以试试 AutoSurvey"
subtitle:   "Because $200 is too expensive to use Deep Research, you can try AutoSurvey"
date:       2025-02-11 09:00:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - agent
    - llm
---

## 背景介绍

最近各种新的大模型辅助科研工具持续不断，在之前的文章中就介绍过 [NVIDIA 的结构化报告生成方案](https://zhuanlan.zhihu.com/p/19404394536)，最近 OpenAI 也退出了类似的产品，叫做 [Deep Research](https://openai.com/index/introducing-deep-research/)，可以根据需要进行深度的调研与信息整理，但是只有 Pro 用户才能享受到，$200 直接劝退，在调研后发现了 [AutoSurvey](https://github.com/AutoSurveys/AutoSurvey) 开源项目，相对之前的 NVIDIA 的方案更加完整，不仅包含了完整的报告生成，还能自动生成文献列表，实际进行二次开发优化，最终可以生成完整的调研报告，实际生成报告的效果类似如下所示：

![AutoSurvey](/img/in-post/autosurvey/preview.png)

下面就对 AutoSurvey 方案的详细介绍。

## 方案介绍

AutoSurvey 的整理流程如下所示：

![AutoSurvey](/img/in-post/autosurvey/overview.png)

主要的撰写流程包含下面三个步骤：

1. 大纲生成；
2. 子模块的撰写；
3. 整合与调优；

整体流程与 NVIDIA 的方案大致相同，但是 AutoSurvey 的方案更加完整，可以支持更加复杂的长报告的生成，并补全了 NVIDIA 方案中没有的文献列表生成。

#### 大纲生成

#### 子模块撰写

#### 整合与调优


