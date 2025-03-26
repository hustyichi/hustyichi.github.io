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
2025 年是大模型应用爆发的一年。从年初的 DeepSeek 吸引大量开发者部署大模型产品，到 Manus 和 MCP 等 Agent 方案持续引发关注，行业内掀起了一股打造爆款大模型应用的热潮。

然而，在这股热潮中，大模型的安全性问题往往被忽视。事实上，AI 应用的安全性是实现完整大模型应用的关键环节。2024 年的一篇文章 [Building A Generative AI Platform](https://huyenchip.com/2024/07/25/genai-platform.html) 中，曾介绍过大模型应用的完整架构：

![arch](/img/in-post/dify-safety/arch.png)

其中，安全模块主要包括`输入围栏(Input guardrails)`和`输出围栏(Output guardrails)`。尽管文章提到了安全机制，但未提供具体的实现方案。在本文中，我们将基于当前流行的大模型应用平台 Dify，探索并实践一套完整的安全性解决方案。我们将从简单方案入手，逐步扩展到更符合生产环境需求的实现。

## 安全性方案
为了实现文本内容的安全性检测，我们可以根据实现成本从低到高，选择以下三种方案：

1. **基于关键词与规则的审查**：通过维护敏感词库并使用正则表达式或文本匹配算法进行检测。此方案实现简单、效率高，但准确率较低。
2. **基于机器学习的内容审核**：利用现有审核模型或训练独立模型，尽管实现复杂，但可以显著提升准确率。
3. **基于人机结合的审核机制**：人工智能审核可能存在遗漏，尤其是模糊边界问题，人工审核能够确保最终的准确性。

人机结合的审核更多体现在工程设计上，因此本文主要聚焦于前两种方案的实践与落地。

## 方案实践

在实践过程中，我们按照成本从低到高的顺序展开，先使用内置方案，再逐步扩展到自定义方案。

### 基于内置方案的实践

Dify 提供了两种开箱即用的安全性方案：基于关键词的审查和基于 OpenAI 的内容审核。这些功能可以在平台的功能页面直接启用：

![default](/img/in-post/dify-safety/default.png)

**关键词方案**

关键词审查是最简单的入门方案。通过配置关键词库和预设输入输出的响应内容，即可快速上手。

实际测试表明，关键词功能符合预期，但在生产环境中存在以下限制：
- 默认关键词只能通过网页端配置，动态更新受限。
- 关键词数量限制为 100 个，难以满足生产需求。

**OpenAI 审核方案**

基于 OpenAI 的审核方案属于机器学习内容审核的范畴。然而，在国内环境中使用此方案存在较大障碍。此外，国内外审核政策差异较大，若要落地生产环境，仍需自定义实现。

### 基于自定义拓展的方案选型

Dify 考虑到内置方案的局限性，提供了两种自定义拓展方式：
- **API 扩展**：通过部署独立的审核服务并提供 API 接口，灵活控制审核逻辑。但此方案需要额外维护服务，增加成本。
- **代码扩展**：直接在 Dify 的代码仓库中实现自定义审核逻辑，无需额外部署，且支持灵活的前端配置。

**API 扩展**

API 扩展的实现较为简单，只需遵循 [官方文档](https://docs.dify.ai/zh-hans/guides/extension/api-based-extension/moderation) 的 API 请求参数规范即可。但维护独立服务的成本较高，适合对灵活性要求较高的场景。

**代码扩展**

代码扩展无需额外部署，且能无缝嵌入现有应用中。通过遵循 [官方实现规范](https://docs.dify.ai/zh-hans/guides/extension/code-based-extension/moderation)，可以实现灵活的前端配置选项，效果如下：

![custom](/img/in-post/dify-safety/custom.png)

### 基于代码扩展的实践

在实现自定义方案时，我们选择关键词库与大模型结合的方案。关键词过滤效率高，可快速拦截违规内容；对于漏网之鱼，再通过大模型进行二次审核，兼顾效率与成本。

考虑到 Dify 应用中通常已部署大模型，我们优先复用现有模型，避免额外部署成本。同时，为满足不同场景需求，可提供纯关键词审核的选项。

**实现步骤**

1. **初始化目录**

   在 Dify 项目的 `api/core/moderation` 下创建目录和文件：

   ```
   api/core/moderation
                ├── custom
                    └── custom.py
                    └── schema.json
   ```

2. **添加前端组件定义文件**

   在 `schema.json` 中定义前端配置项，例如：

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
               "default": "Keyword"
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

3. **添加实现类**

   实现类继承自内置 `Moderation` 类，重写 `moderation_for_inputs` 和 `moderation_for_outputs` 方法。关键词审核可参考官方已有的组件快速实现；大模型审核可通过 `ModelManager` 调用用户配置的默认大模型，这样可以更好的地适配 Dify 框架，避免用户需要额外配置安全围栏的大模型。

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

### 效果验证

完成实现后，通过 ChatGPT 生成一批测试问题验证效果。实际测试表明，大部分场景下均能准确拦截违规内容，但涉及儿童教育的问题国内的大模型没有触发拦截，这种情况也符合国情，基本上符合预期。

![case1](/img/in-post/dify-safety/case1.png)

![case2](/img/in-post/dify-safety/case2.png)

![case3](/img/in-post/dify-safety/case3.png)

## 总结
本文基于 Dify 框架，实践了关键词审核与内容审核的完整实现方案。该方案以最低成本实现自动化审核能力，适用于生产环境。为提升审核效果，可结合人工审核进一步优化。感兴趣的读者不妨亲自实践一下。
