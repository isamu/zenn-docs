---
title: "MulmoCastにimage agentを追加する"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

# MulmoCastにimage agentを追加する

MulmoCastでは、現在、openaiとgoogleのtext2image のapiをサポートしています。
MulmoCastはGraphAIで動いているのでGraphAIのimage agent を追加することで、機能を拡張することが可能です（現在のところ、リポジトリを fork してコードを追加する必要があります）。

# GraphAIのimage agentを作成する

`src/agents/image_foo_agent.ts`を参考に独自の`image_agent`を作成する

`agent`は`inputs`で`prompt`を`string`型として受け取り、`{ buffer }` の形式で PNG 画像ファイルを返すように実装します。

画像の保存処理は agent 側で行う必要はありません。画像のキャッシュや保存は mulmo 側で処理されます。

```
export const imageFooAgent: AgentFunction = async ({ namedInputs }) => {
  const { prompt } = namedInputs;

  ...
  ...
    
  return { buffer: Buffer.from(image_base64, "base64") };
  
};
```

作成した関数は、`agentInfoWrapper`か、もしくは通常の方法でagent化します。


# provider2agent に情報を追加する

`src/utils/provider2agent.ts`に、作成した`image agent`の情報を追加します。

追加方法は既存の他の`image agent`の定義を参考にしてください。

# 環境変数の設定
`agent`の動作に必要な環境変数を追加します。

`src/utils/utils.ts`内の`settings2GraphAIConfig`関数にある`ConfigDataDictionary`に設定を追記してください。

環境変数または設定経由で`graphai config`から`agent`の`config`に値を渡すことができます。

API キーなどの機密情報はこの方法で渡してください。こちらも他の`image agent`を参考にすると実装しやすいです。

# mulmoScriptにproviderとして指定する

scriptsを参考に、imageParamsを設定してください。
