---
title: "SlashGPTでChatGPT搭載のカーナビのデモを30分で作る"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ChatGPT, LLM, Tech, OpenAI, SlashGPT]
published: true
publication_name: "singularity"
---

CES2024でChatGPTを搭載したカーナビゲーションが紹介されていました。

https://www.youtube.com/watch?v=ToZAVlLqzmo&t=849s

カーナビに普通に話しかけることにより、音声認識を使ってカーナビ、エアコン、オーディオなどの操作ができるようです。早速、テキストベースですが[SlashGPT](https://github.com/snakajima/SlashGPT)を使ってこのAIのChatGPT部分を真似してみたいと思います。


まず、カーナビ、エアコン、オーディオの各エージェントを作ります。それぞれのエージェントはFunction callingを使って、ナビやエアコンのAPIを呼び出します。[manifests/main/home.json](https://github.com/snakajima/SlashGPT/blob/main/manifests/main/home.json) や[resources/functions/home.json](https://github.com/snakajima/SlashGPT/blob/main/resources/functions/home.json)を参考にして真似します。


### カーナビ用のエージェントのmanifest.

ルート検索とPOI検索のAPIに対応。サンプルなので、それぞれのAPIが呼ばれたら、呼ばれたことがわかるダミーメッセージを表示します。

```yaml
title: navigation
description: navigation
prompt: |
  Don't make any assumptions about what property values to plug into functions.
  Ask for clarification if a user request is ambiguous.
sample: |
  navigation
modelx: gpt-4-0613
temperature: 0
functions: ./resources/functions/navigation.json
actions: 
  search_route:
    type: message_template
    message: Success. search_route called. route is ....
  search_poi:
    type: message_template
    message: Success. search poi called. poi is is 123, lat = 35, lng = 139
skip_function_result: true
```

### カーナビ用のFunction calling

```json
[
  {
    "name": "search_route",
    "description": "search route",
    "parameters": {
      "type": "object",
      "properties": {
        "distination": {
          "type": "string"
        }
      },
      "required": ["distination"]
    }
  },
  {
    "name": "search_poi",
    "description": "search poi",
    "parameters": {
      "type": "object",
      "properties": {
        "poi_name": {
          "type": "string",
          "description": "poi name"
        }
      },
      "required": ["poi_name"]
    }
  }
]
```


### エアコン用のエージェントのmanifest.

```yaml
title: air conditioner
description: air con
prompt: |
  Don't make any assumptions about what property values to plug into functions.
  Ask for clarification if a user request is ambiguous.
sample: |
  play music
modelx: gpt-4-0613
temperature: 0
functions: ./resources/functions/air_conditioner.json
actions: 
  control_aircon:
    type: message_template
    message: Success. control_aircon called. action is {action}
  set_temperature:
    type: message_template
    message: Success. set_temperature called. temperature set to {temperature}
skip_function_result: true
```

### エアコン用のFunction calling

on/offのスイッチと温度をセットするAPIを用意。

```json
[
  {
    "name": "control_aircon",
    "description": "Control air conditioner",
    "parameters": {
      "type": "object",
      "properties": {
        "action": {
          "type": "string",
          "enum": ["on", "off"]
        }
      },
      "required": ["action"]
    }
  },
  {
    "name": "set_temperature",
    "description": "Set the temperature",
    "parameters": {
      "type": "object",
      "properties": {
        "temperature": {
          "type": "number",
          "description": "Car temperature in celsius."
        }
      },
      "required": ["temperature"]
    }
  }
]
```


### オーディオ用のエージェントのmanifest.


```yaml
title: audio
description: audio
prompt: |
  Don't make any assumptions about what property values to plug into functions.
  Ask for clarification if a user request is ambiguous.
sample: |
  play music
modelx: gpt-4-0613
temperature: 0
functions: ./resources/functions/audio.json
actions: 
  control_audio:
    type: message_template
    message: Success. control_audio called. action is {action}
  change_music:
    type: message_template
    message: Success. change_music called. action is {action}
skip_function_result: true
```

### オーディオ用のFunction calling

音楽の再生、停止、一時停止と、次の曲、前の曲へ移動するAPIをサポートしています。
```json
[
  {
    "name": "control_audio",
    "description": "Play action",
    "parameters": {
      "type": "object",
      "properties": {
        "action": {
          "type": "string",
          "enum": ["play", "stop", "pause"]
        }
      },
      "required": []
    }
  },
  {
    "name": "change_music",
    "description": "Change music",
    "parameters": {
      "type": "object",
      "properties": {
        "action": {
          "type": "string",
          "description": "Change music",
          "enum": ["next", "previous"]
        }
      },
      "required": ["action"]
    }
  }
]
```

### Dispacher

ユーザからの問い合わせに対して、用意した3つのエージェントのうちふさわしいエージェントを選択し、そのエージェントにプロンプトを渡すように動作します。


```json
{
  "title": "car Dispatcher",
  "about": "isamu",
  "temperature": 0,
  "functions": "./resources/functions/dispatcher.json",
  "actions": {
    "categorize": {
      "type": "emit",
      "emit_method": "switch_session",
      "emit_data": {
        "message": "{question}",
        "agent": "{category}"
      }
    }
  },
  "intro": [
    "I am a dispatcher agent. I will find the right agent for your question, and let it answer." 
  ],
  "agents": ["navigation", "air_conditioner", "music"],
  "prompt": [
    "You are responsible in categorize user's question into one of categories below.",
    "Call categorize function with one of categories below.",
    "{agents}"
  ],
  "sample": "Who is the CEO of SpaceX?",
  "sample2": "What is the weather in Seattle today?",
  "sample3": "Please write a summary of the web page at https://www.bbc.com/news/live/world-europe-66049895"
}
```

GitHubでの完成品はこちら。PRで差分を表示しています。
https://github.com/isamu/SlashGPT/pull/29/files

# 対話型カーナビの動作デモ

起動してカーナビモードに
```bash
Activating: Main Dispatcher
Agent(dispatcher): I am a dispatcher agent. I will find the right agent for your question, and let it answer.
You(dispatcher): /switch car
Activating: car Dispatcher
```

経路を聞くとナビモードになり経路検索や、POI検索をする
```
Agent(dispatcher): I am a dispatcher agent. I will find the right agent for your question, and let it answer.
You(dispatcher): 東京までの経路を教えてください。
Activating: navigation
function(search_route): Success. search_route called. route is ....
You(navigation): 近くのコンビニを検索してください
function(search_poi): Success. search poi called. poi is is 123, lat = 35, lng = 139
```

再びdispatcherに戻し、エアコン操作

```
You(navigation): /dispatcher
Activating: car Dispatcher
Agent(dispatcher): I am a dispatcher agent. I will find the right agent for your question, and let it answer.
You(dispatcher): エアコンをつけてください
Activating: air conditioner
function(control_aircon): Success. control_aircon called. action is on
You(air_conditioner): 温度を28度に設定してください
```

自然な会話を試す

```
function(set_temperature): Success. set_temperature called. temperature set to 28
You(air_conditioner): 寒いです
Agent(air_conditioner): 申し訳ありません。温度をもう少し上げますか？
You(air_conditioner): はい
function(set_temperature): Success. set_temperature called. temperature set to 30
```
寒いと温度を上げてくれる。


音楽再生
```
You(air_conditioner): /dispatcher
Activating: car Dispatcher
Agent(dispatcher): I am a dispatcher agent. I will find the right agent for your question, and let it answer.
You(dispatcher): 音楽を聞きたいです
Activating: audio
function(control_audio): Success. control_audio called. action is play
You(music): 次の曲へ。
function(change_music): Success. change_music called. action is next
You(music): この曲は嫌いです
function(change_music): Success. change_music called. action is next
You(music): 前の曲をききたいです
function(change_music): Success. change_music called. action is previous
You(music):
```
嫌いというと、曲をスキップしてくれる。

こんな感じで自然な対話でカーナビを操作することが可能になりました。manifestは英語で記述していますが、日本語でも全く問題なく動作しました。

今回、呼び出すAPIはモックで、メッセージを表示するだけですが、実際に組み込みで作ってAPIを呼び出したり、Rest APIなどで操作できるように実装すれば、ハードウェアを制御することも可能です。これらのエージェントをdispatcherに登録し、ユーザはエージェントを意識することなく「エアコンをつけて」「次の曲」と命令するだけで適切なエージェントにプロンプトが渡されて命令が実行されます。

おおよそ３０分程度で、このエージェントのマニュフェストは作ることが出来ました。

SlashGPTを使うと、Function callingを使ったGPTのデモを圧倒的なスピードで作成することが可能となります。これらに音声認識と音声読み上げ機能を追加すれば、デモにあったようなカーナビにをつくることができます。