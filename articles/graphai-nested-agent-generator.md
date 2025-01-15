---
title: "GraphAI GraphData(ワークフロー)をAgent化するnestedAgentGeneratorの紹介"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---



GraphAIには、ワークフローを定義するGraphDataをagent内で実行するnestedAgentがあります。
nestedAgentが、GraphData内に入れ子で定義されているサブグラフのGraphData（ワークフロー）を実行します。
このため、nestedAgentで使いたいGraphDataを再利用しようと思った場合には、TypeScript側で親GraphDataに子GraphDataを追加するなどの処理が必要で、必ずしも再利用が容易にできませんでした。

これを解決するためにnestedAgentをgeneratorとして再利用し、nestedAgentGeneratorを追加しました。
https://github.com/receptron/graphai/blob/main/agents/vanilla_agents/src/graph_agents/nested_agent.ts

vanilla agentのnested_agentの一部として配置しています。
利用するにはvanilla agent (`@graphai/vanilla`)をインストールしインポートします

```typescript
import { nestedAgentGenerator } from "@graphai/vanilla/lib/graph_agents/nested_agent";
```
::::message
2025/01/15現在の仕様です。変更する可能性があります
::::


これを使うとnestedAgentGeneratorにGraphDataを渡すとそのグラフを実行するagentを作成することができます。
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
