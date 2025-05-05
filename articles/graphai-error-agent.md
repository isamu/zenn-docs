---
title: "GraphAIのエラー周りのメモ"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

# GraphAIのエラー周りのメモ

## agentの 通常の終了時

- node.afterExecuteが呼ばれる
- 状態、結果を更新してonSetResultを呼ぶ
- node.onSetResultを呼び出しtaskManagerのキューの更新をする（waiting list/pending)
- graph.onExecutionCompleteを呼ぶ
  - 通常はisRunningがtrueなので、依存を考慮して次のnodeがよばれる


## agent のエラー時

- retry関数で、retryの有無を判断し、retryしない場合に、graph.onExecutionCompleteを呼ぶ(afterExecuteは呼ばれないので正常終了の処理はされない)
  - エラーになったnode自身をCompleteする。しかし、キューの更新がないので依存タスクは実行されない
  - 通常はisRunningはfalseになる(並列でなにか走っているときにはtrueなのでgraphの処理は続く。その後、他タスクが終わると後述の処理でエラー判定される、）
- graph.onCompleteが呼ばれる
  - errorになったnodeがあるのでrejectされ、GraphAIインスタンスはエラーをスルーする

## agentの処理が終わらない時
  - agentでthrowError指定できるものは例外を投げずにonErrorを含む結果を返すので次の処理に進む
  - 並列で他の処理をしている場合は、その処理が終わるまで処理が続く
  - mapAgentの場合は、それぞれの処理はmapAgent内では別のGraphAIとして実行されるので他のGraphAIインスタンスはエラーの影響を受けずに実行する
  - mapAgentはthrowErrorを指定できるのでthrowError: trueにしないとサブグラフでエラーが発生してもmapAgentの次の処理に進む
  - nestedAgentも同様にthrowErrorが必要