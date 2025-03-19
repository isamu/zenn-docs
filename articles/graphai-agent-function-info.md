---
title: "GraphAI - AgentFunctionInfoについて"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: false
publication_name: "singularity"
---


## AgentFunctionInfoについて

AgentをComputed Nodeから呼び出される関数で、単純に入力(inputsとparams)を受け取り結果を返します。Agentはnpmで配布されますがその関数のみではなく、Agentに関する情報を定義したAgentFunctionInfoとして提供されます。AgentFunctionInfoの１つの要素としてAgentの関数が含まれます。

AgentFunctionInfoには、Agent名、概要、必要な環境変数、サンプルデータ、スキーマなど様々な情報が含まれています。

特にスキーマとサンプルはとても重要です。

GraphAIではそれらの情報を使って

- AgentのUnit Test
- AgentのDocumentの自動生成
- Expressと組み合わせたときに、提供するAgentの情報をAPIとして提供
- AgentFilterと組み合わせて、Agentの入力値のValidation

などなど、様々な機能を自動化しています。

## Test
`@receptron/test_utils`にAgentFunctionInfoを使ったTest Runnerがあります。

```
import { agentTestRunner } from "@receptron/test_utils";
const main = async () => {
  for await (const agentInfo of Object.values(packages)) {
    await agentTestRunner(agentInfo);
  }
};
```

samplesの各sampleのinputs/paramsを使ってAgentを実行し、結果とresultが一致するかテストします。
多くのAgentでこれを使って自動的にUnit testをしています。

GraphAIのレポジトリでは、各Agentで `yarn run test` としてテストしています。

## Document
Agentのスキーマと、サンプルデータ、それらを使ったGOD FormatのサンプルをmdのDocumentとして生成します。
生成されたDocumentはGithub上でまとめられます。

https://github.com/receptron/graphai/blob/main/docs/agentDocs/README.md

ドキュメントを作るツールはnpmパッケージとして提供されており、packageのnpm rootで

```
npx agentdoc
```

と、実行できます。

`package.json`、および`/lib/index`から情報を読み取ってdocumentを作成します。

agentdocのソースはこちら。

https://github.com/receptron/graphai/tree/main/packages/agentdoc

## express

```
app.get(apiPrefix + "/", agentsList(agentDictionary, hostName, apiPrefix));
```
のように、agentDictionaryを渡すと、そのサーバで利用できるAgentの一覧、および、それぞれのAgentの情報を返します。

ソースはこちら。
https://github.com/receptron/graphai-utils/blob/main/packages/express/src/agents.ts

この機能を使うと、他の環境のGraphAI(サーバ、ブラウザ）に対し、そのサーバが提供できるAgentの情報を渡すことができます。呼び出し元のGraphAIはその情報を使ってサーバ上のGraphAIをAPIとして呼び出すことができます。

## AgentFilterによるValidation

`inputSchema`の定義と実際の入力値を使ってagentのvalidatonができます。
ソースはこちら。
https://github.com/receptron/graphai/blob/main/packages/agent_filters/src/filters/namedinput_validator.ts

使い方はこちらを参考にしてください。GraphAIのモノレポでは、CIでこちらを実行して各Agentのsample値をチェックしています。

https://github.com/receptron/graphai/blob/main/packages/agent_filters/tests/validation/test_validator.ts


# AgentFunctionInfoの定義

ソースはこちらにあります。
https://github.com/receptron/graphai/blob/main/packages/graphai/src/type.ts

2025/3/20現在の型はこちら。

```
export type AgentFunctionInfoSample = {
  inputs: any;
  params: DefaultParamsType;
  result: any;
  graph?: GraphData;
};

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

TODO: outputFormat, cacheType, tools, hasGraphDataなどを使っている部分をreferする。

## 開発時に

AgentFunctionInfoは便利なのですが、開発時にはまだ決まってないことも多いので、必ずしも事前にこれらの情報が用意できません。

agentを作るのは

無記名関数で実装する

```
const graphData = {
  agent: ({namedInputs}) =>  {
  }
}
```

関数だけ作って、ダミーのAgentFunctionInfoを入れるagentInfoWrapperを使う

```
const agent = ({namedInputs}) => {
 ...
};
```

Generatorを使ってスケルトンからAgentを作って、sampleを利用しunit testベースで仕上げる

https://zenn.dev/singularity/articles/graphai-create-graphai-agent

などの方法を試してください。