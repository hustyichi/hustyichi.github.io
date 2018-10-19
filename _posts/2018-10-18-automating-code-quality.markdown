---
layout: post
title: '自动化代码质量控制'
subtitle:   "automating code quality"
date:       2018-10-18 22:43:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python

---

## 自动化代码质量控制
代码质量是程序员都比较关心的一件事，如何控制团队的代码质量，靠人工是一件相当难的事情。稍微放松一些，代码质量都惨不忍睹了，最近看了下[Kyle Knapp的演讲](https://www.youtube.com/watch?v=G1lDk_WKXvY&t=9s)，如何利用各种自动化的方式来保证python代码质量，深以为然，记录分享一下。

## 代码质量控制工具
在演讲中，分享了三种工具用于控制代码质量，下面分别进行介绍：

#### flake8
flake8 是一个相当方便而且简单的代码质量检测工具，安装和使用的基本操作可以参照[官方文档](http://flake8.pycqa.org/en/latest/index.html#quickstart)，大家采用pip安装之后，直接执行`flake8 path/to/code`就可以检测项目代码，如果有不符合规范的，flake8就会提示，大家按照提示修改就好啦。运行情况类似：
![flake8](/img/in-post/code-quality/flake8.jpg)

flake8其实是对三个功能的封装，分别如下所示：

1. pycodestyle, python代码风格检查，对于一些不pythonic的代码可以检查出来，比如代码中个空格，代码中使用is_even == True 这种不pythonic的写法，会检测出来
2. pyflakes, 用于检测代码中潜在的bug，比如定义了变量没有使用，引入了没使用过的库等可能隐藏bug的问题  
3. mccabe, 用于检测代码的复杂度，当代码复杂度过高时，比较容易导致bug，因此mccable对复杂度过高的代码进行提示。允许的最大复杂度可以自由设置，当超过最大复杂度，mccable就会警告，建议配置最大复杂度为10

#### pylint
pylint与flake8类似，同样会检测代码质量，对代码风格问题发出警告，帮助检测潜在的bug，一帮情况下比flake8更加严格，但是也更加武断，比如会要求不能使用驼峰变量名，方法前必须方法文档，对一些变量名也会有限制。一般情况下，我们可以根据需要进行配置，从而关闭一些提示。细节使用可以参照[官方文档](http://pylint.pycqa.org/en/latest/)

pylint与flak8有一些共同的部分，也各自有一些自己的部分，两者的比较如下所示：
![pylint-flake8](/img/in-post/code-quality/pylint-flake8.jpg)

可以看到，两者共同的部分是PEP8，以及无效引用，pythonic代码以及变量引用检测部分，除此之外，flake8还有代码复杂度检测，而pylint还有变量名检测，危险模式检测，更严格的错误检测，同时也支持更多自定义配置

#### coverage
代码覆盖率检查，与pytest配合使用，可以确定代码测试覆盖率，帮助我们更好地测试代码。使用也很方便，不仅支持代码覆盖率，还会有测试覆盖的代码行数，方便查看到底哪些代码没有被覆盖到。详细使用可以参考[官方文档](https://coverage.readthedocs.io/en/v4.5.x/)。项目中测试后生成了html格式的统计数据，可以看到效果如下所示：
![coverage_total](/img/in-post/code-quality/coverage_total.jpg)

这个是对不同文件的统计数据的展示，可以方便看到不同文件的覆盖率情况。
![coverage](/img/in-post/code-quality/coverage.jpg)

这个是单个文件的覆盖情况，可以看到飘红即时没有覆盖的代码，目前的代码中没有对raise Error的情况测试到，使用coverage可以帮助我们发现这个问题，从而尽早修复这个问题，从而尽早发现可能隐藏的bug

#### mypy
在演讲中，还有一个被提到的代码质量控制工具就是mypy，这是python3中自带的类型检测工具，可以指定方法的输入和返回值类型，mypy会静态对代码的输入和返回值的类型进行检查，如果两者不相符，那么就可以提前报错。从而提升代码的可靠性

## 本地环境代码质量控制
在本地环境进行代码质量控制时，有了前面这个工具，如何将其更有效和更方便地使用起来呢，在演讲中提到两个tips

#### requirements-dev.txt
使用python的大家都应该会比较了解requirements.txt，使用pip安装了包后，为了保证线上也能正常部署起来，一般会通过`pip freeze > requirements.txt`，将安装的包列表记录下来，后面线上就可以通过`pip install -r requirements.txt`安装项目依赖的包，从而可以正常运行起来。

而requirements-dev.txt其实是说，因为代码质量控制会安装一些新的包，这些是线上部署不需要依赖的，此时可以添加额外的包列表requirements-dev.txt，里面除了线上部署必须的包，也包含代码质量控制的包，从而避免将这些包添加到requirements.txt，从而污染线上的包列表。

#### Makefile
python中也可以使用Makefile执行特定代码检查，从而避免每次需要记住需要执行的代码质量控制相关命令，也方便反复执行。在后续需要执行时，只需要执行特定的make命令即可，方便很多。配置Makefile也很简单，我们在项目中也简单配置如下:

```
lint:    
	flake8 .
	pylint app

coverage: 
	coverage run --source app -m pytest
	coverage report -m
	coverage html
	open htmlcov/index.html
```
在后续使用时，只需要执行`make lint`即可执行flake8和pylint的代码检查，执行`make coverage`执行代码覆盖率的检查，并生成对应的html格式的覆盖率数据

## 项目协作代码质量控制
对于目前的互联网公司，90%以上应该都使用git进行代码管理了，而代码的合并上线大都采用类似github项目上的发起pull request的方式进行提交请求，而由资深员工进行code review之后进行合并。这种方式代码的质量控制就取决于code review的资深员工的态度了，如果他控制代码质量比较严格，那个代码质量就有保证，如果他比较随意，那么代码质量就会惨不忍睹。项目中这种协作方式完全依赖人工控制代码质量，看起来是有一些问题的。

对于这种情况，推荐使用travis进行项目中代码质量的控制，travis会对发起的pull request的代码进行指定的检查，对于检查不通过的代码，是无法被合并的。也就是说，将过去那种完全依赖的人工的方式修改为依赖自动检查+人工的方式，从而减轻code review的压力，将一些低级错误尽早暴露出来。对于检查不通过的pull request，如下所示：
![travis](/img/in-post/code-quality/travis.jpg)

而travis执行的检查可以是使用flake8, pylint和coverage来完成的，同时也可以借助[Codecov](https://codecov.io/)进行pull request代码测试覆盖率的展示，从而方便发起pull request的人修改自己的代码的问题。

## 自动化代码质量控制的好处
1. 机器比人处理更快，也更加准确；
2. 增加新的质量检查很方便，如果发现新的代码检查需求，直接增加至Makefile中即可；
3. 代码必须要先通过质量检查，才能被合并；
4. 减少代码review的时间；

## 最佳实践
1. 持续提升代码质量检查；
2. 在现有的代码质量检查前不要轻易妥协；
3. 自动化的代码质量检查不能保证代码质量，必须还有人工的code review的配合；
