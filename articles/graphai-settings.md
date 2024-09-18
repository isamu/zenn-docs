---
title: "GraphAIの設定値について"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---


# GraphAIでagentに渡す設定について

GraphAIで使う設定値（サーバのURL、パスワード）は、いくつか渡す方法があります。

### environment variables

一般的な環境変数を使います。
サーバやnodeで実行時に

```sh
FOO=abc
```
のように渡して、agentでその環境変数に`process.env`でアクセスします。

利点は、ファイルに値を書かないので安全で、一般的なツールと同じ使い方ができます。

デメリットは、Agent/nodeごとに変更できない、ブラウザでそのagentを使う場合に、少し工夫が必要です。

### GraphAIのconfigにデータを渡す

GraphAIのコンストラクタのoptionにconfigで、各agentに設定値を渡すことができます。

```typescript
  const config = { uid: "abc" };
  const graphai = new GraphAI(graphData, agentDictionary, { agentFilters, config });
```
このconfigは、agentに渡されます

```typescript
export const fooAgent: AgentFunction = async ({ params, namedInputs, config }) => {
 const { uid } = config;
 ...
}
```

セッションごとにデータを渡せるので、uidなどをagentに渡す場合などに使います。

### agentのparamsに渡す

```yaml
version: 0.5
nodes:
  foo:
    agent: fooAgent
    params:
      serverUrl: fooBar
  

```

GraphDataのparamsに渡します。
サーバを動的に変えるなどする場合にはこちらがよいです。

サーバクライアントで使う場合には、通信経路やブラウザで書き換えることが可能なので、書き換えされるとこまるようなデータには不向きです。



