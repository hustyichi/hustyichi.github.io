---
layout: post
title: 'Python虚拟环境管理'
subtitle:   "Virtual environment in python"
date:       2018-09-16 21:43:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
---

## Python 虚拟环境管理
#### Why
管理过多个项目都知道，每个项目会有不同的依赖关系，将这些依赖关系全部保存在全局环境是不切实际的，而且不同项目的依赖关系可能会冲突，比如有些项目依赖python2，有些项目依赖的是python3。因此比较好的方案就是，为每个项目设置隔离的虚拟环境，所需的依赖关系都保存在各自的虚拟环境中，避免互相冲突。这就是为什么我们需要虚拟环境管理的原因。

#### How
早期的Python虚拟环境管理方法就不过多介绍了，大家有兴趣可以自行搜索。本篇文章主要介绍目前Python下依旧在使用，而且广泛使用的虚拟环境管理方法。主要是virtualenv, virtualenvWrapper, pipenv。事实上，这几种虚幻环境的管理都是基于virtualenv，只是封装得更好，功能更强大而已。下面我们会依次进行介绍

## virtualenv
#### 创建虚拟环境
使用virtualenv时，可以先跳转至项目目录`project_path`，然后执行virtualenv 创建虚拟环境

```
cd project_path
virtualenv virtualenv_name
```
此命令会在项目目录下创建一个文件夹，名字叫做`virtualenv_name`，此目录下会包含python拷贝，之后所有的依赖都会保存至此目录中

#### 激活虚拟环境

```
source virtualenv_name/bin/activate
```
通过此命令可以激活虚拟环境，如果正确激活，在终端左侧可以看到`(virtualenv_name)`的提示，在虚拟环境激活状态下，可以安装所需的依赖包，安装的依赖包会保存至项目虚拟环境目录`virtualenv_name `下，不会污染系统全局环境

#### 离开虚拟环境
```
deactivate
```
当完成所需任务后，希望离开当前虚拟环境，可以执行`deactivate`，这样就会离开当前虚拟环境。之后再安装依赖包，就会安装至系统全局环境中

## virtualenvwrapper
virtualenvwrapper是对virtualenv接口的封装，使用起来更加便利。virtualenvwrapper会将虚拟环境的目录统一保存，不需手动管理，使用起来也轻松很多。首先需要进行安装，可以参考[官方安装文档](https://virtualenvwrapper.readthedocs.io/en/latest/install.html)

#### 创建虚拟环境
```
mkvirtualenv virtualenv_name -a project_path
```
在任意目录下执行命令创建虚拟环境，并通过 -a 指定项目路径，后续激活虚拟环境时可以直接跳转至项目路径。使用virtualenvwrapper不需要关心虚拟环境保存的路径，所有采用virtualenvwrapper创建的虚拟环境都会保存至统一的路径

#### 激活虚拟环境
```
workon virtualenv_name
```
使用`workon`命令可以激活虚拟环境，由于通过 -a 指定了项目目录，在激活虚拟环境时，会跳转至项目目录，方便了很多 

#### 离开虚拟环境
```
deactivate
```
当使用结束后，可以采用deactivate离开虚拟环境，与virtualenv使用一样

## pipenv
pipenv是K神最新的项目，可以更加方便地管理虚拟环境，同时也能提供更加强大的包管理能力。因为之前的virtualenv和virtualenvwrapper在安装了依赖关系后，如果希望可以将依赖关系保存下来，都会使用`pip freeze > requirements.txt`，将依赖关系保存至`requirements.txt`文件中。这样其他人就可以利用此文件通过`pip install -r requirements.txt`将所有的依赖关系安装好。pipenv结合pip与virtualenv的优点，可以更加方便与强大地保存依赖关系，同时可以解决仅仅使用requirements.txt保存依赖关系的缺陷，具体可以看看[K神的演讲](https://www.youtube.com/watch?v=GBQAKldqgZs)，自行备梯子一睹K神风采

#### pipenv使用
```
cd project_path
pipenv install xxx
```
使用pipenv时，不需要手动创建虚拟环境，也不需要额外激活虚拟环境，直接安装所需的依赖关系，pipenv会在项目目录下创建Pipfile和Pipfile.lock文件，用于保存依赖关系。

当代码需要在虚拟环境执行时，通过`pipenv run python xx.py`，即可在虚拟环境下执行python文件，因此就不需要显示激活虚拟环境，然后再离开虚拟环境。如果需要在当前命令行洗持续执行虚拟环境下任务，可以通过pipenv shell 生成新的shell，此shell即处于虚拟环境激活状态，可以持续在虚拟环境下执行任务。

