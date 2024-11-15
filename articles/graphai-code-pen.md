---
title: "GraphAIをcodepenで試す"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

GraphAI, jsDelivr経由でhtmlから直接利用できるようになりました。

利用できるパッケージは

- GraphAI本体
  - https://www.jsdelivr.com/package/npm/graphai
- Vanilla Agents
  - GraphAIの基礎となるAgent
  - https://www.jsdelivr.com/package/npm/@graphai/vanilla
- Agent Filters
  - Streamingのためのツール
  - https://www.jsdelivr.com/package/npm/@graphai/agent_filters
- OpenAI Fetch Agent
  - OpenAIをfetchを使ってAPI経由で利用するAgent
  - https://www.jsdelivr.com/package/npm/@graphai/openai_fetch_agent

Codepenで試すこともできます！！

- 素のHTMLで試す
  - https://codepen.io/isamua/pen/yLmwYzX

- Vueで試す
  - https://codepen.io/isamua/pen/bGXZEdR

- VueでStreamingを試す
  - https://codepen.io/isamua/pen/YzmgRYg


GitHubでブラウザ上でopenaiを使ったサンプルのhtmlファイルも配布しています。
```
const openApiKey = "sk-xxxxx";
```
にopenaiのapikeyをセットして、ブラウザで開いてください。

https://github.com/receptron/graphai/blob/main/packages/samples/htmlSample/openai-sample.html



# jsdelivrの必要な組み合わせ

### GraphAIを試す

本体＋Agent

```html
<script src="https://cdn.jsdelivr.net/npm/graphai@0.5.18/lib/bundle.umd.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@graphai/vanilla@0.1.8/lib/bundle.umd.min.js"></script>
```

```typescript
const { GraphAI } = graphai;
const graph = new GraphAI(graph_data, vanilla_agents);
const res = await graph.run();
```

## Vue.js上でcytoscapeを使ってGraphをGUI表示

vue + cytoscape + GraphAI

```html
<script src="https://cdn.jsdelivr.net/npm/vue@3.5.12/dist/vue.global.min.js"></script>

<script src="https://unpkg.com/klayjs@0.4.1/klay.js"></script>
<script src="https://cdn.jsdelivr.net/npm/cytoscape@3.30.3/dist/cytoscape.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/cytoscape-klay@3.1.4/cytoscape-klay.min.js"></script>

<script src="https://cdn.jsdelivr.net/npm/@receptron/graphai_vue_cytoscape@0.0.12/lib/bundle.umd.js"></script>
```

```typescript
const graph_computed = computed(() => {
  return graph_data;
});
const { useCytoscape } = vue_cytoscape;
const { updateCytoscape, cytoscapeRef } = useCytoscape(graph_computed);

const graph = new GraphAI(graph_data, vanilla_agents);
graph.onLogCallback = async ({ nodeId, state }) => {
  updateCytoscape(nodeId, state);
};

```


## Streamingをする

```html
<script src="https://cdn.jsdelivr.net/npm/@graphai/agent_filters@0.0.8/lib/bundle.umd.min.js"></script>
```

```typescript
const { streamAgentFilterGenerator } = graphai_agent_filters;

const message = ref("");
const outSideFunciton = (context, data) => {
  message.value = message.value + data;
};
const agentFilters = [{
  name: "streamAgentFilter",
  agent: streamAgentFilterGenerator(outSideFunciton)
}];

const graph = new GraphAI(graph_data, vanilla_agents, { agentFilters });
```