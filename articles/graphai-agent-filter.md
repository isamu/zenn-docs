---
title: "GraphAI - AgentFilter"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

# GraphAI Agent Filterの紹介
GraphAIは、各Agentを実行するときに、その前後に処理を挟むAgentFilterという機能があります。

イメージとしては、expressのmiddle wareや昔のRailsのaround filterのような機能です。
具体的にはAgentが実行される前後に処理を追加することができます。共通のvalidationやデータのフィルター/加工などで利用することができます。
Agentの実行前と、実行後の結果に対して処理ができ、実行前の処理で実際にAgentを呼び出さないでAgentFilterだけで結果を返すことも可能です。

使い方はstreamAgentFilterというAgentFilterを例にすると

```javascript
const agentFilters = [
  {
    name: "streamAgentFilter",
    agent: streamAgentFilter,
  },
];
```
とfilter名とfilterの関数をarray内にいれて

```javascript
const graphai = new GraphAI(graphData, agents, { agentFilters });
```

と、GraphAIのコンストラクタの３つ目の引数に渡して使います。

AgentFilterの型定義はこちら。contextはAgentでも使っているGraphAIのcontext情報です。

```javascript
export type AgentFilterFunction<ParamsType = DefaultParamsType, ResultType = DefaultResultData, NamedInputDataType = DefaultInputData> = (
  context: AgentFunctionContext<ParamsType, NamedInputDataType>,
  agent: AgentFunction,
) => Promise<ResultData<ResultType>>;
```

前処理をする場合

```javascript
export const streamAgentFilter: AgentFilterFunction = async (context, next) => {
  const { params, debugInfo, filterParams, namedInputs } = context;

  // some streaming process....

  next(context)
};
```


後処理をする場合

```javascript
export const dataFilterAgentFilter: AgentFilterFunction = async (context, next) => {
  const { params, debugInfo, filterParams, namedInputs } = context;

  const result = await next(context);
  // some filter process

  return data;
};
```

途中で処理を止めます。cacheにヒットしたら、agent filter内で処理を完了してagentは呼びません。

```javascript
export const cacheFilter: AgentFilterFunction = async (context, next) => {
  const { params, debugInfo, filterParams, namedInputs } = context;
  if (cached) {
    return cacheData;
  }
  next(context);  
};
```

Agent Filterは特にagent/nodeを指定しない場合は、全てのAgentに適用されます
nodeIds, agentIdsを指定することで、適用されるAgentを制限することは可能です。両方同時に指定可能です。

```javascript
const agentFilters = [
  {
    name: "FooAgentFilter",
    agent: booAgentFilter,
    nodeIds: ["node1", "node12"],
  },
  {
    name: "BarAgentFilter",
    agent: barAgentFilter,
    nodeIds: ["node1", "node12"],
    agentIds: ["aliceAgent", "bobAgent"],
  },
];
```

AgentFilterを複数指定した場合は、arrayの前のものから順に実行されます。


公式のAgentFilterはソース/npmでいくつか提供されています。こちらを参照してください

https://github.com/receptron/graphai/tree/main/packages/agent_filters


demo webでもhttp client/stream のAgentFilterを使っています。

https://github.com/receptron/graphai-demo-web/