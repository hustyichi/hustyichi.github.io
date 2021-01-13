---
layout: post
title: '基于 cookiecutter 的 python 项目模板'
subtitle:   "Python project template based on cookiecutter"
date:       2021-01-12 13:21:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python

---

## Cookiecutter 介绍

使用 Python 这种动态语言进行 web 开发，团队中经常会遇到的问题就是代码的质量比较难控制。Python 语言本身灵活性比较高，不加控制的情况下代码质量可能最后很难维护。而且代码的各方面的标准，比如提示的 lint，代码格式化等等如果不加规范，不同开发者的协作也会比较困难。因此从公司角度来看，一个统一的代码规范是很有必要的。

在 Python 的开发过程中，可以选择公司希望遵循的 Python 开发相关的规范，那么如何更好地将这些规范组织起来呢？

一个比较好的方案就是 cookiecutter，cookiecutter 是一个用于构建工程模板的 python 库，利用 cookiecutter 可以构建一个标准统一的起步 python 项目，同时在构建中还具备一定的灵活性，根据自己需要选择合适配置。下面就对 cookiecutter 进行简要介绍：

#### Cookiecutter 简单实践

[Cookiecutter](https://github.com/cookiecutter/cookiecutter) 是一个跨平台的 Python模板组织工具，可以极其简单地生成一个 Python 起步项目。

使用前需要先安装 cookiecutter，各种安装方式都是支持的，Mac 上可以通过 `brew` 或者 `pip` 安装。

首先你需要寻找符合你要求项目模板，在 github 上通过 cookiecutter 进行搜索即可找到大量的备选模板，我们这边以 cookiecutter 官方维护的一个模板 [cookiecutter-pypackage](https://github.com/audreyfeldroy/cookiecutter-pypackage) 为例进行介绍，找到此模板对应的地址 `https://github.com/audreyfeldroy/cookiecutter-pypackage` 

之后直接执行 `cookiecutter https://github.com/audreyfeldroy/cookiecutter-pypackage` 即可，在执行中会有一些可供选择的配置型，如果不确定可以使用默认值，一路确认下来，你需要的起步项目就完成了。

如果此模板不能完成满足你的需要，你可以选择在构建完成后对工程进行小幅修改，如果需要将此模板推广给全公司使用，直接基于此模板构建一个新的模板进行修改，之后再重新生成工程即可。

#### Cookiecutter 模板

下面可以对 cookiecutter 模板的组织结构进行简单介绍：

```
cookiecutter-something/
├── {{ cookiecutter.project_name }}/  <--------- 项目模板，生成的工程就是这个目录下内容
│   └── ...
├── blah.txt                      <--------- 模板工程之外的文件
│
└── cookiecutter.json             <--------- 提示以及默认值的配置文件，可变的变量都存储在这个文件中
```

通过上面的模板可以生成项目模板目录下的内容，项目的变量名会被输入的内容替换。

另外 cookiecutter 还支持一些更高级的功能，比如前置或后置的检查 Hooks，以及注入实时上下文等，可以根据自己需要进行使用

## 一个合适的项目模板

对于公司而言，有一个标准的项目规范，可以帮助大家统一代码风格，提高整体的代码质量，我们参考了各家的标准化建议，最终提出了一个合适的代码规范，主要包含下面的一些部分：

#### lint

lint 是用于检测代码，有助于团队规则的统一，同时借助 lint 可以规避一些常见的坑。我们最终选择的 lint 是：

- pylint  pylint 基本上是 python 的 lint 中最常见的 lint 了，限制也比较严格，我们根据需要关闭了 `"missing-docstring"`, `"logging-fstring-interpolation"` 
- mypy 此 lint 是 python 中常见的类型检查 lint，通过此 lint 要求方法指定参数的类型

#### 代码格式化

代码格式化是用于保证代码风格的统一，代码的格式化也存在多种选择，我们最终选择了：

- black 这个格式化工具的有名之处在于可配置项比较少，基本上通过去除灵活性来换取高度统一性的代码风格，但是 black 竟然受到了大量 python 程序员的喜爱，可能是动态语言的灵活性让大家更想要某种确定的规范吧
- isort 这是用来调整 python 包的 import 顺序的，按照系统库，第三方库，本地库的顺序进行分开排布

#### 单元测试

单元测试用于保证代码的正确性，我们选择了评价相对更好的：

- pytest 相对于默认的 unittest, pytest 显然更受欢迎，pytest 使用更加便利，同时不仅能支持单元测试，还能支持面向应用的测试
- coverage 我们使用 coverage 用于计算代码单元测试覆盖率，方便保证代码的充分测试

#### 依赖管理

- requirements 在 python 项目中，比较常见的是通过 requirements.txt 进行管理，但是默认的包管理不支持管理直接依赖和间接依赖，因此需要通过 pip-compile 进行区分，而且开发相关的依赖也需要另外通过 requirements-dev.txt，这种管理方式比较不友好，但是考虑到大部分 python 程序员都熟悉这种情况，也可以提供了这种方式
- poetry 这是一种更新的包管理方式，可以比较好地组织好依赖关系，而且已经被纳入 PEP 518 标准了，建议优先采用此方式管理包。同时 poetry 也包含了虚拟环境的管理，使用更加便利

按照上面的规范最终构建的 cookiecutter 模板可以参考 [github](https://github.com/hustyichi/cookiecutter-python-standard) 



## 更多模板

cookiecutter 的使用过程中，上手还是比较简单的，cookiecutter 无非是一个动态替换的机制，没有什么特殊的复杂性。从我的使用的感受来看，cookiecutter 的一个最大价值在于有大量的开发者已经提供了一些现成的工程模板，而且我们可以相对简单地通过已有的工程模板生成新的工程模板，并在团队内部统一规范。

可以参考 [cookiecutter](https://github.com/cookiecutter/cookiecutter/tree/db14e06a1dcc0187beeafde72685c3acef93eb68#a-pantry-full-of-cookiecutters) 提供的大量的代码模板根据自己的需要选择一个基础模板，然后进行必要的修改即可得到所需的上手代码模板。进而生成所需的项目工程。比如对于上面提供的项目模板，如果需要增加相关文档的支持，可以引入 [Sphinx](https://www.sphinx-doc.org/en/master/) ，之后重新生成项目模板即可。



## 总结

从过往的经验来看，有两点总结：

- 项目规范化很有必要，统一的规范省心省力
- 不要重复造轮子，尽量使用成熟方案

使用 cookiecutter 就可以轻松实现