---
title: "GraphAI Agentのキャッシュ"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::


# 複雑なマルチエージェント構築とキャッシュの活用: GraphAIの導入方法

GraphAIを利用すると、LLM（大規模言語モデル）、Webアクセス、ユーザー入力など、複数のエージェントを組み合わせた高度なマルチエージェントシステムを構築できます。しかし、LLMやデータベースアクセス、Webリクエストは処理時間やコストがかかるため、同じ処理結果を効率的に再利用するキャッシュ機能の活用が重要です。

GraphAIでは、この課題に対応するため、**Agent Filter** を用いたエージェント単位のキャッシュ機能を提供しています。

---

## エージェントのキャッシュ分類

GraphAIでは、エージェントの入力値と出力値の特性に基づき、以下の3種類に分類しています。

1. **入力に対して常に同じ出力を生成するエージェント**  
   - 主にデータ処理を担当するエージェント
   - **例**: 行列計算や配列の処理などデータを処理するエージェント
   - **例**: LLM（厳密には毎回出力が異なるが、キャッシュを利用することで再利用可能）  
     - **注意**: 結果が不適切な場合はアプリケーション側でキャッシュをクリアし、再実行する。

2. **内部状態に依存して出力が変化するエージェント**  
   - **例**: ファイルの読み書きや一時的な状態管理を伴う処理
   - **例**: Webエージェント。TTLやhttp header等でcacheの状態を判定する
3. **外部要因やユーザー入力によって動作するエージェント**  
   - **例**: ユーザーからのテキストや画像入力に応じて処理するエージェント

---

## GraphAIにおけるキャッシュ機能の実装

キャッシュは**Agent Filter**を通じて実現されます。以下の特徴があります。

- **キャッシュストレージの管理**  
  GraphAIはキャッシュストレージを提供せず、利用者が任意のストレージを選択可能。  
  GraphAIのエージェントフィルタに`getCache`および`setCache`関数を渡すことでキャッシュ機能を利用します。

- **キャッシュの削除やクリア**  
  キャッシュ管理（削除やクリア）はGraphAIの外側で実装します。

---

## 各エージェントタイプのキャッシュ処理例

### 1. 入力に対して常に同じ出力を生成するエージェント
- **実装方法**  
  Cache Agent Filter内でキャッシュ処理を行います。  
  キャッシュがヒットした場合、エージェントを呼び出さずキャッシュ結果を直接返します。  
- **キャッシュキー生成**  
  `namedInputs`, `params`, `agentId` の組み合わせをJSON形式でハッシュ化したものを使用します。

### 2. 内部状態に依存するエージェント
- **実装方法**  
  Cache Agent Filterでキャッシュ関数を用意し、エージェント内でキャッシュの読み書きを実行します。  
- **キャッシュキー生成**  
  キャッシュキーの設計はエージェント側で実装します。

### 3. 外的要因によるエージェント
- **実装方法**  
  キャッシュは使用しません。

# 使い方

`getCache`および`setCache`関数を実装し、cacheAgentFilterGeneratorに`getCache`および`setCache`関数を渡して、cacheAgentFilterを取得します。

そのagentFilterはGraphAIのコンストラクタに渡します。

```typescript
const setCache = async (key: string, data: any) => {
  ...
  return;
};
const getCache = async (key: string) => {
  ...
  return data;
}

const cacheAgentFilter = cacheAgentFilterGenerator({ getCache, setCache });
```

### pureAgent

pureAgentは、AgentFunctionInfoの`cacheType`に`"pureAgent"`を指定します

```typescript
cacheType: "pureAgent"
```

`pureAgent`は、入力値(inputsとparamas)で結果が一意に決まるので、それらをkeyにして`agent filter`内でキャッシュ結果を返します。
つまりagent側ではキャッシュの実装は不要です。

### impureAgent

impureAgentは、AgentFunctionInfoの`cacheType`に`"impureAgent"`を指定します

```typescript
cacheType: "impureAgent"
```

`pureAgent`は、Agent内部の状態でキャッシュの動作がきまるので`agent`内で、`agent filter`の`set/getCache`関数を使ってキャッシュ操作をします。
cache関数がない場合でも動作するように実装をしておく必要があります。


```typescript
// impureAgent cache example.
const impureSampleAgent: AgentFunction = async ({ namedInputs, filterParams }) => {
  // cache process.
  const { message } = namedInputs;
  const fileKey = .....

  // get cache from cache storage
  if (filterParams.cache) {
    const cacheData = await filterParams.cache.getCache(fileKey);
    if (cacheData) {
      return cacheData;
    }
  }

  // agent process.
  ...
  ...
  ...
  const result = ...

  // after cache process
  if (filterParams.cache) {
    await filterParams.cache.setCache(fileKey, result);
  }
  return result;
};
```

詳しくは、agent_filterのtestコードを参考にしてください。
agent_filters/tests/filters/test_cache_impure.ts

sqlliteを使ったキャッシュのサンプルはこちら

https://github.com/receptron/graphai/blob/main/packages/samples/src/cache/cache_example.ts