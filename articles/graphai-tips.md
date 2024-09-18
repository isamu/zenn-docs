---
title: "GraphAIのTips"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

GraphAIのGraphData作成時に使えるtips

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
    inputs: ["hello", "how are you?"]
    params:
      template:
        - role: "user"
          content: "${0}"
        - role: "assistant"
          content: "${1}"
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
    inputs: [":prompt", ":llm.choices.$0.message.content"]
    params:
      template:
        - role: "user"
          content: "${0}"
        - role: "assistant"
          content: "${1}"
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
    update: ":nextCount"
  nextCount:
    agent: "dataSumTemplateAgent"
    inputs: [:counter, 1]
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

   