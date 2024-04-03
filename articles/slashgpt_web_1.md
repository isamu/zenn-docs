---
title: "SlashGPT webの紹介"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [gpt, ChatGPT, LLM, Tech, SlashGPT]
published: true
publication_name: "singularity"
---

# SlashGPTとは

[SlashGPT](https://github.com/snakajima/SlashGPT/)は[中島聡](https://twitter.com/snakajima)が開発したChatGPTなどのLLMエージェントを手軽に開発するためのツールです。SlashGPTを使えば、jsonファイルを記述するだけでChatGPTを使ったLLM AI エージェントやチャットアプリを手軽に、簡単につくることができます。

# SlashGPT webとは

SlashGPTのmanifestをweb上で編集し、web上で動作確認するツールです。manifestと呼ばれるプロンプトの動作を定義するファイルをfunction callingの機能をWeb上で作成、確認できます。
作成したmanifestはダウンロード可能で、Python版やTypeScript版のSlashGPTでプログラムから利用可能です。

動かすためには、Node.js、ブラウザ、OpenAIのAPI Keyが必要です。

# インストール

GitHub上でMITライセンスで公開されています。

https://github.com/isamu/slashgpt-web

cloneします。

```sh
git clone https://github.com/isamu/slashgpt-web.git
```

packageをいれる

```sh
yarn install
```

起動
```sh
yarn run serve
```

# 動作

画面は2024/04/03現在です。日々開発中ですので、画面や機能は随時変わっていきます。

起動時してブラウザで開きます。画面は左側がプリセットのmanifestデータ、真ん中がmanifestの編集画面、右がLLMとのチャット画面です。
![](https://storage.googleapis.com/zenn-user-upload/2223a6964222-20240403.png)

真ん中あたりにOpenAIのAPI keyを入力する部分があります。OpenAIのサイトからAPI Key取得し入力してください。ブラウザのlocalStorageに保存するので、我々のサーバ等に送られることはありません。localで同じポートで起動するサービスには注意してください。
その他のデータも2023/4/3現在、全てlocalStorageに保存しています。
![](https://storage.googleapis.com/zenn-user-upload/5b697fbbaf8e-20240403.png)

上に戻り、左側の「論文レビュー」のボタンを押します。プリセットの論文解説のmanifestがロードされ、form部分の項目にデータがセットされます。
![](https://storage.googleapis.com/zenn-user-upload/fd6ee211fd4c-20240403.png)

右側にsampleのプロンプトが表示されています。これはmanifestに設定されているsampleのプロンプトで、Set Sampleを押すとチャットのユーザメッセージ部分に入力されます。テストのために、まずこのボタンを押してチャットメッセージをセットしてください。
その後、Sendボタンを押すと、LLMに問い合わせを開始します。

![](https://storage.googleapis.com/zenn-user-upload/c043b9509812-20240403.png)

LLMの処理が終わるとGPTからのメッセージが表示されます。
![](https://storage.googleapis.com/zenn-user-upload/b6384e9ba916-20240403.png)

このマニュフェストは、論文を読んで、その論文の概要をまとめるプロンプトです。Sampleには「Attention Is All You Need」のabstractが入力されていて、その論文の概要が結果として返ってきています。

論文のまとめにはfunction callingを使っていて、LLMに対してfunctionで指定した項目を整理するように命令し、その結果をさらにformatしてLLMで最終的な文章にまとめるようにしています。

このように、オンライン上でfunction callingを使ったAI Agentが簡単に作成可能です。

function callingの結果を使ってAPIを叩いたり、GraphQLを使うことも可能となっています。

以上で、SlashGPT webの動作を説明しました。
