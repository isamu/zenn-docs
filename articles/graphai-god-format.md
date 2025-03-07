---
title: "GraphAI - GOD Format"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

# GOD Format(Graph Object Dot Format)

GOD Formatは、GraphAIデータ構造を表現するための軽量で人間が読みやすいフォーマットです。GraphAI Object Dot (GOD)仕様に基づき、GraphAI内で依存関係を定義したり、データを取得するために使用されます。

GOD Formatの概要
GOD Formatは、GraphAIでのデータ依存関係を簡潔に記述できるフォーマットです。構文はコロン (:) で始まり、ドット (.) で要素を区切ります。基本構造は以下の通りです。

```typescript
:nodeId.prop1.prop2
```

`nodeId`: データを取得する元のノードを指定します。
`prop1`, `prop2`: ノード内のプロパティやネストしたデータのパスを指定します。これらは連結することで深い階層のデータにもアクセス可能です。
配列の場合、要素は `$0`, `$1` などでアクセスできます。

## 使用可能な場所
GOD Formatは以下の箇所で使用できます：

- inputs: ノードのデータ依存関係と入力値を定義するため。
- ~~params: ノードの実行時に必要なパラメータを指定するため。~~ 次のバージョンupで変更予定
- graph: ネストしたGraphを実行する為に、GraphDataを定義するため。
- if/unless: 条件付きでノードを実行するため。
- agent: エージェントを動的に指定するため。
- console (after): ノードの実行結果をログとして出力するため（この場合、nodeIdは省略可能で、.から始まります）。
- result: nestedAgentのために、結果をtransformする（この場合、nodeIdは省略可能で、.から始まります）。

## GOD Formatの使用例

#### 1.基本的な構文

```typescript
:nodeId.prop1.prop2
```
指定されたnodeIdのprop1内にネストされたprop2の値を取得します。

期待されるデータ
```typescript
{
  prop1: {
    prop2: "hello"
  }
}
```

#### 2.配列の要素指定

```typescript
:nodeId.prop1.$0
```
指定されたnodeIdのprop1内の配列の最初の要素を取得します。

期待されるデータ
```typescript
{
 prop1: [1,2,3]
}
```

#### 3.console after

```typescript
console: {
  after: {
    log: ".prop1.prop2"
  }
}
```
console.afterで使用され、現在のノードの実行結果に基づいてprop1やprop2を参照します。この場合、nodeIdは省略されます。

#### 4.動的な文字列挿入
GOD Formatは文字列内での動的な値挿入にも対応しています。例：

```typescript
this is ${:nodeId.prop1.prop2}
```
nodeIdのprop1内にネストされたprop2の値を文字列に挿入します。

## ネスト構造と柔軟性
inputs、およびconsole.afterでは、ネストした記述にも対応しており、複雑なデータ構造を簡潔かつ効率的に記述できます。

例
```typescript
inputs: {
  messages: [
    {
      role: "system",
      content: ":context.person0.system",
    },
    {
      role: "user",
      content: ":context.person1.greeting",
    },
  ],
  context: ":context",
}
```

## 依存関係の管理
GOD Formatで指定されたnodeIdは、Graph実行時の依存関係を定義する重要な役割を果たします。
console.after（ログ目的）を除くすべての箇所で、指定された依存関係はGraphの動作に必要なものとして扱われます。

