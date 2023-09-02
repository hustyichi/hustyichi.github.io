---
layout: post
title: 'FATE Eggroll 源码解析'
subtitle:   "FATE Eggroll source code analysis"
date:       2023-09-04 21:01:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - Federated learning
    - FATE
    - Eggroll
    - Rollsite
---

## 背景介绍
在[之前的文章](https://hustyichi.github.io/2023/03/08/fate-flow-loop/) 中对 FATE 的调度系统 FATE-Flow 从源码角度进行了介绍，FATE 的可视化 [FATE-Board](https://hustyichi.github.io/2023/08/23/fate-board/) 之前也介绍过了，对于 FATE 底层使用的数据传输机制 Eggroll 一直没有过多介绍，[官方文档](https://fate.readthedocs.io/en/latest/zh/architecture/#overview) 上只有 Eggroll 部署的内容，网络上基本搜不到 FATE 底层数据传输 Eggroll 原理分析的内容，而 [Eggroll 工程](https://github.com/FederatedAI/eggroll) 使用的 Python + Java + Scala 的开发语言，导致想了解原理的成本也会更高一些，估计这也是劝退部分人的一些原因。最近刚好有空，就对 Eggroll 源码进行了阅读，整理相关内容在这边，方便大家入门。

注意，本文针对是目前最近发布的 v2.5.1，后续的版本可能会有所不同，但是原理上应该不会有太大的变动。

## 架构介绍
在开始介绍具体的细节前，可以先看看 FATE 建立在 Eggroll 上的架构：

![eggroll](/img/in-post/eggroll/eggroll.png)

通过上面的架构图可以看到，FATE 部署在 Eggroll 上时，是使用 Rollsite 进行数据传输的，Rollsite 就是 Eggroll 很重要的一部分，主要承担各个参与方进行通信的能力，而这篇文章主要就是分析 Rollsite 的实现细节。

FATE 作为一个联邦学习框架，需要承载多方进行机器学习模型的训练，部分模型的大小可能会比较大，而 FATE 也是可以稳定传输的，这篇文章就会揭开其中的秘密。

## Rollsite 原理介绍
在进行源码层面的分析前，先对 Rollsite 的原理进行概述，这样方便部分不想深入到源码层面，但是又对 FATE 的数据传输机制感兴趣的研发同学们。

Rollsite 在数据传输时使用的推拉模式，发送方先调用 `push()` 将数据推送出去，接收方可以异步调用 `pull()` 拉取数据。

#### 数据推送
数据推送的一般流程如下所示：

1. 使用 pickle 对原始数据进行序列化，生成对应的字节数据；
2. 对生成的字节数据进行分片，


#### 数据拉取



