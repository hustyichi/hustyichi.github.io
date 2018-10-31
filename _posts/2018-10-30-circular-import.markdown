---
layout: post
title: 'python中的循环引用'
subtitle:   "circular import in python"
date:       2018-10-30 22:21:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python

---

## 循环引用
循环引用在不同语言都会出现，在python中如果如果出现循环引用，会报错ImportError，在本地创建两个文件，分别为a.py和b.py，然后让他们互相引用，可以看到循环引用的错误，如下所示：
![import_error](/img/in-post/circular-import/import_error.jpg)
一般情况循环引用都是代码存在循环依赖的关系，根据循环引用出错的现场引用路径重构代码，避免循环依赖即可，也没有深入研究循环引用的情况，最近出现一个循环引用，让我觉得很有比较了解清楚其中的原理

## 循环引用案例
将完整的代码进行了简化，存在两个包`my_test`和`my_test2`，包`my_test`中包含三个模块`__init__.py`，`a.py`，`b.py`，代码如下所示

```
# my_test/__init__.py
from . import a
from . import b

# my_test/a.py
空文件

# my_test/b.py
from my_test2 import c
```
而包`my_test2`中包含两个模块`__init__.py`和`c.py`，代码如下所示：

```
# my_test2/__init__.py
空文件

# my_test2/c.py
from my_test import a
```
看起来是完全没有循环引用，但是从其他模块引用`my_test2/c.py`，就会报循环引用的错误，错误如下所示
![import_error](/img/in-post/circular-import/import_one.jpg)
但是更加诡异的是，大佬建议的一个方案是，先引用`my_test`包，然后再引用`my_test2/c.py`模块就不会报错了。效果如下所示：
![import_error](/img/in-post/circular-import/import_two.jpg)


## 循环引用追溯
如何解释上面这种看起来很奇怪的代码，正好手边有《python核心编程》，随手翻了下，在12.5节中有介绍：

> 加载模块会导致这个模块被执行。也就是被执行模块的顶层代码将直接被执行。这通常包含设定全局变量以及类和函数的声明
>
> 一个模块只被加载一次，无论它被导入多少次。这可以阻止多重导入时代码被多次执行

从上面的介绍可以看到，模块的第一次导入（引用）会导致模块的代码会被执行，如果执行中，如果此模块引用了其他模块，又会继续引入，如此循环下去，直到结束所有引用。根据这个，我们可以分析前面错误引用现场，可以看到引用的调用的路线如下所示：

1. 引用`my_test2/c`模块
2. 由于是第一次引用`my_test2/c`模块，此时会执行`my_test2/c.py`文件，此时会执行`from my_test import a`
3. 此时执行了包`my_test/__init__.py`的代码，执行了`from . import b`，引用了模块`my_test/b`
4. 模块`my_test/b`由于是第一次被引用，会执行其中的代码，执行`from my_test2 import c`，循环引用出现，报错

上面的路径最初看起来一些符合预期，但是在第3步就看起来有一些奇怪了，为什么会执行包`my_test/__init__.py`的代码，看起来是引用包内的元素时，可能会执行包初始化的代码，一般搜索加验证之后确定

> 第一次引用包内模块或包时，都会执行包的初始化代码，即包内的`__init__.py`的代码

所以，这个问题看起来就可以解释了，当引用模块`my_test/a`时，先会执行包`my_test`的初始化模块，导致也引用了模块`my_test/b`，从而导致原本没有形成循环引用的情况变成循环引用

同时也能解释为什么先引用包`my_test`，然后再引用`my_test2/c`时，就可以正常工作呢，因为引用`my_test`包的路线如下所示：

1. 引用包`my_test`
2. 由于是第一次引用，会执行包的初始化代码`from . import a from . import b`，从而引用了模块a和b
3. 模块b由于是第一次引用，会执行模块的代码`from my_test2 import c`，从而引用了模块c
4. 模块c由于是第一次引用，会执行模块的代码`from my_test import a`，从而引用模块a，但是此时要注意，此时没有形成环，只是模块a被引用了两次，具体如图所示：

![import_error](/img/in-post/circular-import/import_two_pic.jpg)

而之后再引用my_test2/c`模块时，由于模块c不是第一次引用，因此就不会重复执行，因此也就不会再出现前面的循环引用的情况了

## 总结
python中第一次模块引用会导致模块代码的执行，第一次包引用以及包内模块的引用会导致包初始化代码的执行，因此对于模块内的代码，一定要封装进函数中，避免引用时就意料不到地执行了。

最后，代码中其实没什么玄学，只要找到原理，玄学就顺理成章了，祝愿大家的coding中，少一些玄学，多一些真诚。