---
title: "GraphAI 環境依存(Node.jsとブラウザ）のあるAgentついて"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

GraphAIはTypeScriptで実装されており、AgentはNode.js環境でもブラウザ環境でも動作します。
データ処理やWeb APIへのアクセスといった用途では問題なく動作しますが、ファイル操作(fs module)、パス管理(path)、環境変数(process.env)のアクセスなど環境依存のある処理はNode.js環境に限定されます。
環境依存の処理だけでそれぞれの環境で分けることにより、可能な限り同じエージェントでNode.jsとブラウザの両環境をサポートできるよう、環境依存部分を注入（Injection）して切り替える仕組みを導入しました。

サンプルレポジトリ
https://github.com/receptron/graphai-agents/tree/main/packages/node-browser-detect-agent

npm
https://www.npmjs.com/package/node_browser_detect_agent

以下にサンプルコードのimport例を示します：

```typescript
// ブラウザ環境:
import { nodeBrowserDetectAgent } from "node_browser_detect_agent/lib/browser";
```

```typescript
//Node.js環境:
import { nodeBrowserDetectAgent } from "node_browser_detect_agent/lib/node";
```

これらのエージェントは、それぞれの環境に依存した関数をagent generatorに注入して作成されています。また、エージェント情報を生成するagentFunctionInfoGeneratorを追加し、生成したエージェントを注入することでエージェントに情報を付加する仕組みも構築されています。

このagentを実行すると、環境に応じたメッセージがconsole.logとして出力されます。

# 課題と現状
理想的には、package.jsonのmain、module、browserフィールドを利用して自動的に適切なモジュールが切り替わることが望ましいです。しかし現時点では、moduleフィールドに指定された内容がNode.jsやViteで読み込まれるため、自動切り替えの実現は難しい状況です。

# 提案

この仕組みを参考に、Node.jsおよびブラウザ両環境で動作する環境依存型エージェントを作成してください！
必要に応じてさらに調整いたしますので、ご要望があればお知らせください！


----

# GraphAI Environment-Dependent Agent (Node.js and Browser)

GraphAI is implemented in TypeScript, and its agents work in both Node.js and browser environments. While data processing and Web API access work seamlessly, operations dependent on the environment, such as file handling (fs module), path management (path module), and access to environment variables (process.env), are limited to Node.js.

To support both Node.js and browser environments with the same agent as much as possible, we introduced a mechanism that isolates environment-dependent logic and switches it through injection.

Sample repository:
https://github.com/receptron/graphai-agents/tree/main/packages/node-browser-detect-agent

NPM:
https://www.npmjs.com/package/node_browser_detect_agent

Below are import examples demonstrating how to use the agent in different environments:

```TypeScript
// Browser environment:
import { nodeBrowserDetectAgent } from "node_browser_detect_agent/lib/browser";
```

```TypeScript
// Node.js environment:
import { nodeBrowserDetectAgent } from "node_browser_detect_agent/lib/node";
```

These agents are created by injecting environment-dependent functions into an agent generator. Additionally, an agentFunctionInfoGenerator is included to generate agent-related information. This generator adds information to the agent by injecting the created agent.

When executing this agent, environment-specific messages are output to console.log.

Challenges and Current Status
Ideally, the main, module, and browser fields in package.json could be used to automatically switch to the appropriate module. However, as of now, the module field's content is prioritized in both Node.js and Vite, making automatic switching challenging.

Proposal
Using this setup as a reference, please create an environment-dependent agent that works in both Node.js and browser environments!
Feel free to suggest any adjustments as needed.