
---

データの変換方法まとめ！！

GraphAIの一番のポイントは入力、及び出力を制覇することです。
これを理解し、自由に使いこなせれば、どんな複雑な処理も可能です。


## inputの基本

### GOD Formatで入力（前のノードの出力）を整形する
   TODO: コピペ
### Props funciton
   TODO: コピペ
### utility function
   TODO: 最近追加したので書く

## 入出力の標準化
   説明をかいてGitHub
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

## 結果の変形

### output
    - agentの結果を整形
### passThrough
    - agentの動作に関係なく結果にデータを渡す
      - leteral or god format
      
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
     

## データを変換するagent

### array
  - pop / shift / push / flat / join 
  - array to object(arrayのある要素をkeyにしたobject)
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

### object
  - mergeObject
```
[{ color: "red" }, { model: "Model 3" }]  -> { color: "red", model: "Model 3" }
```
  - lookupDictionaryAgent
    paramsの固定値を、inputsで制御する
```
    const data = { ja: "こんにちは", en: "hello" };
    return data[index];
```    
    paramsのobject固定。keyが可変  
  - copyAgent
    inputsのobjectのなにかデータを引っ張る
    inputsのobject可変。keyはparamで固定値
    keyを指定しない場合はそのまま返す(GUIでのデータ変換でinputsのformatをそのまま返す、という利点を使って利用する)
```
    const key = "name";
    const { user } = namedInputs;
    return user[key];
```
  - property_filter
    - 難しいけどなんか色々できる
    - ソース見ればわかるけど、使いこなせなかった。。。  
###  計算
  - total
```
    {
      inputs: { array: [[{ a: 1, b: -1 }, { c: 10 }], [{ a: 2, b: -1 }], [{ a: 3, b: -2 }, { d: -10 }]] },
      params: {},
      result: { data: { a: 6, b: -4, c: 10, d: -10 } },
    },
```
  - data_sum
```
    {
      inputs: { array: [1, 2, 3] },
      params: {},
      result: { result: 6 },
    },
```

### string
  - ほとんどがprop functionとtemplate記法に移行
  - json_parser
    - prop functionがあるので不要
    - GUIなどでprop functionが使いづらい時に使う
  - string_case_variants
    - lowerCamelCase, snakeCase, kebabCase, normalized
  - string_splitter
    - chunk sizeで区切る。(RAG用？)
  - string_template
    - テンプレート記法ができたの不要
  - string_update_text
    - return oldText ?? newText;

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


# 基本的な入力について

graphai-inputs の記事参照
GODが基本。nodeとproppropetyを指定する（入力として別のnodeのnode名+その結果のデータを指定)

object, array

:node.propeties1.$0.propeties2 (
は、
{
 node: { propeties1: [{propeties2: "value"}]}
}

prop function
データに対して関数を実行

utility function
日付などの



