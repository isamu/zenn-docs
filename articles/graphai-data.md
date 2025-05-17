---
title: "GraphAI / Agent間のデータのやりとり"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: false
publication_name: "singularity"
---


## GraphAIにおけるデータ変換と入出力制御の基本

GraphAIを使いこなす上で最も重要なポイントは、**Agent間のデータのやりとり**です。  
特に、`inputs`の構造やその応用、そしてデータを変換するためのAgentの使い方を理解することが鍵となります。

入力と出力の仕組みを適切に理解し、自在に操ることができれば、**あらゆる複雑な処理を実現することが可能**です。

---

### 入出力を制御する方法

GraphAIでは、以下の3つの手法によってAgentの入出力をコントロールできます：

1. **`inputs`(GOD Format)による入力指定**  
   - 最も基本的な方法
   - できる限りこの方法で必要なデータを整形し、他のAgentからデータを受け取れる
   - Prop functionやutility functionでデータの加工も可能

2. **`output` / `passThrough`による出力整形**
   - Agentの出力形式を調整したい場合に使用します
   - 単純な変換やフィルタリングであれば、こちらで十分対応可能です

3. **データ変換用Agentの利用**
   - 上記の方法で対応できない場合、専用のAgentを使って中間データを変換

---

このように、**入力→処理→出力**の流れを明確に設計し、必要に応じて変換処理を挟むことで、GraphAI上で柔軟かつ強力な処理フローを構築できます。


# inputの基本

## GOD Format(GraphAI Object Dot Format)で入力(前のノードの出力)を整形する
GOD Formatは、GraphAIデータ構造を表現するための軽量で人間が読みやすいフォーマットです。GraphAI Object Dot (GOD)仕様に基づき、GraphAI内で依存関係を定義したり、データを取得するために使用されます。

GOD Formatは、GraphAIでのデータ依存関係を簡潔に記述できるフォーマットです。構文はコロン (:) で始まり、ドット (.) で要素を区切ります。基本構造は以下の通りです。

```typescript
:nodeId.prop1.prop2.$0.prop3
```


`nodeId`: データを取得する元のノードを指定します。
`prop1`, `prop2`: ノード内のプロパティやネストしたデータのパスを指定します。これらは連結することで深い階層のデータにもアクセス可能です。
配列の場合、要素は `$0`, `$1` などでアクセスできます。

TypeScriptで書き換えると以下の処理と同等です。

```typescript
nodeId["prop1"]["prop2"][0]["prop3]
```


## template記法

入力のテンプレートにGOD formatのデータを埋め込むことができます。

```typescript
"hello, I'm ${:userNode.username}"
```

## 構造化されたinputs
GOD formatをtemplateとして使い、またinputs自体が構造化できるので以下のような指定で入力を定義できます。

```typescript
inputs: {
  messages: [
    {
      role: "system",
      content: "hello, you are a ${:context.person0.system}",
    },
    {
      role: "user",
      content: ":context.person1.greeting",
    },
  ],
  context: ":context",
}
```

### Prop funciton

GOD Formatでは、**関数を使ってデータを柔軟に加工**することができ、この機能を**Prop Function**と呼びます。  
データの型(Array、Object、Stringなど)に応じた関数が用意されており、入力整形や前処理に活用できます。

### 基本構文

GOD Formatのデータ参照に、関数をチェーンして記述します

```
:node1.array.length()
:node2.object.toJSON()
```

- 関数は**必ず()**を付けて呼び出します。
- 一部の関数は引数を受け取ることもできます。
- **メソッドチェーン**にも対応しています：

```
:node1.arrayJSON.jsonParse().length()
:node1.markdownWithJSONSample.codeBlock().jsonParse()
```

### 例：主な関数一覧

#### Array向け関数
- `length()`：配列の要素数を返す
- `flat()`：ネストされた配列を1次元に変換
- `toJSON()`：配列をJSON文字列に変換
- `isEmpty()`：空配列かどうかを判定

#### String向け関数
- `codeBlock()`：Markdownのコードブロック内の文字列を抽出
- `jsonParse()`：文字列をJSONとしてパースし、データに変換


### utility function

Prop funcitonはデータを処理をする関数ですが、それに対して単体で動作する関数もあります。utility functionと呼びます。
GOD formatとして利用できます

- `${@loop}` Loop利用時にloop counterの値を数値で取得
- `${@now}` 現在時刻のtimestamp


## 入出力の標準化

Agentごとに似たような処理をしていても、出力されるデータの形式が異なると、それぞれの構造を都度確認する必要があり、開発・保守のコストが増加します。
GraphAIでは、こうした課題を解決するために、**標準的なデータは共通の形式で返す**ことを推奨しています。
これにより、複数のAgentを組み合わせる際の整合性がとれ、デバッグや再利用性が大幅に向上します。

### 標準化のメリット

- Agent間のデータ連携がスムーズになる
- 一貫した形式により、変換や補正の手間が減る
- GUIや自動補完による支援が効きやすくなる
- テストやログ出力のパターンを共通化できる

GraphAIでは、この共通化された構造をベースに**設計パターン**を定義し、Agent開発のガイドラインとして活用しています。

例えば、Arrayでは
```
{
  array: [],
  item: "a",
  items: [],
}
```
llmのpayloadは
```
{
  message: {
    role: "user",
    content: "123"
  }
}
```
などがあります。

