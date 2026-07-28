---
title: "GraphAI - チャットでブラウザを操作するデモ"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

toolsAgent


graphai.injectValue("tools", [...hogeToolsAgent.tools, ...fooToolsAgent.tools]);


llmのagentのように使う。
messageをベースにllmで処理され、llmの結果にtoolsが含まれいると、そのagentと、その関数を実行する
さらにllmの結果にhasNextが含まれると、toolsの実行結果をllmに投げてassistantのメッセージを作ってくれる
hasNextがない場合は、agentの実行結果を返すだけ。
```
llm: {
  isResult: true,
  agent: "toolsAgent",
  inputs: {
    llmAgent: ":llmAgent",
    tools: ":tools",
    messages: ":messages",
    userInput: {
      text: ":prompt",
      message: {
        role: "user",
        content: ":prompt",
      },
    },
  },
},
```

agentはargとfuncを受け取る。
resultをtextで返す。

--
問題点
llmから返答した値は渡せるが、それ以外の値が渡せない
 -> toolsAgentのinputsのバイパスするパラメータを追加する
 -> 全てのagentにそれらの値がたされる。
(argもできればフラットに展開して渡したい) 

返却値が制限されるのと、返却値をうまく次のワークフローにつなげにくい。（データの問題と、どの関数の結果かわからないので）
 -> 呼ばれたagent, 関数、そしてその結果をtoolsの結果として返せば良い？
 


--

spec作って渡す

message

返却地形



--


toolsAgentはtools(function call)をサポートしたllmのagent
toolsのスキーマをllmに渡して、toolsの結果が帰ってきた場合にはtoolsの処理をする
通常のtextの結果が帰ってきた場合には、通常のllmの動作する



---
1 動作とメッセージについて

最初にtools付きでllmに投げる

case1 結果にtool.idがない場合には、通常のllmとして、messagesに、user/assistantのやり取りとしてmessageを追加して返して終了


case2 結果にtool_calls(array)がある場合には

agentName - tool.name(funciton name)
arg - tool.arguments
func - tool.name(funciton name)
data - passthrough from parent

を使ってagentを呼び出し

message -> userInput, llmの結果(tools呼び出し情報), tool call(role = tool, tool_call_id, name, content(呼び出されたagentの結果))の結果

結果にhasNestが含まれる
  2の結果を含めllmに投げる. toolsの結果をllmが処理してボットコメント作成
    これを含めてmessagesに追加

結果にhasNestが含まれない
  2の結果をmessagesに追加


dataに、tool_callsの結果(呼び出されたagentの結果に含まれるdata)を追加して返す

---
toolsから呼び出されるagent

namedInputs
 - agentName - tool.name(funciton name)
 - arg - tool.arguments
 - func - tool.name(funciton name)
 - data - passthrough from parent

result
 - content
 - data
 - hasNext

agentFunctionInfoに、tools

(nested agentはhasGraph )