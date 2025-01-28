---
title: "GraphAI - Agentの紹介(随時更新)"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

# メタパッケージ
他のAgentパッケージをまとめたメタパッケージ。

- https://www.npmjs.com/package/@graphai/agents
- https://www.npmjs.com/package/@graphai/llm_agents
- https://www.npmjs.com/package/@graphai/extra-agents

# LLM系
各種llmのagent.
inputs/result/paramsは各agentで極力互換性があり

- https://www.npmjs.com/package/@graphai/openai_agent
- https://www.npmjs.com/package/@graphai/anthropic_agent
- https://www.npmjs.com/package/@graphai/gemini_agent
- https://www.npmjs.com/package/@graphai/groq_agent
- https://www.npmjs.com/package/@graphai/replicate_agent

- https://www.npmjs.com/package/@graphai/token_bound_string_agent
- https://www.npmjs.com/package/@graphai/openai_fetch_agent
  - openaiのライブラリを使わないでfetchで実装したopenaiのパッケージ
- https://www.npmjs.com/package/@graphai/tools_agent
  - https://github.com/receptron/graphai-demo-web/ のgoogle map/video agentと連携して、tools(function calling)を実行するagent


# 基本機能

- https://www.npmjs.com/package/@graphai/vanilla
  - 他のnpmに依存しない基本機能を提供するagent群. 多くのsampleはこのagent + llmで実装されている
- https://www.npmjs.com/package/@graphai/data_agents
  - データをマージするagent
- https://www.npmjs.com/package/@graphai/sleeper_agents
  - sleepを提供するagent.　主にdebugやtest用
  
# データ変換

- https://www.npmjs.com/package/@graphai/pdf2text_agent

# net service系

- https://www.npmjs.com/package/@graphai/service_agents
  - fetchとwikipedia
- https://www.npmjs.com/package/@graphai/arxiv_agent
- https://www.npmjs.com/package/@graphai/serper_agent

# prompt系
- https://www.npmjs.com/package/@graphai/awesome_chatgpt_prompts_agent
- https://www.npmjs.com/package/@graphai/prompts



# node(サーバ)用
- https://www.npmjs.com/package/@graphai/input_agents
  - ターミナルでユーザとのinteractionを提供する
- https://www.npmjs.com/package/@graphai/vanilla_node_agents
  - nodeでのみ動くvanilla agent. 主にfile操作関連
- https://www.npmjs.com/package/@graphai/shell_utilty_agent
  - shellを実行するagent

# speach系

TTS/STT

- https://www.npmjs.com/package/@graphai/tts_openai_agent
- https://www.npmjs.com/package/@graphai/tts_nijivoice_agent
- https://www.npmjs.com/package/@graphai/stt_openai_agent


# webのユーザinput/event

- https://www.npmjs.com/package/@receptron/event_agent_generator
  - webのformからの入力などのevent処理を扱うagent
  