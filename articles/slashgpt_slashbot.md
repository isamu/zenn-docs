---
title: "SlashBotを使ってterminalでAIにコードレビューをさせる"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ChatGPT, LLM, GitHub, OpenAI, SlashGPT]
published: true
publication_name: "singularity"
---


[SlashGPT](https://github.com/snakajima/SlashGPT/)は[中島聡](https://twitter.com/snakajima)が開発したChatGPTなどのLLMエージェントを手軽に開発するためのツールです。SlashGPTを使えば、jsonファイルを記述するだけでChatGPTを使ったLLMエージェントやチャットアプリを手軽に、簡単につくることができます。

本日、SlashGPTに新機能のSlashBotを追加しました。

SlashBotはUnixやMacのTerminalで標準入力から入力を受け取ってエージェントに問い合わせるボットツールです。

```
cat dog | slashbot  cat_or_doc_agent
```

のように、unixのコマンドをpipeして使うことができます。

cat_or_doc_agentに指定できるagentはSlashGPTのmanifestファイルで記述されたエージェントです。

コードレビューをするエージェント [codereview](https://github.com/snakajima/SlashGPT/blob/main/manifests/main/codereview.yml)を使うと、gitの出力と組み合わせることにより、AIがコードレビューをします。

```
 git show c69f5eeb5944e9007dc6c79839f119a3290b242e  -U9999 | slashbot codereview
```
のように特定のcommitをコードレビューしたり (git に-U9999のoptionをつけると差分以外の情報も含んだソースを渡すことが出来ます)


```
git diff hogeBranch |  slashbot codereview
```

と現在のブランチと、hogeBranchのブランチ間の差分をコードレビューさせることができます。

AIのよるレビューの例です。
```
Agent(codereview): このコミットでは、次の変更が行われました。

- eslintの設定に`@typescript-eslint/ban-ts-comment`が追加されました。
- `pdfjs-dist`が追加されました。
- `dragFile`という関数が追加され、`<div @dragover.prevent @drop.prevent>`内で使用されています。

改善点としては、以下の点が挙げられます。

1. `@typescript-eslint/ban-ts-comment`の設定について
   `@typescript-eslint/ban-ts-comment`は、TypeScriptのコードで不要な`@ts-ignore`コメントを禁止するためのルールです。しかし、このルールを`warn`レベルで設定することで、コード内に不要な`@ts-ignore`コメントが残っている可能性があります。コードを見直し、不要な`@ts-ignore`コメントを削除することをおすすめします。

2. `pdfjs-dist`の使用について
   `pdfjs-dist`は、PDFファイルを表示するためのライブラリです。しかし、このコミットでは`pdfjs-dist`の追加のみで、具体的な使用方法やコードの変更点が示されていません。コード全体を見直し、`pdfjs-dist`を使用するための適切な処理やエラーハンドリングを追加する必要があります。

3. `dragFile`関数の使用について
   `dragFile`関数は、ドラッグアンドドロップイベントに対応するための関数です。ただし、このコミットでは`dragFile`関数が追加されただけであり、具体的な処理や使用方法が示されていません。コード全体を見直し、`dragFile`関数を適切に呼び出す処理やエラーハンドリングを追加する必要があります。

以上の点に注意しながら、コードを見直し、バグや問題を解決するための適切な処理を追加してください。
```

のように、AIがコードレビューをしてくれます。


slashbotを含むSlashGPTのインストールは

```
pip install slashgpt==0.0.7
```

動作にはopenaiのapikeyが必要なので、作業するgitのdirectoryに `.env` を追加して

```
OPENAI_API_KEY=sk-xxxxx
```
のように設定を追加してください。

AIに渡すプロントは [こちら](https://github.com/snakajima/SlashGPT/blob/main/manifests/main/codereview.yml#L2-L6) にあるので、改善案があればPull Requestをお待ちします。

[SlashGPT リリースページ](https://pypi.org/project/slashgpt/0.0.7/)


