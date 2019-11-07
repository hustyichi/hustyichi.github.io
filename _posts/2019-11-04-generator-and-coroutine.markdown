---
layout: post
title: '生成器与协程'
subtitle:   "Generator and Coroutine"
date:       2019-11-04 22:59:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
---

## 背景介绍

在 Python 中由于 GIL 锁的存在，多线程的并发效率不高。为了比较高效地实现并发，在 Python 中一般的方案是采用多进程 + 协程的方案。

协程也被称为纤线程，是一种程序级别的并发控制，多个协程会执行在同一线程中。协程的思想是由程序自身指定中断点，在 IO 操作时，程序可以自行中断，主动放弃 CPU，此时调度另外的协程继续运行。当 IO 就绪后，再调度此程序从中断点继续向下执行。

Python 中的协程是基于生成器实现的，在 Python 3 的版本演化中，生成器与协程的概念反复纠缠。导致这两个概念比较混杂，在这边梳理一下，也希望帮助后面的人少走弯路。

## 生成器 Generator

在 Python 最早的设计中，生成器是内部包含 yield 语句的函数。使用生成器可以比较优雅地实现一个迭代器。关于优雅迭代器的部分可以参考之前的 [一篇博客](https://hustyichi.github.io/2018/08/14/elegant-iterator-in-python/) 。为了更好地梳理整个流程，我们还是简要地回顾一下：

#### yield

yield 是 Python 2 中就开始提供的一个语句，可以程序中设置中断点，并在中断点返回特定的值，之后可以通过 `next()` 方法从中断点恢复程序的执行，运行到下一个中断点或程序执行结束。通过返回调用 `next()` 方法，每次获取一个值，即可实现类似迭代器的功能。

而内部包含 yield 的函数就被称为生成器，生成器的主要价值是实现优雅的迭代器。一个简单的代码如下所示：

```python
def generator_func():
    yield 1
    yield 2

gf = generator_func()
# 1

print(next(gf))     
# 2

print(next(gf))     
```

#### send()

理论上，生成器如果被用于实现迭代器的功能，借助 yield 就已经足够了，但是为了更好地支持协程的功能，在 [pep-0324](https://www.python.org/dev/peps/pep-0342/#new-generator-method-send-value/) 中提到需要增强生成器，支持 `send()` 方法。通过  `send()` 方法我们不仅可以从生成器中获取值，还能给生成器传递值。此时我们不用关心为什么协程需要此方法，只是了解到我们现在得到一个更强大的生成器。

```python
def generator_func():
    val1 = yield 1
    # 11
    
    print(f'val1: {val1}')           
    
    yield val1 + 2

gf = generator_func()
first_val = next(gf)
# 1

print(f'first val: {first_val}')      

second_val = gf.send(first_val + 10)
# 13

print(f'second val: {second_val}')    
```

通过上面的代码可以看到，我们不仅可以通过 `next()` 从生成器中不断获取新的值，还能通过 `send()` 给生成器传递值，生成器可以根据实际得到值生成新的值。从而支持更灵活和复杂的场景。

#### yield from

为了更好地支持协程，生成器被增强，在 [pep-380](https://www.python.org/dev/peps/pep-0380/) 中支持了子生成器(Subgenerator)。这样可以引用其他生成器的，从而写出更加灵活的生成器。类似代码如下所示：

```python
def generator_func():
    yield 1
    yield 2

def new_generator_func():
    yield from generator_func()

ngf = new_generator_func()
# 1

next(ngf)          
# 2

next(ngf)          
```

通过上面的代码，`new_generator_func()` 和 `generator_func()` 函数一样，都是生成器。实现的效果是一样的。在实际中我们可以根据利用不同的生成器组合出更加复杂的生成器。

#### 生成器总结

在上面的生成器的使用与增强中，可以理清生成器的概念。生成器本质是一个生成数据的机制。借助 yield 提供的中断恢复机制，可以不需要一次性生成所需的全部数据，可以不断执行，不断生产新的数据。

在生成器的增强中，虽然对基础的生成器而言不是必须的。但是也并不是加上了 `send()` 机制或 `yield from ` 之后就变成了协程了，而是一种增强版本的生成器。网上不少博客在描述之中对在这个概念上的介绍中完全是胡说八道。

## 协程 Coroutine

协程是一种在程序级别的并发控制，多个协程会在同一线程中执行。由于同一时间一个线程中只有一个协程在执行，因此可以避免加锁的问题。基于最基础的生成器就可以实现协程。当然基于增强型的生成器可以实现更加有实用机制的协程。

下面以一个简单的生产者和消费者协程为例进行介绍：

```python
def consumer():
    while True:
        d = yield 'data from consumer'
        print('[Consumer] get data from producer: %d' % d)

def producer(consumer_obj):
    next(consumer_obj)
    for count in range(5):
        print('[Producer] producing %d' % count)
        consumer_obj.send(count)
    consumer_obj.close()

c = consumer()
producer(c)
```

在上面的代码中，利用生成器实现的协程。可以在无锁的情况下实现从 `producer()` 向 `consumer()` 传递数据。单独看 `consumer()` 还是生成器类型，但是整体是借助生成器实现任务的协作控制。

#### @asyncio.coroutine与yield from

在前面的例子表现，多个协程的协作还比较麻烦，看起来也还和原始的生成器类似，只能通过 `send()` 进行数据传递与任务恢复。在 [pep-3156](https://www.python.org/dev/peps/pep-3156/) 开始引入事件循环，并可以通过 `@asyncio.coroutine` 显示地声明为协程，可以将多个协程在事件循环中运行。

一个简单的例子如下所示：

```python
@asyncio.coroutine
def task1():
    while True:
        print('[Task1] Run task1 at %s' % datetime.now())
        yield from asyncio.sleep(2)

@asyncio.coroutine
def task2():
    while True:
        print('[Task2] Run task2 at %s' % datetime.now())
        yield from asyncio.sleep(2)

loop = asyncio.get_event_loop()
tasks = [task1(), task2()]
loop.run_until_complete(asyncio.gather(*tasks))
loop.close()
```

在上面的任务中 `asyncio.sleep(2)` 模拟异步 IO 操作，可以看到在同一线程下，协程 `task1()` 和 `task2()` 在事件循环中执行，当其中一个协程阻塞在模拟的 IO 操作时，就会放弃 CPU，另一个协程就会被切换执行。利用事件循环可以更充分地利用 CPU 资源，同时可以避免线程切换带来的开销。

#### async 和 await

在上面的代码中，虽然实现了协程的并行执行，但是如果去查看 `task1()`  的类型，会发现依旧是生成器类型（class generator），看起来还是类似生成器的应用。而且使用 `@asyncio.coroutine` 以及 `yield from` 看起来也并不优雅。

在 [pep-0492](https://www.python.org/dev/peps/pep-0492/) 中 Python 提出了全新的 async 和 await 语法，对 `@asyncio.coroutine` 以及 `yield from` 进行了替换，按照新的语法改写上面的代码如下所示：

```python
async def task1():
    while True:
        print('[Task1] Run task1 at %s' % datetime.now())
        await asyncio.sleep(2)

async def task2():
    while True:
        print('[Task2] Run task2 at %s' % datetime.now())
        await asyncio.sleep(2)

loop = asyncio.get_event_loop()
tasks = [task1(), task2()]
loop.run_until_complete(asyncio.gather(*tasks))
loop.close()
```

在使用了全新的语法之后，`task1()` 的类型才变成了协程类型(class coroutine ) ，而从语法上，也与生成器的 `yield` 语法彻底区分开来。从各个方面将协程与生成器进行了区分。

## 总结

这篇文章是在梳理 Python 3 的异步操作时看到了各个技术博主随意使用生成器与协程，看着特别奇怪，新手真的会被各种奇怪的说法绕晕了。初看这两者，在 Python 中由于都是基于 yield 提供的中断与恢复机制，所以看起来确实很相似。但是从使用角度讲，这两者就有很大区别了。生成器本质上一个数据生产器，而协程是一个程序级别的并发控制机制。

当然如果直接从 async 和 await 用起来，那么就不会将协程与生成器弄混了。还是早日拥抱新特性吧。