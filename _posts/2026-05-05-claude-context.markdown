---
layout: post
title: "大型代码库开发的全新解法 - claude-context"
subtitle:   "A New Solution for Large Codebases — claude-context"
date:       2026-05-05 20:30:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - llm
    - agent
    - rag
---

## 背景介绍

简单介绍自 2025 下半年以来的 AI 编程的快速发展，虽然能力得到快速发展，但是依旧在小型项目，新项目中表现更好，原有的大型项目表现会更差。

原因主要来自于大型项目需要关注的信息更多，实际开发时需要考虑的内容更多，大模型很难面面俱到。大模型更像一个很聪明，但是不了解你的个人实际情况的工程师。

在过去的解决方案中，Claude Code 等开发工具选择使用 CLI 工具，比如 Grep 等帮助快速检索相关信息，但是这种方案依赖存在明确的字段精确匹配，要不然就很难发挥作用，当然过往大模型有比较强的理解能力，做了不少的能力适配，提升了构建查询条件的能力，但是依旧主要以精确匹配为主。

项目 [claude-context](https://github.com/zilliztech/claude-context) 提供了全新的解决方案，基于 RAG 提供了更强的代码检索能力，利用多维向量打通自然语言与代码之间的连接，提供更加语义化的检索能力。在官方提供的基准测试中，可以降低 40% 的 Token 消耗，可以降低成本和时间。

![efficiency](/img/in-post/claude-context/mcp_efficiency_analysis_chart.png)

## 核心架构

当前项目的核心架构如下所示：

![arch](/img/in-post/claude-context/Architecture.png)

熟悉 RAG 的工程师应该会很容易理解这个架构，其实就是一个典型的 RAG 方案，只是提供了一个针对代码库的解决方案。

- 左侧的 `Embedding Service` 对应于的项目模型的选型，考虑到当前方案需要实现代码与自然语言的关联，因此向量模型的选型需要针对代码场景进行有针对性的训练，目前选择的模型主要是 OpenAI Embdding, VoyageAI Embdding 模型
- 右侧向量库的选择目前使用的是 Milvus, 针对这种检索场景，准确性主要依赖向量模型的正确选型，知识库只要选择主流的预期都可以正常工作；
- 中间的文本处理主要是代码分片方案的设计，优先使用 AST Parse 这种基于代码语法树的切分方案，AST Parse 分片过大的情况下则基于滑动窗口 + 必要的 Overlap 去执行切分。


## 检索方案
Claude context使用的检索方案。为向量检索+bm25混合检索方案，具体的在 `packages/core/src/context.ts` 中的 `semanticSearch()` 方法中完成，因为底层的使用 Milvus 进行支撑，因此整体的设计极其简单，仅将用户的查询转换为 Milvus 的混合检索的查询即可, 并使用 RRF 进行混合检索的结果排序。 简化后如下所示：

```ts
            const searchRequests: HybridSearchRequest[] = [
                {
                    data: queryEmbedding.vector,
                    anns_field: "vector",
                    param: { "nprobe": 10 },
                    limit: topK
                },
                {
                    data: query,
                    anns_field: "sparse_vector",
                    param: { "drop_ratio_search": 0.2 },
                    limit: topK
                }
            ];

            const searchResults: HybridSearchResult[] = await this.vectorDatabase.hybridSearch(
                collectionName,
                searchRequests,
                {
                    rerank: {
                        strategy: 'rrf',
                        params: { k: 100 }
                    },
                    limit: topK,
                    filterExpr
                }
            );
```



## 代码分片方案

因为Claude-context的主要面对的是代码分片问题。因此需要选择相对合理的分片方案。默认使用的是基于AST的分片方案， AST 分片使用 tree-sitter 解析代码，将代码转换为AST树，然后按语言里的“逻辑单元”进行切分，常见的逻辑单元包括： 函数、类、方法、接口等。这样可以尽可能地避免方法的不完整。具体的实现在 `packages/core/src/splitter/ast-splitter.ts` 中完成。

项目里的 AST 分片方案可以理解成：

**先用 tree-sitter 把代码解析成语法树，再从语法树里提取函数、类、接口等“逻辑代码单元”作为 chunk；如果不支持或失败，再退回字符分片。**

核心文件是 [ast-splitter.ts](/Users/sm4749/works/claude-context/packages/core/src/splitter/ast-splitter.ts:28)。

**整体步骤**

1. **判断语言是否支持 AST 分片**

   `split(code, language, filePath)` 会先调用 `getLanguageConfig(language)`。

   支持的语言包括：

   `javascript`、`typescript`、`python`、`java`、`cpp`、`go`、`rust`、`csharp`、`scala` 等。

   如果不支持，比如 Markdown、Solidity，就直接使用 LangChain fallback 分片。

2. **选择对应的 tree-sitter parser**

   例如：

   - `typescript` 使用 `tree-sitter-typescript`
   - `python` 使用 `tree-sitter-python`
   - `go` 使用 `tree-sitter-go`

   然后执行：

   ```ts
   this.parser.setLanguage(langConfig.parser);
   const tree = this.parser.parse(code);
   ```

3. **根据语言准备“可切分节点类型”白名单**

   项目预定义了每种语言值得切成 chunk 的 AST 节点类型。

   例如 TypeScript：

   ```ts
   [
     'function_declaration',
     'arrow_function',
     'class_declaration',
     'method_definition',
     'export_statement',
     'interface_declaration',
     'type_alias_declaration'
   ]
   ```

4. **递归遍历整棵 AST**

   `extractChunks()` 从根节点开始 DFS 遍历。

   对每个节点检查：

   ```ts
   if (splittableTypes.includes(currentNode.type)) {
       // 提取为 chunk
   }
   ```

   命中后，用 tree-sitter 的位置直接从原始源码中截取：

   ```ts
   const nodeText = code.slice(currentNode.startIndex, currentNode.endIndex);
   ```

   同时记录：

   - `startLine`
   - `endLine`
   - `language`
   - `filePath`

5. **没有命中任何节点时，整文件作为一个 chunk**

   如果一个文件里没有函数、类、接口等可切节点，项目不会丢弃它，而是把整个文件作为一个 chunk。

6. **过大的 chunk 会继续按行拆小**

   `extractChunks()` 只负责 AST 结构切分。

   之后 `refineChunks()` 会检查每个 chunk 的长度。如果超过 `chunkSize`，默认 `2500` 字符，就按行继续拆成更小的 chunk。

7. **给相邻 chunk 添加 overlap**

   最后 `addOverlap()` 会给第 2 个及之后的 chunk 前面补上前一个 chunk 末尾的 `chunkOverlap` 内容，默认 `300` 字符，用来保留上下文连续性。

**例子**

假设有一个 TypeScript 文件 `user.ts`：

```ts
interface User {
  id: string;
  name: string;
}

function formatUser(user: User) {
  return `${user.name} (${user.id})`;
}

class UserService {
  getUser(id: string) {
    return { id, name: "Ada" };
  }

  deleteUser(id: string) {
    return true;
  }
}
```

tree-sitter 会把它解析成类似这样的结构：

```text
program
├─ interface_declaration  // interface User
├─ function_declaration   // function formatUser
└─ class_declaration      // class UserService
   ├─ method_definition   // getUser
   └─ method_definition   // deleteUser
```

项目的 TypeScript 白名单包含：

```ts
interface_declaration
function_declaration
class_declaration
method_definition
```

所以 `extractChunks()` 会产出这些 chunk：

**Chunk 1: interface**

```ts
interface User {
  id: string;
  name: string;
}
```

元数据大致是：

```ts
{
  startLine: 1,
  endLine: 4,
  language: "typescript",
  filePath: "user.ts"
}
```

**Chunk 2: function**

```ts
function formatUser(user: User) {
  return `${user.name} (${user.id})`;
}
```

**Chunk 3: class**

```ts
class UserService {
  getUser(id: string) {
    return { id, name: "Ada" };
  }

  deleteUser(id: string) {
    return true;
  }
}
```

**Chunk 4: method getUser**

```ts
getUser(id: string) {
  return { id, name: "Ada" };
}
```

**Chunk 5: method deleteUser**

```ts
deleteUser(id: string) {
  return true;
}
```




## 动态更新机制

claude-context 面临一个问题，代码经常变动，如何动态更新代码知识库，保证知识库中的代码和实际的代码一致性，每次都全量更新肯定是不现实的，因此需要有相对合理的发现差异并执行动态增量更新的机制。

考虑到基于内容去确定是否发生变化，很容易想到的方案就是基于内容的哈希方案。目前的项目也采用的是类似的方案，对每个文件计算 SHA-256 hash，并把 {relativePath -> hash} 保存到本地快照，后续就可以快速确认代码是是否发生了变化。

但是持续进行文件的逐步比较依旧比较慢。目前实际使用的方案是基于 [merkleDAG](https://docs.ipfs.tech/concepts/merkle-dag/#further-resources) 进行快速的差异判定，其核心思想是借助单个文件的 hash 构造出整个目录的 hash，如果 hash 发生变化，就意味着目录中的文件发生了变化，同时可以递归的去查找发生变化的文件具体是在子目录中的哪一个。这个项目里的 Merkle DAG 比较简单：它不是目录树多层级 Merkle Tree，而是一个 root 节点挂所有文件节点。

生成过程在 buildMerkleDAG()：

创建一个新的 MerkleDAG。
取出所有文件路径。
创建 root 节点，root 的数据是 "root:" + 所有文件 hash 拼接。
每个文件创建一个子节点，数据是：
relative/path.ts:fileHash
简化结构：

root: hashA + hashB + hashC
├── src/index.ts:hashA
├── src/context.ts:hashB
└── README.md:hashC

然后同步时会重新生成一份新的 fileHashes 和 merkleDAG：

```typescript
const newFileHashes = await this.generateFileHashes(this.rootDir);
const newMerkleDAG = this.buildMerkleDAG(newFileHashes);
const changes = MerkleDAG.compare(this.merkleDAG, newMerkleDAG);
```

如果 DAG 不同，再用 compareStates(oldHashes, newHashes) 精确判断：

- 新 map 有、旧 map 没有：新增文件
- 新旧都有但 hash 不同：修改文件
- 旧 map 有、新 map 没有：删除文件

4. 增量更新时先清理过时内容

Context.reindexByChange() 拿到 added / removed / modified 后，处理顺序很关键：

对 removed 文件：删除向量库中该 relativePath 的所有 chunk。
对 modified 文件：先删除旧 chunk。
对 added + modified 文件：重新读取、切块、embedding、写入向量库。

旧内容清理靠 deleteFileChunks()：先按 relativePath == "...file..." 查询 Milvus 中的 chunk id，再按 id 删除。见 context.ts (line 398) 和 context.ts (line 474)。

5. 只重嵌入变更文件，所以快

新增/修改文件会进入 processFileList()，复用完整索引时的切块和批量 embedding 流程，但输入只包含变更文件列表：

```ts
const filesToIndex = [...added, ...modified].map(f => path.join(codebasePath, f));
```

因此不会重新处理整个仓库。大仓库里这就是主要效率来源。


## 总结

本文主要介绍了 Claude-Context 项目。随着 AI 编程进入深水区，AI 编程工具需要解决实际中存在的大的老项目的维护问题。如何更高效地进行代码的检索和匹配，就体现出越来越重要的价值。在这种情况下，诞生了 Claude-Context。虽然目前这个项目的整体架构相对简单，目前的解决方案也算不上完美，但是提供了一个工业级 AI 编程代码检索的比较好的参考机制，可以给大家提供一个良好的借鉴。