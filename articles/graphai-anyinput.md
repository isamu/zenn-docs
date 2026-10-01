---
title: "GraphAI - anyInputについて"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

# anyInputについて

`anyInput`は、複数の入力を受け取る `inputs` に対して、**いずれか1つの入力が到達した時点でエージェントを実行**するためのオプションです。

通常、すべての入力が揃うまで待ってから処理を開始しますが、`anyInput`を使うことで、**最速で到達した入力だけで処理を開始**することが可能になります。  
これにより、反応速度を重視したユースケースや、部分的なデータでも十分な処理において有効です。

```
inputs: {
  a: ":anode.data",
  b: ":bnode.data",
  c: ":cnode.data",
}
```

のような場合、bnodeが先に実行完了した場合には


```
inputs: {
  b: ":bnode.data",
}
```

となります。

# arrayで使う場合

```
inputs: {
  array: [
    ":anode.data",
    ":bnode.data",
    ":cnode.data",
  ]
}
```

だと、通常のケースで考えると、

```
inputs: {
 array: [
   undefined,
   ":bnode.data",
   undefined,
 ]
}
```

となりますが、これだと不便なのでGraphAIでfilterして

```
inputs: {
 array: [
   ":bnode.data",
 ]
}
```
となります。

```
inputs: {
 array: [
   ":bnode.array",
 ]
}
```

のように array を選択するケースでは（LLM の message でよく使いますね）、array の中に array があって不便なので、`arrayFlatAgent` を使うと良いです。

# 通常は if / unless を組み合わせて使う

通常は、その手前で  
`if` / `unless` を使ったノードでデータの分岐を行ってから使用します。


# 利用例(実装サンプル)

https://github.com/receptron/graphai/blob/841ded91c9ef8b07ff67447988e44d85ed2f4010/llm_agents/tools_agent/src/tools_agent.ts#L107

https://github.com/receptron/graphai/blob/841ded91c9ef8b07ff67447988e44d85ed2f4010/packages/samples/src/llm/research.ts#L143

https://github.com/receptron/graphai/blob/841ded91c9ef8b07ff67447988e44d85ed2f4010/packages/samples/graph_data/ollama/weather.yaml#L104

https://github.com/receptron/graphai/blob/841ded91c9ef8b07ff67447988e44d85ed2f4010/agents/vanilla_agents/src/string_agents/string_update_text_agent.ts#L29