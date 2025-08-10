---
title: "GraphAI - ToolsAgentから呼び出されるAgent"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::


## toolsAgentから呼び出されるagentのspec

- namedInputs
  - agentName - tool.name(funciton name)
  - arg - tool.arguments
  - func - tool.name(funciton name)
  - data - passthrough from parent

- result
  - content
  - data
  - hasNext

- agentFunctionInfo
  - toolsにschema
