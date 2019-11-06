---
layout: post
title: '深入了解 gevent'
subtitle:   "Dive into gevent"
date:       2019-04-14 12:26:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
---

## 背景介绍

在 python 的 web 部署中，经常会使用 gunicorn 启动 web 服务，同时，为了并发效率更高，一般会使用 `-w` 指定多个工作进程 (worker processes) , 同时可以通过 `-k` 指定工作进程的类型，目前支持的工作进程的类型包括： sync, eventlet, gevent, tornado, gthread。具体选择工作进程类型可以参考[此博客](<https://medium.com/@genchilu/%E6%B7%BA%E8%AB%87-gunicorn-%E5%90%84%E5%80%8B-worker-type-%E9%81%A9%E5%90%88%E7%9A%84%E6%83%85%E5%A2%83-490b20707f28>) 。在我们的项目中，主要使用 gevent，本篇文章就希望能深入介绍 gevent 在 web 部署中是如何工作以及提升效率的。

gevent 是底层是依赖 greenlet 实现并发提升效率的，而 greenlet 是一个协程机制的实现。下面就从简单到复杂依次进行介绍。

## 协程

协程也被称为微线程，是一种比线程更轻量级的任务调度方式，一个线程内可以有多个协程。

协程是一种可以在子程序内部中断，转而执行其他子程序，之后再从中断点继续执行的机制。比如在 I/O 操作时就可以执行其他子程序，等 I/O 操作数据就绪时继续执行，就可以在单个线程内实现非阻塞式的复用，大大提升效率。

## greenlet

greenlet 是一个轻量级的协程实现，使用的方法简单而清晰。创建 greenlet 实例执行方法，在方法内部可通过 `greenlet.switch()` 切换至其他 greenlet 实例进行执行。一个简单的例子如下所示：

```python
from greenlet import greenlet

def test1():
    print(12)
    gr2.switch()
    print(34)

def test2():
    print(56)
    gr1.switch()
    print(78)

gr1 = greenlet(test1)
gr2 = greenlet(test2)
gr1.switch()
```

上面的代码执行的结果是:

```
12
56
34
```

可以看到上面创建了两个 greenlet 实例 gr1, gr2，首先最后一行调用`gr1.switch()` 启动执行 test1 方法，打印出 12，接着执行 `gr2.switch() ` 从而启动执行 test2 方法，打印出 56，接着执行 `gr1.swich()` 方法执行 test1 方法， 从上次的断点出继续执行，打印出 34，之后就结束了，因此最终 78 不会被打印出来。

可以看到 gevent 实例的运行逻辑很简单，就是切换时保存现场，下一通过`gevent.switch()` 切换回来时，从切换点继续执行。但是并不是所有的 gevent 实例都能执行结束，比如上面的 gr2 就没有执行结束，因为没有切换回来。

#### 父子关系

greenlet 实例之间存在父子关系，当子 greenlet 执行完毕后，父 greenlet 继续执行。在创建 greenlet 时，可以指定其父 greenlet，如果不指定父 greenlet， 那么其父 greenlet 就是主 greenlet(main greenlet)。下面介绍一个简单实例：

```python
from greenlet import greenlet

def test1():
    print(12)
    gr2.switch()
    print(34)

def test2():
    print(56)
    print(78)

gr1 = greenlet(test1)
gr2 = greenlet(test2, parent=gr1)
gr1.switch()
```

可以看到上面的代码执行的结果如下所示：

```
12
56
78
34
```

可以看到建立了两个 greenlet gr1, gr2，其中 gr2 的父 greenlet 是 gr1，gr1 没有指定父 greenlet，因此默认是 主 greenlet。

实际执行时，首先启动 gr1，打印 12，之后切换为 gr2, 打印 56，78，之后 gr2 执行结束，切换为父greenlet gr1，从切换处继续执行，打印34，之后 gr1 执行结束，切换为主 greenlet，继续往下，主 greenlet 也执行结束。

#### greenlet 应用

可以看到 greenlet 思路很清晰，协程的切换接口很易用。但是对于实际的业务开发依旧存在一些不便之处：

1. greenlet 原始的执行方法都需要转换为 greenlet 实例，而且需要使用者管理 greenlet 实例之间的树形关系，对于业务开发而言并不友好；
2. greenlet 的切换需要调用 `greenlet.switch()` 方法进行切换，业务中需要充斥大量的 greenlet 管理代码；

## gevent

gevent 基于 greenlet 库进行了封装，基于 libev 和 libuv 提供了高效的同步API。

对 greenlet 在业务开发中的不便之处，提供了很好的解决方案：

1. 对于 greenlet 实例的管理，不使用树形关系进行组织，隐藏不必要的复杂性；
2. 采用 monkey patching 与第三方库协作，不需要重写原因方法，也不需要手工通过`greenlet.switch()` 切换；

基本使用类似如下所示：

```python
import gevent
from gevent import socket

urls = ['www.google.com', 'www.example.com', 'www.python.org']
jobs = [gevent.spawn(socket.gethostbyname, url) for url in urls]
gevent.joinall(jobs, timeout=2)

print [job.value for job in jobs]
```

上面的示例中使用了 gevent 中两个基础接口：

- `gevent.spawn()` 此方法用于创建一个 greenlet 实例，用于执行特定的方法
- `gevent.joinall()` 用于等待所有的 greenlet 实例执行完毕

上面的例子中创建了三个 greenlet 实例，执行`socket.gethostbyname` 方法，全部执行结束后获取请求的结果。可以看到业务代码使用 gevent 时，不需要关心 greenlet 实例组织的逻辑，由 gevent 统一组织调度起来。

#### monkey patching

通过之前的介绍，gevent 是利用协程实现线程的复用，在子任务出现 I/O 操作时，切换去执行其他子任务，但是在业务代码中，不会手工触发`greenlet.switch()` 去触发切换到另一个子任务，甚至默认的网络库进行网络请求时是阻塞式地请求网络，完全没有办法切换子任务去实现复用，那么 gevent 是如何实现的呢？ 

答案是 monkey patching，gevent.monkey 模块提供了大量的方法与类去替换第三方阻塞式的库的行为，比如替换 socket 库中的行为从而支持非阻塞式的网络请求。

比如希望引入非阻塞式的网络请求，可以实现的方式如下：

1. 直接从 gevent 中引入 socket 库进行使用，代码如下所示

   ```python
   from gevent import socket
   ```

2. 使用系统的 socket 库，并使用 gevent 的 `monkey.patch_socket()` 对其打补丁

   ```python
   from gevent import monkey; monkey.patch_socket()
   ```

如果希望直接用 gevent 对其所支持的第三方库打补丁，使其转换为非阻塞的操作，可以使用如下所示的代码

```python
from gevent import monkey; monkey.patch_all()
```

#### gevent 应用

通过上面的介绍可以看到，gevent 通过接口的封装，让使用者不需要关心 greenlet 实例的调度，易用性得到提升，同时利用 monkey patching，让使用者不用关心任务的切换，从而让业务代码写起来更加方便

## gunicorn 与 gevent

在实际的 web 部署中，我们的使用 gevent 比前面介绍得要更加简单一些，我们甚至都不需要显示去执行 `monkey.patch_all()` 去给第三方库打补丁，直接在 gunicorn 启动 web 服务时，通过 `-k gevent` 去指定工作进程类型即可，后续不需要任何的开发。看起来 gunicorn 帮助我们做了该做的初始化，具体代码是怎么呢 ？
在 github 的项目的 `gunicorn/workers/ggevent.py` 可以看到相关的实现：

```python
def patch(self):
    monkey.patch_all()

    # monkey patch sendfile to make it none blocking
    
    patch_sendfile()

    # patch sockets
    
    sockets = []
    for s in self.sockets:
        sockets.append(socket.socket(s.FAMILY, socket.SOCK_STREAM, fileno=s.sock.fileno()))
    self.sockets = sockets


def init_process(self):
    self.patch()
    super().init_process()
```

可以看到 gunicorn 的初始化工作进程中，调用`self.patch()` 方法，执行了`monkey.patch_all()` 给第三方库加上了补丁，从而保证在使用 gevent 时，可以将支持的第三方的阻塞方法转换为非阻塞方法，从而充分利用协程提供的并发效率。

## 总结

从协程的基础概念，到 greenlet 的协程实现，再到 gevent 的二次封装，再到 gunicorn 与 gevent 的联合使用，最终将一个复杂的协程复用变成一个只需要一个参数 `-k gevent` 就能协作起来大大提升效率的方案，不得不说让人佩服。