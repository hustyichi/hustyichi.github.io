---
layout: post
title: '基于 OpenAPI 的 Web 页面自动渲染实践'
subtitle:   "Practice of Automatic Rendering of Web Pages Based on OpenAPI"
date:       2023-08-13 09:28:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - JSON Schema
    - OpenAPI
---

## 背景信息
最近接手的项目中遇到一个新的需求，需要对原有的机器学习相关的接口进行切换，因为机器学习算法训练包含的参数比较多，结构也比较复杂，而且其中的参数之前存在复杂的动态依赖关系。不过纵使结构上的变化很大，常规情况下只要存在明确的对应关系，接口的切换也不算麻烦。

但是一个麻烦的问题在于前端 Web 页面的展示是基于接口展示的，其中的渲染逻辑会依据接口本身的层次关系以及具体的字段类型动态调整，而前人的技术方案存在严重的扩展障碍，其实现方案如下：

1. 在 Web 后台实现了复杂的接口描述编辑页面，定义了接口的层次关系以及基础的字段类型定义，同时为了前端逻辑的展示的需要，还定义部分特殊的字段类型；
2. Web 前端基于现有的接口描述进行实际渲染，但是考虑到接口描述结构的描述能力限制，还必须增加不少特殊情况的处理；

原有的技术方案存在下面的问题：

1. 接口的变更基本上意味着原有方案的大量改造，因为目前的结构与类型完全不同，意味着定义需要重新调整，更麻烦的是前端页面的渲染逻辑中存在着不少特殊处理，现在都需要进行大量的改造；
2. 本次变更完成之后将来的接口变更依旧需要花费比较多时间，对接口变更的可拓展能力很弱；
3. 后台配置字段存在着依赖人工判断的问题，人工配置字段错误会导致用户大量的请求异常；

## 技术方案探索
根据上面的问题，原有的技术方案基本不可用，需要一种更加具备拓展性的技术方案。预期需要有一套完备的接口描述方案，然后基于这套完备的接口描述文件进行动态渲染。每次用户访问页面时，实时请求接口描述文件，基于这个接口描述文件进行页面渲染。这样后续的接口升级就能平滑过渡了，不需要后台配置，避免了人工配置异常。接下来就是探索这套方案实现的难度。

#### 接口描述方案探索
接口描述方案优先选择成熟的标准，简单调研就发现了 [OpenAPI](https://github.com/fishead/OpenAPI-Specification/blob/master/versions/3.1.0.md) 规范，这是一个与编程语言无关的 RESTful 规范，从应用的广泛层度来看应该可以确认是完备的接口描述规范。而且更友好的是，现有的 Python Web 服务 FastAPI 就是基于 OpenAPI 的，生成应该是比较方便的。

但是实际应用时我们期望裁剪接口数据，仅仅获取接口字段描述相关内容，而不是完整的 OpenAPI 的接口描述信息，参考文档以及 FastAPI 的实现，确认实际的字段描述事实上就是基于 [JSON Schema](https://json-schema.org/understanding-json-schema/index.html) 实现的，从目前来看，JSON Schema 就是通过一套简单的规则可以对现有的字段与接口进行了描述，这不恰好符合需求吗。JSON Schema 描述是怎样的呢，下面举一个例子：

原始接口数据如下所示：

```JSON
{
  "first_name": "George",
  "last_name": "Washington",
  "birthday": "1732-02-22",
  "address": {
    "street_address": "3200 Mount Vernon Memorial Highway",
    "city": "Mount Vernon",
    "state": "Virginia",
    "country": "United States"
  }
}
```

最终的对应的 JSON Schema 描述可能是如下所示的：

```JSON
{
  "type": "object",
  "properties": {
    "first_name": { "type": "string" },
    "last_name": { "type": "string" },
    "birthday": { "type": "string", "format": "date" },
    "address": {
      "type": "object",
      "properties": {
        "street_address": { "type": "string" },
        "city": { "type": "string" },
        "state": { "type": "string" },
        "country": { "type" : "string" }
      }
    }
  }
}
```

通过下面的描述我们可以知道其中的字段类型以及结构关系，基于对应的 JSON Schema 即可自解释确认接口信息。

而且对于 Python 而言，如果已经使用了 Pydantic ，可以基于已有 Pydantic 对象，可以通过 [model_json_schema()](https://docs.pydantic.dev/latest/usage/json_schema/) 方法直接生成 schema 数据，然后就可以通过接口直接返回给前端即可。

#### 渲染方案探索
基于现有的标准 Schema 数据，可以实现通用的渲染能力，后续所有的变更就不需要进行额外的开发了，而且前端可以根据实际需要定义具体的渲染方案，只要符合 JSON Schema 规范，前后端就解耦了。

但是实现一套完整的 JSON Schema 的解析与渲染依旧需要花费不少时间，不过考虑到 JSON Schema 已经是相对成熟的标准，因此调研是否存在相对成熟的解决方案，简单调研就发现，前端存在 JSON Schema Form 的做法。对于复杂的 Form 页面，根据后端提供的 JSON Schema 数据，自动渲染出页面，这不就是我们需要的吗？对现有的 JSON Schema Form 进行了一些整理，开源社区已经实现了很多了，整理在这边：

- https://github.com/lljj-x/vue-json-schema-form 1.7K star，项目在持续迭代中，支持 Vue2
- https://github.com/xaboy/form-create/tree/next 4.9K star，项目在持续迭代中，支持 Vue
- https://github.com/alibaba/formily 9.8k star，阿里开源，项目持续迭代中，支持 Vue，React
- https://github.com/wearebraid/vue-formulate 2.2K star，项目更新频率不高，支持 Vue
- https://github.com/vue-generators/vue-form-generator 2.9K star，项目看起来没有再更新了
- https://github.com/baidu/amis 14.2K star，百度开源，项目持续迭代中，支持 React，可以通过 SDK 支持 Vue
- https://github.com/rjsf-team/react-jsonschema-form 12.8K star，项目在持续迭代中，支持 React，可以使用 https://rjsf-team.github.io/react-jsonschema-form/ 进行在线 JSON Schema 渲染测试
- https://github.com/alibaba/x-render 6.2K star，阿里开源，项目持续迭代中，支持 React
- https://github.com/JakHuang/form-generator 8.1K star，项目没有再更新了
- https://github.com/koumoul-dev/vuetify-jsonschema-form 502 star，项目没有再更新了

最终因为实际项目目前使用 Vue2 开发的，选择了 https://github.com/lljj-x/vue-json-schema-form ，其中包含了在线渲染预览的支持，可以比较方便地确认展示效果。

## 技术方案
根据之前的探索，确定了最终的技术方案：

1. 接口的描述使用 [JSON Schema](https://json-schema.org/understanding-json-schema/index.html)，基于标准协议描述接口字段信息；
2. 后端 Schema 的生成基于 Pydantic 对象的 [model_json_schema()](https://docs.pydantic.dev/latest/usage/json_schema/) 直接生成；
3. 前端基于 JSON Schema Form 进行渲染，实际选择 https://github.com/lljj-x/vue-json-schema-form，并基于其提供的实时在线渲染调整渲染效果；

## 实践中的问题

