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

