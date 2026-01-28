---
title: "MulmoChatでAIと遊べるゲームプラグインを作ろう - Akinatorを例に"
emoji: "🔮"
type: "tech"
topics: ["vue", "react", "typescript", "llm", "ai"]
published: true
publication_name: "singularity"
---

# はじめに

LLM（大規模言語モデル）とのチャットは楽しいですが、テキストだけのやり取りに限界を感じることはありませんか？

「AIにクイズを出してもらいたい」「ゲームを一緒に遊びたい」「グラフィカルな結果を表示したい」

そんな願いを叶えるのが **[MulmoChat](https://github.com/receptron/MulmoChat)** と **[GUIChatPluginTemplate](https://github.com/receptron/GUIChatPluginTemplate)** です。


MulmoChatは何？って人はまずこちらを。

https://www.youtube.com/watch?v=BiWRksxBofE

## プラグイン開発の魅力

**GUIChatPluginTemplate** を使えば、MulmoChatの音声チャットで使えるGUIプラグインを簡単に作れます。

```
テンプレートをコピー
    ↓
Claude Code で自動生成（CLAUDE.md 付き）
    ↓
テキストチャットでデバッグ（yarn dev）
    ↓
yarn build → GitHub/npm で配布
    ↓
MulmoChat で音声チャット対応！
```

**特徴:**

| 開発フェーズ | 環境 | 入力方式 |
|-------------|------|----------|
| 開発・デバッグ | テンプレート (yarn dev) | テキスト |
| 本番 | MulmoChat | テキスト + **音声** |

- **CLAUDE.md 付き**: Claude Code に「〇〇プラグインを作って」と指示するだけで、プラグインを自動生成
- **デバッグ環境内蔵**: Mock Mode でAPIキー不要、Quick Samples でワンクリックテスト
- **そのまま配布**: `yarn build` → GitHub/npm にプッシュするだけで配布完了
- **音声対応**: 開発はテキストで効率よく、本番では音声で自然な体験

この記事では、MulmoChatの仕組みを解説し、実際に[Akinatorゲームプラグイン](https://github.com/isamu/AkinatorGame)を作る過程を通じて、プラグイン開発の方法を紹介します。

# MulmoChatとは

**[MulmoChat](https://github.com/receptron/MulmoChat)** は、LLMの応答にGUIコンポーネントを組み合わせられるチャットアプリケーションです。

特徴：
- **GUI プラグイン**: LLMがツールを呼び出すと、リッチなUIを表示
- **音声入力対応**: テキストだけでなく、音声でもAIと会話できる
- **マルチモーダル**: テキスト・音声・GUI を組み合わせた体験

通常のチャットアプリでは、LLMの応答はテキストのみです。しかしMulmoChatでは、LLMが「ツール」を呼び出すことで、リッチなUIを表示できます。

```
通常のチャット:
ユーザー: クイズ出して
AI: 問題です。日本の首都はどこでしょう？
    A) 大阪  B) 東京  C) 京都  D) 名古屋

MulmoChat:
ユーザー: クイズ出して
AI: [クイズUIコンポーネントが表示される]
    ┌─────────────────────────┐
    │ 問題: 日本の首都は？      │
    │ [A) 大阪] [B) 東京]      │
    │ [C) 京都] [D) 名古屋]    │
    └─────────────────────────┘
```

## MulmoChatのアーキテクチャ

MulmoChatは以下のような流れで動作します：

```
ユーザーメッセージ
       ↓
   LLM (OpenAI等)
       ↓
  tool_calls を含む応答?
       ↓
┌──────┴──────┐
│Yes          │No
↓             ↓
プラグインの    テキスト応答を
execute()実行   そのまま表示
↓
ToolResult を返す
↓
Viewコンポーネントで
GUIを表示
```

ポイントは、LLMが「いつツールを使うか」を判断することです。ユーザーが「クイズ出して」と言えば、LLMはクイズツールを呼び出します。

# GUIChatPluginTemplateとは

**[GUIChatPluginTemplate](https://github.com/receptron/GUIChatPluginTemplate)** は、MulmoChat用のプラグインを簡単に作れるテンプレートです。

:::message alert
**重要**: MulmoChatはVueで作られています。MulmoChatで使うプラグインは**Vue**で実装してください。

テンプレートにはReact版も含まれており、`yarn dev:react`でReact版のテストも可能ですが、MulmoChatではReactプラグインは動作しません。
:::

特徴：
- **チャットデモ統合**: 実際のチャット画面でプラグインをテスト
- **TypeScript**: 型安全な開発
- **CLAUDE.md付き**: Claude Codeでプラグインを自動生成

## ドキュメント

テンプレートには詳細なドキュメントが付属しています：

| ドキュメント | 内容 | 対象 |
|-------------|------|------|
| [Getting Started](https://github.com/receptron/GUIChatPluginTemplate/blob/main/docs/getting-started.md) | 初めてのプラグイン作成チュートリアル | 初心者 |
| [Plugin Development Guide](https://github.com/receptron/GUIChatPluginTemplate/blob/main/docs/plugin-development-guide.md) | プラグイン開発の詳細リファレンス | 全開発者 |
| [AI Development Guide](https://github.com/receptron/GUIChatPluginTemplate/blob/main/docs/ai-development-guide.md) | Claude Code向け最適化ガイド | AI + 開発者 |
| [Advanced Features](https://github.com/receptron/GUIChatPluginTemplate/blob/main/docs/advanced-features.md) | sendTextMessage、viewState等の高度な機能 | 中級者以上 |
| [npm Publishing Guide](https://github.com/receptron/GUIChatPluginTemplate/blob/main/docs/npm-publishing-guide.md) | npm公開とMulmoChatへの統合 | 全開発者 |

日本語版もあります: [Getting Started (日本語)](https://github.com/receptron/GUIChatPluginTemplate/blob/main/docs/getting-started.ja.md)、[Advanced Features (日本語)](https://github.com/receptron/GUIChatPluginTemplate/blob/main/docs/advanced-features.ja.md)

## プラグインの構造

```
src/
├── core/                 # フレームワーク非依存（Vue/React共通）
│   ├── definition.ts     # ツール定義（LLMへの説明）
│   ├── types.ts          # 型定義
│   ├── plugin.ts         # execute関数（メインロジック）
│   └── samples.ts        # テストデータ
├── vue/
│   ├── View.vue          # メイン表示コンポーネント
│   └── Preview.vue       # サイドバーサムネイル
└── react/
    ├── View.tsx
    └── Preview.tsx
```

# Akinatorプラグインを作ってみよう

## Akinatorとは

**Akinator（アキネイター）** は、2007年にフランスで生まれた人気の推理ゲームです。

プレイヤーが頭の中で思い浮かべたキャラクターや有名人を、AIが「はい/いいえ」で答える質問を繰り返すことで当てていきます。「ランプの魔人」というキャラクターが質問し、驚くほど正確に答えを当てることで世界中で話題になりました。

```
あなた: （ドラえもんを思い浮かべる）

Akinator: 「そのキャラクターは日本のものですか？」
あなた: 「はい」

Akinator: 「アニメのキャラクターですか？」
あなた: 「はい」

Akinator: 「青い色をしていますか？」
あなた: 「はい」

Akinator: 「それは... ドラえもん ですね！」
あなた: 「正解！」
```

このゲームをMulmoChat用のプラグインとして実装してみましょう。音声で「はい」「いいえ」と答えるだけで遊べる、自然なインターフェースになります。

## Claude Codeでプラグインを作る

GUIChatPluginTemplateには **CLAUDE.md** が含まれており、Claude Codeに指示するだけでプラグインを自動生成できます。

### Step 1: テンプレートをコピー

```bash
# テンプレートをコピー
git clone https://github.com/receptron/GUIChatPluginTemplate AkinatorGame
cd AkinatorGame
yarn install
```

### Step 2: Claude Codeに指示

Claude Codeを起動して、以下のように指示します：

```
アキネイターゲームのプラグインを作って。

- AIがユーザーに「はい/いいえ」で答える質問をする
- ユーザーの回答をもとにAIが推理して答えを当てる
- カテゴリ: キャラクター、有名人、動物、もの、場所
- 回答ボタン: はい、いいえ、たぶんはい、たぶんいいえ、わからない
- 最大20問で当てる
- スコア表示あり
```

Claude Codeは **CLAUDE.md** を読み込み、プラグインの構造を理解した上で、必要なファイルを自動生成します。

### Step 3: デバッグ

OpenAI API Keyを環境変数に設定して、開発サーバーを起動します。

```bash
# .envファイルを作成
echo "VITE_OPENAI_API_KEY=sk-your-api-key" > .env

# 開発サーバーを起動
yarn dev  # http://localhost:5173
```

チャットで「アキネイターで遊ぼう」と入力すると、実際のLLMの応答を見ながらデバッグできます。Quick Samplesボタンを使えば、ゲームの各フェーズを個別にテストすることも可能です。

### Step 4: ビルドと配布

```bash
# 型チェックとLintを通す
yarn typecheck
yarn lint

# ビルド
yarn build

# GitHubにプッシュ
git add -f dist/
git commit -m "Build Akinator plugin"
git push
```

型チェックとLintが通ればビルドして配布できます。これでMulmoChatで使えるプラグインの完成です！

:::message
**完成品はこちら**: [AkinatorGame リポジトリ](https://github.com/isamu/AkinatorGame)

以下では、Claude Codeが生成したコードの要点を解説します。
:::

## 生成されたコードの解説

Claude Codeは以下のようなコードを生成します。ただし、プロンプトの内容や指示の仕方によって生成されるコードは大きく異なる場合があります。ここでは一例として、各ファイルの役割を解説します。カスタマイズや問題解決に役立ててください。

### ゲームの流れ

1. ユーザーがカテゴリを選ぶ（キャラクター、有名人、動物など）
2. ユーザーが何かを思い浮かべる
3. AIが「はい/いいえ」で答える質問をする
4. ユーザーがボタンで回答
5. AIが推理して答えを当てる

### 型定義 ([types.ts](https://github.com/isamu/AkinatorGame/blob/main/src/core/types.ts))

ゲームに必要なデータ構造です。

```typescript
// 回答の種類
export type AnswerType = "yes" | "no" | "probably_yes" | "probably_no" | "unknown";

// 質問と回答の履歴
export interface QAEntry {
  question: string;
  answer: AnswerType;
}

// ゲームのフェーズ
export type GamePhase = "intro" | "questioning" | "guessing" | "result";

// カテゴリ
export type Category = "character" | "person" | "animal" | "object" | "place";

// ゲーム状態
export interface AkinatorState {
  phase: GamePhase;
  category?: Category;
  questionCount: number;
  maxQuestions: number;
  qaHistory: QAEntry[];
  guess?: string;
  isCorrect?: boolean;
  score?: number;
  message: string;
}
```

ゲームの状態を一つの `AkinatorState` にまとめることで、View側での表示が簡単になります。

### ツール定義 ([definition.ts](https://github.com/isamu/AkinatorGame/blob/main/src/core/definition.ts))

LLMに「このツールは何ができるか」を伝える定義です。

```typescript
import type { ToolDefinition } from "gui-chat-protocol";

export const TOOL_NAME = "akinator_game";

export const TOOL_DEFINITION: ToolDefinition = {
  type: "function",
  name: TOOL_NAME,
  description:
    "Play an Akinator-style guessing game. Ask yes/no questions to guess what the user is thinking of.",
  parameters: {
    type: "object",
    properties: {
      action: {
        type: "string",
        enum: ["start", "answer", "guess", "reveal"],
        description: "Game action",
      },
      category: {
        type: "string",
        enum: ["character", "person", "animal", "object", "place"],
        description: "Category of thing to guess (required for 'start')",
      },
      answer: {
        type: "string",
        enum: ["yes", "no", "probably_yes", "probably_no", "unknown"],
        description: "User's answer to your question",
      },
      guess: {
        type: "string",
        description: "Your guess of what the user is thinking",
      },
      wasCorrect: {
        type: "boolean",
        description: "Whether your guess was correct",
      },
    },
    required: ["action"],
  },
};
```

**重要なポイント：**

- `description` はLLMが読んで理解できるように書く
- `enum` で選択肢を限定することで、LLMの誤動作を防ぐ
- `action` でゲームの進行を制御

### execute関数 ([plugin.ts](https://github.com/isamu/AkinatorGame/blob/main/src/core/plugin.ts))

LLMがツールを呼び出したときに実行されるメイン処理です。

```typescript
export const executeAkinator = async (
  context: ToolContext,
  args: AkinatorArgs,
): Promise<ToolResult<AkinatorData, AkinatorJsonData>> => {
  const { action, category, answer, guess, wasCorrect, actualAnswer } = args;

  // 前回の状態を取得
  const currentResult = context.currentResult as ToolResult<AkinatorData, AkinatorJsonData> | null;
  const prevState = currentResult?.data?.state;

  let state: AkinatorState;
  let instructions: string;

  switch (action) {
    case "start": {
      // ゲーム開始
      state = createInitialState(category);
      instructions = "Game started! Ask your first yes/no question.";
      break;
    }

    case "answer": {
      // ユーザーの回答を記録
      state = {
        ...prevState,
        questionCount: prevState.questionCount + 1,
        qaHistory: [...prevState.qaHistory, { question: prevState.currentQuestion, answer }],
      };
      instructions = "Analyze answers and ask another question or make a guess.";
      break;
    }

    case "guess": {
      // AIの推理
      state = {
        ...prevState,
        phase: "guessing",
        guess,
        message: `私の予想は...「${guess}」です！`,
      };
      instructions = "Wait for user to confirm if correct.";
      break;
    }

    case "reveal": {
      // 結果表示
      const score = calculateScore(prevState.questionCount, wasCorrect);
      state = {
        ...prevState,
        phase: "result",
        isCorrect: wasCorrect,
        score,
      };
      break;
    }
  }

  return {
    toolName: TOOL_NAME,
    data: { state, message: state.message },      // View用
    jsonData: { state, availableActions },         // LLM用
    message: state.message,
    instructions,                                  // LLMへの次の指示
    updating: action !== "start",                  // 既存の結果を更新
  };
};
```

**ToolResultの重要フィールド：**

| フィールド | 用途 |
|-----------|------|
| `data` | Viewコンポーネント用（LLMには見えない） |
| `jsonData` | LLMに見せたいデータ |
| `instructions` | LLMへの次のアクション指示 |
| `updating` | true の場合、新しい結果を追加せず既存を更新 |

### Viewコンポーネント ([View.vue](https://github.com/isamu/AkinatorGame/blob/main/src/vue/View.vue))

ゲームUIを表示するVueコンポーネントです。

```vue
<template>
  <div class="p-8 bg-gradient-to-br from-purple-900 to-blue-900">
    <!-- ヘッダー -->
    <div class="text-center mb-8">
      <div class="text-6xl mb-4">🔮</div>
      <h2 class="text-white text-3xl font-bold">アキネイター</h2>
    </div>

    <!-- メッセージ -->
    <div class="bg-white/10 rounded-xl p-6 mb-6">
      <p class="text-white text-xl text-center">
        {{ gameData.state.message }}
      </p>
    </div>

    <!-- 回答ボタン（質問フェーズ） -->
    <div v-if="gameData.state.phase === 'questioning'" class="grid grid-cols-2 gap-3">
      <button
        v-for="answer in answerOptions"
        :key="answer.value"
        @click="sendAnswer(answer.value)"
        class="py-4 rounded-xl font-bold"
        :class="answer.class"
      >
        {{ answer.label }}
      </button>
    </div>

    <!-- 正解確認（推理フェーズ） -->
    <div v-if="gameData.state.phase === 'guessing'" class="grid grid-cols-2 gap-4">
      <button @click="confirmGuess(true)" class="bg-green-500 text-white">
        🎉 正解！
      </button>
      <button @click="confirmGuess(false)" class="bg-red-500 text-white">
        😅 違う
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch } from "vue";

const props = defineProps<{
  selectedResult: ToolResult;
  sendTextMessage: (text?: string) => void;  // チャットにメッセージ送信
}>();

const gameData = ref<AkinatorData | null>(null);

// 結果が更新されたらデータを反映
watch(
  () => props.selectedResult,
  (newResult) => {
    if (newResult?.toolName === TOOL_NAME && newResult.data) {
      gameData.value = newResult.data as AkinatorData;
    }
  },
  { immediate: true }
);

// 回答を送信
function sendAnswer(answer: string) {
  const label = { yes: "はい", no: "いいえ", ... }[answer];
  props.sendTextMessage(label);  // チャットにテキストとして送信
}

// 推理の正誤を送信
function confirmGuess(wasCorrect: boolean) {
  if (wasCorrect) {
    props.sendTextMessage("正解！当たりです！");
  }
}
</script>
```

**重要なポイント：**

- `sendTextMessage` でユーザーの操作をチャットに送信
- LLMがそのテキストを解釈して次のアクションを決定
- ボタン操作 → テキスト送信 → LLM処理 → ツール呼び出し → View更新

## ゲームの流れを図解

```
[ユーザー] "アキネイターで遊ぼう"
      ↓
[LLM] akinator_game(action="start", category="character")
      ↓
[View] 「キャラクターを当てます！何か思い浮かべて」
      ↓
[LLM] 「日本のキャラクターですか？」
      ↓
[View] ボタン表示 [はい] [いいえ] [たぶん...] [わからない]
      ↓
[ユーザー] 「はい」ボタンをクリック
      ↓
[View] sendTextMessage("はい")
      ↓
[LLM] akinator_game(action="answer", answer="yes")
      ↓
[LLM] 「アニメのキャラクターですか？」
      ↓
   ... 繰り返し ...
      ↓
[LLM] akinator_game(action="guess", guess="ドラえもん")
      ↓
[View] 「私の予想は...ドラえもん！」 [正解！] [違う]
      ↓
[ユーザー] 「正解！」クリック
      ↓
[View] 「🎉 正解！ 8問で当てました！ スコア: 80点」
```

# プラグインの配布

開発したプラグインをMulmoChatで使えるようにする方法を紹介します。

## ビルドとGitHubへの登録

プラグインが完成したら、以下の手順で配布できます：

```bash
# 1. ビルド
yarn build

# 2. distディレクトリをgitに追加
git add -f dist/
git commit -m "Build plugin"
git push
```

**ポイント**: 通常 `dist/` は `.gitignore` に含まれますが、GitHubから直接インストールする場合はビルド済みファイルをリポジトリに含める必要があります。

## MulmoChatへのインストール

MulmoChatにプラグインを追加するには、3つのステップが必要です。

### 1. パッケージのインストール

GitHubから直接インストールする場合：

```bash
yarn add github:username/AkinatorGame
```

npmに公開済みの場合：

```bash
yarn add guichat-plugin-akinator
```

### 2. src/main.ts - CSSのインポート

プラグインのスタイルを読み込みます：

```typescript
// 既存のimport文の後に追加
import "guichat-plugin-akinator/style.css";

createApp(App).use(router).mount("#app");
```

**重要**: CSSをインポートしないと、プラグインのスタイルが適用されません。

### 3. src/tools/index.ts - プラグインの登録

プラグインをインポートしてリストに追加します：

```typescript
// インポート追加
import AkinatorPlugin from "guichat-plugin-akinator/vue";

const pluginList = [
  // 既存のプラグイン...
  OthelloPlugin,
  TicTacToePlugin,
  // 新しいプラグインを追加
  AkinatorPlugin,
];

export const getPluginList = () => pluginList;
```

### まとめ: 3ステップ

| ファイル | 変更内容 |
|----------|----------|
| `package.json` | 依存関係を追加 |
| `src/main.ts` | CSSをインポート |
| `src/tools/index.ts` | プラグインをインポート＆リストに追加 |

```bash
# 実際のコマンド
yarn add github:username/AkinatorGame
# その後、main.ts と tools/index.ts を編集
yarn dev  # 動作確認
```

## 開発環境 vs MulmoChat

| 機能 | 開発環境 (yarn dev) | MulmoChat |
|------|---------------------|-----------|
| 入力方式 | テキストのみ | テキスト + **音声** |
| APIモード | Mock / Real API | Real API |
| プラグイン | 単体テスト | 複数プラグイン連携 |

開発環境はテキスト入力のみですが、**MulmoChatでは音声入力も使えます**。

Akinatorの場合、音声で「アキネイターで遊ぼう」と話しかけるだけでゲームが始まり、「はい」「いいえ」も音声で答えられます。ボタン操作と音声を組み合わせた、より自然な体験が可能です。

```
開発環境:
[テキスト入力] → [LLM] → [GUI表示]

MulmoChat:
[テキスト or 音声] → [LLM] → [GUI表示 + 音声応答]
```

# まとめ

MulmoChatとGUIChatPluginTemplateを使えば、LLMとインタラクティブなGUIを組み合わせたアプリケーションを簡単に作れます。

- **MulmoChat**: LLM + GUI プラグインのチャットアプリ
- **GUIChatPluginTemplate**: プラグイン開発テンプレート
- **Akinator**: 実際のゲーム実装例

ぜひ自分だけのプラグインを作ってみてください！

## リンク

- [MulmoChat](https://github.com/receptron/MulmoChat) - GUI プラグイン対応チャットアプリ
- [GUIChatPluginTemplate](https://github.com/receptron/GUIChatPluginTemplate) - プラグイン開発テンプレート
- [AkinatorGame](https://github.com/isamu/AkinatorGame) - 本記事で作成したプラグイン

---

ご質問やフィードバックがあれば、各リポジトリのIssueでお待ちしています！
