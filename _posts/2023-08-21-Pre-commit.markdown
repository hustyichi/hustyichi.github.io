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
一般而言，统一代码首先需要定义具体的规范，这个每个团队都有所不同，我们选择的规范是 yapf + mypy + isort，定义后规范之后如何规范团队保证标准的实施呢？

1. 鼓励 IDE 中对应的插件的安装，通过直接对应的插件，在编写代码阶段就能实时发现不符合规范的情况，修改成本最低；
2. 通过 Pre-commit 在创建 commit 时执行检查，比进行必要的自动格式化，成本次之；
3. 在发起 Pull-Request 时拉取代码执行检查，并异步返回检查结果，成本稍高一些，但是功能也更完备一些；

而本次主要介绍的就是基于 Pre-commit 进行必要的代码检查与格式化，期间遇到一些或多或少的问题，帮助后人少踩坑吧。

## Pre-commit 简单介绍
[Pre commit](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) 是 git 提供的预提交机制，可以在创建 commit 之前执行预定义的钩子程序，从而方便执行必要的代码检查。

而在实践中可能会需要执行大量的钩子程序，如果来管理这些钩子程序呢，[Pre-commit](https://pre-commit.com/index.html#intro) 就是其中一个应用较多的框架，通过这个框架可以比较方便地管理大量的预提交钩子程序，这样简化了维护成本。如何来使用 Pre-commit 框架呢？

1. [安装 Pre-commit](https://pre-commit.com/#install) 框架，一般情况下 pip 安装下即可；
2. 在工程中添加 `.pre-commit-config.yaml` 文件，需要安装的钩子程序都是维护在这个配置文件中的；
3. 通过 `pre-commit install` 安装对应的钩子程序；

后续在创建 commit 时就会依次安装完成的钩子程序，如果不符合钩子程序对应的规范，则会认为检查失败，commit 无法创建，类似如下所示：

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

在配置文件中主要关注下面的字段：

- `repo`：指定钩子对应的代码库
- `rev`：指定代码仓库对应的版本
- `hooks`：指定代码库中需要需要用到的钩子

Pre-commit 支持的完整的所有的代码仓库与对应的钩子可以见官方的 [Supported Hooks](https://pre-commit.com/hooks.html)

## 具体实践
项目中使用的是 yapf + isort + mypy 的组合，下面的对各个库的实践过程进行具体介绍:

#### yapf
项目中使用的 yapf 对应的配置是定义在 `poetry.yaml` 中的，使用 Pre-commit 时希望直接使用 `poetry.yaml` 中 yapf 对应配置，这样直接调用 yapf 和通过 Pre-commit 执行yapf 对应的格式化方式就是统一的。

在实际配置上 yapf 后，，定义的配置文件如下所示：

```
- repo: https://github.com/google/yapf
  rev: 'v0.31.0'
  hooks:
    - id: yapf
```

测试 Pre-commit 时报错 `toml package is needed for using pyproject.toml as a configuration file`，定位问题时找到了[github issue](https://github.com/pre-commit/mirrors-yapf/issues/15)，看起来是因为 Pre-commit 安装的钩子是放在独立的虚拟环境里面的，这个虚拟环境可能会不存在 toml 包，因此执行 yapf 报错，解决方案就是通过 additional_dependencies 指定对应的包依赖即可，修改配置后即可正确执行，修改后如下所示：

```yaml
- repo: https://github.com/google/yapf
  rev: 'v0.31.0'
  hooks:
    - id: yapf
      additional_dependencies: [toml]
```

