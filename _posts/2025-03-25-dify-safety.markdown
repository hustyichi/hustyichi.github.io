---
layout: post
title: "为 AI 应用打造安全屏障：基于 Dify 的完整实践"
subtitle:   "Building a security barrier for AI applications: a complete practice based on Dify"
date:       2025-03-25 21:30:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - agent
    - llm
---

## 背景介绍
2025 是大模型应用爆发的一年，年初的 DeepSeek 吸引了大量人开始部署大模型产品，之后 Manus, MCP 等 Agent 方案又开始持续吸引大量的注意力，大家都在努力做出一款爆款的大模型应用。

但是在这种热情下，有一个很容易被忽视的问题就是大模型的安全性。但是如果希望实现一个完整的大模型应用，AI 应用的安全性是必不可少的一部分。在 2024 年的一篇文章 [Building A Generative AI Platform](https://huyenchip.com/2024/07/25/genai-platform.html) 就介绍过大模型应用的完整架构：

![arch](/img/in-post/dify-safety/arch.png)

其中对应的安全模块就是`输入围栏(Input guardrails)`和`输出围栏(Output guardrails)`，在文章中虽然介绍了安全机制，但是没有具体的实现方案。在这篇文章中，我们就以目前比较流行的大模型应用平台 Dify 为基础进行实践，我们首先以相对简单方案出发，不断拓展实现更加符合生产环境需求的方案。


## 安全性方案
为了实现文本内容的安全性检测，实现开销从低到高有如下所示的三种方案：

1. 基于关键词与规则的审查：此方案需要维护敏感词库,使用正则表达式、文本匹配算法进行检测，实现简单，匹配效率高，但是准确率较低。
2. 基于机器学习的内容审核：此方案需要需要使用现有审核模型或训练独立的审核模型，实现复杂，但是可以实现较高的准确率。
3. 基于人机结合的审核机制：由于人工智能的审核总会存在遗漏，但是模糊不请的边界问题，人工审核才能保证最终的正确性。

人机结合的审核更多的是工程设计，在本篇文章中我们主要实践和落地前两种方案。

## 方案实践

在进行方案实践时，我们也按照成本从低到高的顺序进行介绍，我们首先使用内置的方案，然后逐步拓展到自定义方案。

### 基于内置方案的实践

Dify 默认提供了两种开箱即用的方案，可以无需额外开发即可使用，分别是基于关键词的审查和基于 OpenAI 的内容审核。可以在功能页面直接开启：

![default](/img/in-post/dify-safety/default.png))

**关键词方案**

如果想快速应用，可以从最简单的关键词审查开始，配置好关键词库以及审查输入输出的预设回复就可以快速上手了。

实际测试下来，关键词的功能符合预期，但是实际在生产环境使用马上就会发现一个问题，默认的关键词只能通过网页端进行配置，动态更新存在明显的限制。而且限制只能配置 100 个，这对于实际的生产环境显然是不够的。

**OpenAI 审核方案**

基于 OpenAI 的审核方案，对应的就是上面的基于机器学习的内容审核方案，但是在国内的环境下使用基本不太可行。而且考虑到国情，国内的审核政策和国外的审核政策还是有比较大的差异，所以如果要落地到生产环境，还是需要自定义的实现方案。


### 基于自定义拓展的方案选型

Dify 估计也考虑到了内置的两种方案大概率是不能满足实际生产环境的需求，所以在底层提供了自定义拓展的方案。目前提供了两种自定义拓展的方案：

- API 扩展
- 代码 扩展

**API 扩展**

基于 API 的扩展就相对自由了，可以部署一个独立的安全性审核服务，之后通过 API 提供给 Dify 现有的服务直接进行调用。这样可以完全自由地控制现有应用的审核逻辑，但是存在一个问题，就是需要维护一套独立的审核服务，带来额外的成本。如果需要支持灵活的参数配置可能甚至还需要维度一套独立的前后端服务。

