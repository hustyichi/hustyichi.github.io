---
layout: post
title: 'FATE Board 执行流程探索'
subtitle:   "FATE Board Execution Process Exploration"
date:       2023-08-23 09:00:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - FATE
---

## 背景介绍
FATE Board 是 FATE 提供的一个工程，用于给 FATE 提供可视化能力，方便在联邦学习训练中实时查看执行状态，更好地定位执行中遇到的问题。

查看 FATE 架构可以看到 FATE Board 是建立在 MySQL 和 FATE Flow Server 的基础上的，看起来数据来源是来自于这两者。FATE Flow Server 在[之前的文章](https://hustyichi.github.io/2023/03/08/fate-flow-loop/) 中已经介绍过，FATE 中隐私计算的主要调度流程都是实现在这个服务中。

![app](/img/in-post/fate-board/fate_arch.png)

FATE Board 代码仓库地址 [https://github.com/FederatedAI/FATE-Board](https://github.com/FederatedAI/FATE-Board), 本文的探索基于 v1.11.1，后续版本可能有所不同

## FATE Board 实现探索
FATE Board 工程中包含前端与后端的实现，前端是基于 Vue 实现的，后端则是基于 Java 实现。本文在探索时主要基于两个场景串联了一下完整的流程，分别是主页面的 job 列表页，以及 job 日志详情，通过查看完整的调用链路，对 FATE Board 建立基础的认识。

#### Job 列表页

![app](/img/in-post/fate-board/list.png)

通过 Chrome 调试模式查看对应的请求，即可比较容易发现获取 job 列表数据对应的请求为 `/job/query/page/new` , 通过对应的接口路径全局搜索可以发现后端的实现为 `src/main/java/com/webank/ai/fate/board/controller/JobManagerController.java` 中的 `queryPagedJob()` 方法，对应的代码实现如下：

```java
public PageBean<Map<String, Object>> queryPagedJobs(PagedJobQO pagedJobQO) {
    String jobId = pagedJobQO.getJobId();
    FlowJobQO flowJobQO = new FlowJobQO();
    if (jobId != null && 0 != jobId.trim().length()) {
        flowJobQO.setJob_id(pagedJobQO.getJobId());
    }

    // 构造请求参数 ...

    // 实际获取数据
    Map<String, Object> jobMap = getJobMap(flowJobQO);

    // ... 冗长的业务处理
}
```

可以看到的真正的数据获取部分基本就是直接调用 `getJobMap()` ，对应的实现如下所示：

```java
private Map<String, Object> getJobMap(Object query) {
    result = flowFeign.post(Dict.URL_JOB_QUERY, JSON.toJSONString(query));

    // ... 冗长的结果转换
}
```
实际的获取是通过一次 HTTP 请求获取到，对应的请求地址为 `/v1/job/list/job`，看情况应该是调用 FATE Flow Server 获取的，在 FATE-Flow 中看到的对应的接口，处于路径 `FATE-Flow/python/fate_flow/apps/job_app.py` 中的 `list_job()`，实际的实现就是一次简单的数据库查询，不再进一步展开。

#### Job 日志

![app](/img/in-post/fate-board/log.png)

通过 chrome 调试模式看到实际获取 Job 日志是通过 websocket 获取的，请求的地址为 `/log/new/202307260855242117390/host/8889/default`，目前来看日志的获取和 job 列表的获取存在一些差异

依旧利用请求地址搜索对应的代码实现，可以确认后端对应的实现路径为 `src/main/java/com/webank/ai/fate/board/websocket/LogWebSocketController.java` 中的 LogWebSocketController 类实现，对于 websocket 的服务端，消息处理都是在 onMessage 实现的，我们可以看到对应的代码实现如下：

```java
@OnMessage
public void onMessage(String message,
                        Session session,
                        @PathParam("jobId") String jobId,
                        @PathParam("role") String role,
                        @PathParam("partyId") String partyId,
                        @PathParam("componentId") String componentId) throws Exception {
    synchronized (session) {
        LogQuery logQuery = JSON.parseObject(message, LogQuery.class);

        // 根据类型主要包含 logSize 和 logCat，其中 logSize 用于获取日志行数，logCat 获取日志内容
        if (logQuery.getType().equals(LogTypeEnum.LOG_SIZE.boardValue)) {
            logSize(session, jobId, role, partyId, componentId, logQuery);
        } else {
            logCat(session, jobId, role, partyId, componentId, logQuery);
        }
    }
}
```

可以看到的通过路径获取 jobId, role, partyId, componentId 的参数，然后调用 `logSize()` 和 `logCat()` 执行实际的处理，我们主要关注日志内容的获取，可以看到 `logCat()` 对应的实现如下所示：

```java
private void logCat(Session session, String jobId, String role, String partyId, String componentId, LogQuery logQuery) {
    // 构造请求
    FlowLogCatReq flowLogCatReq = new FlowLogCatReq();
    flowLogCatReq.setJob_id(jobId);
    flowLogCatReq.setLog_type(Dict.logTypeMap.get(logQuery.getType()));
    flowLogCatReq.setRole(role);
    flowLogCatReq.setParty_id(Integer.valueOf(partyId));
    flowLogCatReq.setComponent_name(componentId);
    flowLogCatReq.setInstance_id(logQuery.getInstanceId());
    flowLogCatReq.setBegin(logQuery.getBegin());
    flowLogCatReq.setEnd(logQuery.getEnd());

    // 实际获取数据
    FlowResponse<List<FlowLogCatResp>> resultFlow = flowLogFeign.logCat(flowLogCatReq);

    // 构造响应数据
    LogContentResponse logContentResponse = new LogContentResponse();
    logContentResponse.setType(logQuery.getType());
    logContentResponse.setData(resultFlow.getData().stream()
            .map(LogContentResponse.LogContent::fromFlowContent)
            .collect(Collectors.toList()));
    try {
        session.getBasicRemote().sendText(JSON.toJSONString(logContentResponse));
    } catch (IOException e) {
        e.printStackTrace();
        logger.error("websocket send error: {}", logContentResponse);
    }
}
```

根据最核心的数据获取是调用 `flowLogFeign.logCat()` ，对应的实现：

```java
@FeignClient(url = RouteTargeter.URL_PLACE_HOLDER + "/v1/log", name = "flowLogFeign", configuration = FeignRequestInterceptor.class)
public interface FlowLogFeign {

    // 构造 http 请求
    @RequestMapping(value = "/cat", method = RequestMethod.POST)
    FlowResponse<List<FlowLogCatResp>> logCat(FlowLogCatReq request);

    @RequestMapping(value = "/size", method = RequestMethod.POST)
    FlowResponse<FlowLogSizeResp> logSize(FlowLogSizeReq request);
}

```

最后兜了一圈，看起来还是转换了一次网络请求，看起来还是发送给了 FATE Flow Server，追踪 FATE-Flow 工程中的对应实现，可以看到对应的网络请求位于 `FATE-Flow/python/fate_flow/apps/log_app.py` 路径下，具体的实现位于 `FATE-Flow/python/fate_flow/utils/log_sharing_utils.py` 中的 `cat_log()` 方法中，实现如下：

```python
def cat_log(self, begin, end):
    line_list = []
    log_path = self.get_log_file_path()
    if begin and end:
        cmd = f"cat {log_path} | tail -n +{begin}| head -n {end-begin+1}"
    elif begin:
        cmd = f"cat {log_path} | tail -n +{begin}"
    elif end:
        cmd = f"cat {log_path} | head -n {end}"
    else:
        cmd = f"cat {log_path}"
    lines = self.execute(cmd)
    if lines:
        line_list = []
        line_num = begin if begin else 1
        for line in lines.split("\n"):
            line = replace_ip(line)
            line_list.append({"line_num": line_num, "content": line})
            line_num += 1
    return line_list
```

可以看到最终就是调用系统的 cat 命令，最终文件对应的内容，整体实现简单直接。


## 总结
通过对 FATE-Board 两个请求的调用链路的跟踪，可以对 FATE-Board 工程有了一些了解，看起来 FATE-Board 是建立在 FATE-Flow 基础上的一个简单可视化，使用的能力基本都是通过 FATE-Flow 提供，而 FATE-Board 仅仅提供必要的数据包装与前端的展示呈现，过程简单清晰。后续如果希望了解 FATE-Board 对应的可视化的能力范围，直接查看 FATE-Flow 对应提供的接口即可
