
GraphAIのGraphDataやAgentを設計する上での設計原則です。

# Agentへの入力(inputs)と結果(result)は標準型にあわせる

Agentは任意の形式データを渡して、任意の形式のデータを返すことができますが、GraphAIで定義している標準の形式に合わせることで、他のユーザの利便性や、GraphDataをAIで作成するときに、扱いが容易になります。

https://github.com/receptron/graphai/blob/main/docs/inputs.md

で定義されています。

配列のデータ
```
{
  array: [], // ベースとなるデータ
  item: "a", // それを操作するデータ(1つの場合)
  items: [], // それを操作するデータ(2つ以上の場合)
}
```

文字列のテキスト
```
{
  text: "123",
}
```

LLMのデータ
```
{
  message: {
    role: "user",
    content: "123"
  }
}
{
  messages: [{
    role: "user",
    content: "123"
  }]
}
```

他はgithubのdocumentを参照してください。
追加したい形式があればPRを送ってください。


# if/unless/anyInput/defaultValueの分岐する処理はnestedAgentGeneratorで隠蔽したAgentを作る(使う）って、極力GraphDataにはかかない


if/unless/anyInputを使うのは少し難しいので。

# 設定値

Agentへの入力は、inputs/params/configがあります。
inputsは他のagentから渡されるデータ、paramsはそのagent固有の設定、configはGraphDataを実行するインスタンス共通の設定です。
configはglobalな設定（全agentに渡される）と、agentごと渡す設定の２つがあります。
それ以外に、サーバ環境で動かすAgentは環境変数で設定値を渡すことが出来ます。
最近のクラウド環境では、secretを環境変数以外で渡す方法もあると思います。

Agentをサーバ/Webの両方で動くように考慮する
GraphDataもサーバ/Webで変更なく動くように意識する

共通化できるように。



design principles
  throwErrorやllm/db設定など、agent全体共通のものは、paramsだけでなくconfigで設定したい。
  config/paramsで同じ設定を受け付けて、config(共通）をparams(個別）で上書きできる、という方針にする。


