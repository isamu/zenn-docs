---
title: "GraphAI - ブラウザ上でユーザの入力を扱うAgent"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

マルチエージェントのワークフローで、任意の場所でユーザの入力を受け付けることは難しいです。
サーバクライアント形式の場合、途中で処理を止めるか(ambient agent), pollingをして入力を待つ必要があります。
いずれの場合でも、検証ために入力条件が変わったり、複数の入力がある場合の処理を考えると、ややこしいことがわかります。

GraphAIでは、ワークフローとAgentの実行をわかることが出来ます。
ワークフローをブラウザで動かせば、ユーザの入力を待つagentを作ることはできそうです。

しかし、ワークフローのagentが環境に依存するように作ってしまうと柔軟性がかけるので、抽象化してGraphAIのワークフローとユーザの入力はできる限り依存しない実装が理想です。

ということで、外部の入力を非同期で待つことが出来るeventAgentを紹介します。

