---
layout: post
title: 'Linux信号机制与docker应用'
subtitle:   "Linux signal in docker"
date:       2018-07-15 22:30:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - docker
    - k8s
    - linux
---

## Linux任务终止信号
在linux系统中，为了终止任务，一般情况下，可能会涉及如下所示的几种信号：

- #### SIGTERM 
SIGTERM是终止命令kill默认的终止信号。此信号是由应用程序捕获的，使用SIGERM也让程序有机会再退出之前做好清理工作，从而优雅地终止

- #### SIGKILL
这是一种不可被捕获或忽略的信号，这是一种可以可靠地杀死进程的方法。由于此信号不会被捕获，可以认为可以很稳定地杀死进程

- #### SIGINT
这是用户在按下中断键（一般是ctrl + c）时，终端驱动程序产生的信号，此信号会被发送给前台进程组，一般用于杀死前台的进程

- #### SIGQUIT
这是用户按下退出键（一般是ctrl + \）时，终端驱动程序产生信号，此信号会被发送给前台进程组，此信号会结束前台进程，同时产生core文件

## signal in docker
在docker使用中，如果想要结束特定的容器一般可以采用docker stop 或 docker kill，那么他们的区别在哪里呢，其实主要的区别在于他们会发送不同的信号

- #### docker stop
docker stop是一种相对优雅的终止任务的方法，此命令会发送SIGTERM信号，正常情况下应用程序会做好清理然后退出，在一段时间(grace period)之后，如果进程没有终止，那么会发送SIGKILL强制结束任务

- #### docker kill
docker kill 则比较暴力了，直接发送SIGKILL结束任务，简单高效，但是可能导致没法执行数据清理导致的问题

事实上还有一个指令也会导致任务的结束，docker rm -f，一般情况下docker rm用户删除已经停止的容器，而加上 -f 则可以终止运行中的容器。此命令事实上也是发送SIGKILL信号，因此也是暴力直接型的

## signal in kubernetes(k8s)
k8s是目前比较火热的多主机的容器化应用，可用通过配置文件进行多主机，多容器的部署与编排，k8s底层是依赖于docker的。在k8s中，更新或关闭pod使用了linux相关的信号机制。下面是关闭时的流程：

> 1. A SIGTERM signal is sent to the main process (PID 1) in each container, and a “grace period” countdown starts (defaults to 30 seconds).
> 2. Upon the receival of the SIGTERM, each container should start a graceful shutdown of the running application and exit.
> 3. If a container doesn’t terminate within the grace period, a SIGKILL signal will be sent and the container violently terminated.

简单说来，k8s中关闭容器时，首先会发送SIGTERM信号，容器收到此信号后，应该做好清理然后退出，在等待一段时间之后（默认是30s），没有退出的容器会收到SIGKILL信号，然后容器就会被强制杀死

因此使用k8s部署的应用中，任务不要过长，否则更新代码时，如果任务不能在30s内完成并退出，可能导致任务处于异常状态。那么如果已经存在长时任务，那么解决方案就是调整SIGTERM到SIGKILL中间等待的时间，将其设置得更长。此时可以调整terminationGracePeriodSeconds参数即可完成。在pod的编排中，设置terminationGracePeriodSeconds: 180，即可将默认等待时间修改为3分钟。
