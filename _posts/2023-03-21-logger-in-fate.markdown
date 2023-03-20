---
layout: post
title: 'FATE Flow 源码解析 - 日志输出机制'
subtitle:   "FATE Flow source code analysis - log mechanism"
date:       2023-03-21 08:12:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - FATE
    - Federated learning
---

## 背景介绍
在 [之前的文章](https://hustyichi.github.io/2023/03/08/fate-flow-loop/) 中介绍了 FATE 的作业处理流程，在实际的使用过程中，为了查找执行中的异常，需要借助运行生成的日志，但是 FATE-Flow 包含的流程比较复杂，对应的日志也很多，而且分散在不同的文件中，在这篇文章中就对 FATE-Flow 的日志机制进行梳理，帮助大家了解 Python 服务中实现一个更加灵活的日志机制



