---
title: "GraphAIの設定値をGraphAIからAgentに渡す"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: false
publication_name: "singularity"
---

Agentは、設定値が必要なものがある

API keyなど
S3のbucketなど。
サービス、アプリ特有のuserIdなどを含むagent


- paramsで渡す
  - 
- 環境変数で渡す


userIdなどの、動作ごとに変わるもの。アプリ依存の変数でユーザごとなど。
api keyなど。サービス固有の値で、サービスごとにグローバルな設定

サーバがで動かす場合、環境変数で渡すかparamsで渡すのが一般的。入力依存の場合のみinputs
ブラウザで動かす場合は環境変数が使えず、paramsで渡す必要があるので大変。
ブラウザとサーバで一貫して

paramsだと、同じagentが複数使われている場合に全てに同じ設定をする必要がある
static nodeで設定して、それを使う、という方法もある。
configで渡すと楽。


GraphData、特に設定値の扱い注意。

