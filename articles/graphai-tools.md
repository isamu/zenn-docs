---
title: "GraphAI - チャットでブラウザを操作するデモ"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

GraphAIを活用し、ブラウザの機能をチャットで操作するデモを作成しました。

具体的には、**tools(function calling)** を用いて会話で以下の機能を実現しています。

- Google Mapsを操作する機能
- 動画の再生をコントロールする機能

GraphAIでは、LLMの処理を内包する`toolsAgent`（@graphai/tools_agent）と、function callingの戻り値で関数を実行するエージェントを組み合わせることで、簡単にfunction calling（tools）を活用したアプリケーションを構築することが可能です。

# デモの動かし方

ソースは[こちら](https://github.com/receptron/graphai-demo-web/) cloneします

```sh
git clone https://github.com/receptron/graphai-demo-web.git
```

```sh
cd graphai-demo-web
yarn install
```
openaiのapikeyとgoogle mapのapi keyをセットします。.envファイルを作成し

```sh
VITE_GOOGLE_MAP_KEY=AAxxxxx
VITE_OPEN_API_KEY=sk-123xxxxx
```
と設定します。google mapはplace apiもenableにする必要があります。localで動かす場合は、google のデモのapi keyなどが参考になります。[こちら](https://jsfiddle.net/gh/get/library/pure/googlemaps/js-samples/tree/master/dist/samples/place-text-search/jsfiddle)

```
yarn run serve
```

で起動します。

![](https://storage.googleapis.com/zenn-user-upload/e7b12a9f1fee-20250127.png =300x)

サイドメニューでGoogleMap/Videoを押します。

# 動画の再生をコントロールするデモの解説 

![google_map_fast](https://github.com/user-attachments/assets/fdb142d8-aa39-4a84-8657-92e971284a4b)


チャットで再生と停止、任意の時間へのスキップを指示することが可能です。

これらの機能は、以下の4つの要素で実現されています。

- **[toolsAgent](https://github.com/receptron/graphai-demo-web/blob/main/src/agents/tools_agent.ts)** - ユーザの入力をLLMに渡し、function callingを呼び出すエージェント。
- **[googleMapAgent](https://github.com/receptron/graphai-demo-web/blob/main/src/agents/google_map_agent.ts)** - function callingから実行される関数を持つエージェント。さらに、function callingのスキーマ情報も保持しています。
- **[tools Graph Data](https://github.com/receptron/graphai-demo-web/blob/main/src/graph/tools.ts)** - 上記エージェントとユーザとの会話を制御するためのグラフデータ（ワークフロー）。
- **[Web UI](https://github.com/receptron/graphai-demo-web/blob/main/src/views/GoogleMap.vue)** - フロントエンドとしてVueを使用したWebページ

GraphAIでは、これらを組み合わせることで、簡単にfunction calling（tools）を活用したアプリケーションを構築することが可能です。

### toolsAgent

tools(スキーマ), userInputs(ユーザからの入力), messages(システムプロンプトを含む会話履歴)を使ってLLMを実行します。  
toolsスキーマに含まれる関数名は、`{実行するagent名}--{関数名}`を指定します。toolsの関数はnamespaceがないため、使える文字に制限があり、`--`で区切っています。  
LLMを実行し、レスポンスにtoolsが含まれる場合、そのagentを呼び出し、`fun(関数名)`, `arg(引数)`をagentに渡します。  
toolsが複数の関数を返してきた場合は、それぞれの関数を実行します。

呼び出したagent（例: GoogleMapAgent）の結果に`hasNext`が含まれる場合には、その結果をさらにLLMに渡して会話を進めます。  
すべての処理が終わると、会話履歴を更新して結果として返します。

### googleMapAgent

toolsAgentから呼び出されるagent. toolsの結果から、setZoom, findPlacesなど関数を指定されると、agent内部でそれに対応する処理を行います。

例:
```TypeScript
  if (func === "setCenter") {
    map.setCenter(arg as google.maps.LatLng);
  }
  if (func === "getCenter") {
    const location = map.getCenter();
    return {
      result: JSON.stringify(location),
      hasNext: true,
    };
  }
```

agentFunctionInfoに、このagentを呼び出すためのtoolsのスキーマを持っています。

```TypeScript
const googleMapAgentInfo = {
  ....
  tools: [
    {
      type: "function",
      function: {
        name: "googleMapAgent--setCenter",
        description: "set center location",
        parameters: {
          type: "object",
   ...
```
GraphAIのコンストラクタにinjectすることで、このスキーマをtoolsAgentに渡します。

また、このAgentはブラウザのgoogle mapのインスタンスを利用するので、graphaiのconfig経由でmap変数を受け取っています。


```TypeScript
    // Vueのコード
    let map: google.maps.Map | null = null;
    onMounted(async () => {
      const { Map: GoogleMap } = (await google.maps.importLibrary("maps")) as google.maps.MapsLibrary;
      map = new GoogleMap(mapRef.value as HTMLElement, {
        ...
      })
    });                          
...
    // GraphAIコンストラクタ
      const graphai = new GraphAI(
      ....
          config: {
            googleMapAgent: {
              map,
            },
          ...
```

```TypeScript
  // Agent
  const { map } = config;
```

### tools Graph Data

https://github.com/receptron/graphai-demo-web/blob/main/src/graph/tools.ts

toolsAgentとeventAgent(ブラウザのイベント- ユーザからの入力)を扱うagentのシンプルな構成になっています。
googleMapAgentはtoolsAgents内部から呼び出されるので、このグラフにはでてきません。

- 余談その1 toolsAgentは、Graph Dataを実行するagentで、toolsAgent自体はGraphDataで実装されています。興味があればソースを見てください)
- 余談その2 eventAgentはGraphAIの外からユーザの入力などのeventを非同期関数として扱う仕組みです。これを応用するとformの入力などをグラフのワークフローの途中にデータとして渡すことが可能となります

### Web UI

一般的なGoogleMapの初期化、ユーザの入力をGraphAIに渡すeventAgent、LLMのStreamを処理するstreamAgentFilter、実行中のグラフの状態をUIに表示するCytoscapeの処理などがVue上に実装されています。


# 動画の再生をコントロールするデモの解説

GoogleMapとの差分はtoolsで実行されるagentと、それに関連するUI部分です。
Vueでは、videoタグのelementをvideoAgentに渡しています。その他はGoogle mapのvueとほとんど同じ処理をしています。
```
          config: {
            videoAgent: {
              videoElement: videoRef.value,
            },
```            

[videoAgent](https://github.com/receptron/graphai-demo-web/blob/main/src/agents/video_agent.ts)は、toolsで呼ばれた関数を元にvideoElementを操作しています。

```TypeScript
  if (func === "pause") {
    videoElement.pause();
  }
  if (func === "play") {
    videoElement.play();
  }
```  

![video_demo_fast](https://github.com/user-attachments/assets/f16aa4dd-f02c-4ba5-aa0c-f1cf50b7b276)



## まとめ

toolsAgentとそれから呼ばれるGoogleMapAgent/VideoAgentを紹介しました。
この２つのサンプルを参考に、toolsで呼ばれるAgentを実装しデモを少し改良することで、ブラウザ上の機能をllmとのチャットで操作することが可能となります。
今回はGoogleMapAgent/VideoAgentを別々に実装しましたが、それぞれのtoolsのスキーマをマージし、toolsAgentにわたすと複数のagentを同時に操作することも可能です。
これを応用して様々なブラウザアプリを作ってください。

この例を参考にすると、作っている既存のwebアプリにllmを組み込むことも可能です。また、今回はllmをブラウザ動かしていますが、llm単体をサーバで動かすことも可能です。
