---
layout: post
title: 'Local，localStack，localProxy深入解析'
subtitle:   "Local, localStack, localProxy in flask and werkzeug"
date:       2018-08-22 13:24:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - flask
---

## Local
#### Local 是什么
根据 [官方文档](http://werkzeug.pocoo.org/docs/0.14/local/) 的介绍，local是提供了线程隔离的数据访问方式，类似于python中的 thread locals。可以理解为存在一个全局的数据，不同线程可以读取和修改它。不同线程之间是彼此隔离的。比如存在一个thread local的数组arr，在线程1往数组arr添加了一个数据，在线程1中读取时可以看到数组中存在一个数据，但是在线程2中读取时看到的数组arr依旧为空。这样带来的好处是很明显的，多线程之间考虑其他线程带来的影响，从而安全地读取和修改数据。

#### 原生thread locals缺陷
1. 不支持其他类型的并发访问，比如协程之间的隔离就没有支持；
2. WSGI不保证每个请求不会共用线程，如果后面请求复用的前面请求的线程，那么就可以获取到前面请求的数据，请求之间的数据隔离无法得到支持；

#### Local 实现
为了实现线程隔离的数据访问，一种比较显而易见的方式就是构建一个dict，根据每个线程的id去创建独立的数据，当访问数据时，根据当前线程的id进行访问，每个线程访问自己的数据即可。这样就实现了线程隔离的数据访问。

```
class Local(object):
    __slots__ = ('__storage__', '__ident_func__')

    def __init__(self):
        object.__setattr__(self, '__storage__', {})
        object.__setattr__(self, '__ident_func__', get_ident)

    # 获取proxy作为名字构建LocalProxy对象
    def __call__(self, proxy):
        return LocalProxy(self, proxy)
    
    # 移除当前线程数据
    def __release_local__(self):
        self.__storage__.pop(self.__ident_func__(), None)
        
	 # 获取name对应的值
    def __getattr__(self, name):
        try:
            return self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)

    # 将name与对应的值value存储至local中
    def __setattr__(self, name, value):
        ident = self.__ident_func__()
        storage = self.__storage__
        try:
            storage[ident][name] = value
        except KeyError:
            storage[ident] = {name: value}

    # 移除name对应的值
    def __delattr__(self, name):
        try:
            del self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)
```

Local引入了gen_ident方法用于获取当前线程的id，将gen_ident方法存储至类的`__ident_func__`属性中，将所有线程的数据存储至`__storage__`属性中，此属性对应的是一个二维dict，每个线程使用一个dict存储数据，而每个线程的dict数据作为`__storage__`的线程id对应的值。因此线程访问数据事实上是访问`__storage__[ident][name]`，其中前面的ident为线程的id，后面name才是用户指定的数据key。而ident是Local自动获取的，用户可以透明进行线程隔离的数据访问与存储。

## LocalStack
#### LocalStack 是什么
LocalStack是一个多线程隔离栈结构，底层采用Local实现，提供push，pop，top接口，用户可以将其理解为普通栈，只是是多线程安全的。

#### LocalStack 实现
众所周知，栈的实现的一种简单方式是采用数组实现，push为数据加入数组尾部，pop为将数组尾部的数据移除，top即获取数组尾部的数据。而为了实现多线程隔离栈，可以使用参考Local，为每个线程建立一个数组，执行操作时，在当前线程对应的数组中执行操作，那么就可以实现多线程隔离的栈了。

```
class LocalStack(object):
   # 内部使用Local进行数据存储
   
   def __init__(self):
        self._local = Local()

    # 获取线程对应的id，直接复用Local的__ident_func__，即使用get_ident获取线程id
    
    def _get__ident_func__(self):
        return self._local.__ident_func__

    # 存储数据，将数据入栈
    
    def push(self, obj):
        rv = getattr(self._local, 'stack', None)
        if rv is None:
            self._local.stack = rv = []
        rv.append(obj)
        return rv

    # 移除栈顶数据
    
    def pop(self):
        stack = getattr(self._local, 'stack', None)
        if stack is None:
            return None
        elif len(stack) == 1:
            release_local(self._local)
            return stack[-1]
        else:
            return stack.pop()

    # 获取栈顶数据
    
    @property
    def top(self):
        try:
            return self._local.stack[-1]
        except (AttributeError, IndexError):
            return None
```

可以看到上面的LocalStack实现的代码，内部使用Local进行数据的存储，由于Local内部每个线程使用dict进行数据存储，为了简单实现，LocalStack使用了线程中dict的其中的key进行数据存储，这个key就是'stack'字符串，这个key对应的值就是一个普通的数组，利用数组可以比较容易就实现栈。可以看到内部的Local数据为：`__storage__[ident]['stack']`，执行push操作时，就是将数值增加到`__storage__[ident]['stack']`对应的数组尾部，而获取top，则是获取`__storage__[ident]['stack']`数组尾部的数据，python中使用-1表示获取数组尾部的数据。

## LocalProxy
#### LocalProxy 是什么
LocalProxy最初是用于为Local为创建一个代理对象，对此代理对象执行的任意操作与直接对Local对象执行操作等价。之后，LocalProxy可以为方法创建代码对象，对此代理对象执行的任意操作与对被代理方法的返回值执行操作等价。下面举例进行介绍：

```
local = Local()
local.name = "val"
proxy = local('name')
len(local.name) == len(proxy)
```
在上面的代码中，使用`local('name')`构造了一个LocalProxy，此LocalProxy与local.name等价，因此可以看到他们的长度相同，都为3。从上面的例子就可以大致看出，LocalProxy就是一个与原有的数据等价的存在，那么LocalProxy存在的优势何在呢。可以看看下面的例子：

```
user_stack = LocalStack()
user_stack.push({'name': 'Bob'})
user_stack.push({'name': 'John'})

def get_user():
  return user_stack.pop()
  
user = get_user()
print user['name']
print user['name']

proxy_user = LocalProxy(get_user)
print proxy_user['name']
print proxy_user['name']
```
执行上面的代码，首先是通过`get_user()`获取到user，两次`print user['name']`，结果都是'John'，后面为`get_user`构建一个`proxy_user`，此`proxy_user`等价于`get_user()`方法的返回值user，因此可以直接通过`proxy_user['name']`获取数据，但是两次执行`print proxy_user['name']`, 可以看到结果分别为'John'，'Bob'。看着是不是很神奇，proxy_user等价于返回值，但是这个值会根据数据进行动态更新。

因此可以看到LocalProxy的优势，如果通过方法获取返回值后，此返回值就会是确定的。但是如果为此方法构建一个代理对象LocalProxy，那么可以将代理对象作为等价于返回值的对象直接操作，但是优势在于根据情况获取的最新的返回值，可以随着实际情况更新。事实上，proxy_user['name']等价于get_user()['name']，这样就避免了方法返回的值不会更新的问题。

#### LocalProxy 实现

```
class LocalProxy(object):
	 __slots__ = ('__local', '__dict__', '__name__', '__wrapped__')

    # 初始化代理，其中local可能是Local类型的对象，或者是可执行方法或可执行的对象，name在local为Local类型的对象时用于作为key访问数据
    def __init__(self, local, name=None):
        object.__setattr__(self, '_LocalProxy__local', local)
        object.__setattr__(self, '__name__', name)

	 # 动态代理对象
    def _get_current_object(self):
    	  # 为可执行方法或对象
        if not hasattr(self.__local, '__release_local__'):
            return self.__local()
        # 为Local对象，通过name获取对应的值
        try:
            return getattr(self.__local, self.__name__)
        except AttributeError:
            raise RuntimeError('no object bound to %s' % self.__name__)

	 # 重载bool方法，就是获取代理对象，然后执行bool判断
    def __bool__(self):
        try:
            return bool(self._get_current_object())
        except RuntimeError:
            return False
    
    # 重载获取属性方法
    def __getattr__(self, name):
        if name == '__members__':
            return dir(self._get_current_object())
        return getattr(self._get_current_object(), name)
    
    # 重载其他基本所有的基本方法 ...
    
```
可以看到带来对象LocalProxy的实现其实很简单，就是利用`_get_current_object()`获取被代理对象，然后执行相应的操作即可。对proxy执行的任意操作，是不是都是直接通过被代理对象执行的。为了保证代理对象可以直接进行操作，LocalProxy重载了所有的基本方法，这样就可以随意对proxy对象执行操作了。

而在`_get_current_object`方法中可以看到，对于可执行对象或方法，就是直接执行获取可执行对象或方法对应的返回值。而对于Local对象，则是获取name作为属性的值。但是要注意的是，所有的获取都是在执行操作的时候获取的，这样就可以随着程序运行动态更新。同样也可以解释了上面的例子中，为方法get_user创建的LocalProxy类型的proxy_user，可以两次执行proxy_user['name']获取到不同值了，因为两次执行时，都会通过_get_current_object执行get_user方法，两次执行的结果不同，返回的值也就不同了。



