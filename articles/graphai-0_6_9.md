---
title: "GraphAI 0.6.9"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

# GraphAI 0.6.9 リリースノート

GraphAI 0.6.9がリリースされました。
主な変更点は以下。

#### 新機能

- **GraphDataのエージェントを動的に指定可能に**  
  `agent: ":node.agentName"` のように記述することで、エージェントを動的に指定できるようになりました。  
  - エージェントは `GraphAI` のコンストラクタ内のバリデータによって処理されますが、このエージェントの有無に関するチェックはスキップされます。

- **AgentFunctionInfoDictionary のバリデータを追加**  
  - `AgentFunctionInfoDictionary` のキーと値の検証が強化されました。  
  - 値が未定義または不正な場合（例: エージェントを含まない場合）、`GraphAI` のコンストラクタでエラーが発生するようになります。

