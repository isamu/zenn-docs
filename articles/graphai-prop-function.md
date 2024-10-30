---
title: "GraphAIのProp function"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

GraphAIでは、`inputs`に「`:nodeId.propId.propId`」という形式で指定することで、依存関係と入力値を定義できます。また、この指定を利用して、データ操作を行う「prop function」を適用する機能も備えています。各データ型に対応したprop functionは以下の通りです。

## Prop Function一覧

### Array（配列型データ）
- `length()`: 配列の要素数を返します。
- `flat()`: ネストされた配列をフラット化して1次元配列として返します。
- `toJSON()`: 配列をJSON形式の文字列に変換して返します。
- `isEmpty()`: 配列が空の場合は`true`を、要素が存在する場合は`false`を返します。
- `join()`: 配列の要素を空文字で連結して文字列として返します。
- `join(,)`: 配列の要素をカンマで区切って連結した文字列として返します。
- `join(-)`: 配列の要素をハイフンで区切って連結した文字列として返します。

### Object（オブジェクト型データ）
- `keys()`: オブジェクトのすべてのキー（プロパティ名）を配列として返します。
- `values()`: オブジェクトのすべての値を配列として返します。
- `toJSON()`: オブジェクトをJSON形式の文字列に変換して返します。

### String（文字列型データ）
- `codeBlock()`: 文字列内のMarkdown形式のコードブロック（```` ``` ````で囲まれた形式）内の文字列のみを返します。
- `jsonParse()`: 文字列をJSONとして解析し、対応するデータ型に変換して返します。
- `toNumber()`: 数字を表す文字列を数値型に変換して返します。

### Number（数値型データ）
- `toString()`: 数値を文字列型に変換して返します。
- `add({integer})`: 指定した整数値を現在の数値に加算して返します。

### Boolean（論理型データ）
- `not()`: 論理値を反転し、`true`は`false`に、`false`は`true`に変換して返します。

