---
title: "GraphAIをNode.jsで試す"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

## GraphAIのNode.jsで試す

GraphAIを動作させるのに必要な最小限のnpmは[graphai](https://www.npmjs.com/package/graphai)と[vanilla agents](https://www.npmjs.com/package/@graphai/vanilla) です。
最も簡単に使えるAgentである[@graphai/vanilla](https://www.npmjs.com/package/@graphai/vanilla) (他にnpmの依存がないagentをvanillaと呼んでいます)を使って簡単な`GraphData`を作り、動かしてみます。

最初にnpmの初期化をして、必要なnpmを入れます。typescriptを実行するためにts-nodeも入れます。

```sh
npm init
yarn add graphai @graphai/vanilla ts-node
```

以下が最小限のスクリプトです。graphai.tsで保存します。

この`GraphData`には２つのnodeが含まれます。
`Static node`の`node1`と`Computed node`の`node2`です。

`Static node`の`node1`で、固定のメッセージを定義します。
そのデータを受け取った`Computed node`の`node2`でcopyAgentを実行します。
`copyAgent`は`inputs`で受け取ったデータをそのまま返却値とて返すagentです。
`node2`は`isResult: true`がセットされているので、GraphAIの実行結果として`node2`の結果を返します。

TypeScriptのresult変数にその結果がセットされます。

```typescript
import { GraphAI } from "graphai";
import * as agents from "@graphai/vanilla";

const graph_data = {
  version: 0.5,
  nodes: {
    node1: {
      value: "hello, GraphAI",
    },
    node2: {
      agent: "copyAgent",
      inputs: {text: ":node1"},
      isResult: true,
    },
  },
};

export const main = async () => {
  const graph = new GraphAI(graph_data, agents);
  const result = await graph.run();
  console.log(JSON.stringify(result));

};

if (process.argv[1] === __filename) {
  main();
}
```

実行します
```shell-session
$ npx ts-node graphai.ts
{"node2": {text: "hello, GraphAI"}}
```

node1で指定して文字列がnode2に渡され、結果として表示されました。

