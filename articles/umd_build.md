---
title: "GraphAIをcodepenで試す"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---


- rollup.config.mjs を作成（コピー）
  - graphai-utils/packages/vue-cytoscape/rollup.config.mjs が一番参考になる
  - 外部パッケージを使う場合はglobalsとexternalの調整
- package.json を編集
  -module / browserを追加
  - buildにrollupを追加
- buildする(lib以下にbuildされる）
- npm publish
- jsdelivrでパッケージを指定してhtmlのscriptタグ取得