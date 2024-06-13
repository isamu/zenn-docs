---
title: "GraphAIの紹介"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

# GraphAIとは

GraphAIは、LLM/RAG/Tools/DBなどの複数のAgentを組み合わせて作る、Multi agentsシステムを効率よく構築するためにフレームワークです。

Multi agentsも数個の単純な組み合わせであればプログラムで記述することは容易です。しかし、複雑な組み合わせて、並列実行やデータの依存関係がある場合、並列で動くMulti agentsを作ることは容易ではありません。
最適化を考えないで作れば作ることはできますが、それぞれの処理に数秒かかるAgentを多く組み合わせるとユーザへのアウトプットが極端に遅くなります。また、スケーラビリティや、サーバ、クライアントでの役割、Stream処理など、現在のAIシステムは様々な要求があります。

GraphAIは

- 複雑な非同期/並列処理とデータの依存関係を解決できる
- TypeScriptで書かれていて、ブラウザとサーバの両方で動きます。
- Agentは１つのコードでサーバ、ブラウザの両方で動かすことができる
- ブラウザから利用する場合、少しの設定を追加するだけでブラウザのAgentを呼び出すだけでサーバでAgentを実行できる
- Streamをブラウザ、サーバでサポートする
- GraphAIを使うために様々なツールやサンプルが揃っている

といった特徴があります。

GraphAIで使うAgentは

- Agentは、それ単体で実装/テストができるので、開発しやすい
- Open Sourceで提供されているAgentを利用できる

です。

現在は、receptronという組織で開発をしています。

## Quick Start

まずは、簡単にGraphAIを使ってみましょう。npmの環境とOpenAIのapi keyを用意してください。

そして、GraphAI の cliをインストールします。

```
npm i -g  @receptron/graphai_cli
```

以下のyamlファイル(Graphデータ)をダウンロードして保存してください。

https://raw.githubusercontent.com/receptron/graphai/main/packages/cli/samples/business_idea_jp.yaml

そのファイルを保存したディレクトリーに.envというファイルをつくってOpenAIのAPI KEYを記述します。

```
OPENAI_API_KEY=sk-xxxx
```

用意は以上です。実行します。

```
$ graphai business_idea_jp.yaml
```

成功していれば、何も表示されないでしばらくまちます。

しばらくすると、「このビジネスアイデアコンテスト」のアイデア、評価、結果が表示されます。

yamlファイルをみると流れはわかると思いますがOpenAIのGPTに問い合わせるAgentが３回実行されています。
GPT(AI)への問い合わせと、その結果を使って更に問い合わせをするという流れです。

問い合わせの内容を変更すると、ビジネスアイデア以外の結果を得ることができるので、改良して使ってみてください。

このように、yamlファイルに定義することで、簡単にAIを使ったAgentを組み合わせて動かすことが可能となります。


## Sample

### yamlのサンプル

graphaiのcliコマンドで実行できる

- https://github.com/receptron/graphai/tree/main/packages/cli/samples

### tsのサンプル

```
npx ts-node  -r tsconfig-paths/register {filename}a
```

- https://github.com/receptron/graphai/tree/main/packages/samples/src
- https://github.com/receptron/graphai/tree/main/agents/llm_agents/samples

## レポジトリとnpm構成

### ソースコード

https://github.com/receptron/graphai

にあります。このレポジトリはモノレポとなっていて、GraphAI本体の他に、各種ツールやAgentが含まれます。

