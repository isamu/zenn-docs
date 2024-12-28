---
title: "GraphAI - GraphAIでエージェントを任意の言語で実装する方法"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::


# Node以外の言語でAgentを作る

GraphAIはTypeScriptで実装されており、エージェント（Agent）も通常TypeScriptで記述されます。しかし、GraphAIにはHTTP経由でエージェントを実行する仕組みが備わっており、この仕組みを応用することで、エージェントを任意のプログラミング言語で実装することが可能です。

# 必要な設定
エージェントをHTTP経由で実行するためには、以下の2つの設定を行う必要があります：

1. `agentFilter` で **HTTP 経由のエージェント実行を指定する**
通常、エージェントはGraphAI内で直接実行されますが、`httpAgentFilter`を指定することで、エージェントを外部サーバ経由で実行する設定に変更します。
1. `bypassAgentIds` で**エージェントの検証をスキップする**
GraphAIのコンストラクタではエージェントの有無を検証する仕組みがありますが、これをスキップするために対象のエージェントIDを `bypassAgentIds` に設定します。


# 設定例

## HTTPエージェントのフィルター設定
外部サーバ上でエージェントを実行するために、`httpAgentFilter` を設定します。サーバが複数ある場合、それぞれのサーバについて設定を行います。

## 検証のスキップ設定
エージェント検証をスキップするエージェントIDを `bypassAgentIds` に追加します。

```TypeScript
import { httpAgentFilter } from "@graphai/agent_filters";

const agentFilters = [
  // Pythonで実装されたAgentの設定
  {
    name: "httpPythonServer",
    agent: httpAgentFilter,
    filterParams: {
      server: {
        baseUrl: "http://localhost:8085/agents",
      },
    },
    agentIds: ["python1Agent", "python2Agent"],
  }
  // Rustで実装されたAgentの設定
  {
    name: "httpRustServer",
    agent: httpAgentFilter,
    filterParams: {
      server: {
        baseUrl: "http://localhost:8080/agents",
      },
    },
    agentIds: ["rustAgent"],
  }
];

// 以下のAgentのvalidationをskipさせる
const bypassAgentIds = ["python1Agent", "python2Agent", "rustAgent"];

const graphai = new GraphAI(
  graph,
  {
    ...agents,
  },
  { agentFilters, bypassAgentIds },
);

```

# 実装の流れ

1. **HTTPサーバを用意**
任意の言語でエージェントのロジックを実装し、HTTPリクエストを受け付けるサーバを構築します。GraphAIからはこのサーバに対してリクエストが送信されます。
1. `httpAgentFilter` にサーバを設定
GraphAIにHTTP経由でエージェントを実行する設定を行います。
1. `bypassAgentIds` を設定
検証をスキップさせたいエージェントIDを追加します。

# メリット

- **多言語対応**: GraphAIのエージェントロジックをTypeScript以外の言語で記述可能。
- **柔軟性**: 外部サーバ上で複雑なロジックや専用の環境を利用可能。
- **分散処理**: サーバを分散させることで負荷分散やスケーラビリティを向上。
- **秘匿性**: エージェントやそれを実行する設定(DBのパスワードなど）を共有せずにAgentを共有可能

これらの設定を行うことで、任意の言語で実装された外部サーバのエージェントをGraphAIで利用できるようになります。

