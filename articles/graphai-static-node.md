---
title: "GraphAI - Static Node と injectValue について"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

static nodeのvalueは、GgraphData内に定義される固定値/変数です。
通常は初期値や定数として使ったりLoopで前の実行結果を受け取り次のloopの値として利用します。

一方、injectValueは、実行時に外部から注入される値です。
static nodeのvalueとして、GraphAIのインスタンスのAPIとして呼び出してStatic Nodeの値をセットします。

また、static nodeのupdate属性にgod記法（`:nodeName.property`など）を記述すると、ループ時に前回の結果を更新することができます。

# パターン別の使い方

## 初期値のみ（ループなし、または2回目以降も同じ値の場合）

```json
{ value: "abc" }
```

毎回 `abc` を返します。

## 初期値と更新対象の値を両方持つ場合

```json
{ value: "abc", update: ":node1.prev1" }
```

１回目に `abc` を、２回目以降はnode1のprev1の値を返します。


## 初期値も更新対象も持たない場合

```json
{ }
```

この場合は、injectValueによって値が更新されることを期待します。
特に、ネストされたnodeでは、親グラフから値が渡されることが想定されています。（本来は定義しなくても動作します）
injectValueによって値がセットされない場合がGraphAI.runでvalidationエラーとなります

## 更新対象の値のみを持つ場合

```json
{ update: "node1.prev1" }
```

この場合、1回目はinjectValueで値がセットされ、2回目以降は別の値が使われます。


:::message alert
重大な注意点
valueとinjectValueは同じではありません。
:::

たとえば次のように定義した場合、

```json
{ value: "123" }
```

この場合、何回ループしても常に"123"が設定され続けます。

一方、injectValueの場合、初回実行時にのみ値がセットされます。
たとえば次のように空の定義でinjectValueだけ渡す場合、

```json
{ }
```

初回はinjectValueの値が使われますが、2回目以降はundefinedになります。
injectValueはあくまでも実行時に値を注入するものであり、nodeの静的な値ではありません。

### selfNodeを使ったupdate指定について

ノード自身の値を参照して更新を行いたい場合は、以下のように記述します。

```json
{ update: ":selfNode" }
```

たとえば、messageというノードの場合は次のように書きます。

```json
nodes: {
  message: { update: ":message" }
}
```

このように設定すれば、初回はinjectValueによって値がセットされ、2回目以降はそのノード自身の最新の値を参照して更新することができます。

