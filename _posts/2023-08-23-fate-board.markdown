---
layout: post
title: 'Fate Board 执行流程探索'
subtitle:   "Fate Board Execution Process Exploration"
date:       2023-08-23 09:00:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - FATE
---

## 背景介绍
FATE Board 是 FATE 提供的一个工程，用于给 FATE 提供可视化能力，方便在联邦学习训练中实时查看执行状态，更好地定位执行中遇到的问题。

查看 FATE 架构可以看到 FATE Board 是建立在 MySQL 和 FATE Flow Server 的基础上的，看起来数据来源是来自于这两者。FATE Flow Server 在之前的文章中已经介绍过，FATE 中隐私计算的主要调度流程都是实现在这个服务中。

![app](/img/in-post/fate-board/fate-arch.png)

FATE Board 代码地址为 [https://github.com/FederatedAI/FATE-Board](https://github.com/FederatedAI/FATE-Board), 本文的探索基于 v1.11.1，后续版本可能有所不同

## FATE Board 实现探索
FATE Board 工程中包含前端与后端的实现，前端是基于 Vue 实现的，后端则是基于 Java 实现。本文在探索时主要基于两个场景串联了一下完整的流程，分别是主页面的 job 列表页，以及 job 日志详情，整体流程不算复杂。

#### Job 列表页

![app](/img/in-post/fate-board/list.png)

查看 Job 列表页通过 Chrome 调试模式查看对应的请求，即可比较容易发现对应的请求为 `job/query/page/new` , 通过对应的接口信息即可轻松发现后端的实现路径为 `src/main/java/com/webank/ai/fate/board/controller/JobManagerController.java` 中的 `queryPagedJob()` 方法，简单跳转后可以看到对应的真正的内容获取为：

```java
private Map<String, Object> getJobMap(Object query) {
    result = flowFeign.post(Dict.URL_JOB_QUERY, JSON.toJSONString(query));
}
```
最终只是一层简单的转发，对应的地址为 `/v1/job/list/job`，追踪至 FATE Flow Server 中看到了对应的实现，处于路径 `FATE-Flow/python/fate_flow/apps/job_app.py` 中的 `list_job()`，最终的在 FATE Flow Server 中仅仅做了必要的数据库查询，最终返回了结果


#### Job 日志

![app](/img/in-post/fate-board/log.png)

