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
统一代码风格首先需要定义参照的规范，每个团队可能会有自己的规范，我们选择的规范是 yapf + mypy + isort，如果保证所有的研发人员都遵循相关规范呢？

1. 鼓励 IDE 中对应的插件的安装，通过直接对应的插件，在编写代码阶段就能实时发现不符合规范的情况，修改成本最低；
2. 通过 Pre-commit 在创建 commit 时执行检查，并进行必要的自动格式化，提供统一的规范约束，成本次之；
3. 在发起 Pull-Request 时拉取代码执行检查，并异步返回检查结果，成本稍高一些，但是功能也更完备一些，不仅能可以进行静态检查，也可以进行必要的自动化测试；

而本次主要介绍的就是基于 Pre-commit 进行必要的代码检查与格式化，期间遇到一些问题，整理出来帮助后人少踩坑吧。

## Pre-commit 简单介绍
[Pre commit](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) 是 git 提供的预提交机制，可以在创建 commit 之前执行预定义的钩子程序，从而方便执行必要的代码检查。

而在实践中可能会需要执行大量的钩子程序，如果来管理这些钩子程序呢，[Pre-commit](https://pre-commit.com/index.html#intro) 就是其中一个应用较多的框架，通过这个框架可以比较方便地管理大量的预提交钩子程序，这样简化了维护成本。如何来使用 Pre-commit 框架呢？

1. [安装 Pre-commit](https://pre-commit.com/#install) 框架，一般情况下 pip 安装下即可；
2. 在工程中添加 `.pre-commit-config.yaml` 文件，需要安装的钩子程序都是维护在这个配置文件中的；
3. 通过 `pre-commit install` 安装对应的钩子程序；

后续在创建 commit 时就会依次执行安装好的钩子程序，如果不符合钩子程序对应的规范，就会检查失败，commit 无法创建，类似如下所示：

![demo](/img/in-post/pre-commit/demo.png)

Pre-commit 的使用主要关注的是配置文件 `.pre-commit-config.yaml` 的定义，配置文件的一个简单例子如下所示：

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
- `hooks`：指定代码库中需要用到的钩子

Pre-commit 支持的完整的所有的代码仓库与对应的钩子见官方 [Supported Hooks](https://pre-commit.com/hooks.html)

## 具体实践
项目中使用的是 yapf + isort + mypy 的组合，之前已经在工程中安装完成，包管理是使用 poetry 实现的，因此格式化工具对应的配置都是定义在 `poetry.yaml` 文件中的。本次使用 Pre-commit 期望也能直接使用原有格式化工具的配置，从而保证与定义好的规范保持一致。

#### yapf
本次在 Pre-commit 中使用 yapf 时，配置文件 `.pre-commit-config.yaml` 中的定义如下所示：

```
- repo: https://github.com/google/yapf
  rev: 'v0.31.0'
  hooks:
    - id: yapf
```

在 Pre-commit 中使用 yapf 时，报错 `toml package is needed for using pyproject.toml as a configuration file`，但是直接调用 yapf 时可以正常执行的。

定位问题后发现，Pre-commit 安装的钩子是放在独立的虚拟环境里面的，这个虚拟环境中没有对应的 toml 包，因此执行 yapf 报错

解决方案就是通过 additional_dependencies 指定对应的包依赖，这样依赖的包才能被正确安装，修改配置后即可正确执行，最终配置如下：

```yaml
- repo: https://github.com/google/yapf
  rev: 'v0.31.0'
  hooks:
    - id: yapf
      additional_dependencies: [toml]
```

#### isort
isort 主要用于进行代码 import 顺序的调整，定义的配置如下所示：

```
- repo: https://github.com/PyCQA/isort
  rev: 5.12.0
  hooks:
    - id: isort
```

#### mypy
项目中主要是使用 mypy 进行静态的代码检查，初始定义的配置如下所示：

```
- repo: https://github.com/pre-commit/mirrors-mypy
  rev: 'v0.910'
  hooks:
  -   id: mypy
```

增加测试代码进行验证后发现问题很多：

1. Pre-commit 中使用的 mypy 虽然是增量提交的，但是很多没有修改的文件中的问题也被提醒出来，导致需要修改的文件特别多；
2. 原本在 `pyproject.toml` 中 mypy 配置中明确排除掉的文件中的问题也会上报出来，而手工执行 mypy 是被正常忽略的；

对于问题 1 定位后发现 mypy 增量提交时会递归对 import 导入的文件同时进行静态类型检查，从静态类型工具的角度是可以理解的，因为需要确认调用方和定义的函数类型是否一致，因此需要递归导入和检查，但是这样就会导致增量提交失去意义，对于已有工程而言在发起新提交时会需要修改大量的文件。

问题 2 的原因其实也与这个 import 循环导入有关，mypy 配置时通过 `exclude` 参数排除掉文件，在 mypy 全量检查时会跳过，但是如果是增量提交，通过 import 导入的文件依旧会执行静态类型检查，此时 `exclude` 就没办法排除掉了

对于提到的这两个问题，github 上有不少人给 mypy 上报了异常，甚至原有 `exclude` 参数不能排除掉 import 文件的机制设计，有开发者提出了 [force exclude](https://github.com/python/mypy/pull/12373) 的 PR，但是截止目前而言，这个想法没有被现有 mypy 的维护者认可

从 mypy 维护者的解释来看，mypy 作为静态类型检查工具，是需要结合执行上下文来尽可能发现不符合静态类型定义的问题，force exclude 会导致没办法根据调用上下文发现类型不匹配的问题，mypy 就失去了意义。从 mypy 作为静态类型检查工具的角度来看，这个解释没有太大问题，但是在 Pre-commit 中使用 mypy 确实就会出现上面所说的那些问题，导致根本不可用。因此 mypy 维护者提出了 [建议解决方案](https://github.com/python/mypy/issues/13916)，方案的解决思路如下：

1. mypy 开启对整个工程的代码检查，即传递参数 `pass_filenames: false`，不要使用增量式传递新增文件的方式；
2. pre-commit 使用独立的虚拟环境去安装 mypy，会导致第三方库的类型检查失效，建议直接使用原有运行虚拟环境中的 mypy 进行检查，通过 `language: system` 进行配置；

最终配置定义如下：

```
- repo: https://github.com/pre-commit/mirrors-mypy
  rev: 'v0.910'
  hooks:
  - id: mypy
    entry: mypy .
    language: system
    pass_filenames: false
```

测试确实能解决掉原先 exclude 不生效的问题，但是由于目前是全量检查，因此需要先对工程中原有的不符合 mypy 规范的进行了修复后，再开启对应的 mypy 检查。虽然解决方案不够完美，需要先进行一轮全局的修复，但是修复后工作良好。

## 总结
通过上面的配置调整，最终在工程中正常配置了 Pre-commit，保证了团队代码风格的一致性，Pre-commit 通过将必要的规范限制在开发环境，保证了对开发人员的统一风格约束，从而提升整体代码质量，有兴趣可以尝试一下
