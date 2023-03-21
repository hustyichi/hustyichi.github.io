---
layout: post
title: 'FATE Flow 源码解析 - 日志输出机制'
subtitle:   "FATE Flow source code analysis - log mechanism"
date:       2023-03-21 08:12:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - FATE
    - Federated learning
---

## 背景介绍
在 [之前的文章](https://hustyichi.github.io/2023/03/08/fate-flow-loop/) 中介绍了 FATE 的作业处理流程，在实际的使用过程中，为了查找执行中的异常，需要借助运行生成的日志，但是 FATE-Flow 包含的流程比较复杂，对应的日志也很多，而且分散在不同的文件中，在这篇文章中就对 FATE-Flow 的日志机制进行梳理，帮助大家了解 Python 服务中实现一个更加灵活的日志机制

## logging 机制
FATE-Flow 中日志主要基于 Python 原生的 [logging 机制](https://docs.python.org/zh-cn/3/library/logging.html)实现，也正是依靠强大而灵活的 logging 机制，FATE-Flow 才实现了丰富的日志查询能力，基础的 logging 知识可以自行查看[官方文档](https://docs.python.org/zh-cn/3/library/logging.html)。

为了了解 FATE-Flow 的日志实现机制，涉及的一个关键概念就是 `Handler`, `Handler` 是实际处理日志的模块，在 logging 中被设计为可插拔模式的，可以根据需要进行注册。举一个简单的例子进行介绍：

```python
# 创建一个 logger 对象

logger = logging.getLogger(name)

# 注册日志处理器 handler1, 将日志输出至 a.log 文件中

handler1 = TimedRotatingFileHandler("a.log")
logger.addHandler(handler1)

# 注册日志处理器 handler2, 将日志输出至 b.log 文件中

handler2 = TimedRotatingFileHandler("b.log")
logger.addHandler(handler2)

# 实际输出日志

logger.info("test log")
```

在上面的例子中会基于 logging 机制创建出日志实例 logger ，并注册了两个 `Handler`，后续输出的日志就会依次被两个 `Handler` 进行处理，即日志最终会输出至文件 `a.log` 和 `b.log` 中，而且在实际运行中可以动态调整 `Handler`，从而动态调整日志处理方式，因此可以实现极其灵活的日志输出方式

理解了这个机制，就很容易理解 FATE-Flow 如何实现将同样的日志内容按照日志类型输出至 `fate_flow_schedule.log` ，`fate_flow_audit.log` 文件中，同时又按照日志等级输出至 `DEBUG.log`, `INFO.log`, `WARNING.log`, `ERROR.log` 文件中了。FATE-Flow 就是通过添加不同的 Handler 实现的

## FATE-Flow 日志
FATE-Flow 的日志主要存在于 `/data/projects/fate/fateflow/logs` 目录下，在此目录下主要分为两种类型的日志目录：

1. fate_flow 目录，其中包含整体的 FATE-Flow 日志；
2. 独立 job 目录的日志，此时会以 job_id 为单位输出所有相关的日志至特定目录中；
最终展示的效果如下所示：

![app](/img/in-post/logger-in-fate/log.png)

其中数字部分是以 job 为单位的日志，右侧 fate_flow 就是 FATE-Flow 整体的日志

其中 fate_flow 中主要是如下所示的日志：

- DEBUG.log
- ERROR.log
- INFO.log
- WARNING.log
- fate_flow_access.log
- fate_flow_audit.log
- fate_flow_database.log
- fate_flow_detect.log
- fate_flow_schedule.log
- fate_flow_stat.log
- stat.log

注意这部分日志因为量可能可能比较大，因此 FATE-Flow 中是按天进行分片，因此最终的效果如下所示的：

![app](/img/in-post/logger-in-fate/fate_flow_log.png)

而 按照 job_id 为单位的日志主要包含如下所示的日志：

- fate_flow_audit.log
- fate_flow_schedule_error.log
- fate_flow_schedule.log
- fate_flow_sql.log
- guest (目录)
- host (目录)

其中 guest/host 是按照 job 执行中参与的角色为单位输出的日志，后续介绍中再进一步展开


## 实现原理
下面按照目录分别介绍 FATE-Flow 不同模块的的日志输出相关的实现

#### fate_flow 目录日志
fate_flow 目录下的日志分为按照类型的日志 `fate_flow_schedule.log`, `fate_flow_audit.log` 等，同时也包含按照日志等级的 `DEBUG.log`, `INFO.log`, `WARNING.log`, `ERROR.log`
事实上，这两种类型的日志中的内容可能有很多相同的内容，只是按照不同维度区分后放在不同的文件中，比如想查询任务调度相关的日志，可以去查询 `fate_flow_schedule.log`，而想查询执行中警告相关的日志，可以去查询 `WARNING.log`, 而任务调度中也可能有处于警告级别的日志，此时在两个文件中都可能会存在。

我们可以先类型相关的日志是如何存入的，这边以 `fate_flow_schedule.log` 为例进行解释：
此日志使用的是 `fate_flow/utils/log_utils.py` 中的 `schedule_logger()` 方法进行日志输出，对应输出至 fate_flow 目录相关的代码如下：

```python
def schedule_logger(job_id=None, delete=False):
    if not job_id:
        return getLogger("fate_flow_schedule")
```

此时调用的是 `fate_arch/common/log.py` 中的 `getLogger()` 方法去获取对应日志实例 logger, 最终会调用 `LoggerFactory.init_logger()` 初始化日志实例，对应的代码如下所示：

```python
def init_logger(class_name):
    # 初始化日志实例 logger，此时对应的 class_name 为 fate_flow_schedule

    logger = LoggerFactory.new_logger(class_name)

    # 创建 logger 对应的 Handler，此 Handler 负责将日志输出至对应文件中

    handler = LoggerFactory.get_handler(class_name)

    # 将 handler 添加至 logger 中

    logger.addHandler(handler)

    LoggerFactory.logger_dict[class_name] = logger, handler

    # 额外配置新 handler，按照等级输出至 `DEBUG.log`, `INFO.log`, `WARNING.log`, `ERROR.log` 就是在这边实现

    LoggerFactory.assemble_global_handler(logger)
    return logger, handler
```

可以看看 handler 是如何输出至文件中的，对应于 `get_handler()` 方法，具体实现如下：

```python
def get_handler(class_name, level=None, log_dir=None, log_type=None, job_id=None):
    # 日志对应的文件

    log_file = os.path.join(LoggerFactory.LOG_DIR, "{}.log".format(class_name))

    # 最终使用 TimedRotatingFileHandler 实现按天的日志搜集

    os.makedirs(os.path.dirname(log_file), exist_ok=True)
    if LoggerFactory.log_share:
        handler = ROpenHandler(log_file,
                                when='D',
                                interval=1,
                                backupCount=14,
                                delay=True)
    else:
        handler = TimedRotatingFileHandler(log_file,
                                            when='D',
                                            interval=1,
                                            backupCount=14,
                                            delay=True)

    return handler
```
可以看到最终就是根据 classname 确定对应的文件路径，并最终创建 `TimedRotatingFileHandler`，此 Handler 会将日志输出至特定文件中，并按天进行日志轮转，最终实现上面图片中看到的按天分片的日志。

而按照日志等级的输出至文件中的实现是在 `LoggerFactory.assemble_global_handler()` 中实现的，起始就是依次创建各个日志层级的 Handler，并添加至 logger 上，对应的实现如下所示：

```python
def assemble_global_handler(logger):
    if LoggerFactory.LOG_DIR:
        # 对所有日志层级，依次进行注册

        for level in LoggerFactory.levels:
            if level >= LoggerFactory.LEVEL:

                # 获得的 level_logger_name 分别是 DEBUG, DEBUG, WARNING, ERROR，最终用于输出的文件命名

                level_logger_name = logging._levelToName[level]

                # 创建特定日志层级的 Handler，添加至 logger 上

                logger.addHandler(LoggerFactory.get_global_handler(level_logger_name, level))
```

而调用的 `LoggerFactory.get_global_handler()` 事实上也是调用 `get_handler()` 方法创建 handler，而指定了 level 的情况下 handler 中会额外使用 `handler.level = level` 仅输出特定日志层级的日志。从而实现了将日志按照层级输出

而其他类型的日志原理都是一样，在 `python/fate_flow/settings.py` 中初始化了对应类型的日志实例 logger，业务中直接使用，具体代码如下所示：

```python
stat_logger = getLogger("fate_flow_stat")
detect_logger = getLogger("fate_flow_detect")
access_logger = getLogger("fate_flow_access")
database_logger = getLogger("fate_flow_database")
```

#### job 日志

job 对应的日志也包含两种，一种是特定的几种类型的日志，比如 `fate_flow_audit.log`, `fate_flow_sql.log`, `fate_flow_schedule.log`，这部分的实现比较简单，具体是在 `fate_flow/utils/log_utils.py` 中，其中每个类型会包含一个特定的方法。

比如 `schedule_logger()` 方法用于获得对应日志实例 logger 输出日志至 `fate_flow_schedule.log`，而 `sql_logger()` 方法用于获得对应的日志实例 logger 输出日志至 `fate_flow_sql.log`，这些方法底层都是调用同样的 `get_job_logger()` 生成对应的日志实例的。具体实现如下所示：

```python
def get_job_logger(job_id, log_type):
    # 主要输出至 fate_flow 目录 和 job_id 相关的目录，包含 job_id 时会以 job_id 为单位分开输出

    job_log_dir = get_fate_flow_directory('logs', job_id)

    # 包含 job_id 会输出至 job_id 目录，audit 应该是请求调用跟踪日志，同时出现在两个 job_id 目录以及 fate_flow 目录

    if log_type == 'audit':
        log_dirs = [job_log_dir, fate_flow_log_dir]
    else:
        log_dirs = [job_log_dir]

    # 创建对应的目录

    os.makedirs(job_log_dir, exist_ok=True)
    os.makedirs(fate_flow_log_dir, exist_ok=True)

    # 生成对应的 logger，并对应匹配创建对应的 handler，从而实现日志输出至特定目录

    logger = LoggerFactory.new_logger(f"{job_id}_{log_type}")

    for job_log_dir in log_dirs:
        # 创建对应的 handler，将日志输出特定文件中，此时文件根据 log_type 命名，此时就是 fate_flow_schedule

        handler = LoggerFactory.get_handler(class_name=None, level=LoggerFactory.LEVEL,
                                            log_dir=job_log_dir, log_type=log_type, job_id=job_id)
        error_handler = LoggerFactory.get_handler(class_name=None, level=logging.ERROR, log_dir=job_log_dir, log_type=log_type, job_id=job_id)
        logger.addHandler(handler)
        logger.addHandler(error_handler)

    return logger

```

可以看到实现原理也是类似的，就是创建对应日志实例 logger，并生成对应的 handler，从而输出至特定文件中

job 中日志还有一部分的日志对应一个独立的目录 guest/host ，这部分是按照 job 训练中参与的角色为单位输出的日志，这些日志是如何被写入文件中的呢？

对应的代码在 `fate_flow/controller/task_controller.py` 中的 `start_task()` 方法，可以看到 task 实际执行的代码如下所示：

```python
backend_engine = build_engine(run_parameters.computing_engine)

# task 执行中对应的日志输出至 get_job_log_directory() 指定的 logs/{job_id}/{role}/{party_id}/{component_name} 对应的目录

run_info = backend_engine.run(task=task,
                                run_parameters=run_parameters,
                                run_parameters_path=run_parameters_path,
                                config_dir=config_dir,
                                log_dir=job_utils.get_job_log_directory(job_id, role, party_id, component_name),
                                cwd_dir=job_utils.get_job_directory(job_id, role, party_id, component_name),
                                user_name=kwargs.get("user_id"))
```

由于不同的执行引擎代码不同，我们以 eggroll 为例去查看对应的 `run()` 执行情况，此方法调用 `WorkerManager.start_task_worker()` 方法，并将日志目录通过 `log_dir` 传入，代码如下所示：

```python
   def start_task_worker(cls, worker_name, task: Task, task_parameters: RunParameters = None,
                          executable: list = None, extra_env: dict = None, **kwargs):
        # 根据 task 执行情况确定最终日志输出的目录 log_dir, 最终使用 logs/{job_id}/{role}/{party_id}/{component_name} 为日志目录

        worker_id, config_dir, log_dir = cls.get_process_dirs(worker_name=worker_name,
                                                              job_id=task.f_job_id,
                                                              role=task.f_role,
                                                              party_id=task.f_party_id,
                                                              task=task)

        # 实际执行的代码对应的文件

        from fate_flow.worker.task_executor import TaskExecutor
        module_file_path = sys.modules[TaskExecutor.__module__].__file__

        # 实际执行进程，使用 python 执行对应的代码

        env = cls.get_env(task.f_job_id, task.f_provider_info)
        if executable:
            process_cmd = executable
        else:
            process_cmd = [env.get("PYTHON_ENV") or sys.executable or "python3"]

        # 执行命令行，通过 log_dir 指定了对应的日志目录

        common_cmd = [
            module_file_path,
            "--job_id", task.f_job_id,
            "--component_name", task.f_component_name,
            "--task_id", task.f_task_id,
            "--task_version", task.f_task_version,
            "--role", task.f_role,
            "--party_id", task.f_party_id,
            "--config", config_path,
            '--result', result_path,
            "--log_dir", log_dir,
            "--parent_log_dir", os.path.dirname(log_dir),
            "--worker_id", worker_id,
            "--run_ip", RuntimeConfig.JOB_SERVER_HOST,
            "--run_port", RuntimeConfig.HTTP_PORT,
            "--job_server", f"{RuntimeConfig.JOB_SERVER_HOST}:{RuntimeConfig.HTTP_PORT}",
            "--session_id", session_id,
            "--federation_session_id", federation_session_id
        ]
        process_cmd.extend(common_cmd)

        # 启动新进程执行对应的命令

        p = process_utils.run_subprocess(job_id=task.f_job_id, config_dir=config_dir, process_cmd=process_cmd,
                                         added_env=env, log_dir=log_dir, cwd_dir=config_dir, process_name=worker_name.value,
                                         process_id=worker_id)
        return {"run_pid": p.pid, "worker_id": worker_id, "cmd": process_cmd}
```

而实际使用 `log_dir` 是在 `fate_flow/worker/base_worker.py` 中完成的，此时会调用 `LoggerFactory.set_directory()` 指定日志输出的父目录，后续即可将任务相关的日志输出至指定的目录中了

## 总结
这边介绍了 FATE-Flow 相关的日志存储情况，相对大部分服务而言，FATE-Flow 中的日志存储都属于比较复杂的了，但是借助 logging 中动态的 Handler 更新机制，可以比较方便地实现不同模块独立的日志文件输出路径，甚至可以实现部分日志输出至日志文件，部分日志输出至终端，有兴趣的也可以进一步了解下
