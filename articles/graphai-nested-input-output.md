---
title: "GraphAI NestedGraphへのデータの受け渡しについて"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

GraphAIは、Agent内でサブグラフを実行する仕組みがあります。それをNestedGraphとよび、 nestedAgent と mapAgentで利用可能です。

### 1. NestedAgent と MapAgent の概要
- **NestedAgent** は **単純にサブグラフを入れ子にするだけ** の構造。親グラフ内で定義されたワークフローを、特定の入力データとともにそのまま実行する。
- **MapAgent** は **データが異なるが同じ処理を並列に動かす仕組み**。配列データを展開し、それぞれの要素に対して同じサブグラフを実行する（MapReduce 的な処理）。

### 2. サブグラフへのデータの受け渡し
#### NestedAgent
- `inputs` を **サブグラフの static node にマッピング** する。
- 例えば `{dataA: 1, dataB: 2}` を渡すと、そのままサブグラフに適用される。

```yaml
version: 0.5
nodes:
  nested:
    isResult: true
    inputs:
      dataA: 1
      dataB: 2
    agent: nestedAgent
    graph:
      version: 0.5
      nodes:
        copy:
          isResult: true
          agent: copyAgent
          inputs:
            dataA: :dataA
            dataB: :dataB
```

このケースではnestedAgent実行時にサブグラフに

```
dataA: {
 value: 1
}
dataB: {
 value: 2
}
```
のstatic nodeが追加されます。その結果

```json
{
  "nested": {
    "copy": {
      "dataA": 1,
      "dataB": 2
    }
  }
}
```

となります。

#### MapAgent
- `rows` に **必ず配列を指定** する必要がある。
- 配列の各要素が `row` としてサブグラフに渡され、**配列の長さだけサブグラフが生成** される（MapReduce 的な処理）。
- `rows` 以外の `inputs` は、そのままサブグラフに渡される。
- `rows` 以外の `inputs` に **配列を指定すると、そのままコピーされ、すべてのサブグラフに渡る**。なので、rowsだけが配列を特殊に扱います。

```yaml
version: 0.5
nodes:
  map:
    isResult: true
    inputs:
      rows: [1,2]
      test: [a,b]
      test2: z
    agent: mapAgent
    graph:
      version: 0.5
      nodes:
        copy:
          isResult: true
          agent: copyAgent
          inputs:
            data: :row
            test: :test
            test2: :test2
```



### 3. サブグラフの結果処理
#### MapAgent の通常の出力
- サブグラフの結果は **配列として返される**。
- そのため、サブグラフ内の特定のノードの結果だけを取得するのは少し不便。

```json
{
  "map": [
    {
      "copy": {
        "data": 1,
        "test": [
          "a",
          "b"
        ],
        "test2": "z"
      }
    },
    {
      "copy": {
        "data": 2,
        "test": [
          "a",
          "b"
        ],
        "test2": "z"
      }
    }
  ]
}
```


#### `compositeResult: true` の利用
- **ノードごとの結果を配列にまとめてくれる**。
- これにより、次の Agent が **特定のノードの結果だけ** を取得しやすくなる。

```yaml
    params:
      compositeResult: true
```

```json
{
  "map": {
    "copy": [
      {
        "data": 1,
        "test": [
          "a",
          "b"
        ],
        "test2": "z"
      },
      {
        "data": 2,
        "test": [
          "a",
          "b"
        ],
        "test2": "z"
      }
    ],
    "copy2": [
      {
        "data": 1,
        "test": [
          "a",
          "b"
        ],
        "test2": "z"
      },
      {
        "data": 2,
        "test": [
          "a",
          "b"
        ],
        "test2": "z"
      }
    ]
  }
}
```

### 4. `output` の利用（NestedGraph & MapAgent）
- **`output` を指定することで、GOD format を使い、出力データの構造をカスタマイズ可能**。
- **NestedGraph での利用が主** だが、MapAgent でも使用可能。
- **GOD format の指定方法**
  - **自分のノードの結果を参照するため、ノードの指定は不要**。
  - `.`（ドット）から始め、**サブノードの `id` や `props` を指定** していく。

```yaml
version: 0.5
nodes:
  nested:
    isResult: true
    inputs:
      test: [a,b]
      test2: z
    agent: nestedAgent
    output: 
      data: .copy.test
      date2: .copy2.test2
    graph:
      version: 0.5
      nodes:
        copy:
          isResult: true
          agent: copyAgent
          inputs:
            test: :test
        copy2:
          isResult: true
          agent: copyAgent
          inputs:
            test2: :test2
```

```json
{
  "nested": {
    "data": [
      "a",
      "b"
    ],
    "date2": "z"
  }
}
```

## まとめ
- **NestedAgent** は単純に **サブグラフを入れ子にする** だけの仕組み。
- **MapAgent** は **データが異なるが同じ処理を並列に動かす（MapReduce 的な処理）**。
- **NestedAgent** は `inputs` を static node にマッピングする。
- **MapAgent** は `rows` を展開し、配列の要素ごとにサブグラフを作成する。
- **MapAgent** で**`compositeResult: true` を指定すると、サブグラフのノードごとの結果を取得しやすくなる**。
- **`output` を指定すると、GOD format に基づいてデータの構造をカスタマイズできる**。

以上が、GraphAI における **NestedGraph と MapAgent のデータの受け渡し & 結果処理** のまとめです！

