---
title: "AI Agentを簡単に効率よく開発/実行するGraphAIの紹介"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

# GraphAIとは

GraphAIとは、TypeScriptで書かれた単機能のAgentと呼ばれるプログラムを、YAMLやJSONファイルに書かれたデータフローグラフの法則に従って順次非同期に実行するプログラムのエンジンです。

LLMやRAGを組み合わせたアプリケーションを作る場合に、組み合わせの数が増えていくと処理の制御が複雑になっていきます。1つの目的のアプリケーションを作る場合はそれに合わせてプログラムを書けば複雑なアプリケーションを作ることも可能ですが、自由に組み合わせて使う場合では、順番の制御などが複雑になっていきます。非同期で効率よく動作するものを作るのはとても困難になります。

GraphAIではエージェントは極力小さく作ることにより、各エージェントはシンプルに保ち、それらの入出力を組み合わせや実行順の制御をGraphAIに任せることより、簡単なプログラムの組み合わせで複雑なアプリケーションを作ることを可能にします。

組み合わせ自体は、YAMLやJSONファイルのデータフローグラフなのでエンジニア以外でも比較的容易にメンテナンスすることが可能です。


## 例

朝起きてから家を出るまでのタスクをプログラムで各例を考えます。左の()内がタスクのid. 右の()は、依存するタスクidとそれぞれの実行にかかる時間。

#### 起きたらすぐするタスク

- (1) コーヒーのお湯をわかす (４分)
- (2) 子供を起こす
  - 3分おきの5回呼ぶと起きる。ただし、3分以内の呼んだり、5分以上間隔が開くと、最初からやり直し
- (3) 朝食をつくる (15分)
- (4) いつも遅れる新聞をとりにいく
  - 5分以上間隔をあけて４回見に行くと、新聞はきている。ただし5分未満で見に行くと 最初からやりなおし

#### 依存タスク

- (5) お湯が湧いたらコーヒーをいれる(1)	(5分)
- (6) 子供が起きたら着替える(2)	(6分)
- (7) 子供が着替えおわったら洗濯機をまわす(6) (13分)
- (8) 朝食ができる && コーヒーができる&& こども着替えおわるの条件を満たすと朝食を食べる(3)(5)(6) (９分)
- (9) 洗濯機終了後、洗濯を干す(7) (5分)
- (10) コーヒーができていて、洗濯が終わっていたらコーヒーを飲み終える(5)(9) (５分)
- (11) 朝食終了 && コーヒーを飲み終えると食器を洗う(10)(8) (４分)
- (12) 朝食終了後、子供が用意する(8) (３分)
- (13) 新聞到着していたら新聞が読む(4) (8分)
- (14) 13までのタスクが全て終わっていたら家出発(1~13)


すべてのタスクは非同期で並列にタスクができるとすると、効率よく14に到達するプログラムを書くことができますか？

GraphAIを使えば、このような複雑な依存のあるタスクも、簡単に制御することが可能となります。


## GraphAIの利用方法

GraphAIを使うには3つの方法があります。

- npmのパッケージでGraphAIをライブラリーとして使う
  - プログラムに組み込んで使う場合はこちらです。
```
yarn add graphai
```

- npmのCLIパッケージでCLIツールとして使う
  - YAMLやJSONでGraphDataを作成し、用意されたエージェントを利用します。
```
npm i -g  @receptron/graphai_cli
```

- GitHubのソースコードを使う
  -  サンプルで用意されているGraphAIを使ったプログラムを実行したり、GraphAI自体を開発する場合に利用します。
```
https://github.com/receptron/graphai.git
```

## CLIで使う

