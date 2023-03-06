---
layout: post
title: 'FATE Flow 源码解析 - 作业提交流程'
subtitle:   "FATE Flow source code analysis - task submission process"
date:       2023-03-08 09:30:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - FATE
    - Federated learning
---

## 背景介绍

Fate 是隐私计算中最有名的开源项目了，从 star 的数量上来看也可以看出来。截止 2023 年 3 月共收获 4.9k 个 star，但是 Fate 一直被认为代码框架复杂，难以理解，作为一个相关的从业者，后续会持续对 Fate 项目的源码进行解析，方便对隐私计算感兴趣的后来者提供一点点帮助。

本文主要基于 FATE-Flow 2022 年 12 月发布的版本 v1.10.0，后续的版本可能略有差异。针对 FATE-Flow 的代码，基于 v1.10.0 的做了一个代码注解的仓库，方便查看具体的代码 [https://github.com/hustyichi/hustyichi.github.io](https://github.com/hustyichi/hustyichi.github.io)

## Fate-Flow 基础介绍

Fate-Flow 是 Fate 项目的重要组成部分，主要用于实现作业的调度，简单理解，我们提交的训练作业会提交给 Fate-Flow，Fate-Flow 会创建对应的作业，并生成多个对应任务，统一进行调度，通过联合多个站点一起进行训练，最终生成对应的结果

Fate-Flow 是作为一个 Web 服务对外提供服务的，对应的初始启动文件为 `FATE-Flow/python/fate_flow/fate_flow_server.py` ，熟悉 flask 的可以看到，最终就是调用 `run_simple` 创建了一个 Web 服务，根据 app 可以找到 Web 服务的主要代码都在 `FATE-Flow/python/fate_flow/apps` 目录下，路由注册的代码如下所示：

```python
# 注册 HTTP 路由，将 Fate-Flow/python/fate_flow/apps 以及 Fate-Flow/python/fate_flow/scheduling_apps 下所有 python 文件

client_urls_prefix = [
    register_page(path)
    for path in search_pages_path(Path(__file__).parent)
]
scheduling_urls_prefix = [
    register_page(path)
    for path in search_pages_path(Path(__file__).parent.parent / 'scheduling_apps')
]
```

可以看到注册的路由主要就是 apps 目录与 scheduling_apps 目录下路由。

一个注意点是：FATE-Flow 是没办法独立运行的，需要作为 FATE 的一部分进行运行。 FATE-Flow 项目部分依赖的代码，比如 `fate_arch` 是存在于 Fate 工程下，对应的路径为 `FATE/python/fate_arch` ，找不到代码时可以联合 Fate 代码仓库进行阅读

## 作业处理流程
作为一个作业调度的服务，最重要的就是完整的处理流程，先厘清这个主线，其他分支就更容易理解了，主要流程如下所示：

#### 作业提交
作业提交是通过 `FATE-Flow/python/fate_flow/apps/job_app.py` 中的 `submit_job` 进行提交的，主要的处理都是通过 `DAGScheduler.submit()` 来完成的，简化版本的代码如下所示：
```python

def submit(cls, submit_job_conf: JobConfigurationBase, job_id: str = None):
    # 没有 id 时默认生成唯一 id

    if not job_id:
        job_id = job_utils.generate_job_id()

    submit_result = {
        "job_id": job_id
    }

    job = Job()
    job.f_job_id = job_id
    job.f_dsl = dsl
    job.f_train_runtime_conf = train_runtime_conf
    job.f_roles = runtime_conf["role"]
    job.f_initiator_role = job_initiator["role"]
    job.f_initiator_party_id = job_initiator["party_id"]
    job.f_role = job_initiator["role"]
    job.f_party_id = job_initiator["party_id"]

    # 通知各个站点 (party) 去创建对应的 job

    status_code, response = FederatedScheduler.create_job(job=job)

    # 更新 job 状态为 WAITING

    job.f_status = JobStatus.WAITING

    # 将 job 状态同步给各个站点（party）

    status_code, response = FederatedScheduler.sync_job_status(job=job)

    return submit_result

```
可以看到提交作业时，主要是在数据库中添加作业表 Job 生成的对应的记录，并将作业的数据与状态同步给各个站点，从而实现联合的训练


#### 资源申请
提交后作业的状态变为 WAITING，在 `DAGScheduler.run_do()` 对 WAITING 状态的作业进行了处理，可以看到如下所示：
```python
def run_do(self):

    # 默认处理 WAITING 状态的第一个创建的 job 进行处理，会分配必要的资源，处理结束状态变为 RUNNING

    jobs = JobSaver.query_job(is_initiator=True, status=JobStatus.WAITING, order_by="create_time", reverse=False)
    if len(jobs):
        job = jobs[0]
        self.schedule_waiting_jobs(job=job, lock=True)
```
可以看到实际处理的方法是 schedule_waiting_jobs() 方法，对应的代码如下所示：
```python
def schedule_waiting_jobs(cls, job):
    job_id, initiator_role, initiator_party_id, = job.f_job_id, job.f_initiator_role, job.f_initiator_party_id,

    # 检查资源依赖关系

    dependence_status_code, federated_dependence_response = FederatedScheduler.dependence_for_job(job=job)
    if dependence_status_code == FederatedSchedulingStatusCode.SUCCESS:

        # 申请相关资源

        apply_status_code, federated_response = FederatedScheduler.resource_for_job(job=job, operation_type=ResourceOperation.APPLY)
        if apply_status_code == FederatedSchedulingStatusCode.SUCCESS:

            # 启动 job 执行，状态更新至 RUNNING

            cls.start_job(job_id=job_id, initiator_role=initiator_role, initiator_party_id=initiator_party_id)

```
在此阶段，会申请作业执行所需的资源，资源申请时会调用依次调用各个站点对应的接口，分配必要的 CPU 与内存资源，对应的接口为 `FATE-Flow/python/fate_flow/scheduling_apps/party_app.py` 中的 `/<job_id>/<role>/<party_id>/resource/apply` 接口，最终调用 `FATE-Flow/python/fate_flow/manager/resource_manager.py` 中的 `resource_for_job()` 方法执行资源的获取，此时会基于数据库表 `EngineRegistry` 去做资源的动态分配限制。具体的分配策略的实现后续专门介绍，这边就不具体展开了。
可以理解为这个阶段结束，作业执行所需的资源就已经被占用，从而保证作业的顺利执行

#### 实际执行




#### 进度更新




## 总结
