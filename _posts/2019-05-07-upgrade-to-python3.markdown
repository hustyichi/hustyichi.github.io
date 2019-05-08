---
layout: post
title: '升级 Python3'
subtitle:   "upgrade to python3"
date:       2019-05-07 8:52:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
---

## 背景介绍

Python 2 目前已经逐渐落伍了，官方会在 2020 元旦放弃 Python 2.x 的支持，而且越来越多的包最新的版本都开始放弃对 Python 2 的支持，对于需要长期更新的项目，升级 Python 3 是一个更加明智的选择。而众所周知，升级 Python 3 是一个痛苦而长期的过程，但是升级完成，又可以享受 Python 3 带来的遍历语法特性。下面就来具体了解一下，相对于Python 2 而言，Python 3 到底有哪些切换痛苦以及全新特性。

## 切换痛苦

这一部分主要介绍由于升级 Python 3 不得不进行的语法或基础概念的调整，这个部分可以借助一些 `flake8` 或 `pylint` 检查发现其中的问题，再借助兼容库比如 `six` 写出兼容性语法。下面具体列出主要的区别：

1. print 从关键字变成方法

   在 Python 2 里面的 `print 'hello'` 写法需要改成 `print('hello')` 

2. unicode 字符串

   Python 2 中存在 unicode 字符串和非 unicode 字符串，Python 3 中统一采用 unicode 字符串。因此 Python 2 中的 `u'hello'` 可以统一改成 `'hello'` 。因为统一采用 unicode 字符串，也就不存在转换了，因此方法`unicode()` 在 Python 3 就不存在了

3. 长整型问题

   Python 2 中为了表示长整型，可能会使用 `x=123000000000L` 的写法，Python 3 统一采用 int 类型表示更大范围，不需要区分整型和长整型了。因此也就不需要采用上面这种奇怪的写法了。

4. 字典方法 `has_key()`

   Python 2 中的 `has_key()` 方法被废除，采用 `'key' in {}` 这种写法

5.  全局方法 `xrange()`

   在 Python 2 中存在 `range()` 和 `xrange()` 方法，其中`range()` 返回列表， 而 `xrange()` 返回迭代器，在 Python 3 中 `xrange()` 方法被移除，但是 `range()` 方法返回迭代器

6. 迭代器替代列表

   下面的方法在 Python 2 中返回 list 列表，在 Python 3 中返回的是迭代器。具体如下如下所示：

   - `dict.keys()`
   - `dict.items()`
   - `dict.values()`
   - `zip()`
   - `filter()`
   - `map()`
   - `range()`

7. 模块重新组织

   在 Python 3 中，部分模块的位置被重新组织，包括 `http` , `urllib` , `dbm` , `xlmrpc` 等，一些第三方工具会帮助你自动进行转换 

8. next() 方法

   在 Python 2 中，使用迭代器的 `next()` 方法获取下一个值，使用类似如下所示 `iter.next()` ，在 Python 3中存在全局方法 `next()` ， 使用类似如下所示 `next(iter)` 

上面列举是的是 Python 2 升级 Python 3 中主要的一些问题，具体可以参考 [官方文档](https://docs.python.org/3/howto/pyporting.html) 了解更多区别

## 全新特性

这部分是 Python 3 提供的一些有用新的语法和功能特性，如果切换了 Python 3， 不妨应用这些新特性，可以提升开发效率

#### async 和 await

在 Python 2 中，协程的使用主要依赖 yield 语法，在 Python 3.5 以后，引入了新的协程语法，async 和 await，利用 async 和 await 可以更好地编写异步任务。很多第三方库协程也利用了此语法进行实现，这也是比较多第三方库只支持 Python 3.4+的原因。

async 的基础使用如下所示：

```python
import asyncio

async def main():
	  print('hello')
		await asyncio.sleep(1)
		print('world')
    
>>> asyncio.run(main())
```

详细的使用可以参考下官方文档[PEP-0492](https://www.python.org/dev/peps/pep-0492/) 以及 [Python 文档](https://docs.python.org/zh-cn/3/library/asyncio-task.html)

####  类型检查 Type hints

在 Python 3.5 中引入了一个新的功能，对方法的输入和输出参数类型进行标记，方便进行静态检查，尽早发现问题。这个功能特性是在 [PEP-0484](https://www.python.org/dev/peps/pep-0484/) 引入，基础的使用如下所示：

```python
def greeting(name: str) -> str:
    return 'Hello ' + name
```

如果实际传入或返回的参数类型出错，这边会直接报错

#### 字面字符串 Literal String

在 Python 3.6 中引入一个新的字符串格式化方法， fstring , 具体参照[PEP-0498](https://www.python.org/dev/peps/pep-0498/) ，提供了一种更加便利的字符串格式化方法，使用方式如下所示：

```python
import datetime

name = 'Fred'
age = 50
anniversary = datetime.date(1991, 10, 12)
print(f'My name is {name}, my age next year is {age+1}, my anniversary is {anniversary:%A, %B %d, %Y}.')
```

最终展示的结果是：

```python
'My name is Fred, my age next year is 51, my anniversary is Saturday, October 12, 1991.'
```

可以看到，这行方式可以直接插入变量，使用起来更加便利

#### 拓展的可迭代解包

这个是在 Python 3.0 中引入的，具体是[PEP-3132](https://www.python.org/dev/peps/pep-3132/) ，使用起来类似如下所示：

```python
>>> a, *b, c = range(5)
>>> b
[1, 2, 3]
```

可以看到通过 * 语法，可以将迭代器切分解包至特定的变量中

## 总结

上面是我碰到的一些 Python 2 与 Python 3 使用中的一些需要克服的不一致的情况，以及一些不错的新特性，可能不一定全面，有兴趣的话，可以去 Python 官方网站全面了解一下，全面拥抱 Python 3。