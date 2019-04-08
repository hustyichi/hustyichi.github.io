---
layout: post
title: 'Celery 3.x 升级至 celery 4.x'
subtitle:   "Celery 3.x upgrade to celery 4.x"
date:       2019-03-26 8:44:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python


---

## 背景介绍

定期升级项目依赖包是个好习惯，依赖包由于将近一年多没有升级了，最近集中整理进行了升级。大部分的升级都比较顺滑，但是 celery 的升级就真的是磕磕绊绊了。按照处理是顺序记录一下，为后来人节省更多时间。

首先介绍下项目中 celery 相关的依赖包，目前项目中主要使用的是 celery 进行异步处理，使用 celery-redbeat 设置定时异步任务，使用 flower 进行异步任务的监控。升级前相关包的版本情况如下：

```python
celery==3.1.25
celery-redbeat==0.10.4
flower==0.9.1
```

## 升级过程

在升级之前，对相关情况进行了调研，发现 celery 和 celery-redbeat 关系紧密，只升级单个容易出现无法启动的问题，于是将 celery 和 celery-redbeat 一起升级，flower 在下一步再进行升级。具体过程如下：

#### celery 官方升级教程

升级第一步，首先去了解了官方给出的[升级教程](http://docs.celeryproject.org/en/4.0/whatsnew-4.0.html#upgrading-from-celery-3-1) ，官方给出的升级步骤主要分为四步：

1. 将 celery 版本升级为 3.1.25，在这个版本中增加了对新的消息协议的正向兼容性， 因此之后可以直接从3.1.25 升级至 4.0，由于原本项目中的 celery 已经处于这个版本，此步骤可省略；

2. 升级配置，使用新的设置名，官方为了保证配置名的一致性，将 celery 的配置进行了更改，但是此次更改是向后兼容的，也就是老的配置名还可以用，但是将来会废弃。完整的对应关系可以见[官方链接](<http://docs.celeryproject.org/en/4.0/userguide/configuration.html#conf-old-settings-map>) ；

3. 阅读官方给出的重要提示，主要是此次版本升级的一些重要变化，如下所示：

   - 不再支持 Python 2.6 ；

   - 这是最后一个支持 Python 2 的主要版本了，从 celery 5.0 开始只支持 Python 3.5+ ；

   - celery 4.x 只支持 Django 1.8 或以上；

   - 不再支持 Windows 和 Jython ；

   - 移除一些原来的功能，包括 `celery.task.http`,` app.mail_admins`, `celery.contrib.batches` ；

   - 由于缺乏支持移除一些功能，不再支持使用 `Django ORM`, `SQLAlchemy`, `CouchDB`, `IronMQ`, `Beanstalk` 作为 broker ；

   - 完全移除一些功能，包括 `--autoreload` , 实验性的线程池功能，`force_execv` ， 使用`amqp` 作为结果存储的功能被废弃；

   - 使用了新的任务消息协议，新的消息协议是默认开启的，而且不是向后兼容的，但是 3.1.25 版本增加了对新协议的兼容性，因此建议先升级 3.1.25 再升级 4.0 ；

   - 小写的配置名，老的配置依旧可用，但是建议升级为新的配置，官方提供了下面的命令帮助自动升级配置；

     ```python
     celery upgrade settings proj/settings.py
     ```

   - 目前是默认的序列化采用 json，如果项目中依赖 pickle 进行序列化，可以使用下面的命令指定 :

     ```python
     task_serializer = 'pickle'
     result_serializer = 'pickle'
     accept_content = {'pickle'}
     ```

   - Task 基类不再自动注册任务了，任务类不再使用特殊的元类自动注册任务了，现在是使用`app.task` 装饰器进行注册。

     如果使用基类任务，需要手工注册任务，代码如下所示：

     ```python
     class CustomTask(Task):
         def run(self):
             print('running')
     CustomTask = app.register_task(CustomTask())
     ```

     最佳实践是使用自定义的任务类复写一般的行为，然后使用任务装饰器实现任务，代码如下：

     ```python
     @app.task(bind=True, base=CustomTask)
     def custom(self):
         print('running')
     ```

   - 任务参数会在调用时进行校验， 如果不一致会报错，但是也支持关闭检查；

   - Redis 事件不是向后兼容的，目前`fanout_patterns` 和 `fanout_prefix` 传输选项是默认开启的。

     没有启动这些标志的 Worker / 监视器是不能看到禁用此标志的 Worker 的。他们依然可以执行任务，单不能彼此接收监视消息。

      为了保持向后的兼容性，你可以在 3.1 的版本中先开启这些标志，然后再升级至 4.0，配置如下：

     ```python
     BROKER_TRANSPORT_OPTIONS = {
         'fanout_patterns': True,
         'fanout_prefix': True,
     }
     ```

   - Redis 的优先级颠倒了，目前优先级0是最低的，优先级9是最高的，这是为了与AMQP保持一致；

   - `Auto-discover` 目前支持 Django app 配置

   - Worker 直接队列不再使用自动删除(auto-delete)

   - 老的命令行程序被移除，安装 celery 不再安装 celeryd, celerybeat 和 celeryd-multi 程序，可以使用新的命令，替代情况如下：

     | 老命令          | 新命令          |
     | --------------- | --------------- |
     | `celeryd`       | `celery worker` |
     | `celerybeat`    | `celery beat`   |
     | `celeryd-multi` | `celery multi`  |

4. 升级至 celery 4.0；

按照上面的流程将 celery 的版本升级至 `celery 4.2`

#### celery-redbeat 配置更新

celery-redbeat 对于 celery 3.x 和 celery 4.x 使用的是不同的配置名，需要根据需要进行更新，pypi 上有[相关的介绍](<https://pypi.org/project/celery-redbeat/>) ，具体如下所示：

1. celery-redbeat 基本配置名针对 celery 3.x 和 celery 4.x 有所不同，对应关系如下：

   | celery 4.x             | celery 3.x             |
   | :--------------------- | ---------------------- |
   | `redbeat_redis_url`    | `REDBEAT_REDIS_URL`    |
   | `redbeat_key_prefix`   | `REDBEAT_KEY_PREFIX`   |
   | `redbeat_lock_key`     | `REDBEAT_LOCK_KEY`     |
   | `redbeat_lock_timeout` | `REDBEAT_LOCK_TIMEOUT` |

2. celery-redbeat 定义具体定时任务列表，在 celery 3.x 中使用的是变量的 `CELERYBEAT_SCHEDULE` ，但是在 celery 4.x 中使用的是变量 `beat_schedule` ， 也需要对应更新

在更新了celery-redbeat 的配置之后，将 celery-redbeat 的版本也更新至最新版 `celery-rebeat 0.12.0`

#### 异常处理 celery 配置生效异常

由于项目中依赖 pickle 进行序列化，因此在前面的步骤中使用新的配置指定使用 pickle 进行序列化，但是在实际启动中发现，依旧报错 datetime 格式序列化出错。在实际测试中发现，在配置文件中加上老的配置参数就能正常序列化了，即加上：

```python
CELERY_TASK_SERIALIZER = 'pickle'
CELERY_RESULT_SERIALIZER = 'pickle'
CELERY_ACCEPT_CONTENT = ['pickle']
```

没有确定是不是 celery 的配置读取还有坑，但是多增加老的配置对运行不会影响 celery 的执行，因此暂时没有深入研究具体原因

#### 异常处理 `'NoneType' object has no attribute 'hostname'`

升级之后重启 celery 时，出现了异常提醒 ` 'NoneType' object has no attribute 'hostname'` ，

查找定位后发现有人在提出过[类似的问题](<https://github.com/celery/kombu/issues/970>) , 现象出现在 `kombu==4.2.1` 升级至`kombu==4.2.2` 之后，kombu 是 celery 包的间接依赖，升级时没有指定版本，确认了本地的 kombu 的版本，确实在升级之后 celery 自动安装的是  `kombu==4.2.2`  

看起来应该是 kombu 新版本引入的bug，于是解决方案就比较明确了，将`kombu==4.2.1` 写入`requirements.txt` ，降级回去之后问题得到解决

#### 异常处理 `RuntimeError('no suitable implementation for this system')`

在本地测试正常执行后，在 docker 环境下使用 gunicorn 启动 web 服务，并使用 eventlet 进行协程并行化，出现了上面的异常 `RuntimeError('no suitable implementation for this system')` 

依旧是一番搜索后发现，有人提出过[类似的问题](<https://github.com/benoitc/gunicorn/issues/1584>) ，在 eventlet 上 github 上，也有人提出[这个issue](<https://github.com/eventlet/eventlet/issues/401>) , 看起来直到目前为止，这个问题依旧没有得到解决。看到大部分给出的意见，这个 bug 应该是 eventlet 引入的。

最终我们抛弃了 eventlet , 投奔了 gevent , 一方面是因为这个 bug，另外一方面也是因为 gevent 使用者更多，更新更加及时，而 eventlet 看起来越来越缺乏维护了，从 eventlet 切换为 gevent 之后问题得到解决

对于希望继续使用 eventlet 的开发者而言，可以将 eventlet 的版本固定在 0.19.0 ， 即指定`eventlet==0.19.0`

## 待解决

在解决了上面的问题中，在本地和测试环境发现的问题就已经都解决了，信心满满地上线，发现依旧依旧出现了下面的两个问题，目前暂时没有解决，因此临时将版本回滚，后续解决问题再上线，到时候再更新此文章

#### 异常待处理 `'list' object has no attribute 'decode'`

这个 bug 在 celery 的 github 上已经[有人报过这个bug](<https://github.com/celery/celery/issues/4363>) , 看起来是新版本的 celery 在使用 redis 作为 backend 时还存在一些问题，官方也没有正面回应这个问题，估计需要后续版本在看能否修复

#### 异常待处理 异步任务没有正常执行

在实际执行后，发现异步任务没有正常执行起来，不确定是否是因为`'list' object has no attribute 'decode'` 的异常，这个问题需要后续进行进一步定位

