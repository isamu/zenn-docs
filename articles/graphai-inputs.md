---
title: "GraphAIのデータ入力について"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

GraphAIでは、Agentを実行するときに前のノードの結果を入力値として受け取ることができます。入力値を受け取ることができるのはComputed Nodeです。一方で、Static Nodeは特別な条件のときに他のノードの結果を受け取ることができますが、通常はvalueを受け取れませんので、注意が必要です。


# GraphAIのComputed Nodeにおける入力について

Computed Nodeは`inputs`という要素を使ってデータを受け取ることができます。`inputs`を指定する方法を説明します。

### 入力値の定義

- **inputsという名前の要素で定義する**  
  `inputs`の値は、arrayまたはobjectのいずれでも記述できます。

```json
nodes: {
  StaticNodeA: {
    value: "hello",
  },
  ComputedNodeB: {
    agent: "bypassAgent",
    inputs: [":StaticNodeA"]
  },
  ComputedNodeC: {
    agent: "bypassAgent",
    inputs: {"A": ":StaticNodeA", "B": ":ComputedNodeB"}
  },
}
```

- **直接値を設定する場合**  
  数値（例: `123`）、文字列（例: `"ABC"`）、オブジェクト（例: `{test: "hello"}`）などを直接入力することができます。

```json
nodes: {
  ...,
  ComputedNodeB: {
    agent: "bypassAgent",
    inputs: [123, "abc", {message: "message"}]
  },
}
```


- **ノードを指定する場合**  
  ノードを指定するには、`":node1"`のように、`:（コロン）`で始まる文字列を使用します。ノードを指定した場合、そのノードが結果を返すまで実行を待機します。言い換えると、`inputs`でノードを指定することで、実行順を決めることができます。

- **特定のプロパティを指定する場合**  
  例えば、`inputs: [":nodeA.property1"]`と記述すると、`nodeA`の結果の`property1`が渡されます。ここで期待するデータ形式は `{ property1: "value" }` のようなオブジェクトです。

```json
nodes: {
  nodeA: {
    value: {
      property1: "value"
    },
  },
  ComputedNodeB: {
    agent: "bypassAgent",
    inputs: [":nodeA.property1"]
  },
}
```


- **配列の要素を指定する場合**  
  `inputs: [":nodeA.result.$0"]` のように記述すると、`nodeA`の結果の中にある配列 `result` の最初の要素が渡されます。ここで `$0` は配列の添字を示しています。例えば、`{ results: ["A", "B"] }` というデータ形式の場合、`"A"` が入力として渡されます。

```json
nodes: {
  nodeA: {
    value: {
      result: ["A", "B"],
    },
  },
  ComputedNodeB: {
    agent: "bypassAgent",
    inputs: [":nodeA.result.$0"]
  },
}
```

- **OpenAIのレスポンス例**  
  OpenAIのレスポンスを処理する場合、例えば `inputs: [":openAI.choices.$0.message.content"]` のように記述します。これにより、OpenAIのレスポンスから `choices` 配列の最初の要素の `message.content` が入力として使用されます。

このように、`inputs` を使って様々な形式で入力値を定義し、ノードの結果や特定のデータを柔軟に扱うことができます。


# GraphAIにおける`inputs`のデータ形式

`inputs`は、arrayまたはobjectのいずれでも記述できます。これにより、データの受け渡し方法が異なります。

### データ形式の定義

- **Arrayの場合**  
  `inputs`がarrayとして定義されると、Agentには`inputs`という名前でarrayが渡されます。例えば、`inputs: [":nodeA"]`と定義すると、Agent内では`inputs`として受け取ります。

- **Objectの場合**  
  `inputs`がobjectとして定義されると、Agentには`namedInputs`という名前でobjectが渡されます。例えば、`inputs: {dataA: ":nodeA"}`と定義すると、Agent内では`namedInputs`として受け取ります。

### ネストされたデータ

- **ネストした部分にノードを指定する場合**  
  `inputs: {dataA: {DataB: ":nodeA"}}`のようにネストされた部分にノードを指定しても、データとして展開されず、`:nodeA`という文字列が直接渡されます。必ず1階層目のvalueに入力ノードを指定するようにしてください。

これにより、`inputs`でデータを渡す際に、適切な形式で受け渡しが行われるようにしてください。




## Static nodeのデータ受取


Static Nodeは、特定の条件下で他のノードの値を受け取ることができます。特に、loopやnested graphの場合に利用されます。

### Loop時のStatic Node

- **Loopの際のStatic Node**  
  Loop処理中にStatic Nodeの値を受け取ることができます。例えば、以下のように定義された場合：

```json
  loop: {
    5,
  },
  nodes: {
    staticNode: {
      value: "init value",
      update: ":lastDataOfloop"
    },
    lastDataOfloop: {
      ...
    },
  }
```



`staticNode`の値は、loopの1回目では `"init value"` となり、2回目以降は `lastDataOfloop` Nodeの結果で上書きされます。

:::message
ComputedNodeと異なり, Static Nodeのvalueに`:nodeName`のようにNodeを指定しても展開されずそのまま文字列として扱われるので使わないように気をつけてください。
入力ノードを指定できるのはupadteのみです。
:::

### Nested Graphの場合

- **暗黙のケース（Static Nodeを定義しない）**

親Agentの入力値が子Graph内のStatic Nodeとして自動的に受け取られます。例：

```json
parentNodeResult: {
  agent: "arrayFlatAgent",
  inputs: {
    array: [":NodeA", ":NodeB"],
  },
},
childGraph: {
  agent: "nestedAgent",
  inputs: { history: ":parentNodeResult.array" },
  isResult: true,
  graph: {
    version: 0.5,
    nodes: {
      nodeB: {
        agent: "bypassAgent",
        inputs: [":history"]
      }
    }
  }
}
```

この場合、`childGraph`の`inputs`で指定された `:parentNodeResult.array` が `namedInputs` で受け取られ、`history` ノードが暗黙的に定義されます。`childGraph`内で `history` ノードを入力値として使うことができます。

- **明示的にStatic Nodeを定義する場合**
明示的にStatic Nodeを定義することで、`update` を使って値を上書きすることができます。例：

```json
parentNodeResult: {
  agent: "arrayFlatAgent",
  inputs: {
    array: [":NodeA", ":NodeB"],
  },
},
childGraph: {
  agent: "nestedAgent",
  inputs: { history: ":parentNodeResult.array" },
  isResult: true,
  graph: {
    version: 0.5,
    loop: {
      5
    },
    nodes: {
      history: {
        value: "",
        update: ":prevResult",
      },
      nodeZ: {
        agent: "bypassAgent",
        inputs: [":history"]
      }
    }
  }
}
```

この場合、`history` ノードを明示的に定義し、`update` を使って値を設定することができます。`update` はloop時に使われるため、2回目以降のloopで指定された内容で上書きされます。


## まとめ

- Computed Node
  - inputsを使ってデータを渡すことができる
  - arrayまたはobjectの形式に応じて異なる方法で渡される。
- Static Node
  - loopやnested graph内で他のノードの値を受け取ることができる
  - updateを使うことで、loop時に値を動的に上書きすることができる。
