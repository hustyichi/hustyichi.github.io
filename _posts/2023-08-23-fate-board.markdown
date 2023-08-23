---
layout: post
title: 'Fate Board 执行流程探索'
subtitle:   "Fate Board Execution Process Exploration"
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

查看 FATE 架构可以看到 FATE Board 是建立在 MySQL 和 FATE Flow Server 的基础上的，看起来数据来源是来自于这两者。FATE Flow Server 在之前的文章中已经介绍过，FATE 中隐私计算的主要调度流程都是实现在这个服务中。

![app](/img/in-post/fate-board/fate_arch.png)

FATE Board 代码地址为 [https://github.com/FederatedAI/FATE-Board](https://github.com/FederatedAI/FATE-Board), 本文的探索基于 v1.11.1，后续版本可能有所不同

## FATE Board 实现探索
FATE Board 工程中包含前端与后端的实现，前端是基于 Vue 实现的，后端则是基于 Java 实现。本文在探索时主要基于两个场景串联了一下完整的流程，分别是主页面的 job 列表页，以及 job 日志详情，整体流程不算复杂。

#### Job 列表页

![app](/img/in-post/fate-board/list.png)

查看 Job 列表页通过 Chrome 调试模式查看对应的请求，即可比较容易发现对应的请求为 `/job/query/page/new` , 通过对应的接口信息即可轻松发现后端的实现路径为 `src/main/java/com/webank/ai/fate/board/controller/JobManagerController.java` 中的 `queryPagedJob()` 方法，简单跳转后可以看到对应的真正的内容获取为：

```java
private Map<String, Object> getJobMap(Object query) {
    result = flowFeign.post(Dict.URL_JOB_QUERY, JSON.toJSONString(query));
}
```
最终只是一层简单的转发，对应的地址为 `/v1/job/list/job`，追踪至 FATE Flow Server 中看到了对应的实现，处于路径 `FATE-Flow/python/fate_flow/apps/job_app.py` 中的 `list_job()`，最终的在 FATE Flow Server 中仅仅做了必要的数据库查询，最终返回了结果


#### Job 日志

![app](/img/in-post/fate-board/log.png)

通过 chrome 调试模式看到实际获取 Job 日志是通过 websocket 获取的，请求的地址为 `/log/new/202307260855242117390/host/8889/default`，目前来看日志的获取是特殊的

利用请求地址搜索对应的代码实现，可以确认后端对应的实现路径为 `src/main/java/com/webank/ai/fate/board/websocket/LogWebSocketController.java` 中的 LogWebSocketController 类实现，对于 websocket 的服务端，消息处理都是在 onMessage 实现的，我们可以看到对应的代码实现如下：

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

我们主要关注日志内容的获取，可以看到 logCat 对应的实现如下所示：

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

根据最核心的内容获取去查看 `flowLogFeign.logCat()` 的实现：

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

最后兜了一圈，看起来还是转换了一次网络请求，看起来还是发送给了 FATE Flow Server
