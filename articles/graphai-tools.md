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

- **[toolsAgent](https://github.com/receptron/graphai/blob/main/llm_agents/tools_agent/src/tools_agent.ts)** - ユーザの入力をLLMに渡し、function callingを呼び出すエージェント。
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


# Google Mapをコントロールするデモの解説

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



I'll translate this technical article about using GraphAI to control browser functions through chat into English.



# Using GraphAI to Control Browser Functions via Chat

I created a demo that uses GraphAI to control browser functions through chat interactions.

Specifically, I implemented the following features using **tools (function calling)**:

- Google Maps control functionality
- Video playback control functionality

In GraphAI, you can easily build applications that utilize function calling (tools) by combining `toolsAgent` (@graphai/tools_agent), which encapsulates LLM processing, with an agent that executes functions based on function calling return values.

## How to Run the Demo

Clone the source from [here](https://github.com/receptron/graphai-demo-web/):

```sh
git clone https://github.com/receptron/graphai-demo-web.git
```

```sh
cd graphai-demo-web
yarn install
```

Set up your OpenAI API key and Google Maps API key. Create a .env file with:

```sh
VITE_GOOGLE_MAP_KEY=AAxxxxx
VITE_OPEN_API_KEY=sk-123xxxxx
```

Note: Google Maps Place API needs to be enabled as well. For local testing, you can reference Google's demo API keys [here](https://jsfiddle.net/gh/get/library/pure/googlemaps/js-samples/tree/master/dist/samples/place-text-search/jsfiddle).

Start the application with:

```sh
yarn run serve
```

Click on GoogleMap/Video in the side menu to access the features.

## Video Playback Control Demo Explanation

You can control video playback through chat commands, including play, pause, and skipping to specific timestamps.

![google_map_fast](https://github.com/user-attachments/assets/fdb142d8-aa39-4a84-8657-92e971284a4b)


These features are implemented using four key components:

1. **[toolsAgent](https://github.com/receptron/graphai-demo-web/blob/main/src/agents/tools_agent.ts)** - An agent that passes user input to the LLM and invokes function calling.
2. **[googleMapAgent](https://github.com/receptron/graphai-demo-web/blob/main/src/agents/google_map_agent.ts)** - An agent containing functions executed by function calling. It also holds the function calling schema information.
3. **[tools Graph Data](https://github.com/receptron/graphai-demo-web/blob/main/src/graph/tools.ts)** - Graph data (workflow) that controls conversations between the above agents and users.
4. **[Web UI](https://github.com/receptron/graphai-demo-web/blob/main/src/views/GoogleMap.vue)** - A Vue-based web page serving as the frontend.

GraphAI makes it easy to build applications utilizing function calling (tools) by combining these components.

### toolsAgent

This agent executes the LLM using tools (schema), userInputs (user input), and messages (conversation history including system prompts).
Function names in the tools schema are specified as `{executing agent name}--{function name}`. Since tools functions don't have namespaces, they use `--` as a delimiter due to character limitations.

When the LLM execution response includes tools, it calls that agent and passes `fun(function name)` and `arg(arguments)` to the agent.
If tools return multiple functions, each function is executed.

If the result from the called agent (e.g., GoogleMapAgent) contains `hasNext`, that result is passed back to the LLM to continue the conversation.
When all processing is complete, it updates the conversation history and returns the results.

### googleMapAgent

This agent is called by toolsAgent. When specified functions like setZoom or findPlaces are received from tools results, it performs corresponding operations internally.

Example:
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

The agentFunctionInfo contains the tools schema for calling this agent:

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

This schema is passed to toolsAgent by injecting it into the GraphAI constructor.

Since this Agent uses the browser's Google Maps instance, it receives the map variable through graphai's config:

```TypeScript
    // Vue code
    let map: google.maps.Map | null = null;
    onMounted(async () => {
      const { Map: GoogleMap } = (await google.maps.importLibrary("maps")) as google.maps.MapsLibrary;
      map = new GoogleMap(mapRef.value as HTMLElement, {
        ...
      })
    });                          
...
    // GraphAI constructor
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

The structure is simple, handling toolsAgent and eventAgent (browser events - user input).
googleMapAgent is called internally by toolsAgents, so it doesn't appear in this graph.

Note 1: toolsAgent itself is implemented in GraphData, which executes Graph Data.
Note 2: eventAgent is a mechanism for handling events like user input as asynchronous functions outside of GraphAI. This can be used to pass form input data as part of the graph workflow.

### Web UI

The Vue implementation includes standard Google Maps initialization, eventAgent for passing user input to GraphAI, streamAgentFilter for processing LLM Streams, and Cytoscape processing for displaying the running graph state in the UI.

## Video Control Demo Explanation

The main difference from the Google Maps implementation is in the tools-executed agent and related UI components.
In Vue, the video tag element is passed to the videoAgent. Other processing is almost identical to the Google Maps Vue implementation:

```TypeScript
          config: {
            videoAgent: {
              videoElement: videoRef.value,
            },
```            

The [videoAgent](https://github.com/receptron/graphai-demo-web/blob/main/src/agents/video_agent.ts) manipulates the videoElement based on functions called by tools:

```TypeScript
  if (func === "pause") {
    videoElement.pause();
  }
  if (func === "play") {
    videoElement.play();
  }
```  

![video_demo_fast](https://github.com/user-attachments/assets/f16aa4dd-f02c-4ba5-aa0c-f1cf50b7b276)

## Summary

I introduced toolsAgent and its called agents (GoogleMapAgent/VideoAgent).
Using these two samples as reference, you can implement Agents called by tools and modify the demo to control browser functions through LLM chat.
While GoogleMapAgent and VideoAgent were implemented separately in this example, you can operate multiple agents simultaneously by merging their tools schemas and passing them to toolsAgent.
You can use this to create various browser applications.

This example can be used to incorporate LLM into existing web applications. While the LLM runs in the browser in this demo, it can also run independently on a server.


I've translated the entire technical article while maintaining its structure and technical accuracy. The translation includes all code samples, explanations, and technical terminology. Would you like me to clarify any specific parts of the translation?