npmのcli用のパッケージ[graphai_cli](https://www.npmjs.com/package/@receptron/graphai_cli)をインストールします

インストールは

```
npm install -g @receptron/graphai_cli
```

です。nodeやnpmがインストールされてない場合は、最初にインストールしてください。

### ヘルプ

引数無しで実行すると、使い方の説明が表示されます。

```
$ graphai
```

```
graphai <yaml_or_json_file>

run GraphAI with GraphAI file.

Positionals:
      --yaml_or_json_file  yaml or json file                            [文字列]

オプション:
      --help     ヘルプを表示                                             [真偽]
      --version  バージョンを表示                                         [真偽]
  -l, --list     agents list
  -s, --sample   agents list                                            [文字列]
  -d, --detail   agent detail                                           [文字列]
  -v, --verbose  verbose log                   [真偽] [必須] [デフォルト: false]
      --log      output log                                             [文字列]
```

### 実行

yamlかjsonのファイル名を指定すると、そのファイルに書かれているData flow graphが実行されます。

```
$ graphai test_base.yml 
```

```
{
  node5: {
    node5: 'output',
    node2: 'output',
    node4: 'output',
    node3: 'output',
    node1: 'output'
  }
}

```


サンプルのyamlファイルは以下からダウンロードしてくだださい。

https://raw.githubusercontent.com/receptron/graphai_cli/main/test_yaml/test_base.yml


### Agentの一覧

GraphAIの標準の状態で使えるAgentの一覧を表示させます。
cliで使うData flow graphのyamlを作る場合は、この一覧にあるAgentをAgentIdに指定して使ってください。

```
$ graphai -l
```

```
Available Agents
* bypassAgent - bypass agent
* dataObjectMergeTemplateAgent - Merge object
* dataSumTemplateAgent - Returns the sum of input values
* dotProductAgent - dotProduct Agent
* echoAgent - Echo agent
* mapAgent - Map Agent
* nestedAgent - nested Agent
* popAgent - pop Agent
* pushAgent - push Agent
* shiftAgent - shift Agent
* slashGPTAgent - Slash GPT Agent
* sleeperAgent - sleeper Agent
* sleeperAgentDebug - sleeper debug Agent
* sortByValuesAgent - sortByValues Agent
* stringEmbeddingsAgent - Embeddings Agent
* stringSplitterAgent - This agent strip one long string into chunks using following parameters
* stringTemplateAgent - Template agent
* tokenBoundStringsAgent - token bound Agent
* totalAgent - Returns the sum of input values
```
## GraphDataをYAMLで書く

### echoAgentを使った簡単なGraphの作成

echoAgent(指定されたデータを出力するAgent)を使って、簡単なGraphファイルを作ります。
GraphAIのGraphファイルの必須項目はversionとnodesです。

versionは、0.3を指定します。(2024/05現在)

nodesは、GraphAIで使う各ノードを書いていきます。
このサンプルではnodesにechoAgentを使った１つのnodeを追加します。
echoAgentはparamsで指定しているユーザからの入力値をそのまま返すAgentです。

追加するnodeは、node1というNode名をつけます。
このnodeは`message: hello`というデータを出力します。

このサンプルのYAMLは１つしかnodeがありませんが、結果を返すnodeはnode1なので、`isResult: true`を追加し、このnodeの結果がこのGraphの結果と指定します。


```yaml
version: 0.2
nodes:
  node1: 
    params:
      message: hello
    agent: echoAgent
    isResult: true
   
```

これをecho.yamlというファイル名で保存して、graphaiのcliで実行します

```sh
$ graphai echo.yaml 
{ node1: { message: 'hello' } }
```

2行目の`message: hello`が表示されていれば成功です。

### inputsを追加する

次に複数のAgentを組み合わせ、inputsで入力を指定したYAMLを作ります。
入力を指定することで依存関係が定義でき、それによって実行順が制御されます。

bypassAgentは入力値をそのまま出力値で返すAgentです。
先程のechoAgentのyamlにnode2を追加します。node2のagentはbypassAgentを指定します。
入力のinputsとして、前のnode1を指定します。inputsは文字列の配列で、node名を指定します。

今回は出力はbypassAgentのnode2なので、node2に`isResult: true`を指定します。node1のisResultは削除します。

```yaml
version: 0.2
nodes:
  node1: 
    params:
      message: hello
    agent: echoAgent
  node2: 
    agent: bypassAgent
    inputs: [":node1"]
    isResult: true
```

実行される順に説明すると、
- node1のechoAgentが実行され、paramsの値を結果として返す
- node1を入力としているnode、node2が実行される。node2の入力値はnode1の実行結果で、それを入力としてうけとる。node2のbypassAgentは入力値をそのまま結果として返すagentなので、node1の結果をそのまま返す。

これを実行するとnode2の結果として`message: 'hello'`が表示されます。
また結果はarrayになっています。

```sh
 $ graphai echo2.yaml
{ node2: [ { message: 'hello' } ] }
```

同じように、今度は入力を増やして試してみます。
node1と同じechoAgentをnode2とし、bypassAgentをnode3にします。

入力のinputsは今度は` ["node1", "node2"]`と２つ指定します。

node3が`isResult: true`です。

```yaml
version: 0.2
nodes:
  node1: 
    params:
      message: hello
    agent: echoAgent
  node2: 
    params:
      message: こんにちは
    agent: echoAgent
  node3: 
    agent: bypassAgent
    inputs: ["node1", "node2"]
    isResult: true
```

```sh
$ graphai echo3.yaml
{ node3: [ { message: 'hello' }, { message: 'こんにちは' } ] }
```

node3の結果として、２つの入力値がそのまま出力されます

このようにinputsを使って、複数のエージェントをつなげていくことができます。

inputsを持つagentは、入力となるnode1, node2のagentの実行結果を待ってから実行します。
入力のagentがデータベースに接続したり、APIを叩くような時間のかかる処理の場合でも、その前の処理が終わるのを待ってから実行されます。

### nodeについて

nodeにはcomputed nodeとstatic nodeの２種類のnodeがあります。

computed nodeはagentを持つnodeで、agentに指定しているプログラムを実行します。

static nodeはデータを持つノードです。valueでデータを指定します。別ページで説明しますが、いくつかの方法で前のノードなどの値を受け取り、次のnodeにわたすことができます。プログラム(agent)は実行しないでデータの受け渡しをするnodeです。

### 用語

- Agent
  - TypeScriptで書かれたプログラム。特定のタスクを非同期に実行し、入力データを処理して出力データを生成する。たとえば、LLMへの問い合わせ、データベースクエリの実行、外部APIの呼び出し、テキストの処理、計算処理、機械学習モデルの利用など。GraphAIでは、各NodeがAgentを呼び出して実行する。

- GraphData
  - YAMLやJSONで書かれた、GraphAIで実行する内容を記述したデータ。NodeとそのNodeで呼び出すAgent、Node同士の繋がりを定義する。

- Node
  - GraphAIが実行する、Agentを実行する最小単位。
  - Computed Node
     - Agentと関連付けられており、特定のAgent関数を非同期に実行し、入力データを処理して出力データを生成する。この種類のノードは、入力データの依存関係が解決されると実行される。
  - Static Node
    - 初期値が指定された値のみを持ち、外部からの値の注入によって更新される。これは、外部プログラムによってノードの結果を手動で設定するために使用される。


## まとめ

以上でGraphAIの紹介と簡単な使い方の説明をしました。
GraphAIはコマンドラインで簡単に動かせるツールが用意されていて、yamlファイルを記述することで簡単に試すことができます。




## FAQ

### GraphAIはどのようなエンジンですか？

 GraphAIは非同期データフロー実行エンジンです。

### GraphAIの主な機能は何ですか？

非同期API呼び出しの管理、データ依存関係の管理、タスクの優先順位管理、マップリデュース処理、エラーハンドリング、リトライ、ロギング。

###  GraphAIを使用してどのようなアプリケーションを構築できますか？

エージェント的アプリケーションを構築できます。これは、宣言型データフローグラフで記述されたエージェントワークフローに基づくアプリケーションです。

### エージェント的アプリケーションとは何ですか？

複数の非同期API呼び出しを行い、それらの間でデータの依存関係を管理するアプリケーション。
###  GraphAIでデータフローを記述する際に使用する形式は何ですか？

YAMLまたはJSON形式で記述します。
### データフローグラフのコンポーネントは何ですか？

ノード（計算ノードと静的ノード）、データソース、エージェント、エージェント関数、インラインエージェント関数。
### GraphAIのデータフローグラフにおいて、ノード間の依存関係はどのように指定しますか？

ノードのinputsプロパティで他のノードを参照することで指定します。

### GraphAIで並列実行を制御する方法は？

グラフのconcurrencyプロパティで並列タスクの最大数を指定します。
