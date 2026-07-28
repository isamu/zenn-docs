---
title: "GraphAIをサーバ/クライアントで動かす"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

Graphaiはサーバ/クライアントで動かすことが出来る

npmで公式のサーバ用のexpress middle wareが提供されている
なので、node.jsのexpressで実装されている
typescriptのagentをサーバでも使っている（agentはブラウザでも動く)

サーバを利用するには

サーバの設定
 - npm install
 - agentと必要な設定をする
 - それらをexpressに渡すだけ

クライアントの設定
 - http agent filterとbypassAgentIdsを設定する


サーバクライアントの

streamingについて
 - non-streaming, streamingの両方をサポート
 - クライアント側のhttpで動作の切り替えができる
   - 公式のagentFilterをクライアントで使う場合は勝手に処理する


入力値
    
endpointについて

agentFunctionInfoに基づいたapi listのapi




