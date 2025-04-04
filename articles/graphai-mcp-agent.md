---
title: "GraphAI - MCP(Model Context Protocol) Agnet"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

GraphAIでMCPをサポートしました。

MCPサーバの呼び出し方法には、ネットワーク経由で呼び出す方法と、プロセス通信を用いる方法の2種類がありますが、今回はプロセス通信（commandで設定する形式）のみを対象としています。

MCPには `init` や `resources` といったメソッドも存在しますが、今回使用しているのは `tools/list` と `tools/call` のみです。通常、この２つのメソッドをサポートしてあればほとんどのMCPサーバが利用可能です。

他のGraphAIエージェントと異なり、MCPは最初にサーバへ接続する必要があります。実際には、ホスト上でコマンドを実行してそのサーバに接続します。そのため、GraphAIの初期化よりも前にMCPの初期化を行う必要があります。

### 1. `mcpConfig` でサービスごとの設定を行います

各サービスに対して、`command` と `args` のペアで設定を記述します。他のアプリケーションでも、同様の形式が利用できると考えられます。

```
export const mcpConfig = {
  filesystem: {
    command: "npx",
    args: ["-y", "@modelcontextprotocol/server-filesystem", path],
  },
  filesystem2: {
    command: __dirname + "/../../../node_modules/@modelcontextprotocol/server-filesystem/dist/index.js",
    args: [path],
  },
};
```

### 2. MCPを初期化してサーバに接続します

GraphAIを動かす前に、MCPの初期化を行い、サーバへの接続を確立しておきます。MCPの起動には時間がかかるため、`setTimeout` や `sleep` などで待機します。
MCPの起動を待たないでGraphAIを実行すると、tools listが空になることがあります。

```
  await mcpInit(mcpConfig)
  await setTimeout(2000);
```
### 3. GraphAIを実行します

Graphデータを定義し、必要なエージェントを登録して実行します。
mcpToolsListAgentは、mcpConfigで設定したサービスの全てのtoolsをマージしたlistを返します。このときにtoolsのnameは、name spaceを考慮しserviceNameをprefixに追加したものを返します。toolsは生データ、llmToolsはopenaiのtoolsの形式に変換したデータです。
mcpToolsCallAgentは、llmから返ってきたtoolsのデータをそのままtools/callで実行します。上記のprefixを追加したname spaceのまま、実行可能です。

```
  const graphData = {
    version: 0.5,
    nodes: {
      list: {
        agent: "mcpToolsListAgent",
        isResult: true,
        console: true,
      },
      llm: {
        agent: "openAIAgent",
        inputs: {
          tools: ":list.llmTools",
          prompt: "get all file list on " + path
        },
      },
      call: {
        console: {before: true, after: true},
        agent: "mcpToolsCallAgent",
        inputs: {
          tools: ":llm.tool"
        },
      },
    },
  };
  const graphai = new GraphAI(graphData, {...vanilla,  mcpToolsListAgent, mcpToolsCallAgent, openAIAgent });
  const result = await graphai.run();
  
```  

実行結果の例は以下の通りです：

```
{
  "response": {
    "content": [
      {
        "type": "text",
        "text": "[FILE] 1.txt\n[FILE] 2.txt"
      }
    ]
  }
}
```


### 4. バッチやサーバ（Expressなど）の終了時にMCPサーバとの接続を切断します

処理の終了時には、MCPとの接続を明示的に切断しておきます。

```
  await setTimeout(500);
  mcpClose();
```



### 今後の対応（TODO）

- サーバ型のMCPサーバをサポートする予定です。
