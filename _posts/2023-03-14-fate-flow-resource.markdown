---
layout: post
title: 'FATE Flow 源码解析 - 资源分配流程'
subtitle:   "FATE Flow source code analysis - resource allocation process"
date:       2023-03-14 07:50:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - FATE
    - Federated learning
---

## 背景介绍
在 [上一篇文章](https://hustyichi.github.io/2023/03/08/fate-flow-loop/) 中介绍了 FATE 的作业处理流程，对于资源管理没有深入介绍，本篇文章就是来补充相关细节
众所周知，FATE 是一个联邦学习框架，涉及了多个参与方进行模型的训练，而各个参与方的训练过程都需要使用 CPU 资源和内存资源，因此在作业执行前需要提前先申请好必要的资源，才能避免在执行中避免因为资源不足而导致执行失败。
而 FATE 中单个作业会包含多个任务，每个任务需要独立使用一定量的资源，因此任务会有独立的资源申请与释放资源的流程

## 资源分配相关流程
在资源的管理中，主要涉及两部分:
1. 资源初始化，在 Fate-Flow 初始化时，根据配置文件初始化对应的资源信息，确定了服务中可供分配的资源总量，这部分是在 `fate_flow_server.py` 中实现，调用链路如下： ConfigManager.load() -> ResourceManager.initialize()
2. 作业的资源申请与释放，会涉及到作业与任务的资源申请与释放。具体的流程如下：

    - 作业资源申请是在 `dag_scheduler.py` 中 `schedule_waiting_jobs()` 中调用 `FederatedScheduler.resource_for_job()` 完成
    - 作业资源释放是在 `job_controller.py` 中调用 `update_job_status()` 时，如果状态处于完成状态，此时会调用 `ResourceManager.return_job_resource()` 释放作业对应的资源
    - 任务资源申请是在 `dag_scheduler.py` 中完成，调用链路如下： schedule_running_job() ->  TaskScheduler.schedule() -> TaskScheduler.start_task() -> ResourceManager.apply_for_task_resource()
    - 任务资源释放是在任务状态更新时调用 `task_controller.py` 中调用 `update_task_status()` 时，如果状态处于完成状态，此时会调用 `ResourceManager.return_task_resource()` 释放资源


## 资源分配源码解析

#### 资源初始化
资源的初始化是在 `ResourceManager.initialize()` 实现，可以看到对应的代码如下所示：

```python
def initialize(cls):
    engines_config, engine_group_map = engine_utils.get_engines_config_from_conf(group_map=True)
    for engine_type, engine_configs in engines_config.items():
        for engine_name, engine_config in engine_configs.items():
            cls.register_engine(engine_type=engine_type, engine_name=engine_name, engine_entrance=engine_group_map[engine_type][engine_name], engine_config=engine_config)
```

通过依次对各种类型的资源调用 `ResourceManager.register_engine()` 进行注册，具体的注册实现代码如下所示：

```python
def register_engine(cls, engine_type, engine_name, engine_entrance, engine_config):
    nodes = engine_config.get("nodes", 1)
    cores = engine_config.get("cores_per_node", 0) * nodes * JobDefaultConfig.total_cores_overweight_percent
    memory = engine_config.get("memory_per_node", 0) * nodes * JobDefaultConfig.total_memory_overweight_percent
    filters = [EngineRegistry.f_engine_type == engine_type, EngineRegistry.f_engine_name == engine_name]
    resources = EngineRegistry.select().where(*filters)

    # 已有资源下，应该是只生效最新的一条资源记录，因为之前对应的资源记录可能使用了一部分，需要对应更新记录

    if resources:
        resource = resources[0]
        update_fields = {}

        # 对应的资源的总量与配置信息，覆盖历史记录即可

        update_fields[EngineRegistry.f_engine_config] = engine_config
        update_fields[EngineRegistry.f_cores] = cores
        update_fields[EngineRegistry.f_memory] = memory

        # f_remaining_cores 表示可用的 CPU 资源，通过比较本次注册的 CPU 资源与之前注册的 CPU 资源的差额，确定可用资源的数量

        update_fields[EngineRegistry.f_remaining_cores] = EngineRegistry.f_remaining_cores + (
                cores - resource.f_cores)

        # f_remaining_memory 表示可用的内存资源，通过比较本次注册的内存资源与之前注册的内存资源的差额，确定可用资源的数量

        update_fields[EngineRegistry.f_remaining_memory] = EngineRegistry.f_remaining_memory + (
                memory - resource.f_memory)
        update_fields[EngineRegistry.f_nodes] = nodes
        operate = EngineRegistry.update(update_fields).where(*filters)
        update_status = operate.execute() > 0

    # 没有对应的资源下，直接新增即可

    else:
        resource = EngineRegistry()
        resource.f_create_time = base_utils.current_timestamp()
        resource.f_engine_type = engine_type
        resource.f_engine_name = engine_name
        resource.f_engine_entrance = engine_entrance
        resource.f_engine_config = engine_config

        resource.f_cores = cores
        resource.f_memory = memory

        # 没有对应的资源时，可用资源 f_remaining_cores 与 f_remaining_memory 与总量 f_cores，f_memory 相等

        resource.f_remaining_cores = cores
        resource.f_remaining_memory = memory
        resource.f_nodes = nodes
        resource.save(force_insert=True)

```

可以看到，对应的资源注册仅仅是在数据库中的表 `EngineRegistry` 中生成对应的记录，后续的分配也是基于 `EngineRegistry` 进行对应的扣减即可

#### 资源申请与释放

梳理资源的申请与释放流程时，先来看看作业的资源申请与释放

**作业资源申请**

作业的资源申请是通过调用 `FederatedScheduler.resource_for_job()` 方法实现的，此时会依次调用各个站点对应的 `/<job_id>/<role>/<party_id>/resource/apply` 接口，此接口会调用 `resource_manager.py` 中的 `apply_for_job_resource()`, 此方法会调用 `resource_for_job()` 方法，申请资源时会将资源从公共的资源池 `EngineRegistry` 分配至作业表 `Job` 上。具体的实现如下所示：

```python

# 计算作业所需的资源

engine_name, cores, memory = cls.calculate_job_resource(job_id=job_id, role=role, party_id=party_id)

# 在 Job 表中增加实际使用的资源，其中 f_cores 表示实际申请的 CPU 资源，f_memory 表示实际申请的内存资源

updates = {
    Job.f_engine_type: EngineType.COMPUTING,
    Job.f_engine_name: engine_name,
    Job.f_cores: cores,
    Job.f_memory: memory,
}
filters = [
    Job.f_job_id == job_id,
    Job.f_role == role,
    Job.f_party_id == party_id,
]

# 其中 f_remaining_cores 表示 Job 中剩余的 CPU 资源，f_remaining_memory 表示 Job 中剩余的内存资源，后续 Task 就是从 Job 中申请资源

updates[Job.f_remaining_cores] = cores
updates[Job.f_remaining_memory] = memory

# f_resource_in_use 表示实际占用资源的标记

updates[Job.f_resource_in_use] = True
updates[Job.f_apply_resource_time] = base_utils.current_timestamp()
operate = Job.update(updates).where(*filters)


# 在 EngineRegistry 减少对应的资源，利用 filter 避免资源超出可用资源 f_remaining_cores，f_remaining_memory

filters = [
    resource_model.f_remaining_cores >= cores,
    resource_model.f_remaining_memory >= memory
]
updates = {resource_model.f_remaining_cores: resource_model.f_remaining_cores - cores,
            resource_model.f_remaining_memory: resource_model.f_remaining_memory - memory}
filters.append(EngineRegistry.f_engine_type == EngineType.COMPUTING)
filters.append(EngineRegistry.f_engine_name == engine_name)
operate = EngineRegistry.update(updates).where(*filters)

```

可以看到的实际的逻辑其实很简单，就是从 `EngineRegistry` 扣减对应的资源 cores, memory, 在 `Job` 表中增加对应的资源。

**作业资源释放**

作业资源的释放与申请是完全相反的，此时会将 `Job` 表中的 `f_resource_in_use` 标记设置为 False，同时在资源池表 `EngineRegistry` 增加对应的资源，实现的代码基本上是申请代码的反向操作，就不过多展示原始代码了

**任务资源申请**

根据前面提到的任务资源的申请流程，最终任务的资源申请是通过调用 `resource_manger.py` 中的 `apply_for_task_resource()` 方法完成的，此方法调用 `resource_for_task()` 方法来完成任务资源分配，对应的实现如下：

```python
# 计算 task 占用的资源

cores_per_task, memory_per_task = cls.calculate_task_resource(task_info=task_info)

# task 的资源操作都是在 Job 表完成的, job 的资源申请是从 EngineRegistry 分配至 Job 表，task 的资源使用时在对应的 job 上完成

filters = [
        resource_model.f_remaining_cores >= cores,
        resource_model.f_remaining_memory >= memory
]
updates = {resource_model.f_remaining_cores: resource_model.f_remaining_cores - cores,
            resource_model.f_remaining_memory: resource_model.f_remaining_memory - memory}
filters.append(Job.f_job_id == task_info["job_id"])
filters.append(Job.f_role == task_info["role"])
filters.append(Job.f_party_id == task_info["party_id"])
filters.append(Job.f_resource_in_use == True)
operate = Job.update(updates).where(*filters)
operate_status = operate.execute() > 0

```

可以看到资源的申请就是消耗 Job 表中 `f_remaining_cores` 和 `f_remaining_memory` 中的剩余的 CPU 和内存资源

**任务资源释放**

任务资源释放与任务资源的申请基本相反，是在 `Job` 中增加可用资源，没有其他特殊操作，因此也就不粘贴原始代码了


## 总结

FATE Flow 的资源分配机制还是比较简单的，在初始化时通过配置文件指定站点可用的资源，调用 `ResourceManager.initialize()` 初始化对应的资源，将可用的资源信息保存至表 `EngineRegistry` 上，作业资源的申请是从 `EngineRegistry` 表分配至 `Job` 表，即从 `EngineRegistry` 表扣减对应的资源，从 `Job` 表增加对应的资源，而任务资源的申请是从 `Job` 表扣减对应的资源

整体的方案相对简单，而且借助 sql 的更新机制鲁棒性相对较高。但是资源的消耗量都是按照配置计算得到的，按照标准化单位计数，并非实际的资源消耗，因此配置不当可能会导致超过可用资源上限从而出现问题。
