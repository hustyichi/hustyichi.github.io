---
layout: post
title: 'FATE Flow 源码解析 - 资源分配流程'
subtitle:   "FATE Flow source code analysis - resource allocation process"
date:       2023-03-14 07:50:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - FATE
    - Federated learning
---

## 背景介绍
在 [上一篇文章](https://hustyichi.github.io/2023/03/08/fate-flow-loop/) 中介绍了 FATE 的作业处理流程，对于资源管理没有深入介绍，本篇文章就是来补充相关细节
众所周知，FATE 是一个联邦学习框架，涉及了多个参与方进行模型的训练，而各个参与方的训练过程都需要使用 CPU 资源和内存资源，因此在作业执行前需要提前先申请好必要的资源，才能避免在执行中避免因为资源不足而导致执行失败

## 资源分配相关流程
在资源的管理中，主要涉及两部分:
1. 资源初始化，在 Fate-Flow 初始化时，根据配置文件初始化对应的资源信息，确定了服务中可供分配的资源总量;
