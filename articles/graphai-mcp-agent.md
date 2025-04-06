---
title: "GraphAI - MCP(Model Context Protocol) Agnet"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

GraphAIでMCPをサポートしました。

MCPサーバの呼び出し方法には、ネットワーク経由で呼び出す方法と、プロセス通信を用いる方法の2種類がありますが、今回はプロセス通信（commandで設定する形式）のみを対象としています。また、このAgentはNode.jsのみをサポートしているので、ブラウザでは動きません。

MCPには `prompt` や `resources` といったメソッドも存在しますが、今回使用しているのは `tools/list` と `tools/call` のみです。通常、この２つのメソッドをサポートしてあればほとんどのMCPサーバが利用可能です。

他のGraphAIエージェントと異なり、MCPは最初にサーバへ接続する必要があります。実際には、ホスト上でコマンドを実行してそのサーバに接続します。そのため、GraphAIの初期化よりも前にMCPの初期化を行う必要があります。

GraphAIでの実装サンプルはこちら。

https://github.com/receptron/graphai-agents/blob/main/protocol/mcp-agent/tests/run_graph.ts

npmはこちら。
https://www.npmjs.com/package/@graphai/mcp_agent

### 1. `mcpConfig` でサービスごとの設定を行います

各サービスに対して、`command` と `args` のペアで設定を記述します。他のアプリケーションでも、同様の形式が利用できると考えられます。

`mcpConfig`はユニークなサービス名(この例では、`filesystem`と`filesystem2`）を`key`とし、`value`に設定値を記述します。以下に例を示しますが、どちらもlocalのファイルシステムを操作する[server-filesystem](https://www.npmjs.com/package/@modelcontextprotocol/server-filesystem)というMCPサーバを利用しています。

`filesystem`は、npmのパッケージをnpxで直接実行する方法、`filesystem2`はそのnpmをインストールした状態でその実行ファイルを指定しています。動作はどちらも同じで通常はどちらか一方を指定すればよいですが、設定のサンプルのため、このような記述にしています。

mcpConfigに複数のmcpサーバを設定することにより、LLMとmcp/toolsを組み合わせて使う場合に、登録されている全てのMCPサーバの機能を使うことができます。ファイルシステムと、GitHub, SlackのMCPサーバを組み合わせてそれぞれのデータを自動的にやり取りするような用途も可能となります。

```TypeScript
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
MCPの初期化する関数は、mcpClientsを返却します。mcpClientsは、mcpを呼び出す時に必要なので、GraphAIのインスタンスにconfigとして渡します。
またMCPサーバの接続を閉じるときにも必要です。

```TypeScript
  const mcpClients = await mcpInit(mcpConfig)
  await setTimeout(2000);
```
### 3. GraphAIを実行します

GraphAIのインスタンス作成時にconfig経由、mcpClientsを渡します。mcpは複数のagentに分かれているのでglobalでmcpClientsを渡します（個別で渡すことも可能ではある)

Graphデータを定義し、必要なエージェントを登録して実行します。
mcpToolsListAgentは、mcpConfigで設定したサービスの全てのtoolsをマージしたlistを返します。このときにtoolsのnameは、name spaceを考慮しserviceNameをprefixに追加したものを返します。toolsは生データ、llmToolsはopenaiのtoolsの形式に変換したデータです。
mcpToolsCallAgentは、llmから返ってきたtoolsのデータをそのままtools/callで実行します。上記のprefixを追加したname spaceのまま、実行可能です。

```TypeScript
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
  const graphai = new GraphAI(graphData, {...vanilla,  mcpToolsListAgent, mcpToolsCallAgent, openAIAgent }, { config: { global: { mcpClients } } });
  const result = await graphai.run();
  
```  

実行結果の例は以下の通りです：

```TypeScript
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
切断時には、mcpClientsを渡します。

```TypeScript
  await setTimeout(500);
  mcpClose(mcpClients);
```

