---
title: "GraphAI - Nested Graph Agentの紹介"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

GraphAIはTypeScriptで記述されたLLM, RAG, Web Accessの機能を持ったAgentと、それらAgentの組み合わせを定義したGraphDataを使って、マルチエージェントシステムを簡単に作ることが可能です。

![GraphData](https://storage.googleapis.com/zenn-user-upload/f8393a413ef9-20240908.png)
*GraphData*


## NestedAgent

特定の機能を持ったGraphDataを作成し、それをAgentとして見立てて別のGraphからAgentとして別のGraphDataを使うことができます。この仕組みをNested Graph(入れ子になったグラフ）と呼びます。

つまり、複数のエージェントの組み合わせで作った１つの機能をGraphDataとして定義し、それを再利用することが可能となります。
再利用目的の他、このような単機能のエージェントを複数組み合わせてマルチエージェントを作ることもできます。

その他、複雑な処理をするために組み合わせ利用することもできます。


Nested GraphはNestedAgentを使い、graph要素に、入れ子にするGraphDataを記述します。

![](https://storage.googleapis.com/zenn-user-upload/92a1c34ac561-20240908.png)
*Nested Graph Agent例*

```json
{
  "version": 0.5,
  "nodes": {
    "source": {
      "value": "Hello World"
    },
    "ragNode": {
      "agent": "bypassAgent",
      "inputs": [":source"]
    },
    "nestedNode": {
      "agent": "nestedAgent",
      "inputs": {
        "inner0": ":ragNode"
      },
      "isResult": true,
      "graph": {
        "nodes": {
          "childNode": {
            "agent": "bypassAgent",
            "inputs": [
              ":inner0"
            ],
            "isResult": true
          }
        }
      }
    }
  }
}
```

この例では、ragAgentやllmAgentの変わりに簡単にテストするためにbypassAgentを使っています。
bypassAgentは入力値をそのまま出力値で返すAgentです。

- ragNodeにsourceの値が入力
- nestedNodeにragNodeの結果が入力({inner0: :ragNode})
- nestしているGraphDataのinner0に:ragNodeが渡る
- childNodeにinner0が渡る

となり、結果、最初のsourceの値が、childNodeのinputsにわたります。

このjsonを保存してcliのgraphaiで実行すると以下の結果が得られます。

```json
{
  "nestedNode": {
    "childNode": [
      [
        "Hello World"
      ]
    ]
  }
}
```


## Map Agent

**Map Agent**は、`nested graph` の応用例の1つであり、複数のネストされたグラフを用意して、並列でデータ処理を行う手法です。

- **Nested Graph**は、グラフがエージェント（agent）としてネストされた単純な構造です。
- 一方、**Map Agent**は、このネストされたグラフを複数コピーして用意し、それぞれに異なるデータを渡して並行処理を行います。

## Map Agentの動作のしくみ

1. **入力として配列（array）を受け取る**  
   Map Agentは入力として配列を受け取り、この配列の各要素を1つ1つ取り出します。

2. **配列の要素ごとにNested Graphを作成**  
   配列の要素の数だけNested Graphを生成し、それぞれのNested Graphに対して個別にデータを渡します。

3. **各Nested Graphで並行処理を実行**  
   各Nested Graphが独立して並行処理を実行します。これにより、複数のデータを効率的に処理することが可能です。

Map Agentの考え方は、多くのプログラミング言語で実装されている`map`関数や、`map-reduce`の並列処理モデルに似ています。

- **`map`関数**: 入力配列の各要素に対して、指定した処理を並行して適用します。
- **`map-reduce`モデル**: 大規模データ処理で使用される手法で、`map`で並行処理を行い、最終的に`reduce`で結果を集約します。

これらの概念に馴染みのある方は、Map Agentの動作も理解しやすいでしょう。


![](https://storage.googleapis.com/zenn-user-upload/e9aa4a5b667f-20240908.png)
*Map Agent例*


## Map Agentの入力

`mapAgent`の入力は、特別な形式を取ります。`namedInputs`を使用して入力を指定し、`rows`を設定する必要があります。`rows`には配列（array）を渡し、この各要素が`mapAgent`内の`nestedGraph`に対して個別に処理されます。

### 入力の設定

具体的には、`mapAgent`の設定で次のように`rows`を指定します：

```yaml
inputs: { rows: ":arrayNode" }
```
この設定により、`rows`の各値（例: `rows[0]`, `rows[1]`, など）が、それぞれの`nestedGraph`に`row`として渡されます。

### Nested Graph内での設定

`nestedGraph`の内部では、次のように`inputs`を定義することで、個別の入力を処理します：

```yaml
inputs: [":row"]
```
この設定により、`rows`から個別の値が`row`としてnestedGraphに順次提供され、並行して処理を行うことが可能になります。


```json
{
  "version": 0.5,
  "nodes": {
    "source1": {
      "value": "Hello World 1"
    },
    "source2": {
      "value": "Hello World 2"
    },
    "ragNode": {
      "agent": "bypassAgent",
      "inputs": [":source1", ":source2"]
    },
    "mapNode": {
      "agent": "mapAgent",
      "inputs": {
        "rows": ":ragNode"
      },
      "isResult": true,
      "graph": {
        "nodes": {
          "childNode": {
            "agent": "bypassAgent",
            "inputs": [
              ":row"
            ],
            "isResult": true
          }
        }
      }
    }
  }
}
```
これを実行すると以下の結果が返ってきます。

```json
{
  "nestedNode": {
    "childNode": [
      [
        "Hello World 1"
      ],
      [
        "Hello World 2"
      ]
    ]
  }
}
```