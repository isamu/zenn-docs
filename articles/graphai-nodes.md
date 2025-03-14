---
title: "GraphAI - Compted NodeとStatic Node"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---


# GraphAI のノードタイプ

GraphAIには、**Staticノード**と**Computedノード**の2種類があります。

## 1. Staticノード
Staticノードは、プログラミング言語における変数のように、値を保持する場所です。  
また、ループ実行時に過去のノードの出力結果を参照できます。

### Staticノードの構造

```yaml
value: <初期値>  # 初期値。テキスト、数値、オブジェクトなど。
update: "<nodeId.propId>"  # ループ時に他のノードの結果を参照  
```

### 特徴
- `update`: `nodeId.propId` のように記述し、ループ時に他ノードの出力を更新できる。
- `console`: `{ before: true, after: true }` を指定すると、ノードの実行前後のデータをデバッグ表示できる。

---

## 2. Computedノード
Computedノードは、特定の計算を実行するエージェントに関連付けられたノードです。  
エージェントの実行結果を取得し、次の処理に渡します。

### Computedノードの構造

```yaml
agent: openAIAgent  # 実行するエージェント  
inputs:  
  text: ":someNode.output.text"  # 他ノードのデータ参照 (GOD format)  
params:  
  temperature: 0.7  # エージェントに渡す設定値  
isResult: true  # GraphAI の実行結果として返す  
console:  
  before: true  # 実行前のデータを表示  
  after: true   # 実行後のデータを表示  
```

### 特徴
- `agent`: 実行するエージェント名（例: `"openAIAgent"`）
- `inputs` / `output`:  
  - **GOD (GraphAI Object Dot) format** を用いて他ノードのデータを指定  
  - 例: `inputs: { text: ":someNode.output.text" }`
  - `output` では `.propId` の形式で結果を変換可能. optionです
- `params`: エージェントに渡すオプション設定
- `isResult`: `true` の場合、GraphAI の実行結果として返される。また、ストリーミング対象ノードとしても指定可能。
- `console`: `{ before: true, after: true }` で実行前後のデータをデバッグ表示

---

このように、**Staticノード**は値を保持・更新し、**Computedノード**はエージェントの計算を実行する役割を持っています。  
各ノードは `GOD format` を用いてデータを受け渡し、GraphAI内での処理をスムーズに連携できます。
