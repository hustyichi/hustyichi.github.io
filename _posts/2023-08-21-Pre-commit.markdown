---
layout: post
title: '基于 Pre-commit 的代码风格统一实践'
subtitle:   "Unified Code Style Practice Based on Pre-commit"
date:       2023-08-21 09:00:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - Pre-commit
---

## 背景信息
一般而言，统一代码首先需要定义具体的规范，这个每个团队都有所不同，我们选择的规范是 `yapf` + `mypy` + `isort`，定义后规范之后如何规范团队保证标准的实施呢？

1. 鼓励 IDE 中对应的插件的安装，通过直接对应的插件，在编写代码阶段就能实时发现不符合规范的情况，修改成本最低；
2. 通过 Pre-commit 在创建 commit 时执行检查，比进行必要的自动格式化，成本次之；
3. 在发起 Pull-Request 时拉取代码执行检查，并异步返回检查结果，成本稍高一些，但是功能也更完备一些；

而本次主要介绍的就是基于 Pre-commit 进行必要的代码检查与格式化，期间遇到一些或多或少的问题，帮助后人少踩坑吧。

## Pre-commit 简单介绍
[Pre commit](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) 是 git 提供的预提交机制，可以在创建 commit 之前执行预定义的钩子程序，从而方便执行必要的代码检查。

而在实践中可能会需要执行大量的钩子程序，如果来管理这些钩子程序呢，[Pre-commit](https://pre-commit.com/index.html#intro) 就是其中一个应用较多的框架，通过这个框架可以比较方便地管理大量的预提交钩子程序，这样简化了维护成本。如何来使用 Pre-commit 框架呢？

1. [安装 Pre-commit](https://pre-commit.com/#install) 框架，一般情况下 pip 安装下即可；
2. 在工程中添加 `.pre-commit-config.yaml` 文件，之后所有的钩子程序都是维护在这个配置文件中的；
3. 通过 pre-commit install 安装对应的钩子程序；

后续在创建 commit 时就会依次执行文件中定义的钩子程序，如果不符合定义的规范，则会检查报错，commit 无法创建，类似如下所示：

![demo](/img/in-post/pre-commit/demo.png)

因此 Pre-commit 的使用主要关注的是配置文件 `.pre-commit-config.yaml` 的定义，配置文件的一个简单例子如下所示：

```yaml
repos:
-   repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v2.3.0
    hooks:
    -   id: check-yaml
    -   id: end-of-file-fixer
    -   id: trailing-whitespace

-   repo: https://github.com/psf/black
    rev: 22.10.0
    hooks:
    -   id: black
```

事实上，大部分情况下最终版本也只需要了解这几个字段概念即可:

- `repo`：指定钩子对应的代码库
- `rev`：指定对应的库版本
- `hooks`：指定代码库中使用的钩子
