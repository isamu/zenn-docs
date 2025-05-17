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

`GOD format` や `output` / `passThrough` では対応しきれないケースでは、**データ変換専用のAgent**を使います。

これらのAgentは通常の LLM Agent と同様に、Nodeとして定義して使います。以下に、Vanilla Agentで提供されている主なデータ変換用Agentを紹介します。

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


### Object/Dictonary/Record関連

#### mergeObjectAgent

複数のオブジェクトをマージして1つにまとめる
inputs
```json
[{ color: "red" }, { model: "Model 3" }]
```
result
```json
{ color: "red", model: "Model 3" }
```
#### lookupDictionaryAgent

paramsに定義した辞書から、inputsで指定されたキーの値を取り出す
JSでの実行イメージ
```typescript
    const data = { ja: "こんにちは", en: "hello" };
    return data[index];
```    

#### copyAgent

inputsのオブジェクトからparamsで指定したキーの値を抽出する
keyを指定しない場合、オブジェクト全体を返す(GUIでのformat変換時に便利)

JSでの実行イメージ
```typescript
    const namedKey = "name";
    const { namedKey } = namedInputs;
    return user[namedKey];
```

#### property_filter

高度なプロパティ抽出を行う(やや複雑なため詳細はソースコード参照)

---

### 計算系

#### totalAgent
多次元配列内のオブジェクトを走査し、キーごとに数値を合計する
```
    {
      inputs: { array: [[{ a: 1, b: -1 }, { c: 10 }], [{ a: 2, b: -1 }], [{ a: 3, b: -2 }, { d: -10 }]] },
      params: {},
      result: { data: { a: 6, b: -4, c: 10, d: -10 } },
    },
```

#### data_sum

数値配列を合計して1つの数値として返す
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


---

## 入れ子構造のAgent（NestedAgent / MapAgent）の入力と出力

ここまでが、一般的な入力・出力の扱いについての解説でした。  
ここからは、**GraphAIを入れ子で実行する特殊なAgent**である `nestedAgent` と `mapAgent` について説明します。


### 概要

`nestedAgent` や `mapAgent` は、内部にサブグラフ（GraphData）を持ち、その中で別のAgentネットワークを実行することで、複雑な処理や並列で同時に同じ処理を可能にします。

- **NestedAgent** は **単純にサブグラフを入れ子にするだけ** の構造。親グラフ内で定義されたワークフローを、特定の入力データとともにそのまま実行する。
- **MapAgent** は **データが異なるが同じ処理を並列に動かす仕組み**。配列データを展開し、それぞれの要素に対して同じサブグラフを実行する（MapReduce 的な処理）。


### 入力の扱い

これらのAgentでは、`inputs` に定義したデータは、**内部のサブグラフに対して暗黙的にStatic Nodeとして渡されます**。
そのため、内部グラフでは `inputs` という名前のノードを明示的に設置しなくても、自動的に外部の `inputs` が反映される仕組みになっています。

### 出力の扱い

出力は、`isResult: true` が指定されたサブグラフのNodeの結果が最終的な出力となり、**親グラフに対してネストされた形で返されます**。
これにより、`nestedAgent` や `mapAgent` の結果も他の通常のNodeと同様に扱うことができます。

この構造を理解することで、GraphAIにおける**再利用性の高いサブグラフ設計**や**動的な並列処理**が可能になります。

### nestedGraphの暗黙の入力

- `inputs` を **サブグラフの static node にマッピング** する。
- 例えば `{dataA: 1, dataB: 2}` を渡すと、そのままサブグラフに適用される。

```yaml
version: 0.5
nodes:
  nested:
    isResult: true
    inputs:
      dataA: 1
      dataB: 2
    agent: nestedAgent
    graph:
      version: 0.5
      nodes:
        copy:
          isResult: true
          agent: copyAgent
          inputs:
            dataA: :dataA
            dataB: :dataB
```

このケースではnestedAgent実行時にサブグラフに

```
dataA: {
 value: 1
}
dataB: {
 value: 2
}
```
のstatic nodeが追加されます。その結果

```json
{
  "nested": {
    "copy": {
      "dataA": 1,
      "dataB": 2
    }
  }
}
```

となります。

### nestedAgentの結果

サブグラフの`isResult: true`で指定した結果が、そのagentの結果としてそのまま渡されます。

