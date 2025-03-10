---
title: "GraphAI - GrapysにおけるNested Graph Agentのinputs/outputsの実装メモ"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---



# Nested Agent を GUI で扱う際の課題

Nested Agent を GUI で扱うのはやや難しい。  
これは、入れ子になった Graph Data を GUI の `inputs` / `outputs` に適切にマッピングする必要があるためである。  
`inputs` / `outputs` は各 Graph Data によって異なるため、Graph Data をもとに決定する必要がある。

## Inputs の処理

- すべてのデータは `Static Node` を経由して渡される。そのため、`Static Node` を参照すればよい。
- サブグラフを作成する際は、`Static Node` にテストデータを入力し、テストを行うことができる。  
  ただし、Nested Agent 内で使用する場合は、値が上書きされる点に注意が必要。  
  （※ただし、入力が接続されていない場合はデフォルト値が適用される。）

### 課題

- ループ処理時のデータ領域も `Static Node` として `update` の値で更新されるため、その `Static Node` が `inputs` に含まれる（例: `messages` など）。  
  - これにより、`inputs` に入力があると、動作に影響を及ぼす可能性がある。
- `inputs` の情報を `metadata` に含め、サブグラフ作成時に指定できるようにするのが望ましいかもしれない。（**TODO: 要検討**）


## Result の処理

- `result` の扱いは特に難しいが、以下のルールが存在する。
  - `isResult: true` となっているノードのみが、サブグラフの結果として親の Nested Agent に渡される。  
  - したがって、`isResult` の設定に注意することで、正しく結果を処理できる。
- もう一つの問題点として、データが `nodeId` をキーとしたオブジェクトで、ネストされたオブジェクトであることが挙げられる。


### GUI におけるデータマッピング

GUI では、単純なオブジェクトを `inputs` / `outputs` にマッピングしている。  `inputs` / `outputs`のオブジェクトのkeyを入力/出力値として、その値をそのまま渡している。
nestedAgentではそのままの結果を使うと、そのkeyはサブグラフの各nodeIdとなるので、その中のデータを使うのには変換が必要となってしまう。
そのため、ネストされたオブジェクトのデータはフラット化する必要がある。
```
{
  "nested": {
    "llm1": {
      "text": "aaa",
      "message": {
         ...some data
      }
    },
    "llm2": {
      "text": "aaa",
      "message": {
         ...some data
      }
    }
  }
}
```
を
```
{
  "nested": {
    "llm1_text": "aaa",
    "llm1_message": {
       ...some data
    }
    "llm2_text": "aaa",
    "llm2_message": {
       ...some data
    }
  }
}
```

このマッピングを実現するために、サブグラフ内で `Graph Data` の `output` を用いて結果を適切に変換（transform）する。  
さらに、この結果情報を `metadata` に格納し、GUI の `profile` として活用することで、より適切なデータの管理が可能になる。

