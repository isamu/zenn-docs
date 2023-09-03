---
title: "SlashGPTのマニュフェストファイルの解説"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [gpt, ChatGPT, LLM, Tech, OpenAI]
published: true
publication_name: "singularity"
---

[SlashGPT](https://github.com/snakajima/SlashGPT/)は、No CodeでLLMエージェント/LLMアプリを作成、利用できるツールです。
マニュフェストファイルと呼ばれるJSONファイルを記述するだけでLLMエージェント/LLMアプリを作成できます。

詳細は[README](https://github.com/snakajima/SlashGPT#manifest-files)にも記載されていますが、それを分野ごとに分類し、ファイルを構成する内容を解説します。

## マニュフェストファイルについて


#### LLMモデル関連

LLMエージェントで利用するLLMモデルを指定

- model (string, optional): LLM model (such as "gpt-4-613", the default is "gpt-3-turbo")
  - そのLLMエージェントで使うモデルを指定する
- temperature (string, optional): Temperature (the default is 0.7)
  - モデルに渡すtemperature

#### CLI表示関連

SlashGPTのCLI動作時に、CLIの出力として使われる設定（LLMエージェントの動作に影響なし）

- title (string, required): Title for the user to see
  - Agent起動時（切替時）に表示される
```
Activating: Main Dispatcher
```

- bot (string, optional): Agent name
   - LLMエージェントからの応答時に表示される名前
```
botname: Hello, I am bot.
```

- you (string, optional): User name. The default is You({agent_name}).
  - 入力のプロンプトや、functionの動作後、再度ユーザの入力を問い合わせる時などに表示される
```
You(simple)
```

#### Sample

LLMの動作検証やユーザへの例として、サンプルクエリ（LLMエージェントの動作に影響なし）

- sample (string, optional): Sample question (type "/sample" to send it)
  - 事前の用意するpromptへの問い合わせの内容を記述する
  - prefixがsampleで始まるkeyは、全てsampleとして実行可能
    - sample_1に記述すれば /sample_1 で利用可能

#### CLI表示 ＋ システムプロンプト

- intro (array of strings, optional): Introduction statements (will be randomly selected)
   - Agent起動時（切替時）に表示され、promptにも渡されるシステムメッセージ

#### システムプロンプト

システムメッセージとしてLLMエージェントに渡すシステムプロンプト

- prompt (array of strings, required): The system prompts which define the agent (required)
  - LLMに渡すシステムプロンプトの文
    - 以下のtemplate記法は、置換される
      - {now}
        - "%Y%m%dT%H%M%SZ" 形式の日付
      - {resource}
        - resource要素のファイルの中身に差し替えられる
        - resource (string, optional): location of the resource file. Use {resource} to paste it into the prompt
      - {random}
        - list内のデータをランダムに置換する
        - list (array of string, optional): {random} will put one of them randamly into the prompt


#### ユーザの入力の変換

ユーザの入力をLLMエージェントにわたすときに、ユーザのメッセージ/クエリーを加工したり、funcitonsの設定をする
  
- form (string): format string to extend user's query (e.g. "Write python code to {question}").
   - LLMに渡すメッセージ。{question}部分がユーザの入力となるため、入力をラップして渡すことができる
- functions (string, optional): location of the function definitions
   - LLMに渡すfunctionsの設定

#### funcitonの返却値関連

LLMエージェントからfunctionの結果が戻ってきた場合の動作を指定する。

動作は大きく分けて２つ。
actionが指定されている場合は、actionに定義されているapiアクセスか、templateによる表示がされる
notebook, moduleが指定されている場合はPythonのコードが実行される

- actions (object, optional): Template-based function processor (see details below)
  - actionsのkeyとfunctionの???がマッチしたものが実行される
    - typeにより動作を指定する
      - rest
      - graphQL
      - data_url
      - message_template

- notebook (boolean): create a new notebook at the beginning of each session (for jupyter2)
   - notebookでPythonを実行する 
- module (string, optional): location of the Python script to be loaded for function calls
  - 指定されているPythonコードのファイルを実行する

- result_form (string): format string to extend function call result.
  - notebook, moduleの実行結果をformatする
 
#### その他

- skip_function_result (boolean): skip the chat completion right after the function call.
  - 通常はfunctionの戻り値はそのままLLMエージェントにわたすがその動作をSKIPする

### embeddings

embeddings関連の設定

- embeddings (object, optional):
  - name (string, optional): index name of the embedding vector database

#### 機能として未利用

- about (string, optional): About the manifest (URL, email, github id, or twitter id)
  - マニフェストファイルなのか説明しています。


## まとめ

マニュフェストを構成する要素を説明しました。

実際のマニュフェストファイルは[こちら](https://github.com/snakajima/SlashGPT/tree/main/manifests/main)にあるので上記の説明を参考にこれらのファイルを参照してください。

