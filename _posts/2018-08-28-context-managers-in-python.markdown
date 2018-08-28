---
layout: post
title: 'Python上下文管理'
subtitle:   "Context managers in python"
date:       2018-08-28 22:31:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
---

## 上下文管理
python的上下文管理可以方便用于管理资源的分配与回收。一般通过with进行使用，比如经常可见的文件操作：

```
with open("path_to_file", "r") as f:
	f.read()
```

在这样的上下文管理下，用户可以打开文件后，可以不用关心的文件的释放，甚至在文件打开成功打开，但是执行出错时，也会正确释放文件资源。事实上这样的代码对应的等价代码为：

```
f = open("path_to_file", "r")
try:
	f.read()
finally:
	f.close()
```

对于就可以知道，采用上下文管理的方式，可以大大缩短代码

## 类实现上下文管理
对于上下文管理，无非是在执行真正的操作前执行预处理，在执行解释后进行清理工作，而且清理工作是必须执行的，即使出现了异常，清理也会执行。在python中，可以利用魔术方法`__enter__`和`__exit__`进行实现，当使用with进行配合时，`__enter__`会在真正操作前执行，而`__exit__`会在操作后执行。实例代码如下：

```
class context_manager(object):

    def __enter__(self):
        print 'Do prepare'
    
    def __exit__(self, *args):
        print 'Do clean up'

with context_manager():
	print 'Do my work'
```
执行上面的代码可以看到，会先打印出'Do prepare'，然后打印出'Do my work'，最后打印出'Do clean up'，可以看到利用`__enter__`和`__exit__`可以轻松实现上下文的管理。在实际中，可以将资源的申请操作写在`__enter__`方法中，将资源的释放写在`__exit__`中。

## 基于生成器实现上下文管理
为了实现上下文管理，还可以使用生成器（如果不理解生成器可以参考[之前的博客](https://hustyichi.github.io/2018/08/14/elegant-iterator-in-python/)）进行实现

```
from contextlib import contextmanager

@contextmanager
def context_manager():
    print 'Do prepare'
    yield
    print 'Do clean up'
    
with context_manager():
	print 'Do my work'
```
可以看到，最初属于与前面相同，比较简单实现了上下文管理器。在yield前执行预处理，在yield后执行清理工作，不需要构建类也实现了上下文管理。但是真的完全等价吗，其实是有不同的，可以将两种实现的`print 'Do my work'`都修改为 raise Exception()，就会发现区别，采用类实现的上下文管理，在出现异常的情况下可以执行清理工作，基于生成器的方式却会直接退出，不会执行清理工作。那么如何修改可以让这种实现方式处理异常情况呢。可以参考如下代码：

```
from contextlib import contextmanager

@contextmanager
def context_manager():
    print 'Do prepare'
    try:
    	yield
    finally:
    	print 'Do clean up'
    
with context_manager():
	print 'Do my work'
```
按照这样方式，捕获所有异常，但是利用finally保证必然会执行清理代码，这样就能达到和前面的类实现的上下文管理器一样完整的上下文管理器功能了。
