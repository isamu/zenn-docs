---
title: "GraphAI - AgentFunctionInfoについて"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: false
publication_name: "singularity"
---


## AgentFunctionInfoについて

AgentをComputed Nodeから呼び出される関数で、単純に入力(inputsとparams)を受け取り結果を返します。
Agentはnpmで配布されますがその関数のみではなく、Agentに関する情報を定義したAgentFunctionInfoとして提供されます。
AgentFunctionInfoの１つの要素としてAgentの関数が含まれます。

AgentFunctionInfoには、Agent名、概要、必要な環境変数、サンプルデータ、スキーマなど様々な情報が含まれています。

特にスキーマとサンプルはとても重要です。

GraphAIではそれらの情報を使って

- AgentのUnit Test
- AgentのDocumentの自動生成
- Expressと組み合わせたときに、提供するAgentの情報をAPIとして提供
- AgentFilterと組み合わせて、Agentの入力値のValidation

などなど、様々な機能を自動化しています。


# AgentFunctionInfoの定義

ソースはこちらにあります。
https://github.com/receptron/graphai/blob/main/packages/graphai/src/type.ts

2025/3/20現在の型はこちら。

```
export type AgentFunctionInfo = {
  name: string;
  agent: AgentFunction<any, any, any, any>;
  mock: AgentFunction<any, any, any, any>;
  inputs?: any; // inputs data schema
  output?: any; // output data schema
  params?: any; // params data schema
  config?: any; // config data schema
  outputFormat?: any;
  tools?: Record<string, any>[]; // function calling(tools) schema.
  samples: AgentFunctionInfoSample[]; // sample data. This is for document and unit test.
  description: string;
  category: string[];
  author: string;
  repository: string;
  license: string;
  cacheType?: CacheTypes;
  environmentVariables?: string[]; // Environment variables required for execution
  hasGraphData?: boolean; // The agent that executes graph data using nestedAgentGenerator is true
  stream?: boolean; // is stream support?
  apiKeys?: string[];
  npms?: string[];
};
```

- agentの本体と、agentに関する情報
- GraphAIの動作のみならず、様々なツールで利用可能


npmで配布されるagentは、実行されるagentの関数と、そのAgentの情報を含む[AgentFunctionInfo](https://github.com/receptron/graphai/blob/ee6878a4cd1d453c0729ee7ffcad63f073427b64/packages/graphai/src/type.ts#L118-L140)の形式配布されます。

AgentFunctionInfoには、Agentの情報(nameやdescription)、サンプルの入出力、入力のスキーマなどの情報が含まれています。Sampleはドキュメントの自動生成やUnit testにも使われます。

Agentを作成してGraphAIで利用するときは、AgentFunctionInfoを作る必要があります。

(* 簡易的にagentを即時関数で使う方法や、AgentFunctionInfoのモックデータを使うなど、開発時には省略する方法もあります。別途説明予定）
