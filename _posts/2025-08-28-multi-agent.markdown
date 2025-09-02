---
layout: post
title: "来自工业界的多 Agent 框架最全细节对比"
subtitle:   "The most complete comparison of multi-agent frameworks from the industry"
date:       2025-08-28 17:00:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - llm
    - agent
---

## 背景介绍
过去的项目涉及 RAG 比较多，在 2024 年整理过 [来自工业界的开源知识库 RAG 项目最全细节对比](https://zhuanlan.zhihu.com/p/707842657)，得到了不少工程师比较好的反馈。最近新项目使用的多 Agent 的技术方案，实际对多 Agent 框架进行了详细了调研，结合最近的项目的具体实践，整理相关内容分享在这边，期望对其他人的框架选型有一些帮助。

在这篇文章中主要对比目前相对成熟或好评较多的多 Agent 框架，主要对比的框架包括 CrewAI、AutoGen、LangGraph、Agno、OpenAI Agents、Pydantic AI、MetaGPT。

- [CrewAI](https://github.com/crewAIInc/crewAI)
- [AutoGen](https://github.com/microsoft/autogen)
- [LangGraph](https://github.com/langchain-ai/langgraph)
- [Agno](https://github.com/agno-agi/agno)
- [OpenAI Agents](https://github.com/openai/openai-agents-python)
- [Pydantic AI](https://github.com/pydantic/pydantic-ai)
- [MetaGPT](https://github.com/FoundationAgents/MetaGPT)

如果只关心技术选型结论，可以直接跳到最后结论部分


## 项目基本情况

框架的基本情况的比较如下所示：

| 项目 |  Star 数量 | 持续维护性 | 社区活跃度 | 上手门槛 |
| --- |  --- | --- | --- | --- |
| [CrewAI](https://github.com/crewAIInc/crewAI) | 36.2k | ⭐️⭐⭐️️ | ⭐️⭐️⭐️ | ⭐️ |
| [AutoGen](https://github.com/microsoft/autogen) | 49.2k | ⭐️⭐️⭐️ | ⭐️⭐️⭐️ | ⭐️⭐️⭐️ |
| [LangGraph](https://github.com/langchain-ai/langgraph) | 17.8k | ⭐️⭐️⭐️ | ⭐️⭐️⭐️ | ⭐️⭐️⭐️ |
| [Agno](https://github.com/agno-agi/agno) | 32.4k | ⭐️⭐️⭐️ | ⭐️⭐️⭐️ | ⭐️ |
| [OpenAI Agents](https://github.com/openai/openai-agents-python) | 14k | ⭐️⭐️️⭐️️ | ⭐️⭐️⭐️ | ⭐️⭐️ |
| [Pydantic AI](https://github.com/pydantic/pydantic-ai) | 11.9k | ⭐️⭐️⭐️ | ⭐️⭐️⭐️ | ⭐️⭐️️ |
| [MetaGPT](https://github.com/FoundationAgents/MetaGPT) | 58.1k | ⭐️⭐️️ | ⭐️⭐⭐️ | ⭐️⭐️⭐️ |

- 项目热度而言，因为剔除了一些相对冷门的多 Agent 框架，基本上热度都比较高，Star 数量最少的也有接近 12k 的 star
- 项目可维护性上，MetaGPT 虽然 Star 数量最多，但是更新频率明显下降，有概率后续不会持续迭代和维护，其他项目依旧在持续高频迭代，从当前来看可维护性基本都是有保障的。
- 项目的上手门槛比较，CrewAI 和 Agno 上手门槛比较低，可以快速上手。LangGraph 基于图的抽象，上手门槛相对较高，AutoGen 涉及概念较多，上手门槛也不低。OpenAI-Agent 和 Pydantic AI 上手门槛适中。
- 匹配场景，MetaGPT 主要用于软件开发的场景，其他多 Agent 框架基本是面向通用场景设计的。

根据现有情况来看，除非针对的是软件开发领域，选型基本可以排除掉 MetaGPT。不考虑业务的情况下，可以根据上手门槛进行一些选择。


## 框架核心能力比较
为了确认多 Agent 框架的核心能力与适用场景，针对每个框架的关键能力进行梳理，主要关注的能力维度包括：

- **Agent 建模能力**: 关注的是框架对 Agent 的抽象能力，是否支持 `Reasoning`（推理）、`Planning`（规划）等能力，以及是否可以灵动地自定义 Agent 的行为、推理逻辑和决策过程。
- **协调与通信能力**：关注的是框架中的 Agent 交互能力， 是简单的顺序链式调用，还是复杂的DAG、黑板模式、订阅发布模式。Agent 之间是如何进行通信的。
- **会话与状态管理能力**：关注的是框架中记忆机制以及会话管理，复杂的业务流程需要有相对复杂的状态管理机制进行支撑。
- **工具生态与集成能力**：关注的是框架提供的外部工具调用能力，内置工具的丰富性以及自定义工具的便捷性。
- **LLM 兼容性**：关注的是框架是否支持多种大模型，以及切换模型供应商的便捷性。
- **可观测性**：是否提供良好的可观测能力，方便进行复杂流程的问题定位以及系统调优；
- **可靠性与容错机制**：关注大模型调用失败或核心组件调用失败时，是否提供重试、降级、超时等机制。

#### CrewAI


#### AutoGen


#### LangGraph


#### Agno


#### OpenAI Agents


#### Pydantic AI


## 结论
