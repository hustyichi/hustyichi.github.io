---
layout: post
title: 'Python优雅迭代器实现'
subtitle:   "Elegant iterator in python"
date:       2018-08-14 23:06:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
---

## 简单迭代器
在python中，为了让类可以迭代访问，需要满足如下所示条件：

1. 在类中需要实现__ iter__()方法，此方法需要返回一个迭代器；
2. 为了类成为迭代器，可通过实现next()方法，此方法可以返回下一个可用的数据，直到没有可用数据，此时抛出StopIteration异常

按照上面的理论，可以实现一个简单的迭代器如下所示：

```
class IterableServer:
    services = [
        {'active': False, 'protocol': 'ftp', 'port': 21},
        {'active': True, 'protocol': 'ssh', 'port': 22},
        {'active': True, 'protocol': 'http', 'port': 80},
    ]

    def __init__(self):
        self.current_pos = 0

    def __iter__(self):
        return self

    def next(self):
        while self.current_pos < len(self.services):
            service = self.services[self.current_pos]
            self.current_pos += 1
            if service['active']:
                return service['protocol'], service['port']
        raise StopIteration
```

可以比较容易分析出来，类IterableServer中存在一个services的列表，希望可以迭代访问其中‘active’为True的service，在上面的代码中，实现了next()方法，因此类IterableServer已经成为了一个迭代器，因此__ iter__()方法可以直接返回self。

在实际使用中，可以直接迭代访问，类似如下：

```
for protocol, port in IterableServer():
    print ("service %s on port %d" % (protocol, port))
```

## 优雅迭代器
按照上面的理论可以简单实现一个迭代器，但是需要维护自己维护迭代的状态，比如current_pos，大多数情况下，此状态是没有额外用途的，因此一种更加优雅的实现方法可以如下所示：

```
class IterableServer:
    services = [
        {'active': False, 'protocol': 'ftp', 'port': 21},
        {'active': True, 'protocol': 'ssh', 'port': 22},
        {'active': True, 'protocol': 'http', 'port': 21},
    ]

    def __iter__(self):
        for service in self.services:
            if service['active']:
                yield service['protocol'], service['port']
```

可以看到在新的实现方法中，没有保存任何中间状态，只是在满足条件时，通过yield将对应的值返回即可

#### yield使用
yield会将函数转换为generator（生成器），函数转换为generator之后，调用此函数不会导致函数执行，而是返回一个迭代器。迭代访问(也可以手动调用next()方法，与之迭代访问等价)生成的迭代器时，才真正开始执行函数，每次执行都会中断与yield语句处，并返回yield指定的值。下一次迭代则从上一次迭代终止处继续执行，直到再次碰到yield语句。当函数执行完成后，会抛出StopIteration异常

```
def yield_func():
    for i in xrange(0, 5):
        yield i
 
for d in yield_func():   # 0, 1, 2, 3, 4
	print d
	        
iter = yield_func()
print iter.next()   # 0
print iter.next()   # 1
print iter.next()   # 2
```

可以看到上面的函数添加了yield语句，转换成了generator，调用此函数会返回迭代器，如果直接循环迭代器访问，即采用for d in yield_func() 会依次获取相应的值0，1，2，3，4，如果手动执行next()方法，那么第一次会得到执行到yield 0处就返回，因此得到迭代值0，第二次从断点处继续执行，会执行到yield 1处返回，得到迭代值1，依次继续，直到函数执行结束

此时再理解前面的IterableServer就比较方便了，在调用IterableServer时，第一次访问的是‘protocol’为ftp的service，由于'active'为False，不会执行yield语句，继续执行，访问'protocol'为'ssh'的service，此时'active'为True，会返回此service的'protocol'与'port'值。当进行下一次迭代时，从此次结束的位置继续，会访问‘protocol’为'http'的service，此时'actice'为True，返回service对应的值。再继续下一次迭代，由于函数执行结束，此时当前方法会抛出StopIteration异常，结束迭代。


  
