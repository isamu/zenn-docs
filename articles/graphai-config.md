---
title: "GraphAIの設定値をGraphAIからAgentに渡す"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

### Agentの設定について

Agentには動作のために設定値が必要な場合があります。代表的な例として、OpenAIのLLMを動作させるためのAPIキーが必要となります。

#### 環境変数を使った設定
- **サーバやバッチ処理の場合**  
  環境変数を使用して、Agent内部で利用しているLLMライブラリが必要とするAPIキーなどの設定値を渡すのが推奨されます。

- **ブラウザでの利用の場合**  
  GraphAIはブラウザ上でもAgentを動作させることが可能ですが、この場合、process.envでの環境変数を使用できません。そのため、以下の方法が用意されています：
  1. **`params`で直接渡す方法**  (非推奨)
     設定値をAgentの`params`に直接指定することが可能です。
  2. **`config`を通じて渡す方法（推奨）**  
     設定値は`config`を使用して渡すのが適切です。

:::message
paramsを使った設定方法は、GraphData内に秘匿情報含まれるため現在は推奨されていません。また、同じAgentを複数回使う場合には、全てのparamsにその設定を含める必要があります。GraphDataをサーバとブラウザで使う場合にも問題があります。
:::

#### `config`を使った設定の仕組み
- `config`はGraphAIのコンストラクタを通じてAgentに設定値を渡す仕組みがあります。
- 仕様変更の影響でGraphAIのバージョン **0.6.14** 以降の利用を推奨します。

#### `config`の主な特徴
- **Agentごとの設定**  
  各Agentに個別の設定値を渡すことが可能です。
  `config: {[agentId]: { someKey: someValue}}`とすると、そのagentIdのAgentだけに値を渡すことが可能です。
- **Global設定**  
  `global`を指定することで、すべてのAgentに共通の設定値を渡すことができます。
  `config: {global: { someKey: someValue}}`
- `Global`と`[agentId]`は同時に利用が可能です
- `[agentId]`で個別に指定できるので、apiKeyのような名前をそれぞれのagentで使うことができ、名前の衝突がありません。
- **Dynamic AgentやNested Graphに対応**  
  動的に決定されるAgentやネストされたグラフにも対応しています。



## 例

globalでサービス共通のuserIdを全てのAgentに渡し、testAgentだけにapiKeyを渡しています。
userIdは他のAgentでも同じように取得できます。

```TypeScript
import GraphAI from 'graphai';

const graphai = new GraphAI(graph, agent, {
  config: {
    global: {
      userId: 123,
    },
    testAgent: {
      apiKey: "someCredentialValue"
    },
    otherAgent: {
      apiKey: "someCredentialValue"
    },
  },
});

// agent
const testAgent: AgentFunction = async (namedInputs, config) => {
  const { userId, apiKey } = config;
  console.log(`User ID: ${userId}`); // 123
  ///
};
```