詳しくはGitHubにある以下のドキュメントをご覧ください：  
🔗 [GraphAI Inputs Guide](https://github.com/receptron/graphai/blob/main/docs/guide/inputs.md)

ご意見・ご要望があれば、ぜひレポジトリにIssueを立てて議論にご参加ください！


## 結果の変形

GraphAIでは、Agentの出力結果を柔軟に調整するための仕組みとして、  
`output` と `passThrough` の2つの設定を提供しています。

これらを活用することで、Agentが返すデータ構造を意図通りに制御でき、次のNodeへの連携がスムーズになります。

---

### output

- Agentの**出力結果を整形**するためのオプションです。
- 自身のNodeの出力のみを対象とし、**他のNodeを参照することはできません**。
- GOD Formatによる指定が可能で、必要な項目だけをマッピングして返すことができます。

```
{ extraText: ".text" }
```

この例では、Agentの出力の `text` プロパティを `extraText` に変換し、それ以外のデータは除外されます。

### passThrough

- Agentの実行結果に**関係なく、指定した値を出力に追加**できます。
- リテラル値の指定も、GOD Formatによる他ノードの参照も可能です。
- 主に **固定値の付加** や **他ノードの出力をマージ** する用途に使います。

```
{ prevResult: ":node1.prop1" }
```

この例では、`:node1.prop1` の値を現在のNodeの出力に `prevResult` として追加します。
passThroughは、指定したデータを追加するので元の結果はそのまま出力されます。
重複するkeyの場合は上書きします

---

※ `output` は Nested Agent の出力整形に、`passThrough` は外部データや固定値の付加に使われることを想定して設計されています。


## データを変換するAgent

`god format` や `output` / `passThrough` では対応しきれないケースでは、
**データ変換専用のAgent**を使います。

これらのAgentは通常の LLM Agent と同様に、Nodeとして定義して使います。
以下に、Vanilla Agentで提供されている主なデータ変換用Agentを紹介します。

---

### Array関連

- `popAgent` / `shiftAgent` / `pushAgent`：配列の操作を行う
- `arrayFlatAgent`：ネストされた配列をフラットにする
- `arrayJoinAgent`：区切り文字で配列を連結する
- `arrayToObjectAgent`：配列内の要素の指定プロパティをキーにしてオブジェクトに変換する
```
  inputs: {
    items: [
      { id: 1, data: "a" },
      { id: 2, data: "b" },
    ],
  },
  params: { key: "id" },
  result: {
    "1": { id: 1, data: "a" },
    "2": { id: 2, data: "b" },
  },
```


### Object関連

- `mergeObjectAgent`：複数のオブジェクトをマージして1つにまとめる
```
[{ color: "red" }, { model: "Model 3" }]  -> { color: "red", model: "Model 3" }
```
- `lookupDictionaryAgent`：paramsに定義した辞書から、inputsで指定されたキーの値を取り出す
```
    const data = { ja: "こんにちは", en: "hello" };
    return data[index];
```    
- `copyAgent`：inputsのオブジェクトから指定したキーの値を抽出する
  - keyを指定しない場合、オブジェクト全体を返す(GUIでのformat変換時に便利)
```
    const key = "name";
    const { user } = namedInputs;
    return user[key];
```

- `property_filter`：高度なプロパティ抽出を行う(やや複雑なため詳細はソースコード参照)

---

### 計算系

- `total`：多次元配列内のオブジェクトを走査し、キーごとに数値を合計する
```
    {
      inputs: { array: [[{ a: 1, b: -1 }, { c: 10 }], [{ a: 2, b: -1 }], [{ a: 3, b: -2 }, { d: -10 }]] },
      params: {},
      result: { data: { a: 6, b: -4, c: 10, d: -10 } },
    },
```
- `data_sum`：数値配列を合計して1つの数値として返す
```
    {
      inputs: { array: [1, 2, 3] },
      params: {},
      result: { result: 6 },
    },
```


### String関連

多くの処理は `prop function` や `template記法` に置き換えられつつありますが、以下のAgentも提供されています：

- `json_parser`：文字列をJSONとしてパース(prop functionの代替)
- `string_case_variants`：文字列を `camelCase`, `snake_case`, `kebab-case` などに変換
- `string_splitter`：指定サイズで文字列を分割(例：RAGのチャンク処理)
- `string_template`：テンプレート記法による文字列生成(現在はtemplate記法推奨)
- `string_update_text`：`oldText` または `newText` のいずれかを返す(null合体的な用途)
    - return oldText ?? newText;

これらのAgentを活用することで、より柔軟で制御可能なデータ処理がGraphAI上で実現できます。



## nested graph
  - isResult
  - このへんもコピペ
### nestedGraphの暗黙の入力
     - inputs -> static node 
### mapAgentの暗黙の入力
     - rows -> row, rowKey
     - inputs -> static node 
### mapAgentの結果
     - normal
       - [{}], [{}]
     - compositeResult: true
       - {node1: [], node2: []}

### ２つを使った例
```
  passThrough: {
    lang: ":lang",
  },
  output: {
    text: ".text",
    lang: ".lang",
  },
```                  

### defaultValue( for if/unless )
     - if/unlessでデータにマッチしなかった場合に結果渡す
     


### GUIツール向けのテクニック
  - GUIは入力を表現するのが難しい
    - 特にArray
    - １つの入力の２つ以上のEdgeを許可すると、途端の難しくなる
      - array -> array
      - item, item -> array
      - array, item -> array (NG)
      - UI上の制限や、判定をが必要になる
    - array -> arrayとitem -> itemだけに制約
    - item, item -> array の変換は変換用のAgent。入力の数は常識的な有限個
      - この変換にcopyAgentが使える
  
- GUIツールを参考にさらなるデータの標準化





