---
title: "GraphAI - OpenAIのクライアントでGraphAIを呼び出す"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---


`@receptron/graphai_express` `1.0.2` で、GraphAIをOpenAIのcompletions互換のAPIとして呼び出すmiddlewareを追加しました。

どういうことかと言うと、npmやpythonのOpenAIのライブラリの`openai.chat.completions.create`や`openai.beta.chat.completions.stream`でGraphAIのワークフローを呼び出せる、ということです。後述する制約がありますが、既存のLLMツールからGraphAIを使うことが簡単になります。

まずはexpressの実装のサンプル。

```TypeScript
import {
  completionRunner,
} from "@receptron/graphai_express";
import * as agents from "@graphai/agents";

const agentDictionary: AgentFunctionInfoDictionary = agents;

const model2graphData = (__model: string) => {
  const graphData = {
    version: 0.5,
    nodes: {
      messages: {
        value: [],
      },
      llm: {
        agent: "openAIAgent",
        params: {
          stream: true,
          isResult: true,
        },
        inputs: {
          messages: ":messages",
        },
        isResult: true,
      },
    },
  };
  return graphData;
};

const onLogCallback = (log: TransactionLog, __isUpdate: boolean) => {
  console.log(log);
};

app.post("/api/chat/completions", completionRunner(agentDictionary, model2graphData, [], onLogCallback));

```

基本的には、`completionRunner`をmiddlewareに追加するだけ。`agentDictionary`はagents, `model2graphData`はmodelをGraphDataに変換する関数。今回はサンプルなので決め打ちですが実際にはmodelと対応するGraphDataを返します。GraphAIでは、llmのmodelはGraphDataの中で指定するので、completionsで渡されるmodel(requred)は、graphDataを指定するために使うようにしました。


OpenAIのtsのサンプルコードは以下

- apiKeyは不要だが、指定しないとライブラリがエラーになるのでdummyを追加。
- baseURLはGraphAI/completionRunnerのサーバを指定。これをprefixとして、/chat/completionsを追加したendpointを呼ぶ
- messagesはopenai(GraphAI)のmessage形式で履歴(システム、ユーザプロンプト）をいれる。
- openai.beta.chat.completions.streamを使って呼び出す

以上で利用可能です。

```TypeScript
import OpenAI from "openai";

const openai = new OpenAI({
  baseURL: "http://localhost:8085/api",
  apiKey: "dummy",
});

const messages = [
  {
    role: "user",
    content: [
      {
        type: "text",
        text: "How can I travel to Mars?",
      },
    ],
  },
] as any;
async function main() {
  const stream = await openai.beta.chat.completions.stream({
    model: "gpt-4o",
    messages,
  });

  for await (const chunk of stream) {
    const token = chunk.choices[0]?.delta?.content;
    if (token) process.stdout.write(token);
  }
  console.log("\n--- done ---");
  const chatCompletion = await stream.finalChatCompletion();
  console.log(JSON.stringify(chatCompletion, null, 2));

}

main().catch(console.error);
```

openAIのクライアントとして使うので制約あり。

- messagesをPOSTしてassistantのmessage（結果を受け取る）
- POSTされたmessagesをGraphDataのmessagesにinjectionするので、messagesのstatic nodeは必須
  - クライアントからPOSTできるもの（渡せるもの)はmessagesだけになる。
- GraphDataで複雑な処理をすることは可能。ただし結果として受け取れるのは１つのmessageだけ。
- Streamingをサポートしているが、openAIのクライアントの制約で１つのllm(Agent)の結果しかうけとれない
  - 並列でstreamingすると混ざった結果となる
  - シリアルの場合は、たぶん２つ連結した結果となるはず
- loop処理をする場合は、completionsを呼び出した結果 + user promptをmessagesに入れて再度postすればよい