对这个方案感兴趣的小伙伴可以参考 [官方文档](https://docs.dify.ai/zh-hans/guides/extension/api-based-extension/moderation) 进行实践，从实际测试来看，只要遵循指定的 API 请求参数即可，实现比较简单。

**代码扩展**

考虑到 API 扩展的部署成本，Dify 还提供了一套基于代码的扩展方案，可以在 Dify 现有的代码仓库中补充自定义的审核机制。只要遵循 [官方实现规范](https://docs.dify.ai/zh-hans/guides/extension/code-based-extension/moderation) 进行实现，可以无缝嵌入现有应用中，无需额外部署，同时还可以支持灵活的前端配置选项。可以实现的效果类似如下所示：

![custom](/img/in-post/dify-safety/custom.png)

### 基于代码扩展的实践

在实现自定义方案时，我们可以进行审核方案的选型，考虑到实际的生产环境的需要，关键词和机器学习结合的方案可能会更合适一些。

关键词虽然不够准确，但是可以高效过滤掉大量的违规内容，对于漏网之鱼，再通过成本更高的机器学习方案进行二次审核，这样可以兼顾效率和成本。同时考虑到 Dify 中的大模型应用一般已经部署了大模型，因此优先考虑直接复用现有的大模型，这样就不会带来额外部署模型的成本。

根据上面的分析，最终最低成本的方案选择的是关键词库 + 基于大模型的安全围栏机制。如果后续发现准确性不够高，可以考虑引入额外的审核模型。当然考虑到响应速度的问题，可以考虑提供给用户选择纯关键词的安全围栏机制。

具体的实现步骤基本遵循 [官方流程](https://docs.dify.ai/zh-hans/guides/extension/code-based-extension/moderation):_

**初始化目录**

在 Dify 项目的 `api/core/moderation` 下新建相关的目录和文件，包含一个前端组件定义文件和一个实现类文件。类似的代码架构为：

```
api/core/moderation
                ├── custom
                    └── custom.py
                    └── schema.json
```

**添加前端组件定义文件**

在 schema.json 中定义前端组件的配置项目，与上面的可视化页面类似的配置项如下所示：

```json
{
    "label": {
        "en-US": "Custom Moderation",
        "zh-Hans": "自定义"
    },
    "form_schema": [
        {
            "type": "select",
            "label": {
                "en-US": "Moderation Type",
                "zh-Hans": "检测类型"
            },
            "variable": "moderation_type",
            "required": true,
            "options": [
                {
                    "label": {
                        "en-US": "Keyword",
                        "zh-Hans": "关键词"
                    },
                    "value": "Keyword"
                },
                {
                    "label": {
                        "en-US": "Keyword & LLM",
                        "zh-Hans": "关键词 & 大模型"
                    },
                    "value": "Keyword & LLM"
                }
            ],
            "default": "Keyword",
            "placeholder": ""
        },
        {
            "type": "paragraph",
            "label": {
                "en-US": "Custom Keywords",
                "zh-Hans": "自定义关键词"
            },
            "variable": "custom_keywords",
            "required": false,
            "default": "",
            "placeholder": "请输入额外的敏感关键词，每行一个"
        }
    ]
}

```

**添加实现类**

实际的实现类可以参考官方关键词的实现方案，继承自内置的 Moderation 类，然后重写其中的 `moderation_for_inputs` 和 `moderation_for_outputs` 方法即可, 这两个方法对应的就是对输入和输出的审核。

关键词的审核比较简单，参考官方的关键词审核的实现即可。对于大模型审核，可以使用 `ModelManager` 提供的    `get_default_model_instance()` 方法获取用户自定义配置的默认大模型，这样可以与现有的 Dify 应用保持一致性。具体的大模型调用实现如下所示：

```python
from core.model_manager import ModelManager

model_manager = ModelManager()
model_instance = model_manager.get_default_model_instance(
    tenant_id=self.tenant_id,
    model_type=ModelType.LLM,
)

prompt_messages = [SystemPromptMessage(content=MODERATION_PROMPT), UserPromptMessage(content=text)]

response = cast(
    LLMResult,
    model_instance.invoke_llm(
        prompt_messages=prompt_messages,
        model_parameters={"temperature": 0.01, "max_tokens": 2000},
        stream=False,
    ),
)

```

## 总结
本文是基于 Dify 框架实现了完整的大模型审核功能，包括关键词审核和内容审核，可以最低成本地实现具体完整的自动化审核能力，在生产环境中，为了保证最终的效果，可能还需要与人工审核配合使用，以提升审核效果。感兴趣的同学可以自行动手实践一下。