- [packagesディレクトリ](https://github.com/receptron/graphai/tree/main/packages)
  - GraphAI本体とcliやテスト用の各種ツール、AgentFilter,全てのAgentを利用できるagentのメタパッケージなど
- [agentsディレクトリ](https://github.com/receptron/graphai/tree/main/agents)
  - GraphAIで利用するagentsがそれぞれの分類ごとに分かれておいてある

各packages/agents以下のディレクトリはそれぞれのnpmのパッケージとして提供されています。

GraphAI本体は https://github.com/receptron/graphai/blob/main/packages/graphai/src/ 以下の

- [graphai.ts](https://github.com/receptron/graphai/blob/main/packages/graphai/src/graphai.ts)
  - GraphAI本体
- [node.ts](https://github.com/receptron/graphai/blob/main/packages/graphai/src/node.ts)
  - Agentと対応するNode
- [task_manager.ts](https://github.com/receptron/graphai/blob/main/packages/graphai/src/task_manager.ts)
  - 並列処理を含むAgentの実行を管理するタスクマネージャー
- [type.ts](https://github.com/receptron/graphai/blob/main/packages/graphai/src/type.ts)
  - 型定義

の４つのファイルで、合わせても1000行程度とコンパクトに作られています。

本体以外にAgent/AgentFilter/Validator/GraphAIの機能を拡張するAgent(nestしたGraphやGraphを並列で動かす）などを組み合わせて使うため、シンプルなエンジンで複雑な処理を行うことが可能となります。

## 用語説明

- Graph
  - 円グラフや棒グラフのグラフではなく、グラフ理論のグラフ。NodeとそれをつなぐEdgeで、それぞれのNodeの関連性を示す
- GraphAI
  - GraphAI本体
- Graphデータ
  - GraphAIで使うGraphを定義したデータ。グラフ理論のNodeはnodeとして、edgeはinputsで定義される。グラフは有向グラフで閉loopなし。
  - yaml, jsonファイルで定義したり、TypeScriptで直接記述も可能。
- Agent
  - GraphAIで実行されるプログラム。各Node(Computed Node)は１つのAgentと対応し、実行する。同じAgentを複数のNodeで使うこともできる。LLMを実行するLLM Agentや、データを処理するpopAgent, stringAgentなどがある。TypeScriptで記述する
- Node
  - GraphのNode。Static NodeとComputed Nodeの２種類がある。
- Static Node
  - データのみを扱うNode。初期値を定義したり、loop(同じグラフを繰り返し処理する）時に、前回実行した結果をデータとして受け取る。
- Computed Node
  - Agentに対応したNode。プログラムを実行する
- Agent Filter
  - 各Agentを実行する前に実行されるプログラム。共通の処理、AgentのデータのValidation, stream処理などができる。
- Node.js
  - JavaScriptの実行環境。GraphAIのnodeと混同しそうなので、Node.jsと表記します。
  
## GraphAIの簡単な動作の流れ

- yamlやjson,TypeScriptで記述されたGraphデータを読み込む
- Graphデータに矛盾がないかvalidationをする。矛盾があればエラーで終了
- 全てのComputed nodeをタスクキューに入れて、依存がないNodeから実行する
- １つのNodeの実行が終わると、それに依存し、他のタスク待ちがないNodeを探し、見つかればそのNodeが実行される。ただし並列動作の上限に達した場合には実行待ちとなる。
- AgentFilterが設定されている場合、各Nodeの実行前にAgentFilterが実行される
- 全てのNodeの実行が終わるとisResultで指定されたNodeの結果が返される。Loop指定がある場合は、最初から実行される。

## npm package

npmパッケージは以下で配布しています。

#### GraphAI本体

https://www.npmjs.com/package/graphai

#### GraphAI関連ツール

receptronのorganization配下で、ツールを提供しています。

yamlやjsonをcliで読み込んで使うツールやexpressのmiddlewareなどがあります。

[receptron organization](https://www.npmjs.com/org/receptron)

#### Agents

graphaiのorganization配下で、Agentを配布しています。
 [graphai organization](https://www.npmjs.com/org/graphai)

- [@graphai/agents](https://www.npmjs.com/package/@graphai/agents)
  - 配布しているnpmを全て盛り込んだメタパッケージ。
  - 含まれるパッケージは、[ここのソース](https://github.com/receptron/graphai/blob/main/packages/agents/src/index.ts)を参照してください
- [@graphai/vanilla](https://www.npmjs.com/package/@graphai/vanilla)
  - 他のnpmに依存しないagent群.
    - arrayの処理、データ処理、文字列やjsonの変換を行うagentなどがあります。
- [@graphai/service_agents](https://www.npmjs.com/package/@graphai/service_agents)  
  - wikipediaやfetchなどネットワークサービスを使うagent.
- [@graphai/llm_agents](https://www.npmjs.com/package/@graphai/llm_agents)
  - llmのagent. llmをapi経由で使うのにapi keyが必要になるのでサーバで使うことを想定しています。
- [@graphai/data_agents](https://www.npmjs.com/package/@graphai/data_agents)
  - データ加工


## Agentのdocument

各Agentの[AgentFunctionInfo](https://github.com/receptron/graphai/blob/ee6878a4cd1d453c0729ee7ffcad63f073427b64/packages/graphai/src/type.ts#L118-L140)に書かれた情報を元に、機械的に生成したdocumentが以下にあります。

[AgentDoc](https://github.com/receptron/graphai/blob/main/docs/agentDocs/README.md)

## AgentFunctionInfoについて

npmで配布されるagentは、実行されるagentを含む[AgentFunctionInfo](https://github.com/receptron/graphai/blob/ee6878a4cd1d453c0729ee7ffcad63f073427b64/packages/graphai/src/type.ts#L118-L140)の形式配布されます。
AgentFunctionInfoには、Agentの情報(nameやdescription)、サンプルの入出力、入力のスキーマなどの情報が含まれています。Sampleはドキュメントの自動生成やUnit testにも使われます。

Agentを作成してGraphAIで利用するときは、AgentFunctionInfoを作る必要があります。

(* 簡易的にagentを即時関数で使う方法や、AgentFunctionInfoのモックデータを使うなど、開発時には省略する方法もあります。別途説明予定）


## 簡単なGraphAIの使い方

こちらのチュートリアルを参考にしてください。英語ですが、ほとんどがソースなので翻訳を使いながらでも簡単に理解できます。

[公式チュートリアル](https://github.com/receptron/graphai/blob/main/docs/Tutorial.md)

このチュートリアルは、graphai_cliを使ってCliでyamlを読み込んでGraphAIを使います。
graphai_cliはGraphAI本体と、@graphai/agentsを含んでいて、全てのAgentを利用できます。


## Sampleの使い方

https://github.com/receptron/graphai/tree/main/packages/samples
以下に公式のサンプルが提供されています。

```
yarn run samples {sampleFile}
```
で実行できます。

T.B.D. fix sample file

<!--  ## inputについて  ## nestしたGraphについて   ## loopについて ## any/ifの使い方 -->



## AgentFilter

AgentFilterは、それぞれのComputed Nodeが実行される前に、なにかの処理を追加することができます。

[@graphai/agent_filters](https://www.npmjs.com/package/@graphai/agent_filters)ではhttpのstreamのためのfilterや、AgentFunctionInfoのinput schemaを使った入力値のvalidateを行うagent filterがあります。


### namedInput Validator

namedInputの値をagentFunctionInfoのinput schemaの情報を元にvalidationします

Testコードでの利用例
https://github.com/isamu/graphai/blob/agentFilter/packages/agent_filters/tests/validation/test_agent_namedinput_validator.ts

```typescript
import { GraphAI } from "graphai";
import * as agents from "@graphai/agents";
import { namedInputValidatorFilter } from "@graphai/agent_filters";

const agentFilters = [
  {
    name: "namedInputValidatorFilter",
    agent: namedInputValidatorFilter,
  },
];

const graph = new GraphAI(graph_data, agents, { agentFilters });
const results = await graph.run();

```

### streamAgentFilterGenerator

filterParamsのstreamTokenCallback関数を通してstreamのデータを受信します。
サーバ、クライアントで利用可能です。

#### server 例

https://github.com/receptron/graphai_utils/blob/main/packages/express/src/express.ts

express server
```typescript
    return async (req: express.Request, res: express.Response) => {
      res.setHeader("Content-Type", "text/event-stream;charset=utf-8");
      res.setHeader("Cache-Control", "no-cache, no-transform");
      res.setHeader("X-Accel-Buffering", "no");

      const callback = (context: AgentFunctionContext, token: string) => {
        if (token) {
          res.write(token);
        }
      };
      const streamAgentFilter = {
        name: "streamAgentFilter",
        agent: streamAgentFilterGenerator<string>(callback),
      };
      const agentFilters = [streamAgentFilter]

      const agentFilterRunner = agentFilterRunnerBuilder(agentFilters);
      const result = await agentFilterRunner(context, agent.agent);

      const json_data = JSON.stringify(result);
      res.write("___END___");
      res.write(json_data);
      return res.end();
   }
```

### web client

https://github.com/isamu/graphai-stream-web/blob/main/src/views/Home.vue

```typescript
const useAgentFilter = (callback: (context: AgentFunctionContext, data: T) => void) => {
  const streamAgentFilter = streamAgentFilterGenerator(callback);

  const agentFilters = [
    {
      name: "streamAgentFilter",
      agent: streamAgentFilter,
      agentIds: streamAgents,
    },
  ];
  return agentFilters;
};   

export default defineComponent({
  setup() {
    const streamingData = ref<Record<string, unknown>>({});

    const callback = (context: AgentFunctionContext, data: string) => {
      const { nodeId } = context.debugInfo;
      streamingData.value[nodeId] = (streamingData.value[nodeId] ?? "") + data;
    };
    const agentFilters = useAgentFilter(callback);
    
    const graphai = new GraphAI(graphData, agents, { agentFilters });
  }
})
```

### httpAgentFilter

グラフのフローで、agentの実行をバイパスし、http経由でサーバのagentを実行します。
Webでのサンプルはこちらにあります。

https://github.com/isamu/graphai-stream-web/blob/main/src/views/Home.vue


### agentFilterRunnerBuilder

GraphAIを使わないでagentFilterとagentを動かすRunnerです。
クライアントからサーバのagentを呼び出すときに、サーバ側で使います。
agentFilterやagentの単体テストでも利用可能です。

expressとtest時のサンプルはこちら。

express
https://github.com/receptron/graphai_utils/blob/main/packages/express/src/express.ts

test
https://github.com/isamu/graphai/blob/agentFilter/packages/agent_filters/tests/filters/test_filter_runner.ts


## Node.jsで使い方

GraphAIを動作させるのに必要な最小限のnpmはgraphaiといずれかのagentです。
最も簡単に使えるAgentのvanilla agents(@graphai/vanilla 他に依存がないagentをvanillaと呼んでいます)を使って簡単なGraphデータを作り、動かしてみます。

最初にnpmの初期化をして、必要なnpmを入れます。typescriptを実行するためにts-nodeも入れます。
```
npm init
yarn add graphai @graphai/vanilla ts-node
```

以下が最小限のスクリプトです。graphai.tsで保存します。
Static nodeのnode1でメッセージを指定して、そのデータを受け取ったComputed nodeのnode2でbypassAgentを実行します。
node2の結果を出力としてかえします。

```typescript
import { GraphAI } from "graphai";
import * as agents from "@graphai/vanilla";

const graph_data = {
  version: 0.5,
  nodes: {
    node1: {
      value: "hello, GraphAI",
    },
    node2: {
      agent: "bypassAgent",
      inputs: [":node1"],
      isResult: true,
    },
  },
};

export const main = async () => {
  const graph = new GraphAI(graph_data, agents);
  const result = await graph.run();
  console.log(JSON.stringify(result));

};

if (process.argv[1] === __filename) {
  main();
}
```

実行します
```sh
$ npx ts-node graphai.ts
{"node2":["hello, GraphAI"]}
```

node1で指定して文字列がnode2に渡され、結果として表示されました。


## Webのサンプル

Vue3で書かれたwebのサンプルです。

https://github.com/receptron/graphai-demo-web

サンプルサイト
https://graphai-demo.web.app/


![](/images/graphai/web-demo.png)

## WebのサンプルStream編

LLMをサーバで動かし、そのレスポンスをstreamで受け取るサンプルです。
サーバ側、クラアント側、両方でstreamをサポートしています。


https://github.com/isamu/graphai-stream-web

localで動作します。

root directoryとserver directoryでnpmのinstall.

```
yarn install
```

vueの起動

```
yarn run serve
```

serverの起動

```
yarn run server
```

で利用できます。

![](/images/graphai/stream.gif)



<!-- ## 透過的にサーバ/クライアントを使う -->
<!-- T.B.D -->

## AgentFunctionInfoについて



## Agentの開発方法


GraphAIで使うAgentの作り方の説明をします。

こちらのサンプルのagentのレポジトリがあります。
https://github.com/isamu/graphai_agent_template

Agentは、Agentの本体とそれをテストするテストコードで構成されます。
Agentの本体をsrc/以下(今回は[src/sample_agent.ts](https://github.com/isamu/graphai_agent_template/blob/master/src/sample_agent.ts)), Agentのテストコードをtest/以下(今回は[tests/test_agent.ts](https://github.com/isamu/graphai_agent_template/blob/master/tests/test_agent.ts))に作ります。

今回作成するAgentは`SampleAgent`という名前です。
動作は、GraphAIからの入力値(設定ファイルのparams)の`params`と、前のagentからの入力値`inputs`、この２つの値をmergeしてobjectとして返す簡単なAgentです。

#### Agent

Agentの本体はこちらです。

```typescript
import { AgentFunction } from "graphai";

export const sampleAgent: AgentFunction = async ({ params, inputs }) => {
  return { params, inputs };
};
```

Agentは必ず`AgentFunction`の型で、非同期(`async`)な関数です。
最初の説明にあったように、`params`と入力をとって、それを返すだけのコードです。

どんなAgentも基本的にはこのように入力値を受け取って、何らかの結果を返す１つの関数です。

#### AgentFunctionInfo

Agentは、AgentFunctionInfoの型のデータとしてGraphAIにわたす必要があります。
AgentFunctionInfoはAgent本体とinputs/params/resultのサンプルの値、Agentのメタ情報を含みます。

[src/sample_agent.ts](https://github.com/isamu/graphai_agent_template/blob/master/src/sample_agent.ts)に追加します。

```
// for test and document
```
以下にテストのサンプル値と、このAgentの情報(`AgentFunctionInfo`)を記載しています。
これらをsampleAgentInfoにまとめて`export`します。

サンプル値はUnit Test(つまり、サンプルのドキュメントとUnit Testの両方を兼ねています）で使います。
後述する開発時にはこのサンプル値を使ってTestRunnerでテストをします。

Agentのファイルは、agentを含むパッケージ情報(`AgentFunctionInfo`)を`defaule export`します。

```typescript
import { AgentFunctionInfo } from "graphai";

const sampleInput = [{message: "hello"}, {message: "test"}];
const sampleParams = { sample: "123" };
const sampleResult = { inputs: sampleInput, params: sampleParams };

const sampleAgentInfo: AgentFunctionInfo = {
  name: "sampleAgent",
  agent: sampleAgent,
  mock: sampleAgent,
  inputs: {
    type: "array",
  },
  samples: [
    {
      inputs: sampleInput,
      params: sampleParams,
      result: sampleResult,
    },
  ],
  description: "Sample agent",
  author: "isamu arimoto",
  repository: "https://github.com/isamu/graphai_doc",
  license: "MIT",
};

export default sampleAgentInfo;

```

複数のサンプル値を用意する場合は

```typescript
  samples: [
    {
      inputs: sampleInput,
      params: sampleParams,
      result: sampleResult,
    },
    {
      inputs: sampleInput2,
      params: sampleParams2,
      result: sampleResult2,
    },
  ],
```
とします。
複数のサンプル値があるときはテストランナーは全てのケースをテストします。

#### Unit Test

Agentのパッケージの情報を使ってAgent単体のUnit Testをします。
GraphAIに含まれるagentTestRunnerにsampleAgentInfoを渡してUnit Testを実行します。
agentTestRunnerは、sampleAgentInfoに含まれるサンプル値を使ってAgentを実行します。
この関数は、内部で`node:test`を使っています。
inputs, paramsのペアを使ってAgentを実行、結果とresultが一致すればテストは成功です。

```typescript
import sampleAgentInfo from "@/sample_agent";
import { agentTestRunner } from "@receptron/test_utils";

agentTestRunner(sampleAgentInfo);
```

package.jsonにテスト実行のスクリプトがあるので

```
yarn run test
```

でテストを実行します。

#### 開発

最初にAgentを開発するとき、この[レポジトリ](https://github.com/isamu/graphai_agent_template)をforkして使うと良いです。
Agent作成に必要な設定は `package.json`, `eslintrc.js`, `.prettierrc`, `tsconfig.json` に設定済みです

`src/sample_agent.ts`をベースに必要な実装を追加していき、期待すべき`samples`を更新、追加しながらUnit Testを動かします。
Unit TestをPassし、期待すべき動作がするようになればAgentは完成です。


#### Agentの受け取るデータ

sampleAgentでは、Agentの関数で`{ params, inputs }`を受け取りました。
実際は`AgentFunctionContext`の情報を受け取っています。

- params
  - GraphDataのyaml/jsonのparamsで与えられる値
- inputs
  - GraphDataのinputsで指定される値。前に実行したAgentの結果や、static nodeでの値、inputsに書かれた値など。
  - inputsはArrayで、入力の数と同じ長さ。
- namedInputs
   - inputsのobject(dictonary)時
   
基本的にはこのうちの２つ(paramsと(inputs or namedInputs))を入力として受け取り、Agentの処理をします。結果はreturnで返します。
Agentの結果は、次に実行されるAgentのinputsなどで利用されます。(inputの記述方法は別途解説します)

inputsとinputsはGraphDataで
- `inputs: [":node1", ":node2"]` とarrayにした場合はinputs,
- `inputs: {v1: ":node1", v2: ":node2"}` とした場合にはobjectでnamedInputs
が渡されます。同時に２つがくることはありません。

- agentFilters
  - AgentFilterという仕組みを使う場合に使う
- debugInfo
  - debug時などに必要な情報

で、それ以外はNestedGraphで使う特別なデータなので、通常は利用しません。


## Agentのdocument生成とテスト

AgentFunctionInfoに含まれる方法を使って、agentのunit testを実行する、documentを自動生成することが可能です。

Test runnerは、`@receptron/test_utils`に含まれています。
```
import { agentTestRunner } from "@receptron/test_utils";

const main = async () => {
  await agentTestRunner(agentFunctionInfo);
};

main();
```

Documentの自動生成はこちらを参考に。今後、利用しやすい形式で提供予定です。

https://github.com/receptron/graphai/blob/main/packages/cli/src/docs.ts



## Graphデータの作り方

