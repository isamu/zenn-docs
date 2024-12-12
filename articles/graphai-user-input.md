---
title: "GraphAI Agentにおけるユーザ入力の課題と解決策"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---


# GraphAIにおけるユーザ入力の課題と解決策

GraphAIにとってユーザからの入力は、とても難しい課題です。  
というのも、GraphDataのワークフローはブラウザ、CLI、バッチのいずれの環境でも動作することを前提としているため、特定の入力方法を指定すると動作環境が固定されてしまうからです。  
現状では、Agentを通じてユーザから入力を受け付けると、動作環境が固定されてしまうことが課題として認識されています。

入力を受け取る方法は以下の2つがあります。

- Agentを通じて入力を受け取る方法
- GraphDataのワークフロー実行前に入力を受け付け、Static nodeに値をinjectionする方法

---

## Agentで入力値を受け取る

Agentを使用して入力値を受け取る場合、CLIでは`@inquirer/input`を使った対話型の方法が、ブラウザでは特別な方法でformから入力値を受け取る方法が用いられます。

### CLIでの入力

CLIでは`@graphai/input_agents`を使用します。

実装例は以下のリンクをご参照ください。  
[サンプル実装（chat.ts）](https://github.com/receptron/graphai/blob/main/packages/samples/src/interaction/chat.ts)

### ブラウザでの入力

ブラウザではAgent単体で入力を受け付けることが難しいため、AgentのGeneratorを用意し、Agentと入力を紐づける必要があります。  
具体的には、Agentが入力状態になるとブラウザに通知し、フォームでの入力値がSubmitされるとAgentのPromiseが解決される仕組みです。

`text-input-agent-generator`というパッケージを利用することで、ブラウザ上で簡単に入力値を受け取ることが可能です。

[vueのサンプルはこちら](https://github.com/receptron/graphai-utils/blob/main/packages/vue-text-input-agent-generator/README.md)

これを使用することで、ワークフローの任意のタイミングでブラウザを介してユーザからの入力を受け取ることができます。

---

## ワークフローの初期値として渡す方法

Static nodeはTypeScriptのGraphAIインスタンスで`injectValue`を呼び出すことで値を更新できます。  
これにより、ユーザの入力値をGraphのNodeに初期値として設定することが可能です。

ただし、この方法では初期値として値を渡すのみで、ワークフローの途中で値を更新することはできません。

---

## 手動ループを活用した入力値の適用

GraphAI 0.6.6からmanual loopをサポートしました。  
通常、ループはワークフローの1回目の実行結果をStatic nodeの更新として受け取り、その値を更新して次回のワークフローを実行します。

`initializeGraphAI`と`setPreviousResults`を使用することで、プログラム側でこの仕組みを制御することが可能です。  
v以下のコード例では、ワークフローの途中でユーザ入力値を反映させる方法を示しています。

### 実装例

```typescript
const result = await graph.run();

graph.initializeGraphAI();  // 必要に応じてグラフを初期化
graph.setPreviousResults(result); // 前回の結果を設定（これによりStatic nodeの更新が適用される）
graph.injectValue("userInput", someValue); // ユーザ入力値を設定

const result2 = await graph.run();  // 再度runすることで実質的なループが可能
```

この方法はブラウザでのワークフロー動作時だけでなく、Expressサーバ上での動作にも適用可能であり、幅広い応用が期待できます。