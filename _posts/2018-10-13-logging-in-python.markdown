---
layout: post
title: 'logging入门'
subtitle:   "logging in python"
date:       2018-10-13 18:56:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python

---

## 基本概念
logging是python中功能强大的日志系统，可以方便地格式化日志信息，对于python programmer来说，logging就是线上定位bug的神兵利器。本文部分内容参考自 [Mario Corchero的演讲](https://www.youtube.com/watch?v=Pbz1fo7KlGg)，有兴趣可以自备梯子观看。

logging包含的基本结构包括logger, handler, filter, formatter，基本结构关系如下所示：
![full logging](/img/in-post/logging/full_logging.jpg)

- Logger

logger用于提供日志的接口。一般情况下，我们会通过logging.getLogger()获取对应的Logger，然后调用的对应的debug(), info(), warn()等接口生成日志

- Handler

Handler处理Logger生成的日志，可以将日志打印至终端，也可以将日志输出至特定的文件。

- Formatter

Formatter决定日志展示的格式

- Filter

Filter对日志进行过滤

#### 层次关系
在实际使用中，Logger会配置Handler进行日志的处理。而Logger之间存在父子层次关系，当子Logger生成日志后，如果设置为可以向上传递日志，那么不仅子Logger对应的Handler会处理此日志，父Logger对应的Handler也会处理此日志。

在python的Logging中，Logger的层次关系是通过`.`来划分层次，比如通过`abc = Logging.getLogger('abc')`创建了叫做abc的Logger，之后继续配置了`def = Logging.getLogger('abc.def')`，那么def就是abc的子Logger，如果def配置了`propagate=1`指定日志可以向上传递，那么def生成的日志不仅会被def指定的Handler进行处理，也会被abc对应的Handler进行处理。

## 基础配置
对Logger的配置主要分成主要分成三种方式：

1. 代码配置，可以在生成Logger后，调用对应的方法进行配置
2. 文件配置，通过在配置ini文件中指定配置信息，然后通过fileConfig()方法指定配置信息
3. dictconfig配置，python3.2后引入的新配置方式，可以采用任意格式存储配置信息，然后转化为字典，通过dictConfig()方法指定配置信息

关于配置文件的相关配置细节，可以参考[云游道士的博客](https://www.cnblogs.com/yyds/p/6885182.html)

## logger实践
在web应用中，使用Logger的一个比较明显的痛点就是单次网络请求一连串相关的任务无法串联起来，导致连续操作的追踪比较麻烦。在我们的实践中，也遇到的这个问题，我们的解决方案是对系统的Logger进行了二次封装，提供系统同样接口，其他人就可以透明使用。具体方案如下：

由于我们采用的flask架构，在所有网络请求的before_request()方法中，为单次网络请求生成唯一的id，然后将对应id存储至全局变量g中，而Logger的封装函数中，会从全局变量g中获取对应id，将id增加至日志前，其他与默认Logger保持一致，其他人就可以透明使用此方法了。
