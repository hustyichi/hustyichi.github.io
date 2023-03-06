---
layout: post
title: 'FATE Flow 源码解析 - 作业提交处理流程'
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

本文主要基于 FATE-Flow 2022 年 12 月发布的版本 v1.10.0，后续的版本可能略有差异。针对 FATE-Flow 的代码，基于 v1.10.0 的做了一个代码注解的仓库，方便查看具体的代码 [https://github.com/hustyichi/FATE-Flow](https://github.com/hustyichi/FATE-Flow)

## Fate-Flow 基础介绍

FATE-Flow 是 Fate 项目的重要组成部分，主要用于实现作业的调度，整体的设计可以查看 [官方文档](https://federatedai.github.io/FATE-Flow/latest/zh/fate_flow/)
FATE 中提交的训练作业会提交给 Fate-Flow，由 FATE-Flow 统一进行调度执行，最终生成所需训练结果

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

可以看到注册的路由主要就是 apps 目录与 scheduling_apps 目录下的路由。

一个注意点：FATE-Flow 是没办法独立运行的，需要作为 FATE 的一部分执行。 FATE-Flow 项目部分依赖的代码，比如 `fate_arch` 是存在于 Fate 工程下，对应的路径为 `FATE/python/fate_arch` ，找不到代码时可以联合 Fate 代码仓库进行阅读

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

    # 通知各个站点 (party) 去创建对应的作业 job 以及对应的任务 tasks

    status_code, response = FederatedScheduler.create_job(job=job)

    # 更新 job 状态为 WAITING

    job.f_status = JobStatus.WAITING

    # 将 job 状态同步给各个站点（party）

    status_code, response = FederatedScheduler.sync_job_status(job=job)

    return submit_result

```
可以看到提交作业时，主要是在数据库中的作业表 Job 生成对应的记录，并将作业的数据与状态同步给各个站点，并根据作业 job 的信息初始化生成对应的任务 task，最终实际执行时是以任务为单位进行的



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
可以理解为这个阶段结束，作业执行所需的资源就已经被占用，从而保证后续作业的顺利执行

#### 实际执行
实际作业的执行是在 `DAGScheduler.run_do()`中完成的，处理的状态是在 RUNNING，可以看到如下所示：
```python
def run_do(self):
    # 默认处理所有 RUNNING 状态的 job

    jobs = JobSaver.query_job(is_initiator=True, status=JobStatus.RUNNING, order_by="create_time", reverse=False)
    for job in jobs:
        self.schedule_running_job(job=job, lock=True)
```
可以看到实际的作业执行是在 `schedule_running_job()` 中完成的，此方法真正的任务执行是通过调用 `TaskScheduler.schedule()` 完成的，对应的代码如下所示：

```python

def schedule(cls, job, dsl_parser, canceled=False):
    initiator_tasks_group = JobSaver.get_tasks_asc(job_id=job.f_job_id, role=job.f_role, party_id=job.f_party_id)
    waiting_tasks = []

    # 获取就绪的 tasks

    for initiator_task in initiator_tasks_group.values():
        if initiator_task.f_status == TaskStatus.WAITING:
            waiting_tasks.append(initiator_task)


    # 执行所有就绪的 tasks

    for waiting_task in waiting_tasks:
        status_code = cls.start_task(job=job, task=waiting_task)


def start_task(cls, job, task):
    # 申请 task 相关的资源

    apply_status = ResourceManager.apply_for_task_resource(task_info=task.to_human_model_dict(only_primary_with=["status"]))
    if not apply_status:
        return SchedulingStatusCode.NO_RESOURCE

    # 更新状态为 RUNNING , 并同步给各个站点

    task.f_status = TaskStatus.RUNNING
    update_status = JobSaver.update_task_status(task_info=task.to_human_model_dict(only_primary_with=["status"]))
    FederatedScheduler.sync_task_status(job=job, task=task)

    # 实际调用参与方执行 task

    status_code, response = FederatedScheduler.start_task(job=job, task=task)

```
可以看到作业的执行事实上是依次获取所有就绪的任务 Task，然后执行 `FederatedScheduler.start_task()` 去执行 task 做成所需完成的功能，而 `FederatedScheduler.start_task()` 事实上就是发起一次请求调用 `party_app.py` 中的 `/<job_id>/<component_name>/<task_id>/<task_version>/<role>/<party_id>/start` 接口完成的
实际的任务执行是在 `TaskController.start_task()` 中完成，对应的代码如下所示：
```python
def start_task(cls, job_id, component_name, task_id, task_version, role, party_id, **kwargs):
    task_info = {
        "job_id": job_id,
        "task_id": task_id,
        "task_version": task_version,
        "role": role,
        "party_id": party_id,
    }

    # 根据 id 获取对应的任务

    task = JobSaver.query_task(task_id=task_id, task_version=task_version, role=role, party_id=party_id)[0]

    task_info["engine_conf"] = {"computing_engine": run_parameters.computing_engine}

    # 根据执行环境选择对应的 engine，目前主要是 eggroll 和 spark，默认是 eggroll

    backend_engine = build_engine(run_parameters.computing_engine)

    # 实际执行对应的任务，对于 eggroll，会启动新进程执行 python/fate_flow/worker/task_executor.py 脚本

    run_info = backend_engine.run(task=task,
                                    run_parameters=run_parameters,
                                    run_parameters_path=run_parameters_path,
                                    config_dir=config_dir,
                                    log_dir=job_utils.get_job_log_directory(job_id, role, party_id, component_name),
                                    cwd_dir=job_utils.get_job_directory(job_id, role, party_id, component_name),
                                    user_name=kwargs.get("user_id"))

    # 更新 task 相关的执行情况，执行正常的情况下状态为 RUNNING

    task_info.update(run_info)
    task_info["start_time"] = current_timestamp()

    cls.update_task(task_info=task_info)
    task_info["party_status"] = TaskStatus.RUNNING
    cls.update_task_status(task_info=task_info)

```
简单理解任务 Task 最终只是根据作业在独立进程中完成特定命令的执行，最终作业就是一系列任务的执行的组合。当所有任务完成时，作业也就完成了


#### 进度更新
前面提到作业 job 的执行事实上仅仅是一系列对应的任务 task 的执行，因此 FATE-Flow 的进度更新也是根据任务 task 的完成的数量占所有 task 的数量来确定的。具体的代码如下：
```python
def schedule_running_job(cls, job: Job, force_sync_status=False):
    # 调度 job 进行执行

    task_scheduling_status_code, auto_rerun_tasks, tasks = TaskScheduler.schedule(job=job, dsl_parser=dsl_parser, canceled=job.f_cancel_signal)

    # 更新 job 执行的进度以及状态

    tasks_status = dict([(task.f_component_name, task.f_status) for task in tasks])
    new_job_status = cls.calculate_job_status(task_scheduling_status_code=task_scheduling_status_code, tasks_status=tasks_status.values())

    # 根据 job 中已完成 task 的数量与总 task 的数量确定完成的进度

    total, finished_count = cls.calculate_job_progress(tasks_status=tasks_status)
    new_progress = float(finished_count) / total * 100

    if new_job_status != job.f_status or new_progress != job.f_progress:
        # 通知参与方更新 job 执行的进度信息

        if int(new_progress) - job.f_progress > 0:
            job.f_progress = new_progress
            FederatedScheduler.sync_job(job=job, update_fields=["progress"])
            cls.update_job_on_initiator(initiator_job=job, update_fields=["progress"])

        # 有状态变化时通知相关方更新 job 状态信息

        if new_job_status != job.f_status:
            job.f_status = new_job_status
            FederatedScheduler.sync_job_status(job=job)
            cls.update_job_on_initiator(initiator_job=job, update_fields=["status"])

    # 处理结束，执行必要资源回收

    if EndStatus.contains(job.f_status):
        cls.finish(job=job, end_status=job.f_status)
```
可以看到最终就是调用 `calculate_job_progress()` 计算特定作业 job 中任务 task 完成的数量，最终确定完成的进度。
所有的处理处理结束时，调用 `finish()` 执行必要的资源回收


## 总结
本文对 FATE-Flow 的作业的完整执行流程进行了梳理，为了简化删除了大量异常分支的处理，有兴趣的可以结合实际的 FATE-Flow v1.10.0 的源码进行查看，应该会更有裨益
