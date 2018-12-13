---
layout: post
title: 'Mock与模块引入问题'
subtitle:   "mock and module import"
date:       2018-12-13 21:35:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
---

## 问题描述

在最近的单元测试中，使用Mock模块时发现一个一个奇怪的问题，当使用`from module import func` 后，如果使用Mock去模拟对应的方法时，执行的依旧是原始方法，而不是模拟的方法。下面使用代码解释一下：

在项目中存在两个python文件，其中一个是`action.py` 文件，代码为：

```python
def my_func():
    return 'original func'
```

另一个是`test.py` 文件，其代码文件为：

```python
from mock import patch
from .action import my_func

def test_func():
    with patch('path_to_action.my_func', return_value='new value'):
        print my_func()

```

使用pytest执行上面的测试代码，最终执行的结果是`original func` ，说明mock没有成功

但是采用另一种引入方式，将`test.py` 修改为如下所示：

```python
from mock import patch
from . import action

def test_func():
    with patch('path_to_action.my_func', return_value='new value'):
        print action.my_func()
```

执行结果是`new value` ，说明这种方式是可以正常mock成功的。



## import

按照之前使用的情况来看，`import module` 和`from module import func` 是类似，但是从mock的情况来看，这两种使用方式存在一些细微的区别，那么区别到底是什么呢？

一般搜索之后知乎上刘同周给出的[解释](https://zhuanlan.zhihu.com/p/30836117)，这两种方式执行的操作是不同的。

- import module

  此方式引入模块时，执行的操作是：

  1. **创建新的命名空间，用作在相应源文件中定义的所有对象的容器**。在模块重定义的函数和方法在使用global语句时将访问该命名空间。
  2. 在新创建的命名空间中执行模块中包含的代码。
  3. 在调用函数中创建名称来引用模块命名空间。这个名称与模块的名称相匹配。

- from module import func

  使用此方式引入方法时，**将模块中的具体定义加载到当前命名空间中**。from语句相当于import，但它不会创建一个名称来引用新创建的模块命名空间，而是将对模块中定义的一个或多个对象的引用放到当前命名空间中

所以这两种引入造成的区别是命名空间的不同，因此import的不同结果可能就是命名空间造成的。可以做出如下的解释：

1. 采用`import module` 引入时，会根据原始action模块创建命名空间，调用`my_func()` 方法时，是在原始的命名空间下执行的，而mock中的`patch()` 方法使用的就是原始action模块对应命名空间，因此这种方式是可以正常模拟的。
2. 采用`from module import func` 的方式引入时，会将原始方法加载到当前test模块的命名空间中，而mock中的`patch()` 方法中使用的还是原始action模块对应的命名空间，因此这种方式不能正确模拟方法。

那么如何验证这种猜测呢，既然`from module import func` 是将原始方法加载到当前test模块的命名空间中，可以在mock的`patch()` 方法中模拟的当前test模块的`my_func()` 方法，应该就可以正常工作了。那么将`test.py` 的代码修改为如下所示：

```python
from mock import patch
from .action import my_func

def test_func():
    with patch('path_to_test.my_func', return_value='new value'):
        print my_func()
```

测试结果是`new value` ，说明猜测正确

## 结论

这个问题还是对python的引入机制了解不够深入造成的。下面详细介绍一下：

- 采用`import module` 引入时，会创建新的命名空间，此命名空间与模块的名称一致，最终访问引入模块上的方法时，访问的是原始模块对应的路径，因此这种方式引入时，上面的`my_func()` 方法对应的命名空间为`path_to_action.my_func`，其中`path_to_action` 是`action` 模块的路径
- 采用`from module import func` 引入时，不会创建新的命名空间，会将引入的方法加载到当前命名空间中。这样访问引入模块的方法时，访问的是当前模块的路径。因此这种方式引入时，上面的`my_func()` 方法对应的命名空间为`path_to_test.my_func`， 其中`path_to_test` 是`test` 模块的路径

最后，代码中没有魔法，了解清楚背后的原理，一切就了然于胸了。

