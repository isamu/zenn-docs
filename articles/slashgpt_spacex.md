---
title: "SlashGPTのFunction callingの動作原理"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [gpt, ChatGPT, LLM, Tech, OpenAI]
published: true
publication_name: "singularity"
---

[SlashGPT](https://github.com/snakajima/SlashGPT/)は[中島聡](https://twitter.com/snakajima)が開発したChatGPTなどのLLMエージェントを手軽に開発するためのツールです。SlashGPTを使えば、jsonファイルを記述するだけでChatGPTを使ったLLMエージェントやチャットアプリを手軽に、簡単につくることができます。

OpenAIから2023年6月に発表されたFunction callingにも対応し、プログラムを一切書くとことなく、Function callingとAPIサービスを連携させたLLMエージェントを作成することが可能です。

今回はspacexエージェントを例に、Function callingの使い方や動作原理を解説します。

# Function callingを使ったマニュフェストの定義

SlashGPTのFunction callingの使い方を理解するには、事前に用意されているマニュフェストを参考にするのがよいです。
ここでは例として`/spacex`で利用できるspacexエージェントのマニュフェストを参照します。
レポジトリのmanifests/main/spacex.jsonにspacexのマニュフェストがあります。
マニュフェストから動作に影響がある部分を抜き出します。

```
{
  "title": "SpaceX Information",
  "functions": "./resources/functions/graphql.json",
  "resource": "./resources/templates/spacex.json",
  "actions": {
    "call_graphQL": {
      "graphQL": true,
      "url": "https://spacex-production.up.railway.app/graphql"
    }
  },
  "prompt": [
    "You are an expert in GraphQL and use call_graphQL function to retrieve necessary information.",
    "Ask for clarification if a user request is ambiguous.",
    "Here is the schema of GraphQL query:",
    "{resource}"
  ]
}
```

リソースファイルの
```
 "resource": "./resources/templates/spacex.json"
```
は、GraphQLのスキーマが定義されています。
resourceで定義されているので、promptの{resource}が置換されこのファイルが展開されます。
"Here is the schema of GraphQL query: {ファイルの中身〜〜〜}"のようになります


```
  "functions": "./resources/functions/graphql.json"
```
は、GPTに渡すFunction callingの定義で、LLMがクエリーに対して一致する情報があると、ここで定義した形式で情報を返してくれます。これがLLMに対してFunction callingを指定する肝となる部分です。このファイルの中を見ると

```
[
  {
    "name": "call_graphQL",
    "description": "access graphQL endpoint",
    "parameters": {
      "type": "object",
      "properties": {
        "query": {
          "type": "string",
          "description": "graphQL query"
        }
      },
      "required": ["query"]
    }
  }
]
```
となっています。call_graphQLという関数名で、queryという文字型のデータを含むオブジェクトを返すようになっています。コメントにあるようにgraphQLのendpointへのクエリーとなっています。
さて、LLMの返却値にこの結果が含まれているときに、このcall_graphQLをどのように使うかし調べます。
再びマニュフェストを見るとactionsに

```
  "actions": {
    "call_graphQL": {
      "type": "graphQL",
      "url": "https://spacex-production.up.railway.app/graphql"
    }
  },
```
と、定義されています。call_graphQLが返ってきた場合には、urlに定義しているendpointに対してgraphQLのクエリーで問い合わせをします。
問い合わせた結果を、再び最初のユーザの質問と一緒にLLMになげることで、最終的な回答を作成します。


# 実際の動作

```
./SlashGPT.py 
```
でSlashGPTを起動し

```
/spacex
```

とエージェントを切り替えて、/sampleを動かしましょう。そうすると自動的に質問をします。

```
Who is CEO of SpaceX?
```

が呼び出され、resource, functionsを含んだクエリーをOpenAIのLLMに投げます。

すると,LLMからは

```
{
  "name": "call_graphQL",
  "arguments": "{\n  \"query\": \"{ company { ceo } }\"\n}"
}
```

という結果が返ってきます。この結果のargumentsをパースしオブジェクトに復号しspacexのgraphQLサーバにクエリーとして投げます。

spacexのサーバーからは
```
{"company": {"ceo": "Elon Musk"}}
```
と返ってきます。この結果を再びLLMに投げると

```
The CEO of SpaceX is Elon Musk.
```

と、完璧な応えが得られます。

余談ですが、試しに

```
CEOはだれ？
```

と日本語で聞くと

```
SpaceXのCEOはElon Muskです。
```
と、日本語で結果が得られます。


# まとめ

spacexのマニュフェストを参考にFunction callingを使って外部のAPIと連携するサンプルをみました。これを参考に、マニュフェストを変更することで、プログラムを一切書くとことなくFunction callingを使ったLLMエージェントを作成することが可能となります。


### 関連記事

- [SlashGPTの使い方](https://zenn.dev/singularity/articles/slashgpt_tutorial_1)
- [SlashGPTのマニュフェストファイルの解説](https://zenn.dev/singularity/articles/slashgpt_tutorial_2)
- [SlashGPTのプラグインを使って新しいLLMを追加する](https://zenn.dev/singularity/articles/slashgpt_llm_engine)
- [SlashGPTを使ってチョムスキーさんと田原さんを討論させる](https://zenn.dev/singularity/articles/slashgpt_agents)
- [SlashGPTのFunction callingの詳細](https://zenn.dev/singularity/articles/slashgpt_function_calling)

- [SlashGPTで、No-CodeでAIエージェントを作る方法](https://zenn.dev/snakajima/articles/adf436f7f794b3) by 中島さん
- [SlashGPT + Jupyter](https://zenn.dev/singularity/articles/718cba24bf1275) by kozayupapaさん



