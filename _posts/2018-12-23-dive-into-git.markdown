---
layout: post
title: 'Git笔记'
subtitle:   "dive into git"
date:       2018-12-23 15:31:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - tools
---

## 基础概念

使用过git的都会有一些基础的感觉，git管理的代码会涉及到几个存储区域。具体如下图所示：

![repository](/img/in-post/git/repository.jpeg)

其中包含三个不同的的代码存储区域：

- 工作区，工作区就是可视的代码目录。一般情况下，我们修改了代码就是更改工作区的代码
- 暂存区，对于需要提交的代码，会通过`git add` 加入暂存区
- 代码分支，进行实际提交时，使用`git commit` 将暂存区中的代码一次性加入代码分支

比较上面三个存储区域的代码的不同之处可以使用下面的命令：

- `git diff` 比较工作区和暂存区的不同，可以工作区中没有被暂存的修改
- `git diff --cached` 比较暂存区和代码分支的不同，可以看到被暂存但是没有被合并的修改
- `git diff HEAD` 比较工作区和代码分支的不同，可以看到全部没有合并的修改

![diff](/img/in-post/git/diff.png)

对照于上面的概念，git中代码文件的生命周期如下所示：

![lifecycle](/img/in-post/git/lifecycle.png)

分别对代码文件的状态进行解释：

- Untracked,  未追踪，git中新增或新删除的代码文件就处于未追踪状态。
- Unmodified, 未修改，表示代码已经被合并进代码分支，是最终状态
- Modified, 已修改，表示工作区中的代码文件已经被修改，但是没有被暂存，也没有加入代码分支
- Staged, 已暂存，表示工作区中的代码文件已经被暂存，准备合并进代码分支

一般情况下，新文件的状态生命周期为：

**未追踪 > 已暂存 > 未修改**

旧文件的状态生命周期为：

**未修改 > 已修改 > 已暂存 > 未修改**

## 命令理解

了解上面的基础概念之后，之前的一些看起来模棱两可的命令就可以更好地理解了。

#### reset 比较

在使用git时，如果想要发现修改不符合预期，可能会使用`git reset` 执行重置，执行重置时可能会使用`git reset --soft` , `git reset` , `git reset --hard` , 到底哪一个执行后才是符合预期的呢。根据上面的概念进行比较一下：

- `git reset --soft` 此命令仅重置代码分支中的代码，工作区和暂存区的代码保持不变，此时被重置的代码文件会处于已暂存的状态
- `git reset` 此命令与`git reset --mixed` 等同，此命令会重置代码分支和暂存区的代码，工作区的代码保持不变，此时被重置的代码会处于已修改或未追踪状态
- `git reset --hard` 此命令会重置代码分支，暂存区，工作区的代码，被重置的代码修改会丢失。

通过上面的比较，可以理解，`git reset --soft` 和 `git reset` 都是很安全的，因为修改都是被保存的，而`git reset --hard` 会抛弃所有位置的代码修改，因此是不安全的。

#### checkout

在进行文件修改但是还没有暂存代码时，git可能会提示`git checkout -- <file>` 命令，此命令会做什么呢？它又是否安全呢？

事实上，这种情况下代码修改因为没有加入暂存区，自然也不会进入代码分支中，因此代码修改进行在工作区中，而`git checkout -- <file>` 就是抛弃在工作区中的修改。而执行之后，修改在任意地方都不存在了，因此也很难再恢复。理解此命令所做的事情之后，也可以理解，此命令也是不安全的

## 一些感受

最近闲来无事，看了一下[git官方文档](https://git-scm.com/book/zh/v2/%E8%B5%B7%E6%AD%A5-%E5%85%B3%E4%BA%8E%E7%89%88%E6%9C%AC%E6%8E%A7%E5%88%B6)，对一些基础概念熟悉了一下，感觉一些原来很模糊的命令会更清晰一些。对武侠的内功的理解有一些新的感受，武功再花里胡哨，没有内功一样发挥其威力。git也很多命令，各个命令又会有不同的参数，没办法全部记住，但是理解了一些基础的概念，才能更好地理解运用git的各个命令。技术之路，任重道远。