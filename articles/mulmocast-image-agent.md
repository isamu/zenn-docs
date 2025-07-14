---
title: "MulmoCastにimage agentを追加する"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, mulmocast, LLM, Tech, GraphAI]
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
export const newImageAgent: AgentFunction = async ({ namedInputs }) => {
  const { prompt } = namedInputs;

  ...
  ...
    
  return { buffer: Buffer.from(image_base64, "base64") };
  
};
```

作成した関数は、`agentInfoWrapper`か、もしくは通常の方法でagent化します。

## imageAgents に登録する

以下のいずれかの方法で、imageAgents に agent を追加します：

- `src/actions/images.ts`にある`imageAgents`に直接追加する
  - 追加するとGraphAIのインスタンスに渡すAgentとして登録されます。
- または、generateImages 関数や images 関数の options 引数として imageAgents を渡す
  - ライブラリとしてMulmoCastを使う場合は、この方法で外から追加できます。

このときのagent名は、この後の設定でのagentNameやdictonaryのkeyとなります。

# provider2agent に情報を追加する

`src/utils/provider2agent.ts`に、作成した`image agent`の情報を追加します。

```
export const provider2ImageAgent = {
  openai: {
    agentName: "imageOpenaiAgent",
    defaultModel: "gpt-image-1",
    models: ["dall-e-3", "gpt-image-1"],
  },
  yourImageAgentProvider: {
    agentName: "newImageAgent",
    defaultModel: "image1",
    models: ["image1", "image2"],
  },
};  
```  
keyはAgent名と一致させる必要があります。graphaiのagentsにわたすときのkeyです。
defaultModelやmodelsはschemaのvalidationで使われます。

追加方法は既存の他の`image agent`の定義を参考にしてください。


# 環境変数の設定
`agent`の動作に必要な環境変数を追加します。

`src/utils/utils.ts`内の`settings2GraphAIConfig`関数にある`ConfigDataDictionary`に設定を追記してください。

```
  const config: ConfigDataDictionary<DefaultConfigData> = {
    newImageAgent: {
      apiKey: getKey("IMAGE", "NEW_IMAGE_API_KEY"),
    },
```
keyはprovider2ImageAgentのagentNameと一致させる必要があります。
環境変数または設定経由で`graphai config`から`agent`の`config`に値を渡すことができます。

agentのconfigにapiKeyとして値が渡されます。
IMAGE_NEW_IMAGE_API_KEY もしくは、NEW_IMAGE_API_KEYの環境変数で設定してください。


API キーなどの機密情報はこの方法で渡してください。こちらも他の`image agent`を参考にすると実装しやすいです。


# mulmoScriptにproviderとして指定する

scriptsを参考に、imageParamsを設定してください。


以上で、image agentを追加して利用することができます。


imagesの処理でmovieを作成できますが、これもほぼ同じですので興味があればソースを参考にすれば追加可能です。
