---
layout: post
title: 'flask 信号机制实现'
subtitle:   "Signal in flask"
date:       2019-04-21 21:06:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
---

## 背景

flask 从 0.6 开始支持信号机制，具体的功能实现是依赖 [blinker](<https://github.com/jek/blinker>) ， 通过使用 blinker，在 flask 内实现了一个简单的观察者模式。在 flask 中，可以对 flask 内的一些 [内置的信号](<https://dormousehole.readthedocs.io/en/latest/api.html#core-signals-list>) 可以监察，注册相关的处理方法，在信号发出时触发。使用起来十分方便，这篇文章就来具体探索一下 blinker 的信号实现机制

## 信号使用

在 blinker 中，信号机制的使用十分简单，通过 `Signal.connect()`  注册信号处理方法， 即增加观察者，之后可以通过 `Signal.send()` 发出信号，此时所有的信号处理方法会被依次触发。具体的使用如下所示：

```python

# 生成信号
ready = signal('ready')

# 注册信号
def subscriber(sender):
    print("Got a signal sent by %r" % sender)

ready.connect(subscriber)

# 发出信号
ready.send('sender A')
```

上面的情况最后会执行`subscriber()` 方法，在本例子中看起来有些多余，完全可以直接调用`subscriber()` 方法。但是在实际中，信号的注册方法可以是很多个，并且可以动态变化，使用了 blinker 提供的观察者模式，信号触发者不用关心一系列观察者方法的调用，从而解耦信号触发者和观察者。

## 具体实现

blinker 中代码的实现也十分简单，对其主要代码简化后如下所示：

```python
class Signal(object):
    def __init__(self, doc=None):
        self.receivers = {}                  # 建立 receiver 的 id 与 receiver 的映射
        self._by_sender = defaultdict(set)   # 保存 sender_id 与 receiver 对应 id 列表的映射

    # 注册信号触发的方法，可以用 sender 进行过滤
    # 如果指定了 sender, 则只会在此信号触发并传递与 sender 等同的参数时响应
    # 如果不指定 sender, 那么会在此信号触发传递任意参数都会响应
    def connect(self, receiver, sender=ANY, weak=True):
        receiver_id = hashable_identity(receiver)

        if sender is ANY:
            sender_id = ANY_ID
        else:
            sender_id = hashable_identity(sender)

        self.receivers.setdefault(receiver_id, receiver) #存储 receiver_id 与 receiver 映射
        self._by_sender[sender_id].add(receiver_id) # 存储 sender_id 与 receiver_id 列表映射
        return receiver

    # 触发信号注册的方法
    def send(self, *sender, **kwargs):
        if not self.receivers:
            return []

        return [(receiver, receiver(sender, **kwargs)) # 依次调用参数 sender 对应注册的方法
                for receiver in self.receivers_for(sender)]

    # 获取 sender 对应的注册的方法
    def receivers_for(self, sender):
        if self.receivers:
            sender_id = hashable_identity(sender)
            if sender_id in self._by_sender:
                ids = (self._by_sender[ANY_ID] |
                       self._by_sender[sender_id]) # 获取 sender_id 或任意参数对应注册的方法
            else:
                ids = self._by_sender[ANY_ID].copy()

            for receiver_id in ids:
                receiver = self.receivers.get(receiver_id) # 获取 receiver_id 对应的方法
                yield receiver

```

#### 信号注册

通过上面的代码可以看到，通过 `Signal` 类创建信号后，可以调用`Signal.connect()` 方法注册处理方法，在注册时，可以使用 `sender` 参数指定只对特定的参数才会触发执行，还是对此信号的任意参数的触发都执行，默认是对任意参数都会触发执行。注册之后就会存储相应的注册信息至`Signal` 类对应的`receivers`  和 `_by_sender` 属性中

#### 信号触发

在执行`Signal.send()` 方法触发信号时，会从 `_by_sender` 属性中查找到此次触发参数对应的注册的方法的 id 列表，然后利用 id 列表从 `receivers` 中查找到注册的处理方法，依次执行注册的处理方法。

对 blinker 的简单理解为可以通过将注册的方法列表保存至对象的属性中，之后可以简单的触发这个方法列表。

## 总结

上面的介绍是一个 blinker 中的一个简单实现，blinker 内部还有一些更复杂一些，包括存储原始触发方法的弱引用优化，有兴趣的同志们可以深入研究一下。但是最核心的内容无非就是这些了。可以看到，一个被广泛使用的库也没有做什么神奇的事情。越来越能理解 linux 的哲学，每个工具都应该追求做少做好。