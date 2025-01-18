---
title: "GraphAI 既知の問題点"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---


# GraphAIのinjection Value

GraphAIのStatic nodeに外から値を注入するinjectValueがあります。
これで注入された値は、そのグラフをloopで動作させた場合に、２周目以降はinjectされた値が消える(valueで上書きされる。)ので想定しない動作になる可能性があります。
これを回避するには、injectValueを使わないで、GraphAIのコンストラクタにわたす前にGraphDataを更新して値をセットするか

```TypeScript
staticNode: {
  update: ":staticNode"
},
```

とupdateで、自身のnodeを指定することで、２回目以降も注入された値を使うことができます。

# anyInputとDynamic Agent

anyInputは、依存しているnodeのいずれか１つが実行完了すると、他のnodeの完了を待たずにすぐにそのAgentを実行します。
通常はinputs部分のみにGOD formatを使うことを想定します。Dynamic AgentではAgent部分もGOD Formatで依存のかけるので、このケースではinputs、もしくはagentが決まった段階で、someAgentが実行されるので意図しない動作となります。

```TypeScript
someAgent: {
 anyInput: true,
 agent: ":depAgent",
 inputs: {
   array: [":node1", ":node2"]
 }
},
```
