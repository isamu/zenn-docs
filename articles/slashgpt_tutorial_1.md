---
title: "SlashGPTの使い方"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [gpt, ChatGPT, LLM, Tech]
published: true
publication_name: "singularity"
---

# SlashGPTとは

[SlashGPT](https://github.com/snakajima/SlashGPT/)は[中島聡](https://twitter.com/snakajima)が開発したChatGPTなどのLLMエージェントを手軽に開発するためのツールです。SlashGPTを使えば、jsonファイルを記述するだけでChatGPTを使ったLLMエージェントやチャットアプリを手軽に、簡単につくることができます。

出来上がったjsonファイル（これからはマニフェストと呼びます）は、簡単に共有することができます。マニフェストファイルは他の人と共有する他、SlashGPTをライブラリとして利用する場合にエージェントの設定としてそのまま使うことが可能です。

# インストール

SlashGPTはコマンドライン（ターミナル）で使うツールです。コマンドラインを使っていくつかのツールを使いインストールをします。SlashGPTの開発者はMacを使っているので、現時点ではMacでの利用を推奨しています。(Windows, Linux利用者のフォードバックをお待ちしています！）

## 事前準備

- gitクライアント
- python
- pip

を入れておきます。

## インストール

### ソースをダウンロード

```
git clone https://github.com/snakajima/SlashGPT.git
```

### パッケージのインストール

```
cd SlashGPT
pip install -r requirements.txt
```

### API KEYを環境変数に設定

.envファイルを作成して、OPENAI_API_KEYを設定する

```.env
OPENAI_API_KEY=xxxx
```

(OPENAI_API_KEYは openaiのサイトから持ってきます。https://platform.openai.com/account/api-keys クレジットカードの登録が必要です。OPENAI_API_KEYは他人と共有しないように注意してください)


# 使い方

### 起動

OSのコマンドラインから起動します。

```
./SlashGPT.py 
```

```
AssertionError: OPENAI_API_KEY environment variable is missing from .env
```
と出る場合は、OpenAPのAPI KEYが未設定なので、設定をして再度起動してください

```
You(dispatcher):
```
と出た場合は、起動成功です。

以下、全てSlashGPTのプロンプトに入力をしてください。

#### ヘルプ1

```
/help
```
SlashGPTの使い方が表示されます

```
/switch main:      Switch to the manifest set in main (default)
/switch roles1:    Switch to the manifest set in roles1 (original)
/switch roles2:    Switch to the manifest set in roles2
~~~

Agents:
/aiagents     AI Agents
/browser      Internet Browser
/cal          Calendar
~~~

```

前半はSlashGPTが用意しているコマンドの一覧、後半のAgents以下はマニュフェストファイルで用意されているエージェントの一覧です。

#### ヘルプ2

```
/help {agent_name}
```

各エージェントのヘルプです。実際にはヘルプではなくマニュフェストファイルの中身が表示されます。
どのようなサンプルがあるかなどを確認することができます。

```
You(browser): /help spacex
{
  "title": "SpaceX Information",
  "description": "Anything about SpaceX",
  "bot": "spacex",
  "about": "snakajima",
  "temperature": 0,
  "functions": "./resources/functions/graphql.json",
  "resource": "./resources/templates/spacex.json",
  "actions": {
    "call_graphQL": {
      "graphQL": true,
      "url": "https://spacex-production.up.railway.app/graphql"
    }
  },
  "model": "gpt-3.5-turbo-16k-0613",
  "sample": "Who is CEO of SpaceX?",
  "sample1": "What kind of information are you able to get?",
  "sample2": "Get all the information of last 10 launches",
  "sample3": "Next Launch",
  "sample4": "Get names of all rockets",
  "sample5": "List all launchpads",
  "prompt": [
    "You are an expert in GraphQL and use call_graphQL function to retrieve necessary information.",
    "Ask for clarification if a user request is ambiguous.",
    "Here is the schema of GraphQL query:",
    "{resource}"
  ]
}
```


#### サンプルの動かし方

```
/sample
````

マニュフェストに用意されているサンプルを動作させます

```
You(spacex): /sample
Who is CEO of SpaceX?
{
  "name": "call_graphQL",
  "arguments": "{\n  \"query\": \"{ company { ceo } }\"\n}"
}
function(call_graphQL): {"company": {"ceo": "Elon Musk"}}
spacex: The CEO of SpaceX is Elon Musk.
```

#### 自分で質問する

```
You(spacex): {question}
```
ChatGPTを使うように質問をします。

```
You(spacex): 宇宙に行くスケジュールを教えてください
{
  "name": "call_graphQL",
  "arguments": "{\n  \"query\": \"{ launchesUpcoming { mission_name launch_date_local } }\"\n}"
}
function(call_graphQL): {"launchesUpcoming": [{"mission_name": "USSF-44", "launch_date_local": "2022-11-01T09:41:00-04:00"}, {"mission_name": "Starlink 4-36 (v1.5)", "launch_date_local": "2022-10-20T10:50:00-04:00"}, {"mission_name": "Galaxy 33 (15R) & 34 (12R)", "launch_date_local": "2022-10-08T19:05:00-04:00"}, {"mission_name": "Hotbird 13F", "launch_date_local": "2022-10-15T01:22:00-04:00"}, {"mission_name": "Hotbird 13G", "launch_date_local": "2022-11-02T23:24:00-04:00"}, {"mission_name": "Galaxy 31 (23R) & 32 (17R)", "launch_date_local": "2022-11-07T19:00:00-05:00"}, {"mission_name": "Eutelsat 10B", "launch_date_local": "2022-11-14T19:00:00-05:00"}, {"mission_name": "ispace Mission 1 & Rashid", "launch_date_local": "2022-11-21T19:00:00-05:00"}, {"mission_name": "CRS-26", "launch_date_local": "2022-11-18T17:00:00-05:00"}, {"mission_name": "Starlink 4-37 (v1.5)", "launch_date_local": "2022-10-31T20:00:00-04:00"}, {"mission_name": "O3b mPower 1,2", "launch_date_local": "2022-10-31T20:00:00-04:00"}, {"mission_name": "SWOT", "launch_date_local": "2022-12-04T16:00:00-08:00"}, {"mission_name": "SES-18 & SES-19", "launch_date_local": "2022-10-31T20:00:00-04:00"}, {"mission_name": "Transporter-6", "launch_date_local": "2022-11-30T19:00:00-05:00"}, {"mission_name": "TTL-1", "launch_date_local": "2022-11-30T16:00:00-08:00"}, {"mission_name": "WorldView Legion 1 & 2", "launch_date_local": "2022-11-30T19:00:00-05:00"}, {"mission_name": "Viasat-3 & Arcturus", "launch_date_local": "2022-11-30T19:00:00-05:00"}, {"mission_name": "O3b mPower 3.4", "launch_date_local": "2022-11-30T19:00:00-05:00"}]}
spacex: 以下は、SpaceXの将来の宇宙への打ち上げスケジュールの一部です：

1. USSF-44 - 2022年11月1日 09:41（現地時間）
2. Starlink 4-36 (v1.5) - 2022年10月20日 10:50（現地時間）
3. Galaxy 33 (15R) & 34 (12R) - 2022年10月8日 19:05（現地時間）
4. Hotbird 13F - 2022年10月15日 01:22（現地時間）
5. Hotbird 13G - 2022年11月2日 23:24（現地時間）
6. Galaxy 31 (23R) & 32 (17R) - 2022年11月7日 19:00（現地時間）
7. Eutelsat 10B - 2022年11月14日 19:00（現地時間）
8. ispace Mission 1 & Rashid - 2022年11月21日 19:00（現地時間）
9. CRS-26 - 2022年11月18日 17:00（現地時間）
10. Starlink 4-37 (v1.5) - 2022年10月31日 20:00（現地時間）

なお、これは一部の予定であり、変更される可能性があります。最新の情報については、SpaceXの公式ウェブサイトや関連ニュースソースをご確認ください。

```

### エージェントの切り替え

```
/{agent_name}
```

helpを参考にエージェントを切り替えます。用意されてるマニュフェストファイルを読み込んで新しいエージェントに切り替わります。

```
/home2
```

### LLMエンジンの切り替え

```
/llm {llm_engine}
```

```
/llm
/llm: gpt3,gpt31,gpt4,llama2,llama270,vicuna,palm
```
利用できるLLMを確認して

```
/llm gpt31
```
LLMエンジンを切り替えます。
設定等が必要な場合もあるので、その場合はwarningがでます。
PALMやREPLICATEのAPI KEY/Tokenが必要です。詳しくはSlashGPTの[README](https://github.com/snakajima/SlashGPT/#initialization)を参照してください

### 気持ちを切り替える

```
/new
```
SlashGPTはエージェントごとに会話履歴をLLMに投げるので、履歴をリセットできます


### 履歴を見る

```
/history
```
エージェントと対話している履歴を確認します


### システムプロンプトを見る

```
/prompt
```

システムプロンプトとしてLLMに投げる内容を確認できます

### マニュフェストセットを切り変え

```
/switch {dir_name}
```

マニュフェストはmanifestsのサブディレクトリごとに管理されているので、そのサブディレクトリを切り替えます。

### ログ

```
/verbose
```

詳細なログを出力するようにします。
トグルになっているので、再度コマンドを実行するとverboseがoffになります。


### 音声出力
```
/audio
```
音声出力を音にします
```
/audio ja
```
とすると日本語で出力されます。


### functions
```
/functions
```

エージェントに設定されてるfunctionsが出力されます

### 履歴のインポート

```
/import
```

```
/import 1
```

履歴のインポートができます

### テスト！！

```
/autotest
```

SlashGPTの機能をテストする自動テストです。主に開発者向けの検証用途です。


### まとめ

これで一通りの使い方の説明は終わりです。
用意されているagentをsampleを使って、試してください！

### 関連記事

- [SlashGPTのマニュフェストファイルの解説](https://zenn.dev/singularity/articles/slashgpt_tutorial_2)
- [SlashGPTのFunction callingの動作原理](https://zenn.dev/singularity/articles/slashgpt_spacex)
- [SlashGPTのプラグインを使って新しいLLMを追加する](https://zenn.dev/singularity/articles/slashgpt_llm_engine)
- [SlashGPTを使ってチョムスキーさんと田原さんを討論させる](https://zenn.dev/singularity/articles/slashgpt_agents)