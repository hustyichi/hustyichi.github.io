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

## logging 机制
FATE-Flow 中日志主要基于 Python 原生的 [logging 机制](https://docs.python.org/zh-cn/3/library/logging.html)实现，也正是依靠强大而灵活的 logging 机制，FATE-Flow 才实现了丰富的日志查询能力，完整的原理可以自行查看官方文档。

为了了解 FATE-Flow 的日志实现机制，涉及的一个关键概念就是 `Handler`, `Handler` 是实际处理日志的模块，在 logging 中被设计为可插拔模式的，可以根据需要进行注册。举一个简单的例子进行介绍：

```python
# 创建一个 logger 对象

logger = logging.getLogger(name)

# 注册日志处理器 handler1, 将日志输出至 a.log 文件中

handler1 = TimedRotatingFileHandler("a.log")
logger.addHandler(handler1)

# 注册日志处理器 handler2, 将日志输出至 b.log 文件中

handler2 = TimedRotatingFileHandler("b.log")
logger.addHandler(handler2)

# 实际输出日志

logger.info("test log")
```

在上面的例子中会基于 logging 机制创建出日志实例 logger ，并注册了两个 `Handler`，后续输出的日志就会依次被两个 `Handler` 进行处理，即日志最终会通过输出至文件 `a.log` 和 `b.log` 中，而且在实际运行中可以动态调整 `Handler`，从而动态调整日志处理方式，因此可以实现极其灵活的日志输出方式

理解了这个机制，就很容易理解 FATE-Flow 如何实现将同样的日志内容按照日志类型输出至 `fate_flow_schedule.log` ，`fate_flow_audit.log` 文件中，同时又按照日志等级输出至 `DEBUG.log`, `INFO.log`, `WARNING.log`, `ERROR.log` 文件中了

## FATE-Flow 日志
FATE-Flow 的日志主要存在于 `/data/projects/fate/fateflow/logs` 目录下，在此目录下主要分为两种类型的日志目录：

1. fate_flow 目录，其中包含整体的 FATE-Flow 日志；
2. job 执行的日志，此时会以 job_id 为单位输出所有相关的日志至此作业运行中产生的日志；
最终展示的效果如下所示：

![app](/img/in-post/logger-in-fate/log.png)

其中数字部分是以 job 为单位的日志，右侧 fate_flow 就是 FATE-Flow 整体的日志

其中 fate_flow 中主要是如下所示的日志：

- DEBUG.log
- ERROR.log
- INFO.log
- WARNING.log
- fate_flow_access.log
- fate_flow_audit.log
- fate_flow_database.log
- fate_flow_detect.log
- fate_flow_schedule.log
- fate_flow_stat.log
- stat.log

注意这部分日志因为量可能可能比较大，因此 FATE-Flow 中是按天进行分片，因此最终的效果如下所示的：

![app](/img/in-post/logger-in-fate/fate_flow_log.png)

而 按照 job_id 为单位的日志主要包含如下所示的日志：

- fate_flow_audit.log
- fate_flow_schedule_error.log
- fate_flow_schedule.log
- fate_flow_sql.log
- guest
- host

其中 guest/host 是按照 job 中训练中参与的角色为单位输出的日志，后续介绍中再进一步展开


