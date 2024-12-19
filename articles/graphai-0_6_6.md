---
title: "GraphAI 0.6.6"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

# GraphAI 0.6.6 リリースノート

GraphAI 0.6.6がリリースされました。
主な変更点は以下。

## console.afterへのformatオプション追加

0.6.5でoutputにformat指定が可能になった機能と同様の指定方法が、`console.after`にも適用可能になりました。
これにより、たとえば以下のような形式でconsole出力に対してformatが行えます：

```typescript
console: {
  after: { text: ".somedata.text" }
}
```

このformat処理は、output format適用前のデータを対象とするため、outputとは別に個別で指定する必要があります。

## 手動ループ対応（loopなしでの複数回実行）

loopをGraphAIの設定で直接指定しなくても、graph.run()を複数回呼び出すことで、グラフ全体を繰り返し処理することが可能になりました。
以下はその使用例です：

```typescript
const result = await graph.run();

graph.initializeGraphAI();  // 必要に応じてグラフを初期化
graph.setPreviousResults(result); // 前回の結果を設定（これによりstatic nodeのupdateが適用）

const result2 = await graph.run();  // 再度runすることで実質的なループが可能
```

この手順により、明示的なloopパラメータなしでも、前回の出力を引き継いでグラフを更新し、複数回の処理を実行できます。

実際の利用例はテストコードを参考にしてください。
https://github.com/receptron/graphai/blob/main/packages/graphai/tests/units/test_manually_loop.ts