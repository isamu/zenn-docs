---
title: "GraphAI abortの機能を追加"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

# GraphAI 0.6.20 リリースノート

GraphAI 0.6.20がリリースされました。大きな変更点として、abort()が追加されました。
グラフフローの実行中に、このAPIを呼ぶと、処理を途中で止めることが出来ます。

```TypeScript
const graphAI = new GraphAI(graphData, agents, graphOptions);

const runGraphAI = async () => {
  try {
    const result = graphAI.run();
    ...
  } catch (e) {
    ...
  }
};

const stopGraphAI = () => {
  graphAI.abort();
  ...
}
```

- abort()を呼ぶと、run()はすぐにrejectを返す。
- 実行中のagentは、最後まで実行される。
- その後のagentは実行されない。

以下、関連パッケージを最新にすると有効になります。
- nestedAgent/mapAgentなどのsubGraphを実行するagentも同じようにabortされる (@graphai/vanilla@0.2.12)
- streamingの処理もabortしたときに、データを返さなくなる。 (@graphai/agent_filters@0.2.0)


# 開発者向け情報

- subGraphを実行するagentを作るときは、そのagent内で親agentにsubGraphを登録/削除する必要があります

https://github.com/receptron/graphai/blob/main/agents/vanilla_agents/src/graph_agents/nested_agent.ts

```TypeScript
  debugInfo.subGraphs.set(graphAI.graphId, graphAI);
  const results = await graphAI.run(false);
  debugInfo.subGraphs.delete(graphAI.graphId);

```

- streamingのようなagent実行中に、ユーザやgraphaiの外にデータを渡す処理をするagent/agent filterを作るときは、agentのstatusがExecutingかどうか確認して処理をする必要があります。

https://github.com/receptron/graphai/blob/main/packages/agent_filters/src/filters/stream.ts

```TypeScript
  context.filterParams.streamTokenCallback = (data: T) => {
    if (context.debugInfo.state === NodeState.Executing) {
      callback(context, data);
    }
  };
```