Graph Data
```yaml
version: 0.5
nodes:
  nested:
    isResult: true
    inputs:
      test: [a,b]
      test2: z
    agent: nestedAgent
    graph:
      version: 0.5
      nodes:
        copy:
          isResult: true
          agent: copyAgent
          inputs:
            test: :test
        copy2:
          isResult: true
          agent: copyAgent
          inputs:
            test2: :test2
        copy3:
          agent: copyAgent
          inputs:
            test2: :test2
```

結果
```json
{
  "nested": {
    "copy": [
      "a",
      "b"
    ],
    "copy2": "z"
  }
}
```


### MapAgenthの暗黙の入力

- `rows` に **必ず配列を指定** する必要がある。
- 配列の各要素が `row` としてサブグラフに渡され、**配列の長さだけサブグラフが生成** される（MapReduce 的な処理）。
- `rows` 以外の `inputs` は、そのままサブグラフに渡される。
- `rows` 以外の `inputs` に **配列を指定すると、そのままコピーされ、すべてのサブグラフに渡る**。なので、rowsだけが配列を特殊に扱います。


```yaml
version: 0.5
nodes:
  map:
    isResult: true
    inputs:
      rows: [1,2]
      test: [a,b]
      test2: z
    agent: mapAgent
    graph:
      version: 0.5
      nodes:
        copy:
          isResult: true
          agent: copyAgent
          inputs:
            data: :row
            test: :test
            test2: :test2
```

２つの処理が並列で実行され、それぞれのGraphDataには以下のデータがStatic Nodeとしてinjectされます。
```
{
  data: 1
  test: [a,b]
  test2: z
}
```

```
{
  data: 2
  test: [a,b]
  test2: z
}
```


### 3. サブグラフの結果処理
#### MapAgent の通常の出力
- サブグラフの結果は **配列として返される**。
- そのため、サブグラフ内の特定のノードの結果だけを取得するのは少し不便。

```json
{
  "map": [
    {
      "copy": {
        "data": 1,
        "test": [
          "a",
          "b"
        ],
        "test2": "z"
      }
    },
    {
      "copy": {
        "data": 2,
        "test": [
          "a",
          "b"
        ],
        "test2": "z"
      }
    }
  ]
}
```


#### `compositeResult: true` の利用
- **ノードごとの結果を配列にまとめてくれる**。
- これにより、次の Agent が **特定のノードの結果だけ** を取得しやすくなる。

```yaml
    params:
      compositeResult: true
```

```json
{
  "map": {
    "copy": [
      {
        "data": 1,
        "test": [
          "a",
          "b"
        ],
        "test2": "z"
      },
      {
        "data": 2,
        "test": [
          "a",
          "b"
        ],
        "test2": "z"
      }
    ],
    "copy2": [
      {
        "data": 1,
        "test": [
          "a",
          "b"
        ],
        "test2": "z"
      },
      {
        "data": 2,
        "test": [
          "a",
          "b"
        ],
        "test2": "z"
      }
    ]
  }
}
```

# Appendix

### defaultValue

少し特殊な例でdefaultValueという機能があります。
これは、if/unlessでnodeを制御するときに、条件にマッチしなかった場合の返す値を指定します。
これを使わないと、if/unlessでデータフローをコントローするときに、if/unlessのそれぞれのnodeを作成し、それらのnodeからの入力とanyInputを指定したnodeが必要となります。defaultValueを使うとそのような複雑な構成は不要となります。

TODO: anyinput とdefaultValueを使った例

### GUIツール向けのテクニック
  - GUIは入力を表現するのが難しい
    - 特にArray
    - １つの入力の２つ以上のEdgeを許可すると、途端の難しくなる
      - array -> array
      - item, item -> array
      - array, item -> array (NG)
      - UI上の制限や、判定をが必要になる
    - array -> arrayとitem -> itemだけに制約
  - このような問題を回避するためにGrapysでは、１つの入力には１つのEdgeしか接続できないようにしている。
  - そのため複雑な入力は、直接GUI上では要しない。
  - inputsのobjectの整形もGUIからagentにデータを渡す段階で、内部に変換コードを持たせている
  - item, item -> array の変換は変換用のAgent。入力の数は常識的な有限個
    - この変換にcopyAgentが使える
  - 詳しくはGrapysのコードを参照
  
- これらのGUIツールのテクニックを活用すると、通常のGraphDataでも汎用的な作りができる





