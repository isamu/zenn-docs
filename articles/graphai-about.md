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

他、サンプルのwebレポジトリではクライアント側でstreamを受信するagent filterや、クライアント側でAgentを実行するときに、動的にサーバ/クライアントでのAgentを割り振って、透過的にAgentを実行するAgenなどを使っています。

(この透過的なサーバ/クライアントについては別途説明します)

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


## WebのサンプルStream編

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


## 透過的にサーバ/クライアントを使う

T.B.D

## Agentの開発方法

T.B.D

## Agentのdocument生成とテスト

T.B.D

## Graphデータの作り方

