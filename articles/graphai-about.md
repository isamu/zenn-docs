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

にあります。このレポジトリはモノレポとなっていて、GraphAI本体の他に、各種ツールやAgentが含まれます。GraphAI本体と各種ツールはpackages以下に、Agentはagents以下においてあります。
それぞれのツール/Agentはnpmで提供されています。

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
  - GraphAIで使うGraphを定義したデータ。グラフ理論のNodeはnodeとして、edgeはinputsで定義される。グラフは有向グラフでloopなし。
- Agent
  - GraphAIで実行されるプログラムの関数。各Node(Computed Node)は１つのAgentを実行する。同じAgentを複数のNodeで使うこともできる。LLMを実行するLLM Agentや、データを処理するpopAgent, stringAgentなどがある。TypeScriptで記述する
- Node
  - GraphのNode。
- Static Node
  - データのみを扱いNode. 初期値を定義したり、loop(同じグラフを繰り返し処理する）時に、前回実行したデータを受け取る。
- Computed Node
  - Agentに対応したNode。プログラムを実行する
- Agent Filter
  - 各Agentを実行する前に実行されるプログラム。共通の処理、AgentのデータのValidation, stream処理などができる。

## 簡単な動作の流れ

- yamlやjsonでGraphデータを読み込む
- Graphデータに矛盾がないかvalidationをする
- タスクキューに入れて、依存がないNodeから実行する
- Nodeの実行が終わると、それに依存している次のNodeが実行される
- AgentFilterが設定されている場合、各Nodeが実行前にAgentFilterが実行される
- 全てのNodeの実行が終わるとisResultで指定されたNodeの結果が返される

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

各Agentについては、機械的に生成したdocumentが以下にあります。
[AgentDoc](https://github.com/receptron/graphai/blob/main/docs/agentDocs/README.md)

## 簡単なGraphAIの使い方

こちらのチュートリアルを参考にしてください。英語ですが、ほとんどがソースなので翻訳を使いながらでも簡単に理解できます。

このチュートリアルは、graphai_cliを使ってCliでyamlを読み込んでGraphAIを使います。
graphai_cliはGraphAI本体と、@graphai/agentsを含んでいて、全てのAgentを利用できます。

[公式チュートリアル](https://github.com/receptron/graphai/blob/main/docs/Tutorial.md)

## Sampleの使い方

https://github.com/receptron/graphai/tree/main/packages/samples
以下に公式のサンプルが提供されています。

```
yarn run samples {sampleFile}
```
で実行できます。

<!--  ## inputについて  ## nestしたGraphについて   ## loopについて ## any/ifの使い方 -->

## AgentFilter


## Webのサンプル


## 透過的にサーバ/クライアントを使う

## Agentの開発方法


## Agentのdocument生成とテスト
