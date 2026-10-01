---
title: "SlashGPTのFunction callingの詳細"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ChatGPT, LLM, Tech, OpenAI, SlashGPT]
published: true
publication_name: "singularity"
---


[SlashGPT](https://github.com/snakajima/SlashGPT/)は[中島聡](https://twitter.com/snakajima)が開発したChatGPTなどのLLMエージェントを手軽に開発するためのツールです。SlashGPTを使えば、jsonファイルを記述するだけでChatGPTを使ったLLMエージェントやチャットアプリを手軽に、簡単につくることができます。

OpenAIから2023年6月に発表されたFunction callingにも対応し、プログラムを一切書くとことなく、Function callingとAPIサービスを連携させたLLMエージェントを作成することが可能です。
SlashGPTはFunction callingの返却値を使い、さまざまな動作をNoCodeで使うことができます。


SlashGPTでのFunction callingの大まかな動作の流れを説明すると、

- LLMに問い合わせをするときに、マニュフェストのfunctionに定義されている関数の情報を投げる
- LLMへの問い合わせに対して戻り値にfunction_callが含まれると、マニュフェストの情報に従って処理をする

となります。

マニュフェストでは、戻ってきたfunction_callに対して
- actions
- notebook
- modules
の３つの方法の処理を定義できます。actionsは更にいくつかのタイプの定義があります。

## actions

actionsは、SlashGPTで定義しているいくつかのタイプのactionを実行します。
大きく3種類。
- rest/graphQLのapiを呼び出す(rest/graphQL)
- templateを処理して文字列を返す(data_url, message_template)
- emitと呼ばれる、sessionの外側での処理をする（SlashGPTではdispatcherでのagentの切り替えで使っています）

actinsはobjectであり、keyが関数名、valueがその関数の動作を定義しています。
valueにはtypeが定義され、typeとして指定できるのはrest, graphQL, data_url, message_template, emitです。typeごとの動作を見ていきます。

#### rest

actionに定義されているurlとmethodのrest apiを呼びます。optionでheadersも追加可能です。headerに認証情報などを追加できます
bodyやqueryにはfunction callingから返ってきたargumentsを渡します。

具体的に[currency](https://github.com/snakajima/SlashGPT/blob/main/manifests/main/currency.json)のマニュフェストを見てきましょう。まずはfunctionsの定義です。
funcitonsの定義はcurrencyのようにマニュフェストに直接書くこともできますし、ファイル名を書いて別ファイルから読み込むことも可能です。

```
  "functions": [
    {
      "name": "convert",
      "description": "Convert one currency to another",
      "parameters": {
        "type": "object",
        "properties": {
          "from": {
            "type": "string",
            "description": "The currency to convert from"
          },
          "to": {
            "type": "string",
            "description": "The currency to convert to"
          },
          "amount": {
            "type": "string",
            "description": "Amount to convert"
          }
        },
        "required": ["from", "to", "amount"]
      }
    }
  ]
```
convertという関数が定義され、form, to, ammountという変数があり、それらが必須となっています。

実際に動かします。
```
/sample
```
を動かすと、LLMからはfunctionsの指示に従い

```
{
  "name": "convert",
  "arguments": "{\n  \"from\": \"USD\",\n  \"to\": \"JPY\",\n  \"amount\": \"1\"\n}"
}
```
とFunction callが返ってきます。マニュフェストのactionsでは同じくconvertという関数が定義されています。

```
  "actions": {
    "convert": {
      "type": "rest",
      "url": "https://today-currency-converter.oiconma.repl.co/currency-converter?from={from}&to={to}&amount={amount}"
    }
  },
```

type=restが指定されているので、restとして動作します。
urlのquery部分(from, to, amount)がFunction callで戻ってきたargumentsにより置換され、(methodの指定がないので）GETで呼ばれます。

このrest apiを呼ぶと

```
{"date":"2023-09-04","historical":false,"info":{"rate":146.04441},"motd":{"msg":"If you or your company use this project or like what we doing, please consider backing us so we can continue maintaining and evolving this project.","url":"https://exchangerate.host/#/donate"},"query":{"amount":1,"from":"USD","to":"JPY"},"result":146.04441,"success":true}
```
のような為替の結果が返ってきます。この結果を再度LLMにわたすと

```
GPT: 1 USD is equivalent to 146.04441 JPY.
```
となります。


#### graphQL

type=graphQLの場合は、actionに定義されているurlをgraphQLとして呼びます。
argumentsをそのまま渡すので、Function CallingでGraphQLに対応したデータを取得する必要があります。

graphQLを使っている[spacex](https://github.com/snakajima/SlashGPT/blob/7ddb7b2b7c2421bb06387b73304d424e0efd1f69/manifests/main/spacex.json)のactionsの定義。
```
  "actions": {
    "call_graphQL": {
      "type": "graphQL",
      "url": "https://spacex-production.up.railway.app/graphql"
    }
  },
```

具体的な動作は[こちら](https://zenn.dev/singularity/articles/slashgpt_spacex#実際の動作)にあります。

#### data_url

actionに定義されているtemplateファイルを読み込み、actionのmime_type, mesageとargumentsを置換してdate_url形式のデータを返します。

[cal(カレンダー)のマニュフェスト](https://github.com/snakajima/SlashGPT/blob/main/manifests/main/cal.json)で使われています。
functionsは少し大きいので全て掲載することは割愛しますが、make_eventと、send_invitationの関数が定義されています。[詳細はこちら](https://github.com/snakajima/SlashGPT/blob/c8edb81c1a894c16bfb0241390b414918d4beac9/resources/calendar.json)

calは、プロンプトがユーザの予定を聞いて、必要な情報が揃ったらiCal用のdata_urlを作成してメールを送るようなLLMアプリです。

actionsの定義は以下。

```
  "actions": {
    "make_event": {
      "type": "data_url",
      "template": "./resources/action_templates/calendar.ics",
      "mime_type": "text/calendar",
      "message": "{{ 'result':'Success', 'invitation_link': '{url}' }}"
    },
    "send_invitation": {
       "type": "message_template",
       "message":"Success. The invitation was sent to {recipients}"
     }
  },
```

make_eventとsend_invitationはそれぞれ異なるタイプが指定されています。
actionsはこのように関数ごとに異なる動作を指定することも可能です。
message_templateについては次のセクションで説明します。

/sampleを動かすと、LLMからは

```
{
  "name": "make_event",
  "arguments": "{\n  \"SUMMARY\": \"Meeting with Tim Cook\",\n  \"DESCRIPTION\": \"Discussing new product launch\",\n  \"DTSTART\": \"20230704T20:00:00Z\",\n  \"DTEND\": \"20230704T20:30:00Z\",\n  \"LOCATION\": \"Apple Headquarters\"\n}"
}
```

が返ってきます。
actionsのmake_eventの定義に従い [./resources/action_templates/calendar.ics](https://github.com/snakajima/SlashGPT/blob/7ddb7b2b7c2421bb06387b73304d424e0efd1f69/resources/calendar.ics)のファイルで開いて置換、されに置換された文字列はマニュフェストのmessageのurl部分に置き換えれて、最終的に

```
{ 'result':'Success', 'invitation_link': 'data:text/calendar;charset=utf-8,BEGIN%3AVCALENDAR%0AVERSION%3A2.0%0APRODID%3A-%2F%2FMy+Calendar%2F%2FNONSGML+v1.0%2F%2FEN%0ABEGIN%3AVEVENT%0ADTSTART%3A20230704T20%3A00%3A00Z%0ADTEND%3A20230704T20%3A30%3A00Z%0ASUMMARY%3AMeeting+with+Tim+Cook%0ADESCRIPTION%3ADiscussing+new+product+launch%0ALOCATION%3AApple+Headquarters%0AEND%3AVEVENT%0AEND%3AVCALENDAR' }
```
という結果が返ってきます。

さらにこのマニュフェストはmessage_templateの処理を続けます。


#### message_template

actionに定義されているmessageの内容をargumentsで置換したデータを返します。

calでの処理の続きを見ると、
```
    "send_invitation": {
       "type": "message_template",
       "message":"Success. The invitation was sent to {recipients}"
     }
```
となっていて、date_typeの結果をLLMになげると

```
{
  "name": "send_invitation",
  "arguments": "{\n  \"invitation_link\": \"data:text/calendar;charset=utf-8,BEGIN%3AVCALENDAR%0AVERSION%3A2.0%0APRODID%3A-%2F%2FMy+Calendar%2F%2FNONSGML+v1.0%2F%2FEN%0ABEGIN%3AVEVENT%0ADTSTART%3A20230704T20%3A00%3A00Z%0ADTEND%3A20230704T20%3A30%3A00Z%0ASUMMARY%3AMeeting+with+Tim+Cook%0ADESCRIPTION%3ADiscussing+new+product+launch%0ALOCATION%3AApple+Headquarters%0AEND%3AVEVENT%0AEND%3AVCALENDAR\",\n  \"recipients\": [\"tim@apple.com\"]\n}"
}
```
が返ってきます。
actionのtypeに従いmessage_templateとして結果が実行されます。
実際にはmessageのrecipientsが置換され

```
 Success. The invitation was sent to ['tim@apple.com']
```

というメッセージになり、最終的にそれをうけてLLMが

```
iCal: I have scheduled a meeting with Tim Cook on July 4th at 1pm Pacific Time for 30 minutes. The meeting will take place at his office. I have sent the invitation to Tim Cook at tim@apple.com.
```
という送信結果を返します。


## notebook

マニュフェストのnotebookがtrueの場合、argumentsにあるPythonコードをnotebookで実行されます。code interpreterのような動作ですね。
この機能を使う場合は、funciton callingから返ってくるargumentsなどはSlashGPTのSPECに合わせる必要があるので[jupyter](https://github.com/snakajima/SlashGPT/blob/7ddb7b2b7c2421bb06387b73304d424e0efd1f69/manifests/main/jupyter.json)のマニュフェストを参考にfuncitonsを指定する必要があります。

```
"functions": "./resources/functions/jupyter.json"
```
functionの中身は[こちら](https://github.com/snakajima/SlashGPT/blob/7ddb7b2b7c2421bb06387b73304d424e0efd1f69/resources/jupyter.json)。
run_python_code関数のcodeでpythonのコードを渡す必要があります。codeで渡されたものが実行されます。


では、jupyterの/sampleを動かします。
データを読み込む命令をします。

```
Load './data/Electric_Vehicle_Population_Data.csv'
```

すると、LLMからはfunction callとしてpythonコードが返ってきます。run_python_codeはそのコードを実行する関数で、codeの実際のコードが入っています。
```
{
  "name": "run_python_code",
  "arguments": "{\n  \"code\": [\n    \"import pandas as pd\\n\",\n    \"df = pd.read_csv('./data/Electric_Vehicle_Population_Data.csv')\\n\",\n    \"df.head()\"\n  ],\n  \"query\": \"Load './data/Electric_Vehicle_Population_Data.csv'\"\n}"
}
```
この関数をnotebookで実行します。詳しい動作は、[jupyter_runtime.py](https://github.com/snakajima/SlashGPT/blob/7ddb7b2b7c2421bb06387b73304d424e0efd1f69/lib/jupyter_runtime.py#L68C9-L68C9)で確認をしてください。

実行後、

```
function(run_python_code):    VIN (1-10)  ... 2020 Census Tract
0  1N4AZ0CP5D  ...      5.303508e+10
1  1N4AZ1CP8K  ...      5.303509e+10
2  5YJXCAE28L  ...      5.303301e+10
3  SADHC2S1XK  ...      5.306701e+10
4  JN1AZ0CP9B  ...      5.306104e+10

[5 rows x 17 columns]
```

が返ってきます。



## module

moduleを指定するとそのファイルに書かれたpythonコードの関数が実行されます。
今まで紹介したaction, notebookとは異なり、自分で書いたプログラムを呼び出すことができます。

browserのマニュフェストを見ます。

```
  "functions": "./resources/functions/browser.json",
  "module": "./resources/module/browser.py",
```
と、funcitonsとmoduleのファイル名が定義されています。

functionsは
```
[
  {
      "name": "fetch",
      "description": "Fetch the content of web page",
      "parameters": {
          "type": "object",
          "properties": {
              "url": {
                  "type": "string",
                  "description": "url of the web page"
              }
          },
          "required": ["url"]
      }
  }
]
```
と、fetch関数にurlを渡す関数が定義されています。
browser.pyを見ると

```
def fetch(url: str):
    response = requests.get(url)

    # Check if the request was successful
    if response.status_code == 200:
    ~~~~
```
と、渡されたurlを開いて、データを取得するfetch関数が書かれています。

sampleを動かすと

```
Please write a summary of the web paga at https://www.bbc.com/news/live/world-europe-66049895
```
と、LLMにwebページの要約をお願いする

```
{
  "name": "fetch",
  "arguments": "{\n  \"url\": \"https://www.bbc.com/news/live/world-europe-66049895\"\n}"
}
```
LLMからはfetch関数にurlを渡す関数が返ってきます。

SlashGPTではmoduleの情報を元にbrowser.pyのコードが実行され
```
function(fetch): {"result": "success", "text": "\nAnd we'll keep you signed in.\nSign in  or  Register\nYou need one to watch ....}
```
と、webページのhtmlのデータが戻ってきます。これをGPTが要約し、

```
GPT: The web page at https://www.bbc.com/news/live/world-europe-66049895 provides live updates on the unrest and riots happening in France. The page includes information about the number of arrests, the deployment of police officers, calls for calm from public figures, and incidents of violence and looting in various cities. It also mentions the shooting of a 17-year-old boy by police, which sparked the protests. The page is regularly updated with the latest developments.
```

となりました。


# まとめ

SlashGPTのFunction callingの動作を説明しました。
マニュフェストでシンプルな設定をするだけで、Function callingを使った様々な動作をすることができます。


### 関連記事

- [SlashGPTの使い方](https://zenn.dev/singularity/articles/slashgpt_tutorial_1)
- [SlashGPTのマニュフェストファイルの解説](https://zenn.dev/singularity/articles/slashgpt_tutorial_2)
- [SlashGPTのFunction callingの動作原理](https://zenn.dev/singularity/articles/slashgpt_spacex)
- [SlashGPTのプラグインを使って新しいLLMを追加する](https://zenn.dev/singularity/articles/slashgpt_llm_engine)
- [SlashGPTを使ってチョムスキーさんと田原さんを討論させる](https://zenn.dev/singularity/articles/slashgpt_agents)

- [SlashGPTで、No-CodeでAIエージェントを作る方法](https://zenn.dev/snakajima/articles/adf436f7f794b3) by 中島さん
- [SlashGPT + Jupyter](https://zenn.dev/singularity/articles/718cba24bf1275) by kozayupapaさん
