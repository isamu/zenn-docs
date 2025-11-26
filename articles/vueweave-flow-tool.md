---
title: "VueWeave: Vue 3用の現代的なノードベースフローエディタを作りました"
emoji: "🌊"
type: "tech"
topics: ["vue", "vuejs", "typescript", "opensource", "ui"]
published: true
---

# VueWeave: Vue 3用の現代的なノードベースフローエディタを作りました

こんにちは！今回、**VueWeave**というVue 3用のノードベースフローエディタをオープンソースとして公開しました。

https://github.com/receptron/grapys/tree/main/packages/vueweave

https://vueweave.com/

## 🚀 なぜ作ったのか

私は現在、[**GraphAI**](https://graphai.info/)というAIワークフローツールを開発しています。開発を始めてすぐに、ビジュアルなフローエディタが必要になりました。

具体的には以下のような要件がありました：
- エッジ（接続線）の入力を細かく制御したい
- ノードの見た目や動作を深くカスタマイズしたい
- Vue 3で動作する必要がある

### 既存ツールの課題

#### React Flow
- React専用（Vueでは使えない）
- **エッジバリデーションの柔軟性が低い** - 複数の入出力ポートを追加したり、ポートごとに異なるバリデーションルールを設定するのが困難

#### Vue Flow
- React FlowのVue版として期待したが...
- **React Flowと同様、エッジバリデーションが柔軟でない**

#### LiteGraph
- ノードのカスタマイズが複雑すぎる
- アーキテクチャが古い
- TypeScriptサポートが不十分

**どのツールも自分の要件を満たせない**ことに気づき、結局UIを完全に自作することにしました。

### Grapysから汎用ライブラリへ

自作したUIでGrapys（GUIツール）を無事に完成させることができました。その過程で「これは他の開発者にも役立つはず」と考え、汎用的なライブラリとして切り出したのが**VueWeave**です。

## ✨ VueWeaveの特徴

**デモサイト**: https://vueweave.com/

実際に動作するデモを見ながら読むとわかりやすいです！

### 1. ゼロコンフィグで動作

最小限のコードでフローエディタが動きます：

```vue
<script setup lang="ts">
import { GraphCanvasBase } from "vueweave";
import "vueweave/style.css"; // スタイルのインポート
</script>

<template>
  <GraphCanvasBase />
</template>
```

これだけで、ドラッグ&ドロップ可能なノードと接続線を持つフローエディタが完成します。

**デモ**: https://vueweave.com/minimal

### 2. 柔軟なエッジバリデーション

これがVueWeaveを作った主な理由の一つです。各入力ポートごとに異なるバリデーションルールを細かく制御できます：

```typescript
import { useFlowStore } from "vueweave";

const flowStore = useFlowStore();

// カスタムバリデーション
const validateConnection = (expectEdge, existingEdges) => {
  const targetNode = flowStore.nodes.find(
    n => n.nodeId === expectEdge.target.nodeId
  );

  // ノード全体で1つの接続のみ許可（全ポート排他的）
  if (targetNode.type === "singleInput") {
    // 複数のポート（input1, input2）があっても、どれか1つだけ接続可能
    const allEdges = flowStore.edges.filter(
      e => e.target.nodeId === expectEdge.target.nodeId
    );
    return allEdges.length === 0;
  }

  // ポートごとに1つの接続のみ許可
  if (targetNode.type === "onePerPort") {
    // input1, input2, input3など各ポートに1つずつ接続可能
    const samePortEdges = existingEdges.filter(
      e => e.target.index === expectEdge.target.index
    );
    return samePortEdges.length === 0;
  }

  // 同じポートに複数の接続を許可
  if (targetNode.type === "multiple") {
    // 1つのポートに何本でも接続可能
    return true;
  }

  // その他のカスタムロジック（型チェックなど）
  return true;
};
```

**ポイント**: 同じノードに複数の入力ポート（input1, input2など）を持たせた上で、ポートごとに異なるルール（排他的 or 複数許可）を設定できます。これがReact/Vue Flowでは難しい部分です。

**デモ**: https://vueweave.com/validation

### 3. Vueスロットによる簡単なカスタマイズ

ノードのデザインはVueのスロットで自由にカスタマイズできます：

```vue
<script setup lang="ts">
import { GraphCanvasBase, NodeBase, type NodeColorConfig } from "vueweave";
import "vueweave/style.css";

const nodeColors: NodeColorConfig = {
  source: {
    main: "bg-purple-400",
    header: "bg-purple-500",
    mainHighlight: "bg-purple-200",
    headerHighlight: "bg-purple-300",
  },
  processor: {
    main: "bg-green-400",
    header: "bg-green-500",
    mainHighlight: "bg-green-200",
    headerHighlight: "bg-green-300",
  },
};

const getInputs = (nodeData) => {
  return nodeData.type === "source" ? [] : [{ name: "input" }];
};

const getOutputs = (nodeData) => {
  return [{ name: "output" }];
};
</script>

<template>
  <GraphCanvasBase :node-styles="{ colors: nodeColors }">
    <template #node="{ nodeData }">
      <NodeBase :inputs="getInputs(nodeData)" :outputs="getOutputs(nodeData)">
        <template #header>
          <div class="w-full rounded-t-md py-2 text-center text-white">
            {{ nodeData.nodeId }}
          </div>
        </template>
        <template #body-main>
          <div class="p-2">
            <div class="font-semibold">{{ nodeData.type }}</div>
          </div>
        </template>
      </NodeBase>
    </template>
  </GraphCanvasBase>
</template>
```

**デモ**: https://vueweave.com/styled

### 4. TypeScriptファースト

完全な型安全性とIntelliSenseサポート：

```typescript
import type { GUINodeData, GUIEdgeData } from "vueweave";

interface MyNodeData {
  value: string;
  count: number;
}

const node: GUINodeData<MyNodeData> = {
  type: "custom",
  nodeId: "node1",
  position: { x: 100, y: 100 },
  data: {
    value: "Hello",
    count: 42,
  },
};
```

### 5. Piniaによる状態管理

Vueの公式状態管理ライブラリPiniaを使用：

```typescript
import { useFlowStore } from "vueweave";

const flowStore = useFlowStore();

// データの初期化
flowStore.initData(nodes, edges, {});

// ノードの追加
flowStore.pushNode({
  type: "processor",
  nodeId: "new-node",
  position: { x: 200, y: 200 },
  data: { name: "New Node" },
});

// ノードの削除（インデックス指定）
flowStore.deleteNode(0);

// エッジの追加
flowStore.pushEdge({
  type: "edge",
  source: { nodeId: "node1", index: 0 },
  target: { nodeId: "node2", index: 0 },
});

// Undo/Redo
flowStore.undo();
flowStore.redo();

// 現在の状態確認
console.log(flowStore.nodes);
console.log(flowStore.edges);
console.log(flowStore.undoable); // true/false
console.log(flowStore.redoable); // true/false
```

### 6. 組み込みのUndo/Redo

すべての操作で自動的に履歴が記録されます：

```vue
<template>
  <div>
    <button @click="flowStore.undo()" :disabled="!flowStore.undoable">
      元に戻す
    </button>
    <button @click="flowStore.redo()" :disabled="!flowStore.redoable">
      やり直し
    </button>
  </div>
</template>
```

### 7. Tailwind CSSサポート

Tailwindのクラスで簡単にスタイリング：

```typescript
import type { NodeColorConfig } from "vueweave";

const nodeColors: NodeColorConfig = {
  important: {
    main: "bg-red-400",
    header: "bg-red-600",
    mainHighlight: "bg-red-200",
    headerHighlight: "bg-red-400",
  },
};
```

エッジの色もノードタイプやnodeIdで指定可能：

```typescript
import type { EdgeColorConfig } from "vueweave";

const edgeColors: EdgeColorConfig = {
  source: "#a855f7",      // ソースノードからの接続は紫
  processor: "#22c55e",   // プロセッサーからは緑
  output: "#3b82f6",      // 出力からは青
  "special-node": "#ef4444", // 特定のnodeIdも指定可能
  default: "#9ca3af",     // デフォルト色
};

// GraphCanvasBaseに渡す
<GraphCanvasBase :node-styles="{ colors: nodeColors, edgeColors: edgeColors }">
```

## 📚 実践例

### 基本的な使い方

```vue
<script setup lang="ts">
import { onMounted } from "vue";
import { useFlowStore, GraphCanvasBase, NodeBase, type GUINodeData } from "vueweave";
import "vueweave/style.css";

const flowStore = useFlowStore();

onMounted(() => {
  flowStore.initData(
    [
      {
        type: "source",
        nodeId: "input1",
        position: { x: 50, y: 100 },
        data: { value: "データA" },
      },
      {
        type: "processor",
        nodeId: "process1",
        position: { x: 300, y: 100 },
        data: { operation: "transform" },
      },
      {
        type: "output",
        nodeId: "output1",
        position: { x: 550, y: 100 },
        data: { format: "json" },
      },
    ],
    [
      {
        type: "edge",
        source: { nodeId: "input1", index: 0 },
        target: { nodeId: "process1", index: 0 },
      },
      {
        type: "edge",
        source: { nodeId: "process1", index: 0 },
        target: { nodeId: "output1", index: 0 },
      },
    ],
    {}
  );
});

const getInputs = (nodeData: GUINodeData) => {
  return nodeData.type === "source" ? [] : [{ name: "input" }];
};

const getOutputs = (nodeData: GUINodeData) => {
  return nodeData.type === "output" ? [] : [{ name: "output" }];
};
</script>

<template>
  <GraphCanvasBase>
    <template #node="{ nodeData }">
      <NodeBase :inputs="getInputs(nodeData)" :outputs="getOutputs(nodeData)">
        <template #header>
          <div class="w-full rounded-t-md py-2 text-center text-white">
            {{ nodeData.nodeId }}
          </div>
        </template>
        <template #body-main>
          <div class="p-2">
            <div class="font-semibold">{{ nodeData.type }}</div>
          </div>
        </template>
      </NodeBase>
    </template>
  </GraphCanvasBase>
</template>
```

### ノードパラメータの編集

ノード内に入力フィールドを配置できます：

```vue
<template>
  <GraphCanvasBase>
    <template #node="{ nodeData }">
      <NodeBase :inputs="getInputs(nodeData)" :outputs="[{ name: 'output' }]">
        <template #body-main>
          <div class="p-2">
            <!-- 文字列入力 -->
            <input
              v-if="nodeData.type === 'source'"
              v-model="nodeData.data.value"
              class="w-full rounded border p-1"
              @mousedown.stop
            />

            <!-- 数値入力 -->
            <input
              v-if="nodeData.type === 'processor'"
              v-model.number="nodeData.data.count"
              type="number"
              class="w-full rounded border p-1"
              @mousedown.stop
            />

            <!-- セレクト -->
            <select
              v-if="nodeData.type === 'output'"
              v-model="nodeData.data.format"
              class="w-full rounded border p-1"
            >
              <option value="json">JSON</option>
              <option value="csv">CSV</option>
              <option value="xml">XML</option>
            </select>
          </div>
        </template>
      </NodeBase>
    </template>
  </GraphCanvasBase>
</template>
```

**重要**: `@mousedown.stop`を付けることで、入力フィールドクリック時にノードがドラッグされるのを防ぎます。

### バリデーションの実例

複数のバリデーションルールを組み合わせた例：

```typescript
const validateConnection = (expectEdge, existingEdges) => {
  const sourceNode = flowStore.nodes.find(
    n => n.nodeId === expectEdge.source.nodeId
  );
  const targetNode = flowStore.nodes.find(
    n => n.nodeId === expectEdge.target.nodeId
  );

  // ルール1: 単一入力ノード（全体で1つの接続のみ）
  if (targetNode.type === "singleInput") {
    const allEdges = flowStore.edges.filter(
      e => e.target.nodeId === expectEdge.target.nodeId
    );
    return allEdges.length === 0;
  }

  // ルール2: ポートごとに1つの接続
  if (targetNode.type === "onePerPort") {
    const samePortEdges = existingEdges.filter(
      e => e.target.index === expectEdge.target.index
    );
    return samePortEdges.length === 0;
  }

  // ルール3: 無制限の接続
  if (targetNode.type === "multiple") {
    return true;
  }

  // ルール4: 型マッチング
  if (sourceNode.type === "typeA" || sourceNode.type === "typeB") {
    if (targetNode.type !== "output" && sourceNode.type !== targetNode.type) {
      return false; // 型が一致しない
    }
    return true;
  }

  return true;
};
```

## 🎯 こんな用途に最適

VueWeaveは以下のようなアプリケーションに適しています：

- **AIパイプラインデザイナー** - LLMやエージェントのワークフローを視覚的に構築
- **データフロー図** - データ処理の流れを可視化
- **ワークフローエディタ** - ビジネスプロセスの自動化
- **ビジュアルプログラミングツール** - ノードベースのプログラミング環境
- **プロセス管理システム** - 承認フローや業務フローの設計

実際に、GraphAIではAIエージェントのワークフローを視覚的に設計・実行するために使用しています。

## 🔄 React Flow・Vue Flow・LiteGraphとの比較

| 機能 | VueWeave | Vue Flow | React Flow | LiteGraph |
|------|----------|----------|------------|-----------|
| フレームワーク | Vue 3 | Vue 3 | React | フレームワーク非依存 |
| TypeScript | ✅ 完全サポート | ✅ 完全サポート | ✅ 完全サポート | ⚠️ 部分的 |
| 状態管理 | Pinia (公式) | 内部管理 | Zustand | 独自実装 |
| カスタマイズ | Vueスロット (簡単) | Vueコンポーネント | Reactコンポーネント | 複雑なAPI |
| ポートごとのバリデーション | ✅ 柔軟に制御可能 | ⚠️ 限定的 | ⚠️ 限定的 | ✅ 可能 |
| エッジバリデーション | ✅ 完全カスタマイズ | ⚠️ 基本的な制御のみ | ⚠️ 基本的な制御のみ | ✅ 可能 |
| Undo/Redo | ✅ 組み込み | ❌ 手動実装必要 | ❌ 手動実装必要 | ✅ あり |
| ライセンス | MIT | MIT | MIT | MIT |

VueWeaveの最大の強みは：
- **Vue開発者にとって自然な書き方**（Composition API、スロット、Pinia）
- **複数の入力ポートに対して個別にバリデーションルールを設定可能**
  - 例：input1は排他的、input2は複数接続OK、input3は型チェックなど
  - React/Vue Flowではこのような細かい制御が困難
- **柔軟なエッジバリデーション**が簡単に実装できる

## 📦 インストールと使い始め

```bash
npm install vueweave
```

```vue
<script setup lang="ts">
import { GraphCanvasBase } from "vueweave";
import "vueweave/style.css"; // CSSを忘れずに！
</script>

<template>
  <GraphCanvasBase />
</template>
```

たったこれだけです！

**重要**: `import "vueweave/style.css"` を必ず追加してください。これがないとスタイルが適用されません。

## 🔗 リンク

- **デモサイト**: https://vueweave.com/
- **GitHub**: https://github.com/receptron/grapys/tree/main/packages/vueweave
- **API仕様**: [SPEC.md](https://github.com/receptron/grapys/blob/main/packages/vueweave/SPEC.md)
- **npm**: https://www.npmjs.com/package/vueweave

### デモページ

- [Minimal Example](https://vueweave.com/minimal) - 最小構成
- [Interactive Example](https://vueweave.com/interactive) - ノード追加・編集
- [Validation Example](https://vueweave.com/validation) - バリデーション例
- [Styled Example](https://vueweave.com/styled) - スタイリング例

## 🙏 フィードバックをお待ちしています！

VueWeaveはまだ始まったばかりです。皆さんの実際のユースケースやフィードバックが、より良いツールへと成長させてくれます。

以下のような情報をぜひお寄せください：

- 実際に使ってみた感想
- 欲しい機能
- バグ報告
- ドキュメントの改善提案
- 実装例の共有

GitHubのIssueやPull Requestをお待ちしています！

また、Discord（[Singularity Society](https://discord.gg/XqmAYxm2Xf)）でも議論できます。

## 🎉 まとめ

VueWeaveは、**Vue 3で本格的なノードベースフローエディタを簡単に構築できる**ライブラリです。

- ゼロコンフィグで動作開始
- 柔軟なエッジバリデーション
- Vueスロットによる簡単カスタマイズ
- TypeScript完全サポート
- Piniaによる状態管理
- 組み込みUndo/Redo

GraphAIでの実戦経験から生まれたVueWeaveが、皆さんの開発にも役立てば嬉しいです。

ぜひ試してみて、フィードバックをお願いします！

一緒に、Vue エコシステムをより豊かにしていきましょう！ 🚀
