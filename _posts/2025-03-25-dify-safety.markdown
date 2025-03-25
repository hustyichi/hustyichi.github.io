---
layout: post
title: "为 AI 应用打造安全屏障：基于 Dify 的完整实践"
subtitle:   "Building a security barrier for AI applications: a complete practice based on Dify"
date:       2025-03-25 21:30:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - agent
    - llm
---

## 背景介绍
2025 是大模型应用爆发的一年，年初的 DeepSeek 吸引了大量人开始部署大模型产品，之后 Manus, MCP 等 Agent 方案又开始持续吸引大量的注意力，大家都在努力做出一款爆款的大模型应用。

但是在这种热情下，有一个很容易被忽视的问题就是大模型的安全性。但是如果希望实现一个完整的大模型应用，AI 应用的安全性是必不可少的一部分。在 2024 年的一篇文章 [Building A Generative AI Platform](https://huyenchip.com/2024/07/25/genai-platform.html) 就介绍过大模型应用的完整架构：

![](/img/in-post/dify-safety/arch.png)

其中对应的安全模块就是`输入围栏(Input guardrails)`和`输出围栏(Output guardrails)`，在文章中虽然介绍了安全机制，但是没有具体的实现方案。在这篇文章中，我们就以目前比较流行的大模型应用平台 Dify 为基础进行实践，我们首先以相对简单方案出发，不断拓展实现更加符合生产环境需求的方案。


## 安全性方案
为了实现文本内容的安全性检测，有如下所示的三种方案：
1. 基于关键词与规则的审查
2. 基于机器学习/深度学习的内容审核
3. 基于人机结合的审核机制
