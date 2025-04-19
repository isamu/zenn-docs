---
title: "GraphAIのTips"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

GraphAIのGraphData作成時に使えるtips


### mapAgentを使う場合は、params: { compositeResult: true }を使う

通常mapAgentを使うと、それぞれインスタンスの結果がarrayとして返ってきます。

```json
nodeId: [{
 childNode: {
   // some result1
 }
}, {
 childNode: {
   // some result2
 }
}]
```

`compositeResult: true`を追加すると

```json
nodeId: {
  childNode: [{
   // some result1
  }, {
   // some result2
  }]
}
```
となります。


### graphai.run()で、結果の型を指定する

genericで型指定ができます。

```TypeScript
const result = await graphai.run<{date: string}>()

const output = result.node?.data; // nodeが、{date: string}となる
```


### LLMの履歴をloopで回す

```yaml
version: 0.5
loop:
  count: 5
nodes:
  history:
    value: []
    update: ":nextHistory.array"
  messageData:
    agent: "stringTemplateAgent"
    inputs: { user: "hello", assistant: "how are you?" }
    params:
      template:
        - role: "user"
          content: "${user}"
        - role: "assistant"
          content: "${assistant}"
  nextHistory:
    agent: "arrayFlatAgent"
    inputs: {array: [:history, :messageData]}
    console:
      after: true
```      

### 入力のテンプレート

inputsは、arrayか１段階のobjectのみ、入力値のデータに変換できるので、nestしているデータを構成したい場合は、stringTemplateAgentで加工する

```yaml
  messageData:
    agent: "stringTemplateAgent"
    inputs: { user: ":prompt", assistant: ":llm.choices.$0.message.content" }
    params:
      template:
        - role: "user"
          content: "${user}"
        - role: "assistant"
          content: "${assistant}"
```          

### loopのcounter
Loop処理時のcount処理

```yaml
version: 0.5
loop:
  count: 5
nodes:
  counter:
    value: 0
    update: ":nextCount.result"
  nextCount:
    agent: "dataSumTemplateAgent"
    inputs: { array: [:counter, 1] }
    console:
      after: true
  
```

### デバック

yamlやjsonで、各ノードのデータをprintする

```yaml
  console:
    after: true
```

typescriptで実装時は、即時関数で扱うこともできる

```typescript
  debug: {
    agent: (args: any) => {
      console.log(args);
    },
    inputs: { json: ":jsonParse", a: ":nextHistory", b: ":counter", c: ":prompt" },
  },
````

   