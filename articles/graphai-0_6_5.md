---
title: "GraphAI 0.6.5"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

GraphAI 0.6.5がリリースされました。
主な変更点は以下。

## 新機能と変更点

### 1. Agent 関数の引数の変更
- `agent` 関数の引数の構造が変更されました。新しい仕様に合わせて使用方法を調整してください。

### 2. Static Value の更新
- `static value` で `undefined` が許可されるようになりました。
  - 値が未定義の場合、値の注入（inject value）を行うか、`update` を定義する必要があります。
  - `("value" in node)` による判定は使用できなくなりました。代わりに `!("agent" in node)` を使用してください。

### 3. Dynamic Params の拡張
- `dynamic params` のネストされた要素のデータ更新機能がサポートされました。これにより、`named inputs` と同等の挙動を示します。
- 将来的に、`params` は廃止され、`inputs` に統合される予定です。

### 4. Agent での `inputs` の廃止
- `agent` 内で `inputs` フィールドが使用できなくなりました。今後は `named inputs` のみがサポートされます。

### 5. ネストされたグラフ向け Context データ形式の変更
- ネストされたグラフで使用する `context` のデータ形式が変更されました。この変更に対応するよう実装を調整してください。

### 6. if/unless を使った Default Value のサポート
- `if/unless` を使用してノードに `defaultValue` を設定できるようになりました。
  - `if/unless` の条件を満たさない場合に、`defaultValue` を使用してグラフのフローを進めることができます。
  - これにより、従来の `if/unless/anyInputs` を組み合わせた処理を、`if + defaultValue` を使用して簡潔に記述できます。

---

これらの変更を確認し、最新バージョンの GraphAI に対応するためにプロジェクトを更新してください。
