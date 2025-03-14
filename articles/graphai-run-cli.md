---
title: "GraphAIをcli(コマンドライン)で試す"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

GraphAIは、cli（コマンドライン), nodeのプログラム、ブラウザでのjavascriptのいずれの環境でも動作します。
cliでの簡単な試し方を解説します。

## GraphAI CLI

まずは、簡単にGraphAIを使ってみましょう。node, npmの環境とOpenAIのapi keyを用意してください。

そして、GraphAI の cliをインストールします。

```sh
npm i -g  @receptron/graphai_cli
```

以下のyamlファイル(Graphデータ)をダウンロードして保存してください。

https://raw.githubusercontent.com/receptron/graphai/refs/heads/main/packages/samples/graph_data/openai/business_idea_jp.yaml

そのファイルを保存したディレクトリーに.envというファイルをつくってOpenAIのAPI KEYを記述します。

```
OPENAI_API_KEY=sk-xxxx
```

用意は以上です。実行します。

```sh
graphai business_idea_jp.yaml
```

成功していれば、何も表示されないでしばらくまちます。

しばらくすると、「このビジネスアイデアコンテスト」のアイデア、評価、結果が表示されます。

yamlファイルをみると流れはわかると思いますがOpenAIのGPTに問い合わせるAgentが３回実行されています。
GPT(AI)への問い合わせと、その結果を使って更に問い合わせをするという流れです。

問い合わせの内容を変更すると、ビジネスアイデア以外の結果を得ることができるので、改良して使ってみてください。

このように、yamlファイルに定義することで、簡単にAIを使ったAgentを組み合わせて動かすことが可能となります。


## Sample

### yamlのサンプル

graphaiのcliコマンドで実行できるyamlサンプルです。`@receptron/graphai_cli`がインストール済みであれば

```sh
graphai {filename}
```

で実行できます。

https://github.com/receptron/graphai/tree/main/packages/samples/graph_data

