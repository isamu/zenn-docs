---
title: "GraphAI GraphData(ワークフロー)をAgent化するnestedAgentGeneratorの紹介"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---



GraphAIには、ワークフローを定義する`GraphData`をagent内で実行するnestedAgentがあります。
nestedAgentは`GraphData`内に入れ子で定義されているサブグラフの`GraphData`（ワークフロー）を実行します。
このため、nestedAgentで使いたい`GraphData`を再利用しようと思った場合には、TypeScript側で親`GraphData`に子`GraphData`を追加するなどの処理が必要で、必ずしも再利用が容易にできませんでした。

これを解決するためにnestedAgentをgeneratorとして再利用し、`nestedAgentGenerator`を追加しました。
https://github.com/receptron/graphai/blob/main/agents/vanilla_agents/src/graph_agents/nested_agent.ts

vanilla agentのnested_agentの一部として配置しています。
利用するにはvanilla agent (`@graphai/vanilla`)をインストールしインポートします

```typescript
import { nestedAgentGenerator } from "@graphai/vanilla/lib/graph_agents/nested_agent";
```
::::message
2025/01/15現在の仕様です。変更する可能性があります
::::


これを使うと`nestedAgentGenerator`に`GraphData`を渡すとそのグラフを実行するagentを作成することができます。
使い方は簡単で

```typescript
const newAgent = nestedAgentGenerator(graphData);
```
とgraphDataを渡すだけです。

これでnewAgentは、graphDataを実行するagentとなります。
agentFunctionInfoを追加しパッケージ化すればnpmでagentとして配布も可能です。

```typeScript
const newAgentInfo = {
  name: "newAgent",
  agent: newAgent,
  mock: newAgent,
  samples: [],
  description: "",
  category: [],
  author: "",
  repository: "",
  tools: [],
  license: "",
};

export default newAgentInfo;
```

利用時にも１つポイントがあります。
現状、サブグラフをもつagentでないと、nestedAgentに必要な情報が渡らないのでダミーのサブグラフを親グラフに追加しておく必要があります。
```typeScript
    nestedGraphWorkFlow: {
      agent: "newAgent",
      inputs: {},
      graph: { version: 0, nodes: {} },
    },
```

---

GraphAI has a `nestedAgent` that executes `GraphData` within an agent to define workflows.
The `nestedAgent` executes subgraph `GraphData` (workflows) that are nested and defined within the main `GraphData`.

As a result, when you try to reuse `GraphData` for the `nestedAgent`, you need to process it on the TypeScript side, such as adding child `GraphData` to the parent `GraphData`. This means reuse is not always straightforward.

To address this issue, we have introduced `nestedAgentGenerator`, which allows the `nestedAgent` to be reused as a generator.
https://github.com/receptron/graphai/blob/main/agents/vanilla_agents/src/graph_agents/nested_agent.ts

It is included as part of the nested_agent in the vanilla agent.
To use it, install and import the vanilla agent (@graphai/vanilla).


```typescript
import { nestedAgentGenerator } from "@graphai/vanilla/lib/graph_agents/nested_agent";
```
::::message
This is the specification as of January 15, 2025. It is subject to change.
::::

By using this, you can create an agent that executes a graph by passing `GraphData` to the `nestedAgentGenerator`.

It's simple to use—just pass the following:


```typescript
const newAgent = nestedAgentGenerator(graphData);
```
and graphData.



With this, newAgent becomes an agent that executes the provided graphData.

By adding agentFunctionInfo and packaging it, you can distribute it as an agent via npm.


```typeScript
const newAgentInfo = {
  name: "newAgent",
  agent: newAgent,
  mock: newAgent,
  samples: [],
  description: "",
  category: [],
  author: "",
  repository: "",
  tools: [],
  license: "",
};

export default newAgentInfo;
```

There is one important point to note when using it.

Currently, unless the agent contains a subgraph, the necessary information for the `nestedAgent` cannot be passed. Therefore, you need to add a dummy subgraph to the parent graph in advance.

```typeScript
    nestedGraphWorkFlow: {
      agent: "newAgent",
      inputs: {},
      graph: { version: 0, nodes: {} },
    },
```
