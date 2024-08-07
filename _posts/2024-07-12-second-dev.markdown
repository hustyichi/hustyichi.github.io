---
layout: post
title: "基于开源项目二次开发可行方案"
subtitle:   "The solution for secondary development based on open source projects"
date:       2024-07-12 09:00:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - rag
    - agent
---

## 背景介绍
一般情况下我们不需要进行开源项目的二次开发，因为开源项目往往会提供良好的封装，可以通过依赖包或 API 服务的形式引入项目中。如果开源项目存在一些问题，我们往往可以通过给开源项目提供 PR 来解决，这样就可以尽可能减少二次开发开源项目的问题。

但是某些情况下可能会需要基于开源项目开发自己的服务，需要一个相对长周期二次开发。而且因为定位不同，代码很难直接合并至开源项目，这种情况下就存在两种情况：

1. 不需要开源项目后续的更新，这种情况比较简单，直接拉取代码进行开发就好，当然要注意开源项目的版权信息；
2. 需要开源项目后续的代码更新，这种情况就会更加麻烦，因为二次开发与开源项目的持续迭代很容易造成冲突，而且随着时间的推移这种冲突会越来越多，最终导致无法同步；

本篇文章就来讨论下情况 2 下可能的一些处理方案，希望对大家有所帮助。

## 建议方案

#### 上策：插件机制

先调研项目是否支持插件机制，基于官方提供的插件机制是解决这种问题的最佳方案。通过插件机制，可以避免直接修改开源项目的代码。这种情况下就可以避免后续的冲突问题。以之前调研过的大模型应用开发基础框架 [Dify](https://zhuanlan.zhihu.com/p/706381113) 为例：

如果需要扩充 Dify 现有的能力，满足更多业务场景，建议方案就是通过 Dify 提供的自定义工具：

![custom-tool](/img/in-post/second-dev/custom_tool.png)

通过实现一个自定义的基础服务，并通过 API 形式接入 Dify 中。只要 Dify 的自定义工具机制没有太多的变化，预期后续可以持续稳定使用，这个是建议的上策。

#### 中策：模块化开发

如果你希望实现的插件能力官方没有提供，而确实又有相关需求，那么可以考虑模块化的开发方案。

将你需要实现的功能封装为一个独立的模块，通过依赖包或 submodule 的形式引入项目中。

- 依赖包形式：将所需要实现的功能进行良好设计，作为一个 pypi 包引入项目中，这样可以尽可能避免代码层面的直接耦合，后续的更新会相对容易；
- submodule 形式：通过 git 提供的 [Submodule](https://git-scm.com/book/en/v2/Git-Tools-Submodules) 机制，将代码引入现有项目，这样新增的功能模块可以独立开发，根据需要定期更新至现有项目中，代码冲突的可能性比较低；

唯一可能造成冲突的就是对现有功能的模块化的调用部分，这部分的代码量较少，处理也会更容易一些。

#### 下策：直接二次开发

如果必须要对原有项目进行非模块化的功能改造，那么维护的成本就会大不少，这种情况下建议遵循下面的方案尽可能减少冲突：

1. 基于 git rebase 进行改造，这样可以方便区分原有项目的代码与新增的二次开发的代码，持续的冲突处理会更方便一些，要不然持续的 merge 冲突很快就让代码无法区分；
2. 定期同步和小步迭代，这样每次冲突更少，避免冲突过多无法处理的问题；
3. 代码风格与原有项目保持一致，要不然修改后的自动格式化会导致所有文件都被修改了，后续冲突就无法处理了；

即使如此，随着持续的开发，修改的内容持续增加，依旧会导致维护越来越困难，迟早要面临放弃同步开源项目的后续更新的问题。

## 总结

本文主要讨论了基于开源项目二次开发的一些方案，分为上中下三策，如果不是非做不可的情况，建议不要选择下策，维护的成本会持续叠加，最终无法维护。
