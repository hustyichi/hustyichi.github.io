---
layout: post
title: '实用的装饰器库decorator'
subtitle:   "decorator"
date:       2018-11-17 12:39:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python

---

## decorator
今天介绍的是一个已经存在十年，但是依旧不红的库decorator，[github地址](https://github.com/micheles/decorator)。这个库可以帮你做什么呢？可以帮你更方便地写python装饰器代码，不了解装饰器的可以先去阅读[廖神的博客](https://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/001386819879946007bbf6ad052463ab18034f0254bf355000)，更重要的是，让python中被装饰器装饰后的方法长得更像装饰前的方法。这个需求，不得不说，确实很小众，但是需要的时候自己完全从头写出来就相当麻烦了。因此，来了解一下吧。本文主要内容都是参考官方文档，英文和中文一样流利的可以直接阅读[官方文档](https://decorator.readthedocs.io/en/latest/tests.documentation.html)

## 基础定义
在介绍定义前，先补充一个基础概念：`signature`，直译过来是签名，python中每个方法都有自己的签名，根据[PEP 362](https://www.python.org/dev/peps/pep-0362/)可以了解到，方法的签名包含方法的所有必要信息，以及相应的参数信息，在python3中，签名中还包含方法注释，用于标识参数的类型。了解到上面的定义之后，目前的装饰器可以分成两种：

1. 签名不变的装饰器，此装饰器是一个可执行对象，输入和输出都是方法，而且输入和输出的方法签名相同。
2. 签名改变的装饰器，此装饰器会改变输入方法的签名，或者输出的就不是可执行的方法。

签名改变的装饰器会有特定的用途，比如内置的`staticmethod`和`classmethod`装饰器，但是大部分场景下，我们需要的都是签名不变的装饰器，因为装饰器的一个很重要的便利之处在于：很方便地增强方法的功能。但是一般情况下，我们写出来的装饰器都会改变装饰器签名，下面会详细介绍。

## 原始装饰器
原始的装饰器一般简单写法是两层嵌套，一个简单的例子如下所示：

```
def log(func):
    def wrapper(*args, **kw):
        print 'before run'
        return func(*args, **kw)
    return wrapper
```
上面的装饰器实现一个简单功能，就是在被装饰的代码执行前多打印一行文字，对应于实际中方法执行前的预处理，后续我们使用此装饰器装饰我们的代码，装饰的代码如下所示：

```
@log
def now(msg):
    print '2018-11-17 {}'.format(msg)
```
我们此时运行`now()`方法，看起来一切正常。但是执行下面的代码：

```
print now.__name__
print inspect.getargspec(now)
```
最终看到的效果是：
![原始装饰器](/img/in-post/decorator/original.png)
看到方法名改变了，`now()`方法的方法名变成了`wrapper`，而且参数表中参数支持也变成了`args`和`kw`，原因在于装饰器log装饰了`now()`方法后，实际执行的是`now = log(now)`，而`log()`方法实际返回的就是`wrapper()`方法，只是`wrapper()`方法也执行了原始`now()`方法的代码而已，因此装饰器实际上是通过创建一个新方法执行旧方法的代码而已。这种方法可以帮我们完成所需的功能。但是按照上面的定义，这是一个签名改变的装饰器。一般情况下，我们会使用`functools.wraps()`装饰器帮我们解决上面`__name__`的问题，但是参数表依旧改变了。

## decorator库
如果使用decorator库，上面的例子可以写成如下所示：

```
from decorator import decorator

@decorator
def log(func, *args, **kw):
    print 'before run'
    return func(*args, **kw)
```
decorator库将原始嵌套的写法改造成单层的写法，因此两层的参数合并了，使用新的log装饰器装饰原来的`now()`方法，执行上面代码，最终效果如下所示：
![decorator装饰器](/img/in-post/decorator/decorator.png)
可以看到，被装饰的对象最终展示出来的是`__name__`与参数与原始的`now()`方法完全一致，使用decorator库实现的装饰器真正实现了签名不变。

同时decorator库还提供了[contextmanager](https://decorator.readthedocs.io/en/latest/tests.documentation.html#contextmanager)，与系统的contextmanager功能一致，不了解系统contextmanager的可以阅读我[之前的文章](https://hustyichi.github.io/2018/08/28/context-managers-in-python/)，同时decorator库提供的contextmanager还提供`__call__`方法的增强，可以将原始的上下文代码如下所示：

```
def func():
	with context_manager():
		print 'Do my work'
```
变成如下所示的代码：

```
@context_manager
def func()
	print 'Do my work'
```
即python3.2中实现的`GeneratorContextManager`，直译是上下文管理生成器。而decorator提供的contextmanager支持python2，可以简化上下文管理的代码。

## 总结
decorator是一个小众需求，因为原始的装饰器虽然是会导致签名改变，但是大部分场景下我们使用都是Ok的，也可以完成我们的功能。但是如果有需要了解原始参数表的装饰器，比如多层装饰器，那么直接使用decorator库吧，省心省力。