---
title: "GraphAI Streaming"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

# GraphAI Streamingについて

GraphAIを使うことで、サーバやクライアント単体での処理、さらにはサーバとクライアントの連携によるLLMのプロンプトなどのストリーミング処理を透過的に扱うことができます。 

ここで「透過的」というのは、GraphAIが標準で用意しているExpressのmiddlewareやHTTP/StreamingのAgent Filterを利用することで、サーバとクライアント、またはストリーミング処理を意識することなく利用できることを指します。

一度実装した処理は、クライアントからサーバに変更する場合や、サーバが変更される場合でも、ほとんど実装を変えることなく、設定を変更するだけで移行が可能となります。この柔軟性により、開発者はストリーミング処理の実装に煩わされることなく、ビジネスロジックに集中できるようになります。

# Streaming処理の概要

1. **データの逐次受け渡し**  
   Agentが実行中、データが生成されるたびに、callback関数を通じてAgentフィルターに渡されます。データは一括で処理されるのではなく、逐次callback関数が呼ばれ、その都度処理が行われます。

2. **callback関数内での処理**  
   callback関数では、Contextから`nodeId`、`agentId`、`data`といった情報を受け取り、それぞれのデータに対して個別の処理が行われます。以下は、利用する環境ごとの処理の例です。

   - **ブラウザの場合**  
     callback関数で受け取ったデータをブラウザに逐次表示します。これにより、リアルタイムでのデータ更新が可能になります。

   - **Express（Webサーバ）の場合**  
     callback関数で受け取ったデータを加工し、HTTPレスポンスとして返します。この逐次処理により、データが生成されるたびに即座に応答が返せるため、APIとしての利用シナリオにも適しています。

このような仕組みによって、Agent実行中にリアルタイムでのデータ処理と表示・応答が可能となります。



## Agent内で、agentFilterにデータを渡す

https://github.com/receptron/graphai/blob/e720821dff1a4f59423dbb02db64aaec3a2a61eb/llm_agents/openai_agent/src/openai_agent.ts#L120-L125


## agentFilter

ここでは、Streaming処理用の`agentFilter`を生成する関数 `streamAgentFilterGenerator` の使い方について説明します。この関数を使うことで、ユーザはcallback関数を指定するだけで、データをリアルタイムで処理できる`agentFilter`を取得できます。

https://github.com/receptron/graphai/blob/e720821dff1a4f59423dbb02db64aaec3a2a61eb/packages/agent_filters/src/filters/stream.ts#L3-L13

### 使用方法

1. **callback関数を定義する**  
   `context`と`data`を引数に取るcallback関数を作成します。この関数は、Agentがデータを受け取るたびに呼び出され、リアルタイムでデータを処理できます。

```typescript
   const myCallback = (context, data) => {
     console.log("Data received:", data);
     // 必要な処理をここに記述
   };
```   

2. **streamAgentFilterを取得する**  
   `streamAgentFilterGenerator`にcallback関数を渡すことで、データを逐次処理する`agentFilter`が生成されます。この`agentFilter`は、Agentが実行中にデータをリアルタイムで処理します。

```
const myAgentFilter = streamAgentFilterGenerator(myCallback);
```

以上で、`streamAgentFilterGenerator`を使ったagentFilter処理のセットアップは完了です。
このagentFilterをGraphAIのコンストラクタのagentFiltersに渡します。
callback関数を通じて、データを逐次処理する仕組みを簡単に構築できます。



## Streaming処理について

### 1. GraphAIを直接使う場合（ブラウザ）

- AgentFilter経由のcallback関数を使用して、ストリーミングデータを受け取ります。
- 全体の結果は`graphai.run()`から受け取ります。
- ストリーミングと結果の処理は実装側で制御できるため、delimiterやデータ形式について特に考慮する必要はありません。

### 2. Expressの場合

- HTTPの仕組み上、文字列が順次送られてきます。
- 標準では、token（文字列）がストリーミングで逐次流れ、終了時には`__END__`というdelimiterを挟んで、結果（content）がJSON形式の文字列で返ってきます。
- Tokenの処理やdelimiter、contentの処理は、Expressにcallback関数を渡すことで変更可能です。

## expressの制御

Expressは、ストリーミングサーバ、ノンストリーミングサーバ、そして両方をサポートするサーバのそれぞれに対応したmiddlewareを提供しています。両方のサポートをするmiddlewareを使用することで、Agentがストリーミングに対応していても、HTTPヘッダーを通じてストリーミングの制御が可能です。

具体的には、HTTPヘッダーの以下の有無によって判定が行われます：

- `Content-Type` が `text/event-stream` であるかどうかです。



## 以下参考ソース

- express のcallback関数例

https://github.com/receptron/graphai-utils/blob/b302835d978ce1017c6e105898431eda28adcbd4/packages/express/src/agents.ts#L122-L135

- expressの実装

https://github.com/receptron/graphai-utils/tree/main/packages/express/src

streamAgentFilterGenerator
https://github.com/receptron/graphai/blob/main/packages/agent_filters/src/filters/stream.ts



- httpAgentFilter の実装(GraphAI Agentのagent Filter形式のclinet)

https://github.com/receptron/graphai/blob/main/packages/agent_filters/src/filters/http_client.ts