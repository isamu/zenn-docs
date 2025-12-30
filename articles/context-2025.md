---
title: "Context Management 2025 - 完全ガイド"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech]
published: true
publication_name: "singularity"
---

このディレクトリには、**2024-2025年におけるContext Management（コンテキスト管理）の進化**と、それに基づく**モダンなアーキテクチャ設計**に関する包括的なドキュメントが含まれています。

Roo Code、Anthropic Claude、および最新のAIエージェント設計の実装パターンから抽出したベストプラクティスを統合しています。

# 01. Context Management の進化 (2023-2024 → 2025)

## はじめに

2023-2024年のLLMアプリケーションは、比較的単純なContext Management（コンテキスト管理）で動作していました。しかし、以下の課題が顕在化しました：

- **長期セッションでの情報損失**: 古いメッセージを削除すると重要な文脈が失われる
- **トークン制限への機械的対応**: しきい値を超えたら先頭から削除するだけ
- **復元不可能**: 一度削除したデータは戻せない
- **ツールの無秩序**: すべてのツールが常に見える状態
- **プロトコル非対応**: Native Toolsなどの新しい仕様に対応できない

2024年後半から2025年にかけて、**Context Engineering**という概念が確立され、より高度で柔軟なアプローチが標準になりました。

このドキュメントでは、この1年間の進化を7つのポイントに分けて詳しく解説します。

---

## 1年前のシンプルなアプローチ

### アーキテクチャ

```typescript
// 2023-2024年の典型的な実装
interface OldContextManagement {
  // チャットログをそのまま配列で保持
  messages: Message[]

  // 利用可能なツール一覧（静的）
  tools: Tool[]

  // 設定
  config: {
    maxMessages: number      // 最大メッセージ数（例: 100）
    contextWindow: number    // トークン制限（例: 4096）
  }
}

class SimpleContextManager {
  private messages: Message[] = []

  // メッセージ追加
  addMessage(message: Message) {
    this.messages.push(message)

    // 上限チェック（単純）
    if (this.messages.length > this.config.maxMessages) {
      // 古いものから削除
      this.messages.shift()  // ❌ 永久に失われる
    }
  }

  // API送信用
  getMessages(): Message[] {
    return this.messages  // そのまま返す
  }

  // ツール取得
  getTools(): Tool[] {
    return this.tools  // 常に同じ
  }
}
```

### 問題点

#### 1. 情報の永久的な損失

```typescript
// メッセージが100件を超えると...
messages.length = 101

// 最初のメッセージを削除
messages.shift()  // ❌ このメッセージは永久に失われる

// 結果: 重要な文脈情報が消える
// - プロジェクトの初期要件
// - ユーザーの重要な指示
// - 過去の決定事項
```

#### 2. トークンカウントの不正確さ

```typescript
// 単純なカウント
const tokenCount = messages.reduce(
  (sum, msg) => sum + msg.content.length / 4,  // ❌ 粗い推定
  0
)

// 問題:
// - 実際のトークン数と大きくずれる
// - tool_use/tool_resultのトークン数を考慮していない
// - 画像のトークン数が不正確
```

#### 3. 復元不可能

```typescript
// 編集や削除
messages = messages.filter(msg => msg.id !== deleteId)  // ❌

// 問題:
// - 誤って削除したら戻せない
// - チェックポイント機能を実装できない
// - デバッグが困難
```

#### 4. ツールの無秩序

```typescript
const tools = [
  readFileTool,
  writeFileTool,
  deleteFileTool,     // ❌ 常に見える（危険）
  executeTool,        // ❌ 常に見える（危険）
  searchTool,
  analyzeTool
]

// 問題:
// - フェーズに関係なく全ツールが見える
// - 権限チェックなし
// - 誤操作のリスク
// - トークンの無駄
```

---

## 7つの主要な進化

### 進化1: 非破壊的管理 → データ損失ゼロ

#### Before: 物理削除

```typescript
// 古い方法: 削除したら戻せない
class OldManager {
  deleteOldMessages(cutoff: number) {
    this.messages = this.messages.filter(
      msg => msg.timestamp > cutoff
    )
    // ❌ 削除されたメッセージは永久に失われる
  }
}
```

#### After: タグ付け管理

```typescript
// 新しい方法（Roo Code方式）
interface ApiMessage {
  role: "user" | "assistant"
  content: string | ContentBlock[]
  ts: number

  // 非破壊的なタグ
  condenseParent?: string      // このサマリーに置き換えられた
  truncationParent?: string    // このマーカーで隠された
  isSummary?: boolean          // サマリーメッセージか
  isTruncationMarker?: boolean // トランケーションマーカーか
  condenseId?: string          // サマリーの一意ID
  truncationId?: string        // マーカーの一意ID
}

class ModernManager {
  // メッセージは削除せずタグ付け
  hideMessages(messagesToHide: Message[], markerId: string) {
    return this.messages.map(msg => {
      if (messagesToHide.includes(msg)) {
        // タグ付けするだけ（削除しない）
        return {
          ...msg,
          truncationParent: markerId  // ✅ タグ付け
        }
      }
      return msg
    })
  }

  // API送信時にフィルタリング
  getEffectiveHistory(messages: ApiMessage[]): ApiMessage[] {
    // 存在するマーカーIDを収集
    const existingMarkers = new Set(
      messages
        .filter(m => m.isTruncationMarker && m.truncationId)
        .map(m => m.truncationId)
    )

    // truncationParentを持ち、対応するマーカーが存在する場合のみ除外
    return messages.filter(msg => {
      if (msg.truncationParent) {
        return !existingMarkers.has(msg.truncationParent)
      }
      return true  // それ以外は含める
    })
  }

  // 復元（マーカー削除でタグをクリーンアップ）
  restoreMessages(markerId: string) {
    // マーカーを削除
    const withoutMarker = this.messages.filter(
      msg => !(msg.isTruncationMarker && msg.truncationId === markerId)
    )

    // 孤立したタグをクリーンアップ
    return withoutMarker.map(msg => {
      if (msg.truncationParent === markerId) {
        const { truncationParent, ...rest } = msg
        return rest  // ✅ タグを削除して復元
      }
      return msg
    })
  }
}
```

#### 具体例: マーカーによる隠蔽

```typescript
// 初期状態
const messages = [
  { id: 1, role: "user", content: "Hello" },
  { id: 2, role: "assistant", content: "Hi!" },
  { id: 3, role: "user", content: "How are you?" },
  { id: 4, role: "assistant", content: "I'm good!" },
  { id: 5, role: "user", content: "Great!" }
]

// トランケーション実行（50%削除）
const truncationId = "uuid-abc-123"
const tagged = [
  { id: 1, role: "user", content: "Hello" },  // 最初は保持
  { id: 2, role: "assistant", content: "Hi!", truncationParent: "uuid-abc-123" },  // タグ
  { id: 3, role: "user", content: "How are you?", truncationParent: "uuid-abc-123" },  // タグ
  {
    role: "user",
    content: "[Truncation: 2 messages hidden]",
    isTruncationMarker: true,
    truncationId: "uuid-abc-123",
    ts: messages[3].ts - 1  // 保持メッセージの直前
  },  // マーカー挿入
  { id: 4, role: "assistant", content: "I'm good!" },
  { id: 5, role: "user", content: "Great!" }
]

// API送信時
const effective = getEffectiveHistory(tagged)
// => [
//   { id: 1, ... },
//   { role: "user", content: "[Truncation: 2 messages hidden]", ... },
//   { id: 4, ... },
//   { id: 5, ... }
// ]

// 復元（マーカー削除）
const restored = restoreMessages("uuid-abc-123")
// => 元の5件すべてが復元される ✅
```

#### 利点

✅ **データ損失なし**: すべてのメッセージが保持される
✅ **復元可能**: いつでも過去の状態に戻せる
✅ **チェックポイント統合**: Shadow Gitと完全に統合可能
✅ **デバッグ容易**: 全履歴を確認できる
✅ **監査証跡**: すべての操作を追跡可能

---

### 進化2: 単純削除 → 知的圧縮（Condensation）

#### Before: 機械的削除

```typescript
// 古い方法
class OldCompressor {
  compress(messages: Message[], limit: number) {
    const tokenCount = this.countTokens(messages)

    if (tokenCount > limit) {
      // 先頭から削除
      const toRemove = Math.floor(messages.length * 0.3)  // 30%削除
      return messages.slice(toRemove)  // ❌ 機械的に削除
    }

    return messages
  }
}

// 問題:
// - 重要な情報も容赦なく削除
// - 文脈が分断される
// - 情報が完全に失われる
```

#### After: AI要約 + フォールバック

```typescript
// 新しい方法（二段階アプローチ）
class ModernCompressor {
  async manageContext(
    messages: ApiMessage[],
    tokenCount: number,
    contextWindow: number
  ): Promise<ContextResult> {
    const threshold = contextWindow * 0.75  // 75%

    if (tokenCount < threshold) {
      return { messages }  // 何もしない
    }

    // 第1段階: Condensation（AI要約）
    if (this.config.autoCondenseContext) {
      try {
        const result = await this.condense(messages)

        // 成功チェック
        if (result.newTokens < tokenCount) {
          return {
            messages: result.messages,
            summary: result.summary,
            condenseId: result.condenseId,
            reduction: tokenCount - result.newTokens
          }
        }
      } catch (error) {
        console.warn("Condensation failed, falling back to truncation")
        // フォールスルー
      }
    }

    // 第2段階: Truncation（スライディングウィンドウ）
    return this.truncate(messages, 0.5)  // 50%削除
  }

  private async condense(
    messages: ApiMessage[]
  ): Promise<CondenseResult> {
    // 保持するメッセージ（最新3件）
    const keepMessages = messages.slice(-3)

    // 要約対象
    const toSummarize = messages.slice(0, -3)

    if (toSummarize.length === 0) {
      throw new Error("Not enough messages to summarize")
    }

    // LLMで要約生成
    const summary = await this.llm.generate(`
以下の会話を要約してください。

重要な情報:
1. Previous Conversation: 会話の流れ
2. Current Work: 現在作業していたこと
3. Key Technical Concepts: 技術的な概念
4. Relevant Files and Code: 関連ファイル
5. Problem Solving: 解決した問題
6. Pending Tasks: 未完了のタスク

会話履歴:
${toSummarize.map(m => \`\${m.role}: \${m.content}\`).join('\\n')}

500語以内で要約してください。
`)

    // サマリーメッセージ作成
    const condenseId = crypto.randomUUID()
    const summaryMessage: ApiMessage = {
      role: "assistant",
      content: [
        {
          type: "reasoning",
          text: "Condensing conversation context. The summary captures key information."
        },
        {
          type: "text",
          text: summary
        }
      ],
      ts: keepMessages[0].ts - 1,  // 保持メッセージの直前
      isSummary: true,
      condenseId
    }

    // 中間メッセージにタグ付け
    const tagged = messages.map((msg, i) => {
      if (i === 0) return msg  // 最初は保持
      if (i >= messages.length - 3) return msg  // 最後3件は保持

      // 中間はタグ付け
      return {
        ...msg,
        condenseParent: condenseId
      }
    })

    // サマリー挿入
    const result = [
      tagged[0],  // 最初のメッセージ
      ...tagged.slice(1, -3),  // タグ付きの中間メッセージ
      summaryMessage,  // サマリー
      ...keepMessages  // 最新3件
    ]

    // トークン数計算
    const newTokens = await this.countTokens(result)

    return {
      messages: result,
      summary,
      condenseId,
      newTokens
    }
  }
}
```

#### 具体例: Condensation の動作

```typescript
// 初期状態（12メッセージ）
const messages = [
  { id: 0, role: "user", content: "プロジェクト初期化して" },
  { id: 1, role: "assistant", content: "npm init実行しました" },
  { id: 2, role: "user", content: "Reactをインストール" },
  { id: 3, role: "assistant", content: "インストール完了" },
  { id: 4, role: "user", content: "App.tsxを作成" },
  { id: 5, role: "assistant", content: "作成しました" },
  { id: 6, role: "user", content: "ボタンコンポーネント追加" },
  { id: 7, role: "assistant", content: "Button.tsxを作成" },
  { id: 8, role: "user", content: "スタイル調整" },
  { id: 9, role: "assistant", content: "CSSを更新" },
  { id: 10, role: "user", content: "ビルド実行" },  // 最新3件
  { id: 11, role: "assistant", content: "成功しました" }  // 最新3件
]

// Condensation実行後
const condensed = [
  { id: 0, role: "user", content: "プロジェクト初期化して" },  // 最初は保持

  // 中間メッセージ（タグ付き）
  { id: 1, role: "assistant", content: "...", condenseParent: "uuid-123" },
  { id: 2, role: "user", content: "...", condenseParent: "uuid-123" },
  // ... id: 3-8 すべて condenseParent: "uuid-123"
  { id: 9, role: "assistant", content: "...", condenseParent: "uuid-123" },

  // サマリーメッセージ
  {
    role: "assistant",
    content: [
      { type: "reasoning", text: "Condensing conversation context..." },
      {
        type: "text",
        text: `
Previous Conversation: ユーザーがReactプロジェクトのセットアップを依頼
Current Work: UIコンポーネント（Button.tsx）の作成とスタイリングを完了
Key Technical Concepts: React, TypeScript, CSS
Relevant Files:
  - package.json: 依存関係
  - App.tsx: メインコンポーネント
  - Button.tsx: ボタンコンポーネント
Problem Solving: プロジェクト初期化からコンポーネント作成まで順調に完了
Pending Tasks: なし（ビルド実行予定）
`
      }
    ],
    isSummary: true,
    condenseId: "uuid-123"
  },

  // 最新3件（保持）
  { id: 10, role: "user", content: "ビルド実行" },
  { id: 11, role: "assistant", content: "成功しました" },
  { id: 12, role: "user", content: "テスト実行して" }  // 新しいメッセージ
]

// API送信時（フィルタ後）
const effective = getEffectiveHistory(condensed)
// => [
//   { id: 0, ... },
//   { role: "assistant", content: [...], isSummary: true },  // サマリー
//   { id: 10, ... },
//   { id: 11, ... },
//   { id: 12, ... }
// ]

// トークン削減
// Before: 12メッセージ ≈ 3000トークン
// After:  5メッセージ ≈ 1000トークン
// 削減率: 67% ✅
```

#### 利点

✅ **大幅なトークン削減**: 70-90%の削減が可能
✅ **情報保持**: 重要な情報はサマリーに含まれる
✅ **文脈の連続性**: 会話の流れが維持される
✅ **失敗時のフォールバック**: Truncationで確実に削減
✅ **復元可能**: サマリーを削除すれば元に戻る

---

### 進化3: フラットな履歴 → 階層的State管理

#### Before: フラットな配列

```typescript
// 古い方法
interface OldState {
  messages: Message[]  // すべて同じレベル
  currentTool: Tool | null
  userInput: string
}

// 問題:
// - すべてが同じ優先度
// - 古い情報と新しい情報の区別なし
// - 圧縮戦略が限定的
```

#### After: 階層的構造（Context Engineering）

```typescript
// 新しい方法（Anthropic方式）
interface LayeredState {
  // L0: System / Policy（最高優先度、不変）
  system: {
    role: string                    // システムロール
    policies: Policy[]              // 安全ポリシー
    constraints: Constraint[]       // 制約
    auditRequirements: AuditConfig  // 監査要件
  }

  // L1: Task contract（タスク定義）
  task: {
    goal: string                    // 最終目標
    successCriteria: Criterion[]    // 成功条件
    outputFormat: Schema            // 出力形式
    deadline?: Date                 // 期限
  }

  // L2: Runtime State（実行時状態）
  runtime: {
    phase: "planning" | "executing" | "reflecting" | "verifying"
    permissions: Permission[]       // 現在の権限
    environment: "dev" | "staging" | "prod"
    failureHistory: Failure[]       // 失敗履歴
  }

  // L3: Memory（記憶、複数種類）
  memory: {
    // 短期ワーキングメモリ
    shortTerm: {
      workingVariables: Record<string, any>  // 作業変数
      recentTurns: Message[]                 // 直近3-5ターン
    }

    // エピソード記憶（出来事）
    episodic: {
      summaries: Summary[]      // 要約
      decisions: Decision[]     // 意思決定
      milestones: Milestone[]   // マイルストーン
    }

    // 意味記憶（事実）
    semantic: {
      facts: Fact[]             // 確定事実
      definitions: Definition[] // 定義
      preferences: Preference[] // ユーザー設定
      projectKnowledge: Knowledge[]  // プロジェクト知識
    }

    // 手続き記憶（やり方）
    procedural: {
      skills: Skill[]           // スキル
      templates: Template[]     // テンプレート
      checklists: Checklist[]   // チェックリスト
      procedures: Procedure[]   // 手順
    }
  }

  // L4: Evidence（観測・検索結果）
  evidence: {
    observations: Observation[]  // ツール実行結果
    ragResults: RAGResult[]      // RAG検索結果
    citations: Citation[]        // 引用
    measurements: Measurement[]  // 測定値
  }

  // L5: Work Buffer（作業領域、最も変動的）
  workBuffer: {
    plan: Step[]                // 計画
    diff: Change[]              // 差分
    hypotheses: Hypothesis[]    // 仮説
    draft: string              // 下書き
  }
}
```

#### Context Builder: 階層を組み立てる

```typescript
class ContextBuilder {
  async build(state: LayeredState): Promise<Context> {
    // 各層を構造化されたXMLで構築
    return `
<context>
  <!-- L0: System（常に最優先） -->
  <system>
    <role>${state.system.role}</role>
    <policies>
      ${state.system.policies.map(p => `<policy>${p}</policy>`).join('\n')}
    </policies>
  </system>

  <!-- L1: Task -->
  <task>
    <goal>${state.task.goal}</goal>
    <success_criteria>
      ${state.task.successCriteria.map(c => `<criterion>${c}</criterion>`).join('\n')}
    </success_criteria>
  </task>

  <!-- L2: Runtime State -->
  <runtime_state>
    <phase>${state.runtime.phase}</phase>
    <environment>${state.runtime.environment}</environment>
    <permissions>
      ${state.runtime.permissions.map(p => `<permission>${p}</permission>`).join('\n')}
    </permissions>
  </runtime_state>

  <!-- L3: Memory（圧縮・フィルタ済み） -->
  <memory>
    <!-- 短期ワーキングメモリ（最新） -->
    <short_term>
      <recent_turns>
        ${state.memory.shortTerm.recentTurns.map(t =>
          `<turn role="${t.role}">${t.content}</turn>`
        ).join('\n')}
      </recent_turns>
    </short_term>

    <!-- エピソード記憶（要約済み） -->
    <episodic>
      ${state.memory.episodic.summaries
        .filter(s => this.isRelevant(s, state.task))
        .map(s => `<summary>${s.text}</summary>`)
        .join('\n')}
    </episodic>

    <!-- 意味記憶（関連する事実のみ） -->
    <semantic>
      ${state.memory.semantic.facts
        .filter(f => this.isRelevant(f, state.task))
        .map(f => `<fact source="${f.source}">${f.content}</fact>`)
        .join('\n')}
    </semantic>
  </memory>

  <!-- L4: Evidence（新しい観測を優先） -->
  <evidence>
    <recent_observations>
      ${state.evidence.observations
        .filter(o => Date.now() - o.timestamp < 60000)  // 1分以内
        .map(o => `
          <observation tool="${o.tool}" timestamp="${o.timestamp}">
            ${o.result}
          </observation>
        `).join('\n')}
    </recent_observations>

    <rag_results>
      ${state.evidence.ragResults.map(r => `
        <result relevance="${r.relevance}">
          <source>${r.source}</source>
          <content>${r.content}</content>
        </result>
      `).join('\n')}
    </rag_results>
  </evidence>

  <!-- L5: Work Buffer（現在の作業） -->
  <work_buffer>
    <plan>
      ${state.workBuffer.plan.map((step, i) => `
        <step id="${i}" status="${step.status}">
          ${step.description}
        </step>
      `).join('\n')}
    </plan>
  </work_buffer>
</context>
`
  }

  private isRelevant(item: any, task: Task): boolean {
    // ベクトル類似度やキーワードマッチで関連性判定
    const similarity = this.cosineSimilarity(
      this.embed(item.content),
      this.embed(task.goal)
    )

    return similarity > 0.3  // 閾値
  }
}
```

#### トークン予算配分

```typescript
class TokenBudgetAllocator {
  allocate(state: LayeredState, maxTokens: number): BudgetAllocation {
    const budget = maxTokens * 0.9  // 10%バッファ

    // 階層別の予算配分
    return {
      system: {
        budget: budget * 0.05,      // 5% - 重要だが短い
        priority: Priority.CRITICAL
      },

      task: {
        budget: budget * 0.05,      // 5%
        priority: Priority.CRITICAL
      },

      runtime: {
        budget: budget * 0.03,      // 3%
        priority: Priority.HIGH
      },

      memory: {
        shortTerm: {
          budget: budget * 0.20,    // 20% - 直近の文脈
          priority: Priority.HIGH
        },
        episodic: {
          budget: budget * 0.15,    // 15% - 要約済み
          priority: Priority.MEDIUM
        },
        semantic: {
          budget: budget * 0.10,    // 10% - 関連する事実
          priority: Priority.MEDIUM
        },
        procedural: {
          budget: budget * 0.07,    // 7% - 必要なスキル
          priority: Priority.MEDIUM
        }
      },

      evidence: {
        observations: {
          budget: budget * 0.25,    // 25% - 最新の観測
          priority: Priority.HIGH
        },
        ragResults: {
          budget: budget * 0.10,    // 10%
          priority: Priority.MEDIUM
        }
      },

      workBuffer: {
        budget: budget * 0.10,      // 10%
        priority: Priority.MEDIUM
      }
    }
  }

  async fitToBudget(
    content: any,
    budgetTokens: number
  ): Promise<any> {
    const currentTokens = await this.countTokens(content)

    if (currentTokens <= budgetTokens) {
      return content  // 予算内
    }

    // 予算超過 → 圧縮
    if (Array.isArray(content)) {
      // 配列の場合: 優先度でソート→上位を保持
      const withPriority = content.map(item => ({
        item,
        priority: this.calculatePriority(item)
      }))

      withPriority.sort((a, b) => b.priority - a.priority)

      // 予算内に収まるまで保持
      let accumulated = 0
      const result = []

      for (const { item } of withPriority) {
        const tokens = await this.countTokens(item)

        if (accumulated + tokens <= budgetTokens) {
          result.push(item)
          accumulated += tokens
        } else {
          break
        }
      }

      return result
    } else if (typeof content === 'string') {
      // 文字列の場合: 要約
      return await this.summarize(content, budgetTokens)
    }

    return content
  }
}
```

#### 利点

✅ **明確な優先度**: どの情報が重要かが明確
✅ **柔軟な圧縮**: 層ごとに最適な戦略を適用
✅ **効率的なトークン使用**: 予算配分により最適化
✅ **拡張性**: 新しい層を追加しやすい
✅ **関連性フィルタ**: タスクに関連する情報のみ保持

---

### 進化4: 静的ツール → 動的ツール投影

#### Before: すべてのツールを常に提供

```typescript
// 古い方法
class OldToolManager {
  getTools(): Tool[] {
    // 常に全ツールを返す
    return [
      readFileTool,
      writeFileTool,
      deleteFileTool,      // ❌ 常に見える
      executeTool,         // ❌ 常に見える
      searchTool,
      analyzeTool,
      deployTool,          // ❌ 常に見える（危険）
      rollbackTool         // ❌ 常に見える（危険）
    ]
  }
}

// 問題:
// 1. セキュリティリスク
// 2. トークンの無駄
// 3. LLMの混乱（選択肢が多すぎる）
// 4. 誤操作のリスク
```

#### After: 状態に応じた動的投影

```typescript
// 新しい方法
class DynamicToolProjector {
  project(
    state: LayeredState,
    allTools: Tool[]
  ): Tool[] {
    let tools = [...allTools]

    // ステップ1: フェーズフィルタリング
    tools = this.filterByPhase(tools, state.runtime.phase)

    // ステップ2: 権限フィルタリング
    tools = this.filterByPermission(tools, state.runtime.permissions)

    // ステップ3: 環境に応じたスキーマ制限
    tools = this.restrictSchema(tools, state.runtime.environment)

    // ステップ4: Tool budget（トークン効率化）
    tools = this.applyToolBudget(tools, state)

    return tools
  }

  // フェーズ別フィルタリング
  private filterByPhase(
    tools: Tool[],
    phase: RuntimeState['phase']
  ): Tool[] {
    const phaseToolMap: Record<string, string[]> = {
      planning: [
        'search',      // 情報収集
        'analyze',     // 分析
        'estimate',    // 見積もり
        'readFile'     // ファイル読み込み
      ],

      executing: [
        'writeFile',   // ファイル書き込み
        'execute',     // 実行
        'modify',      // 修正
        'test'         // テスト
      ],

      reflecting: [
        'evaluate',    // 評価
        'test',        // テスト
        'compare',     // 比較
        'measure'      // 測定
      ],

      verifying: [
        'verify',      // 検証
        'audit',       // 監査
        'approve'      // 承認
      ]
    }

    const allowedCategories = phaseToolMap[phase] || []

    return tools.filter(tool =>
      allowedCategories.includes(tool.category)
    )
  }

  // 権限別フィルタリング
  private filterByPermission(
    tools: Tool[],
    permissions: Permission[]
  ): Tool[] {
    return tools.filter(tool => {
      // ツールに必要な権限をチェック
      const requiredPermissions = tool.requiredPermissions || []

      return requiredPermissions.every(required =>
        permissions.some(p => p.includes(required))
      )
    })
  }

  // 環境に応じたスキーマ制限
  private restrictSchema(
    tools: Tool[],
    environment: RuntimeState['environment']
  ): Tool[] {
    if (environment === 'production') {
      return tools.map(tool => this.addProductionSafety(tool))
    }

    return tools
  }

  private addProductionSafety(tool: Tool): Tool {
    // 例: deleteFileツール
    if (tool.name === 'deleteFile') {
      return {
        ...tool,
        parameters: {
          ...tool.parameters,
          properties: {
            ...tool.parameters.properties,
            path: {
              type: 'string',
              // 本番環境では許可されたパスのみ
              enum: [
                '/tmp/allowed-file-1.txt',
                '/tmp/allowed-file-2.txt'
              ],
              description: 'File path (production: whitelist only)'
            }
          }
        }
      }
    }

    // 例: executeツール
    if (tool.name === 'execute') {
      return {
        ...tool,
        parameters: {
          ...tool.parameters,
          properties: {
            ...tool.parameters.properties,
            command: {
              type: 'string',
              // 本番環境では安全なコマンドのみ
              enum: [
                'npm test',
                'npm run build',
                'npm run lint'
              ],
              description: 'Command (production: safe commands only)'
            }
          }
        }
      }
    }

    return tool
  }

  // Tool budget適用
  private applyToolBudget(
    tools: Tool[],
    state: LayeredState
  ): Tool[] {
    // タスクの緊急度・重要度に応じてツール数を制限
    const urgency = this.calculateUrgency(state.task)

    if (urgency === 'high') {
      // 緊急時は重要なツールのみ
      return tools
        .sort((a, b) => b.priority - a.priority)
        .slice(0, 5)  // 上位5件
    }

    // 通常時はトークン予算内に収める
    const budgetTokens = 2000  // ツール定義用のトークン予算
    let accumulated = 0
    const result: Tool[] = []

    for (const tool of tools) {
      const toolTokens = this.estimateToolTokens(tool)

      if (accumulated + toolTokens <= budgetTokens) {
        result.push(tool)
        accumulated += toolTokens
      } else {
        break
      }
    }

    return result
  }
}
```

#### 具体例: フェーズごとのツール変化

```typescript
const allTools = [
  { name: 'search', category: 'search', priority: 80 },
  { name: 'readFile', category: 'readFile', priority: 90 },
  { name: 'writeFile', category: 'writeFile', priority: 85 },
  { name: 'deleteFile', category: 'deleteFile', priority: 50, requiredPermissions: ['admin'] },
  { name: 'execute', category: 'execute', priority: 70 },
  { name: 'test', category: 'test', priority: 75 },
  { name: 'deploy', category: 'deploy', priority: 60, requiredPermissions: ['deploy'] }
]

// Planning フェーズ
const state1 = {
  runtime: {
    phase: 'planning',
    permissions: ['read', 'search'],
    environment: 'dev'
  }
}

const tools1 = projector.project(state1, allTools)
// => [search, readFile]
// ✅ 情報収集系のみ
// ✅ write/delete/execute は見えない

// Executing フェーズ
const state2 = {
  runtime: {
    phase: 'executing',
    permissions: ['read', 'write', 'execute'],
    environment: 'dev'
  }
}

const tools2 = projector.project(state2, allTools)
// => [readFile, writeFile, execute, test]
// ✅ 実行系が見える
// ✅ deploy は権限不足で見えない

// Executing フェーズ（本番環境）
const state3 = {
  runtime: {
    phase: 'executing',
    permissions: ['read', 'write', 'execute', 'admin'],
    environment: 'production'
  }
}

const tools3 = projector.project(state3, allTools)
// => [
//   readFile,
//   writeFile,
//   deleteFile (with enum restrictions),  // ✅ パス制限
//   execute (with enum restrictions),     // ✅ コマンド制限
//   test
// ]
```

#### 利点

✅ **セキュリティ強化**: フェーズ・権限・環境で制限
✅ **トークン効率**: 必要なツールのみ提供
✅ **誤操作防止**: 危険なツールを隠す
✅ **LLMの精度向上**: 選択肢が絞られて判断しやすい
✅ **監査証跡**: どのツールがいつ使えたか記録

---

### 進化5: tool_use単体 → tool_use/tool_resultペア保持

#### Before: ペアが崩れる

```typescript
// 古い方法: 要約時にtool_useが失われる
async summarize(messages: Message[]): Promise<Message[]> {
  const toSummarize = messages.slice(0, -3)
  const keep = messages.slice(-3)  // 最新3件保持

  const summary = await this.llm.generate(...)

  return [
    { role: 'assistant', content: summary },  // サマリー
    ...keep  // 最新3件
  ]
}

// 問題: 最初のkeepメッセージがtool_resultを含む場合
// Before:
// [
//   { role: 'assistant', content: [..., tool_use] },  // 要約対象
//   { role: 'user', content: [..., tool_result] }     // 保持
// ]

// After:
// [
//   { role: 'assistant', content: 'Summary...' },     // tool_use がない
//   { role: 'user', content: [..., tool_result] }     // ❌ ペアが崩れる
// ]

// エラー: "tool_result requires matching tool_use"
```

#### After: ペア保持（Native Tools対応）

```typescript
// 新しい方法（Roo Code方式）
async summarize(
  messages: Message[],
  useNativeTools: boolean
): Promise<Message[]> {
  const keep = messages.slice(-3)
  const toSummarize = messages.slice(0, -3)

  // Native Tools使用時: tool_useブロック抽出
  let toolUseBlocks: ToolUseBlock[] = []
  let reasoningBlocks: ReasoningBlock[] = []

  if (useNativeTools) {
    const result = this.extractToolBlocks(keep, messages)
    toolUseBlocks = result.toolUseBlocks
    reasoningBlocks = result.reasoningBlocks
  }

  // サマリー生成
  const summaryText = await this.llm.generate(...)

  // サマリーメッセージ構築
  const summaryContent: ContentBlock[] = [
    // 合成reasoning（DeepSeek対応）
    {
      type: 'reasoning',
      text: 'Condensing conversation context. Summary captures key information.'
    },

    // 元のreasoningブロック保持
    ...reasoningBlocks,

    // サマリーテキスト
    {
      type: 'text',
      text: summaryText
    },

    // tool_useブロック保持（重要！）
    ...toolUseBlocks
  ]

  return [
    {
      role: 'assistant',
      content: summaryContent,
      isSummary: true
    },
    ...keep
  ]
}

// tool_useブロック抽出
private extractToolBlocks(
  keepMessages: Message[],
  allMessages: Message[]
): ExtractResult {
  // 最初のkeepメッセージをチェック
  const firstKeep = keepMessages[0]

  // tool_resultを含むか？
  const hasToolResult = this.hasToolResultBlocks(firstKeep)

  if (!hasToolResult) {
    return { toolUseBlocks: [], reasoningBlocks: [] }
  }

  // 直前のassistantメッセージを取得
  const keepStartIndex = allMessages.indexOf(firstKeep)
  const precedingIndex = keepStartIndex - 1

  if (precedingIndex < 0) {
    return { toolUseBlocks: [], reasoningBlocks: [] }
  }

  const precedingMessage = allMessages[precedingIndex]

  if (precedingMessage.role !== 'assistant') {
    return { toolUseBlocks: [], reasoningBlocks: [] }
  }

  // tool_useブロックを抽出
  const toolUseBlocks = this.getToolUseBlocks(precedingMessage)

  // reasoningブロックを抽出（DeepSeek対応）
  const reasoningBlocks = this.getReasoningBlocks(precedingMessage)

  return { toolUseBlocks, reasoningBlocks }
}

private getToolUseBlocks(message: Message): ToolUseBlock[] {
  if (!Array.isArray(message.content)) {
    return []
  }

  return message.content.filter(
    block => block.type === 'tool_use'
  ) as ToolUseBlock[]
}

private getReasoningBlocks(message: Message): ReasoningBlock[] {
  if (!Array.isArray(message.content)) {
    return []
  }

  return message.content.filter(
    block => block.type === 'reasoning'
  ) as ReasoningBlock[]
}
```

#### 具体例: ペア保持の動作

```typescript
// 初期状態
const messages = [
  { id: 1, role: 'user', content: 'ファイルを読んで' },
  { id: 2, role: 'assistant', content: 'readFileツールを使います' },
  // ... 多数のメッセージ ...

  // 直前のassistantメッセージ（tool_use含む）
  {
    id: 98,
    role: 'assistant',
    content: [
      { type: 'reasoning', text: 'Using readFile to check content' },
      {
        type: 'tool_use',
        id: 'tool_abc',
        name: 'readFile',
        input: { path: 'test.js' }
      }
    ]
  },

  // 最初のkeepメッセージ（tool_result含む）
  {
    id: 99,
    role: 'user',
    content: [
      {
        type: 'tool_result',
        tool_use_id: 'tool_abc',
        content: 'console.log("hello")'
      }
    ]
  },  // ← keep開始位置

  { id: 100, role: 'assistant', content: 'ファイル内容を確認しました' },
  { id: 101, role: 'user', content: '次は何をする？' }
]

// 要約実行（useNativeTools: true）
const result = await summarize(messages, true)

// 結果
// [
//   {
//     role: 'assistant',
//     content: [
//       // 合成reasoning
//       { type: 'reasoning', text: 'Condensing conversation context...' },
//
//       // 元のreasoningブロック（保持）
//       { type: 'reasoning', text: 'Using readFile to check content' },
//
//       // サマリーテキスト
//       { type: 'text', text: 'Previous conversation: ユーザーがファイル読み込みを依頼...' },
//
//       // tool_useブロック（保持！）
//       {
//         type: 'tool_use',
//         id: 'tool_abc',
//         name: 'readFile',
//         input: { path: 'test.js' }
//       }
//     ],
//     isSummary: true
//   },
//
//   // keep（最新3件）
//   {
//     id: 99,
//     role: 'user',
//     content: [
//       {
//         type: 'tool_result',
//         tool_use_id: 'tool_abc',  // ✅ 対応するtool_useがサマリーに含まれている
//         content: 'console.log("hello")'
//       }
//     ]
//   },
//   { id: 100, role: 'assistant', content: '...' },
//   { id: 101, role: 'user', content: '...' }
// ]

// ✅ tool_use/tool_resultペアが維持される
// ✅ Native Toolsプロトコルに準拠
// ✅ DeepSeek対応（reasoningブロック含む）
```

#### 利点

✅ **Native Toolsプロトコル準拠**: tool_use/resultペア維持
✅ **DeepSeek/Z.ai対応**: reasoning_contentを保持
✅ **API互換性**: エラーなく動作
✅ **文脈保持**: ツール呼び出しの意図が残る

---

### 進化6: 直接削除 → MessageManager統合

#### Before: 直接操作（危険）

```typescript
// 古い方法: 直接削除
class OldMessageHandler {
  deleteMessage(messageId: string) {
    // clineMessagesから削除
    this.clineMessages = this.clineMessages.filter(
      m => m.id !== messageId
    )  // ❌ 直接削除

    // apiConversationHistoryから削除
    this.apiConversationHistory = this.apiConversationHistory.filter(
      m => m.id !== messageId
    )  // ❌ 直接削除

    // 問題:
    // 1. サマリーが孤立する
    // 2. マーカーが孤立する
    // 3. タグが孤立する
    // 4. チェックポイントと不整合
  }
}
```

#### After: MessageManager統合

```typescript
// 新しい方法（Roo Code方式）
class MessageManager {
  constructor(
    private state: LayeredState,
    private checkpointService: CheckpointService
  ) {}

  async rewindToTimestamp(
    ts: number,
    options: RewindOptions = {}
  ): Promise<void> {
    const { includeTargetMessage = false } = options

    // ステップ1: 削除されるコンテキストイベントIDを収集
    const removedIds = this.collectRemovedContextEventIds(ts)

    // ステップ2: clineMessagesを削除
    await this.truncateClineMessages(ts, includeTargetMessage)

    // ステップ3: apiConversationHistoryを削除 + クリーンアップ
    await this.truncateApiHistory(ts, removedIds)

    // ステップ4: チェックポイント同期
    await this.syncCheckpoint(ts)
  }

  // コンテキストイベントID収集
  private collectRemovedContextEventIds(
    cutoffTs: number
  ): ContextEventIds {
    const condenseIds = new Set<string>()
    const truncationIds = new Set<string>()

    // cutoff以降のclineMessagesを走査
    for (const msg of this.state.clineMessages) {
      if (msg.ts >= cutoffTs) {
        // condense_contextイベント
        if (msg.say === 'condense_context' && msg.contextCondense?.condenseId) {
          condenseIds.add(msg.contextCondense.condenseId)
        }

        // sliding_window_truncationイベント
        if (msg.say === 'sliding_window_truncation' && msg.contextTruncation?.truncationId) {
          truncationIds.add(msg.contextTruncation.truncationId)
        }
      }
    }

    return { condenseIds, truncationIds }
  }

  // apiConversationHistory削除 + クリーンアップ
  private async truncateApiHistory(
    cutoffTs: number,
    removedIds: ContextEventIds
  ): Promise<void> {
    let history = [...this.state.apiConversationHistory]

    // ステップ1: レースコンディション対策（タイムスタンプ調整）
    const actualCutoff = this.adjustCutoffTimestamp(history, cutoffTs)

    // ステップ2: タイムスタンプでフィルタ
    history = history.filter(m => !m.ts || m.ts < actualCutoff)

    // ステップ3: 孤立したサマリー削除
    if (removedIds.condenseIds.size > 0) {
      history = history.filter(msg => {
        if (msg.isSummary && msg.condenseId && removedIds.condenseIds.has(msg.condenseId)) {
          console.log(`[MessageManager] Removing orphaned summary: ${msg.condenseId}`)
          return false  // 削除
        }
        return true
      })
    }

    // ステップ4: 孤立したマーカー削除
    if (removedIds.truncationIds.size > 0) {
      history = history.filter(msg => {
        if (msg.isTruncationMarker && msg.truncationId && removedIds.truncationIds.has(msg.truncationId)) {
          console.log(`[MessageManager] Removing orphaned marker: ${msg.truncationId}`)
          return false  // 削除
        }
        return true
      })
    }

    // ステップ5: タグクリーンアップ
    history = this.cleanupOrphanedTags(history)

    // 保存
    this.state.apiConversationHistory = history
    await this.saveApiHistory(history)
  }

  // 孤立したタグのクリーンアップ
  private cleanupOrphanedTags(messages: Message[]): Message[] {
    // 存在するサマリー/マーカーIDを収集
    const existingCondenseIds = new Set(
      messages
        .filter(m => m.isSummary && m.condenseId)
        .map(m => m.condenseId)
    )

    const existingTruncationIds = new Set(
      messages
        .filter(m => m.isTruncationMarker && m.truncationId)
        .map(m => m.truncationId)
    )

    // 孤立したタグを削除
    return messages.map(msg => {
      let needsUpdate = false

      // condenseParentチェック
      if (msg.condenseParent && !existingCondenseIds.has(msg.condenseParent)) {
        needsUpdate = true
      }

      // truncationParentチェック
      if (msg.truncationParent && !existingTruncationIds.has(msg.truncationParent)) {
        needsUpdate = true
      }

      if (!needsUpdate) {
        return msg
      }

      // タグを削除して復元
      const { condenseParent, truncationParent, ...rest } = msg
      const result: Message = rest

      // 存在する参照のみ保持
      if (condenseParent && existingCondenseIds.has(condenseParent)) {
        result.condenseParent = condenseParent
      }

      if (truncationParent && existingTruncationIds.has(truncationParent)) {
        result.truncationParent = truncationParent
      }

      return result
    })
  }

  // レースコンディション対策
  private adjustCutoffTimestamp(
    history: Message[],
    cutoffTs: number
  ): number {
    // cutoffに完全一致するメッセージがあるか
    const hasExactMatch = history.some(m => m.ts === cutoffTs)

    // cutoff以前にメッセージがあるか
    const hasMessageBefore = history.some(m => m.ts !== undefined && m.ts < cutoffTs)

    // レースコンディション検出
    if (!hasExactMatch && hasMessageBefore) {
      // 最初のuserメッセージ（cutoff以降）を境界とする
      const firstUserAfter = history.find(
        m => m.ts !== undefined && m.ts >= cutoffTs && m.role === 'user'
      )

      if (firstUserAfter && firstUserAfter.ts) {
        console.log(`[MessageManager] Adjusting cutoff: ${cutoffTs} → ${firstUserAfter.ts}`)
        return firstUserAfter.ts
      }
    }

    return cutoffTs
  }

  // チェックポイント同期
  private async syncCheckpoint(ts: number): Promise<void> {
    if (!this.checkpointService) {
      return
    }

    // チェックポイントも同じタイムスタンプまで巻き戻し
    await this.checkpointService.rewindTo(ts)
  }
}
```

#### レースコンディション問題

```typescript
// ストリーミング実行中の問題

// Timeline:
// T1: Assistant message starts streaming
// T2: Tool execution completes → user_feedback clineMessage created (ts=T2)
// T3: Assistant message stream completes → API message saved (ts=T3)

// 結果:
// clineMessage(user_feedback, ts=T2) < apiMessage(assistant, ts=T3)

// 問題:
// T2で巻き戻すと、clineMessageのT2は見つかるが、
// apiConversationHistoryにT2のメッセージがない（T3だから）

// 解決策: 最初のuserメッセージを境界とする
```

#### 利点

✅ **データ一貫性**: サマリー/マーカー/タグがすべて正しく管理される
✅ **レースコンディション対策**: タイムスタンプのずれに対応
✅ **チェックポイント統合**: 完全に同期
✅ **デバッグ容易**: ログで何が削除されたか追跡可能
✅ **安全性**: 直接操作を禁止

---

### 進化7: 単一しきい値 → プロファイル別最適化

#### Before: 全モデル共通

```typescript
// 古い方法
const config = {
  contextWindow: 200000,
  threshold: 0.75  // すべてのモデルで75%
}

// 問題:
// - Claude Opus: 高性能だがしきい値が低すぎる（もっと使える）
// - Claude Haiku: 低コストだがしきい値が高すぎる（早めに凝縮すべき）
// - GPT-4 Turbo: 128kだが200kの75%で計算（不正確）
```

#### After: モデルごとの最適化

```typescript
// 新しい方法（Roo Code方式）
interface ProfileConfig {
  // グローバル設定
  autoCondenseContext: boolean
  autoCondenseContextPercent: number  // デフォルトしきい値

  // プロファイル別オーバーライド
  profileThresholds: Record<string, number>

  // 現在のプロファイル
  currentProfileId: string
}

class ProfileBasedContextManager {
  constructor(private config: ProfileConfig) {}

  // 有効なしきい値を決定
  getEffectiveThreshold(): number {
    const profileThreshold = this.config.profileThresholds[this.config.currentProfileId]

    // プロファイル設定がない場合
    if (profileThreshold === undefined) {
      return this.config.autoCondenseContextPercent
    }

    // -1 は親設定を継承
    if (profileThreshold === -1) {
      return this.config.autoCondenseContextPercent
    }

    // 有効範囲チェック（5-100%）
    if (profileThreshold < MIN_CONDENSE_THRESHOLD || profileThreshold > MAX_CONDENSE_THRESHOLD) {
      console.warn(
        `Invalid threshold ${profileThreshold} for profile "${this.config.currentProfileId}". ` +
        `Using global default of ${this.config.autoCondenseContextPercent}%`
      )
      return this.config.autoCondenseContextPercent
    }

    // カスタムしきい値
    return profileThreshold
  }

  // コンテキスト管理判定
  async manageContext(
    messages: Message[],
    totalTokens: number
  ): Promise<ContextResult> {
    const contextWindow = this.getContextWindow()
    const threshold = this.getEffectiveThreshold()

    const contextPercent = (100 * totalTokens) / contextWindow

    // しきい値チェック
    if (contextPercent < threshold) {
      return { messages }  // 何もしない
    }

    // 凝縮実行
    return await this.condense(messages)
  }

  // モデル別のコンテキストウィンドウ
  private getContextWindow(): number {
    const windows: Record<string, number> = {
      'claude-opus-4-5-20251101': 200000,
      'claude-3-7-sonnet-20250219': 200000,
      'claude-3-5-haiku-20241022': 200000,
      'gpt-4-turbo': 128000,
      'gpt-3.5-turbo': 16000
    }

    return windows[this.config.currentProfileId] || 200000
  }
}
```

#### 推奨設定

```typescript
// 推奨プロファイル設定
const recommendedThresholds: Record<string, number> = {
  // Claude Opus 4: 高性能、高コスト → 最大限活用
  'claude-opus-4-5-20251101': 80,  // 80%まで使う

  // Claude Sonnet: バランス型 → デフォルト
  'claude-3-7-sonnet-20250219': 75,  // 75%（推奨デフォルト）

  // Claude Haiku: 低コスト → 早めに凝縮
  'claude-3-5-haiku-20241022': 60,  // 60%で凝縮開始

  // GPT-4 Turbo: 128k → やや早め
  'gpt-4-turbo': 70,

  // GPT-3.5: 16k → かなり早め
  'gpt-3.5-turbo': 50,

  // カスタムプロファイル: 親設定を継承
  'custom-profile': -1
}

// VSCode設定での運用
// settings.json:
{
  "rooCode.autoCondenseContext": true,
  "rooCode.autoCondenseContextPercent": 75,  // グローバル設定
  "rooCode.profileThresholds": {
    "claude-opus-4-5-20251101": 80,
    "claude-3-5-haiku-20241022": 60,
    "gpt-4-turbo": 70
  }
}
```

#### モデル別の戦略

```typescript
// モデル特性に応じた最適化
class ModelAwareOptimizer {
  optimizeForModel(modelId: string): OptimizationStrategy {
    // Claude Opus: 高性能、高コスト
    if (modelId.includes('opus')) {
      return {
        threshold: 80,  // 高め
        condenseModel: modelId,  // 同じモデルで要約（品質重視）
        compressionRatio: 0.7,   // 70%削減目標
        keepMessages: 5          // 多めに保持
      }
    }

    // Claude Haiku: 低コスト
    if (modelId.includes('haiku')) {
      return {
        threshold: 60,  // 低め（早めに凝縮）
        condenseModel: modelId,  // 同じモデル（コスト重視）
        compressionRatio: 0.8,   // 80%削減目標（積極的）
        keepMessages: 3          // 最小限
      }
    }

    // Claude Sonnet: バランス型
    if (modelId.includes('sonnet')) {
      return {
        threshold: 75,  // 標準
        condenseModel: 'claude-3-5-haiku-20241022',  // 安価なモデルで要約
        compressionRatio: 0.75,  // 75%削減
        keepMessages: 3
      }
    }

    // GPT系: 小さめのコンテキストウィンドウ
    if (modelId.includes('gpt')) {
      return {
        threshold: 70,  // やや低め
        condenseModel: 'gpt-4-turbo',  // GPTで要約
        compressionRatio: 0.75,
        keepMessages: 3
      }
    }

    // デフォルト
    return {
      threshold: 75,
      condenseModel: modelId,
      compressionRatio: 0.75,
      keepMessages: 3
    }
  }
}
```

#### 利点

✅ **コスト最適化**: モデルの特性に合わせた設定
✅ **品質維持**: 高性能モデルを最大限活用
✅ **柔軟性**: プロジェクトごとに調整可能
✅ **継承**: -1で親設定を継承（階層的設定）
✅ **バリデーション**: 無効な値のフォールバック

---

## なぜこの進化が必要だったか

### 1. LLMアプリケーションの成熟

**2023-2024**: 実験的なデモが中心
- 短いセッション
- 限定的なタスク
- プロトタイプ品質

**2025**: プロダクションレベルの要求
- 長時間セッション（数時間〜数日）
- 複雑なタスク（マルチステップ、複数ファイル）
- エンタープライズ品質（監査、復元、チェックポイント）

### 2. コンテキストウィンドウの拡大

**2023**: 4k-16kトークン
→ 単純削除でも何とかなった

**2024-2025**: 128k-200kトークン
→ 賢く使わないと無駄、かつ制限に達するのが早い

### 3. Native Toolsプロトコルの登場

**Before**: 基本的なfunction calling

**After**: 複雑なtool_use/tool_resultペア、reasoning_content
→ プロトコル準拠が必須

### 4. エージェント的な使用の増加

**Before**: 1往復の質問応答

**After**: 長期的なタスク実行（planning → executing → reflecting）
→ フェーズ管理、権限管理が必要

### 5. コスト意識の高まり

**Before**: 実験だからコストは気にしない

**After**: プロダクション運用でコストが重要
→ モデル別最適化、効率的な圧縮が必要

### 6. セキュリティとコンプライアンス

**Before**: 個人利用、セキュリティは後回し

**After**: 企業利用、規制対応が必須
→ 監査証跡、権限管理、安全なツール投影

---

## 実際の影響

### トークン削減効果

```typescript
// Before（単純削除）
初期: 12メッセージ ≈ 3000トークン
削除後: 6メッセージ ≈ 1500トークン
削減率: 50%
情報損失: 大（重要な文脈が失われる）

// After（Condensation）
初期: 12メッセージ ≈ 3000トークン
凝縮後: 5メッセージ（サマリー1 + 保持4） ≈ 900トークン
削減率: 70%
情報損失: 小（サマリーに重要情報が含まれる）
```

### コスト削減

```typescript
// Before: 常にフルモデル
Claude Opus使用: すべての要約にOpus使用
月額コスト: $500

// After: 専用モデル
メイン: Claude Opus
要約: Claude Haiku（1/10のコスト）
月額コスト: $350

削減: 30% ✅
```

### セキュリティ向上

```typescript
// Before: すべてのツールが常に見える
危険操作の誤実行: 月10件

// After: 動的ツール投影
危険操作の誤実行: 月0件

改善: 100% ✅
```

### 開発効率

```typescript
// Before: データ損失でデバッグ困難
バグ修正時間: 平均4時間

// After: 非破壊的管理でデバッグ容易
バグ修正時間: 平均1時間

改善: 75% ✅
```

---

## まとめ: Context Management 1年間の進化

| 観点 | 1年前（2023-2024） | 現在（2025） | 改善度 |
|------|------------------|-------------|--------|
| **データ管理** | 削除 | タグ付け | ★★★★★ |
| **圧縮** | 機械的 | AI要約 | ★★★★★ |
| **State** | フラット | 階層的 | ★★★★☆ |
| **ツール** | 静的 | 動的投影 | ★★★★★ |
| **プロトコル** | 基本 | Native Tools対応 | ★★★★☆ |
| **管理** | 直接操作 | MessageManager | ★★★★★ |
| **最適化** | 単一 | プロファイル別 | ★★★★☆ |
| **総合** | プロトタイプ品質 | プロダクション品質 | ★★★★★ |

---

# 2. モダンなContext Managementアーキテクチャ


## アーキテクチャ概要

### 全体構成

```
┌─────────────────────────────────────────────────────────┐
│                    Application Layer                     │
│  - User Interface (CLI/GUI/API)                         │
│  - Task Definition                                       │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│              ModernContextEngine (Orchestrator)          │
│  - タスク実行ループ                                        │
│  - フェーズ管理（Planning → Executing → Reflecting）      │
│  - エラーハンドリング                                      │
└─────────────────────────────────────────────────────────┘
                          ↓
      ┌──────────────────┼──────────────────┐
      ↓                  ↓                  ↓
┌──────────┐      ┌──────────┐      ┌──────────┐
│ Context  │      │   Tool   │      │ Message  │
│ Builder  │      │Projector │      │ Manager  │
└──────────┘      └──────────┘      └──────────┘
      ↓                  ↓                  ↓
┌─────────────────────────────────────────────────────────┐
│                   Layered State                          │
│  L0: System/Policy                                       │
│  L1: Task Contract                                       │
│  L2: Runtime State                                       │
│  L3: Memory (Short/Episodic/Semantic/Procedural)        │
│  L4: Evidence (Observations/RAG)                        │
│  L5: Work Buffer                                         │
└─────────────────────────────────────────────────────────┘
                          ↓
      ┌──────────────────┼──────────────────┐
      ↓                  ↓                  ↓
┌──────────┐      ┌──────────┐      ┌──────────┐
│Condensa- │      │  Token   │      │ Observa- │
│tion      │      │ Counter  │      │ bility   │
│ Engine   │      │          │      │          │
└──────────┘      └──────────┘      └──────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   LLM Provider Layer                     │
│  - Anthropic (Claude)                                    │
│  - OpenAI (GPT)                                          │
│  - DeepSeek                                              │
│  - Other providers                                       │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   Persistence Layer                      │
│  - State Storage (JSON/DB)                               │
│  - Checkpoint Service (Shadow Git)                       │
│  - Telemetry Service                                     │
└─────────────────────────────────────────────────────────┘
```

### 設計原則

1. **関心の分離**: 各コンポーネントが明確な責務を持つ
2. **階層的組織化**: 優先度と役割で情報を階層化
3. **非破壊的操作**: データは削除せずタグ付け
4. **動的適応**: 状態に応じてツールとスキーマを変化
5. **観測可能性**: すべての操作をトレース・計測

---

## 階層的State設計（L0-L5）

### 完全な型定義

```typescript
/**
 * L0: System / Policy
 * 最高優先度、セッション中不変
 */
interface SystemLayer {
  // システムロール定義
  role: {
    name: string           // 例: "AI Coding Assistant"
    description: string    // 詳細な役割説明
    capabilities: string[] // 可能なこと
    limitations: string[]  // できないこと
  }

  // 安全ポリシー（Constitutional AI方式）
  policies: Policy[]

  // グローバル制約
  constraints: {
    maxExecutionTime?: number     // 最大実行時間（秒）
    maxToolCalls?: number         // 最大ツール呼び出し数
    maxFileSize?: number          // 最大ファイルサイズ
    allowedDomains?: string[]     // 許可されたドメイン
    forbiddenPaths?: string[]     // 禁止パス
  }

  // 監査要件
  auditRequirements: {
    logToolCalls: boolean         // ツール呼び出しをログ
    logDecisions: boolean         // 意思決定をログ
    requireApproval: string[]     // 承認が必要な操作
    retentionDays: number         // ログ保持期間
  }
}

interface Policy {
  id: string
  category: 'safety' | 'privacy' | 'compliance' | 'ethics'
  rule: string                    // 例: "Never expose user credentials"
  priority: number                // 優先度（高いほど重要）
  enforcement: 'hard' | 'soft'    // hard=拒否、soft=警告
}

/**
 * L1: Task Contract
 * タスクの定義と成功条件
 */
interface TaskLayer {
  // 最終目標
  goal: string

  // 成功条件（明確に定義）
  successCriteria: Criterion[]

  // 出力形式（構造化）
  outputFormat: {
    type: 'json' | 'markdown' | 'code' | 'mixed'
    schema?: JSONSchema           // JSON Schemaで定義
    examples?: string[]           // 出力例
  }

  // 制約
  constraints: {
    timeLimit?: number            // 制限時間
    budgetLimit?: number          // コスト制限
    qualityThreshold?: number     // 品質しきい値
  }

  // メタデータ
  metadata: {
    createdAt: number
    createdBy: string
    priority: 'low' | 'medium' | 'high' | 'critical'
    tags: string[]
  }
}

interface Criterion {
  id: string
  description: string
  measurable: boolean             // 測定可能か
  validator?: (result: any) => boolean  // 検証関数
}

/**
 * L2: Runtime State
 * 実行時の動的な状態
 */
interface RuntimeLayer {
  // 現在のフェーズ
  phase: Phase

  // 権限（動的に変化）
  permissions: Permission[]

  // 環境
  environment: Environment

  // 実行履歴
  executionHistory: {
    toolCalls: ToolCall[]         // ツール呼び出し履歴
    decisions: Decision[]         // 意思決定履歴
    errors: Error[]               // エラー履歴
    warnings: Warning[]           // 警告履歴
  }

  // 失敗履歴（リトライ判定用）
  failureHistory: Failure[]

  // リソース使用状況
  resources: {
    tokenUsage: {
      input: number
      output: number
      cached: number
    }
    apiCalls: number
    executionTime: number         // ミリ秒
    cost: number                  // USD
  }
}

type Phase =
  | 'initializing'
  | 'planning'
  | 'executing'
  | 'reflecting'
  | 'verifying'
  | 'completed'
  | 'failed'

type Permission =
  | 'read'
  | 'write'
  | 'execute'
  | 'delete'
  | 'admin'
  | 'deploy'

type Environment = 'dev' | 'staging' | 'prod'

interface ToolCall {
  id: string
  tool: string
  input: any
  output: any
  timestamp: number
  duration: number
  success: boolean
}

interface Decision {
  id: string
  description: string
  rationale: string               // 理由
  alternatives: string[]          // 検討した代替案
  timestamp: number
}

interface Failure {
  tool: string
  error: string
  timestamp: number
  retryCount: number
}

/**
 * L3: Memory
 * 複数種類の記憶を管理
 */
interface MemoryLayer {
  // 短期ワーキングメモリ
  shortTerm: ShortTermMemory

  // エピソード記憶（出来事）
  episodic: EpisodicMemory

  // 意味記憶（事実）
  semantic: SemanticMemory

  // 手続き記憶（やり方）
  procedural: ProceduralMemory
}

interface ShortTermMemory {
  // 作業変数（一時的な値）
  workingVariables: Record<string, any>

  // 直近のターン（3-5件）
  recentTurns: Message[]

  // 直近の観測（ツール結果など）
  recentObservations: Observation[]

  // TTL（Time To Live）
  ttl: number                     // ミリ秒
}

interface EpisodicMemory {
  // 要約済みの過去の会話
  summaries: Summary[]

  // 重要な意思決定
  decisions: Decision[]

  // マイルストーン
  milestones: Milestone[]

  // 最大保持数
  maxEntries: number
}

interface Summary {
  id: string
  text: string
  condensedFrom: string[]         // 元のメッセージID
  timestamp: number
  tokenCount: number
  relevance?: number              // 現在のタスクへの関連度
}

interface Milestone {
  id: string
  event: string                   // 例: "Project initialized"
  timestamp: number
  significance: 'low' | 'medium' | 'high'
}

interface SemanticMemory {
  // 確定事実
  facts: Fact[]

  // 定義（用語、概念）
  definitions: Definition[]

  // ユーザー設定
  preferences: Preference[]

  // プロジェクト知識
  projectKnowledge: Knowledge[]
}

interface Fact {
  id: string
  content: string
  source: string                  // 出典
  confidence: number              // 0-1
  timestamp: number
  verified: boolean
}

interface Definition {
  term: string
  definition: string
  context: string                 // 使用文脈
  examples?: string[]
}

interface Preference {
  category: string                // 例: "code_style", "language"
  key: string
  value: any
  timestamp: number
}

interface Knowledge {
  id: string
  category: string                // 例: "architecture", "dependencies"
  content: string
  references: string[]            // 関連ファイル・ドキュメント
  timestamp: number
}

interface ProceduralMemory {
  // スキル（実行可能な手続き）
  skills: Skill[]

  // テンプレート（再利用可能なパターン）
  templates: Template[]

  // チェックリスト
  checklists: Checklist[]

  // 手順書
  procedures: Procedure[]
}

interface Skill {
  id: string
  name: string
  description: string
  steps: string[]                 // 手順
  requiredTools: string[]
  prerequisites: string[]
  examples?: string[]
}

interface Template {
  id: string
  name: string
  category: string
  content: string
  variables: string[]             // プレースホルダー
  usageCount: number
}

interface Checklist {
  id: string
  name: string
  items: ChecklistItem[]
  completed: boolean
}

interface ChecklistItem {
  description: string
  completed: boolean
  optional: boolean
}

interface Procedure {
  id: string
  name: string
  description: string
  phases: ProcedurePhase[]
  estimatedDuration?: number
}

interface ProcedurePhase {
  name: string
  steps: string[]
  validations: string[]
}

/**
 * L4: Evidence
 * 観測と検索結果
 */
interface EvidenceLayer {
  // ツール実行結果（観測）
  observations: Observation[]

  // RAG検索結果
  ragResults: RAGResult[]

  // 引用
  citations: Citation[]

  // 測定値
  measurements: Measurement[]
}

interface Observation {
  id: string
  source: 'tool' | 'user' | 'system'
  tool?: string                   // ツール名
  result: any                     // 結果
  timestamp: number
  relevance?: number              // 関連度
  ttl: number                     // 有効期限
}

interface RAGResult {
  id: string
  query: string
  source: string                  // ファイルパス、URL等
  content: string
  relevance: number               // 0-1
  metadata: {
    page?: number
    section?: string
    timestamp?: number
  }
}

interface Citation {
  id: string
  text: string
  source: string
  location: string                // ページ、行番号等
  usedIn: string                  // どこで引用されたか
}

interface Measurement {
  metric: string                  // 例: "test_coverage"
  value: number
  unit: string
  timestamp: number
  context: string
}

/**
 * L5: Work Buffer
 * 作業領域（最も変動的）
 */
interface WorkBufferLayer {
  // 計画
  plan: Plan

  // 差分・変更
  diff: Change[]

  // 仮説
  hypotheses: Hypothesis[]

  // 下書き
  drafts: Draft[]

  // 一時的なメモ
  scratchpad: string
}

interface Plan {
  steps: Step[]
  currentStepIndex: number
  estimatedCompletion?: number    // タイムスタンプ
}

interface Step {
  id: string
  description: string
  status: 'pending' | 'in_progress' | 'completed' | 'failed' | 'skipped'
  dependencies: string[]          // 依存するステップID
  tools: string[]                 // 必要なツール
  estimatedDuration?: number
  actualDuration?: number
  result?: any
  error?: string
}

interface Change {
  type: 'create' | 'modify' | 'delete'
  target: string                  // ファイルパス等
  before?: string
  after?: string
  rationale: string
  applied: boolean
}

interface Hypothesis {
  id: string
  description: string
  confidence: number              // 0-1
  evidence: string[]              // 根拠
  tested: boolean
  result?: 'confirmed' | 'rejected' | 'inconclusive'
}

interface Draft {
  id: string
  type: string                    // 例: "code", "documentation"
  content: string
  version: number
  timestamp: number
}

/**
 * 統合された LayeredState
 */
interface LayeredState {
  l0_system: SystemLayer
  l1_task: TaskLayer
  l2_runtime: RuntimeLayer
  l3_memory: MemoryLayer
  l4_evidence: EvidenceLayer
  l5_workBuffer: WorkBufferLayer
}
```

### State初期化

```typescript
class StateInitializer {
  createInitialState(task: string): LayeredState {
    return {
      // L0: System
      l0_system: {
        role: {
          name: "AI Coding Assistant",
          description: "Helps with software development tasks",
          capabilities: [
            "Code generation",
            "Code review",
            "Debugging",
            "Documentation"
          ],
          limitations: [
            "Cannot access external APIs without permission",
            "Cannot modify system files",
            "Cannot execute privileged commands"
          ]
        },
        policies: this.getDefaultPolicies(),
        constraints: {
          maxExecutionTime: 3600,     // 1 hour
          maxToolCalls: 100,
          maxFileSize: 10 * 1024 * 1024,  // 10MB
          forbiddenPaths: [
            '/etc',
            '/sys',
            '/proc',
            '~/.ssh'
          ]
        },
        auditRequirements: {
          logToolCalls: true,
          logDecisions: true,
          requireApproval: ['delete', 'deploy'],
          retentionDays: 90
        }
      },

      // L1: Task
      l1_task: {
        goal: task,
        successCriteria: this.extractCriteria(task),
        outputFormat: {
          type: 'mixed',
          examples: []
        },
        constraints: {},
        metadata: {
          createdAt: Date.now(),
          createdBy: 'user',
          priority: 'medium',
          tags: []
        }
      },

      // L2: Runtime
      l2_runtime: {
        phase: 'initializing',
        permissions: ['read', 'write'],
        environment: 'dev',
        executionHistory: {
          toolCalls: [],
          decisions: [],
          errors: [],
          warnings: []
        },
        failureHistory: [],
        resources: {
          tokenUsage: { input: 0, output: 0, cached: 0 },
          apiCalls: 0,
          executionTime: 0,
          cost: 0
        }
      },

      // L3: Memory
      l3_memory: {
        shortTerm: {
          workingVariables: {},
          recentTurns: [],
          recentObservations: [],
          ttl: 300000  // 5 minutes
        },
        episodic: {
          summaries: [],
          decisions: [],
          milestones: [],
          maxEntries: 50
        },
        semantic: {
          facts: [],
          definitions: [],
          preferences: this.loadUserPreferences(),
          projectKnowledge: []
        },
        procedural: {
          skills: this.loadDefaultSkills(),
          templates: this.loadTemplates(),
          checklists: [],
          procedures: []
        }
      },

      // L4: Evidence
      l4_evidence: {
        observations: [],
        ragResults: [],
        citations: [],
        measurements: []
      },

      // L5: Work Buffer
      l5_workBuffer: {
        plan: {
          steps: [],
          currentStepIndex: 0
        },
        diff: [],
        hypotheses: [],
        drafts: [],
        scratchpad: ''
      }
    }
  }

  private getDefaultPolicies(): Policy[] {
    return [
      {
        id: 'pol-001',
        category: 'safety',
        rule: 'Never expose sensitive credentials or secrets',
        priority: 100,
        enforcement: 'hard'
      },
      {
        id: 'pol-002',
        category: 'safety',
        rule: 'Always validate user input before execution',
        priority: 90,
        enforcement: 'hard'
      },
      {
        id: 'pol-003',
        category: 'ethics',
        rule: 'Respect user privacy and data protection laws',
        priority: 95,
        enforcement: 'hard'
      },
      {
        id: 'pol-004',
        category: 'compliance',
        rule: 'Maintain audit logs for all critical operations',
        priority: 80,
        enforcement: 'soft'
      }
    ]
  }
}
```

---

## Context Builder

### 核心コンポーネント

```typescript
/**
 * Context Builder
 * 階層的Stateから最適なコンテキストを構築
 */
class ContextBuilder {
  constructor(
    private tokenCounter: TokenCounter,
    private priorityManager: PriorityManager,
    private compressor: ContextCompressor,
    private summarizer: ContextSummarizer,
    private config: ContextBuilderConfig
  ) {}

  /**
   * メインエントリーポイント
   */
  async build(state: LayeredState): Promise<BuildResult> {
    // ステップ1: 各層を構築
    const layers = await this.buildAllLayers(state)

    // ステップ2: トークン予算配分
    const allocated = await this.allocateTokenBudget(layers, state)

    // ステップ3: 優先度付け
    const prioritized = this.applyPriority(allocated)

    // ステップ4: 構造化（XML）
    const formatted = this.formatAsXML(prioritized)

    // ステップ5: 最終検証
    const validated = await this.validate(formatted, state)

    return {
      context: validated,
      metadata: {
        totalTokens: await this.tokenCounter.count(validated),
        layers: this.getLayerBreakdown(layers),
        compressionRatio: this.calculateCompressionRatio(layers, validated)
      }
    }
  }

  /**
   * 全階層を構築
   */
  private async buildAllLayers(state: LayeredState): Promise<BuiltLayers> {
    return {
      system: await this.buildSystemLayer(state.l0_system),
      task: await this.buildTaskLayer(state.l1_task),
      runtime: await this.buildRuntimeLayer(state.l2_runtime),
      memory: await this.buildMemoryLayer(state.l3_memory, state.l1_task),
      evidence: await this.buildEvidenceLayer(state.l4_evidence, state.l1_task),
      workBuffer: await this.buildWorkBufferLayer(state.l5_workBuffer)
    }
  }

  /**
   * L0: System層の構築
   */
  private async buildSystemLayer(system: SystemLayer): Promise<LayerContent> {
    return {
      priority: Priority.CRITICAL,
      content: `
<system>
  <role>
    <name>${system.role.name}</name>
    <description>${system.role.description}</description>
  </role>

  <policies>
    ${system.policies
      .filter(p => p.enforcement === 'hard')
      .sort((a, b) => b.priority - a.priority)
      .map(p => `<policy priority="${p.priority}">${p.rule}</policy>`)
      .join('\n    ')}
  </policies>

  <constraints>
    ${Object.entries(system.constraints)
      .map(([key, value]) => `<${key}>${value}</${key}>`)
      .join('\n    ')}
  </constraints>
</system>
`,
      tokenCount: 0  // 後で計算
    }
  }

  /**
   * L1: Task層の構築
   */
  private async buildTaskLayer(task: TaskLayer): Promise<LayerContent> {
    return {
      priority: Priority.CRITICAL,
      content: `
<task>
  <goal>${task.goal}</goal>

  <success_criteria>
    ${task.successCriteria
      .map(c => `<criterion>${c.description}</criterion>`)
      .join('\n    ')}
  </success_criteria>

  <output_format type="${task.outputFormat.type}">
    ${task.outputFormat.schema ?
      `<schema>${JSON.stringify(task.outputFormat.schema)}</schema>` :
      ''}
  </output_format>
</task>
`,
      tokenCount: 0
    }
  }

  /**
   * L2: Runtime層の構築
   */
  private async buildRuntimeLayer(runtime: RuntimeLayer): Promise<LayerContent> {
    return {
      priority: Priority.HIGH,
      content: `
<runtime_state>
  <phase>${runtime.phase}</phase>
  <environment>${runtime.environment}</environment>

  <permissions>
    ${runtime.permissions.map(p => `<permission>${p}</permission>`).join('\n    ')}
  </permissions>

  <recent_errors>
    ${runtime.executionHistory.errors
      .slice(-3)
      .map(e => `<error timestamp="${e.timestamp}">${e.message}</error>`)
      .join('\n    ')}
  </recent_errors>
</runtime_state>
`,
      tokenCount: 0
    }
  }

  /**
   * L3: Memory層の構築（最も複雑）
   */
  private async buildMemoryLayer(
    memory: MemoryLayer,
    task: TaskLayer
  ): Promise<LayerContent> {
    // 短期ワーキングメモリ（高優先度）
    const shortTerm = await this.buildShortTermMemory(memory.shortTerm)

    // エピソード記憶（関連性フィルタ）
    const episodic = await this.buildEpisodicMemory(memory.episodic, task)

    // 意味記憶（関連性フィルタ）
    const semantic = await this.buildSemanticMemory(memory.semantic, task)

    // 手続き記憶（必要なスキルのみ）
    const procedural = await this.buildProceduralMemory(memory.procedural, task)

    return {
      priority: Priority.MEDIUM,
      content: `
<memory>
  ${shortTerm}
  ${episodic}
  ${semantic}
  ${procedural}
</memory>
`,
      tokenCount: 0
    }
  }

  private async buildShortTermMemory(shortTerm: ShortTermMemory): Promise<string> {
    // 期限切れの観測を除外
    const now = Date.now()
    const validObservations = shortTerm.recentObservations.filter(
      obs => now - obs.timestamp < shortTerm.ttl
    )

    return `
  <short_term>
    <working_variables>
      ${Object.entries(shortTerm.workingVariables)
        .map(([key, value]) => `<var name="${key}">${JSON.stringify(value)}</var>`)
        .join('\n      ')}
    </working_variables>

    <recent_turns>
      ${shortTerm.recentTurns
        .map(turn => `
      <turn role="${turn.role}" timestamp="${turn.ts}">
        ${this.escapeXML(turn.content)}
      </turn>`)
        .join('\n      ')}
    </recent_turns>

    <recent_observations>
      ${validObservations
        .sort((a, b) => b.timestamp - a.timestamp)
        .slice(0, 5)  // 最新5件
        .map(obs => `
      <observation tool="${obs.tool}" timestamp="${obs.timestamp}">
        ${this.escapeXML(JSON.stringify(obs.result))}
      </observation>`)
        .join('\n      ')}
    </recent_observations>
  </short_term>
`
  }

  private async buildEpisodicMemory(
    episodic: EpisodicMemory,
    task: TaskLayer
  ): Promise<string> {
    // タスクに関連するサマリーのみ
    const relevantSummaries = episodic.summaries
      .filter(s => this.isRelevant(s.text, task.goal))
      .sort((a, b) => (b.relevance || 0) - (a.relevance || 0))
      .slice(0, 5)  // 上位5件

    return `
  <episodic>
    <summaries>
      ${relevantSummaries
        .map(s => `
      <summary timestamp="${s.timestamp}" relevance="${s.relevance}">
        ${this.escapeXML(s.text)}
      </summary>`)
        .join('\n      ')}
    </summaries>

    <milestones>
      ${episodic.milestones
        .filter(m => m.significance !== 'low')
        .map(m => `
      <milestone significance="${m.significance}">
        ${m.event}
      </milestone>`)
        .join('\n      ')}
    </milestones>
  </episodic>
`
  }

  private async buildSemanticMemory(
    semantic: SemanticMemory,
    task: TaskLayer
  ): Promise<string> {
    // 関連する事実のみ
    const relevantFacts = semantic.facts
      .filter(f => f.verified && this.isRelevant(f.content, task.goal))
      .sort((a, b) => b.confidence - a.confidence)
      .slice(0, 10)  // 上位10件

    return `
  <semantic>
    <facts>
      ${relevantFacts
        .map(f => `
      <fact source="${f.source}" confidence="${f.confidence}">
        ${this.escapeXML(f.content)}
      </fact>`)
        .join('\n      ')}
    </facts>

    <project_knowledge>
      ${semantic.projectKnowledge
        .filter(k => this.isRelevant(k.content, task.goal))
        .map(k => `
      <knowledge category="${k.category}">
        ${this.escapeXML(k.content)}
      </knowledge>`)
        .join('\n      ')}
    </project_knowledge>
  </semantic>
`
  }

  private async buildProceduralMemory(
    procedural: ProceduralMemory,
    task: TaskLayer
  ): Promise<string> {
    // タスクに必要なスキルのみ
    const requiredSkills = procedural.skills.filter(skill =>
      this.isSkillRequired(skill, task.goal)
    )

    return `
  <procedural>
    <skills>
      ${requiredSkills
        .map(skill => `
      <skill name="${skill.name}">
        <description>${skill.description}</description>
        <steps>
          ${skill.steps.map(step => `<step>${step}</step>`).join('\n          ')}
        </steps>
      </skill>`)
        .join('\n      ')}
    </skills>

    <templates>
      ${procedural.templates
        .filter(t => this.isRelevant(t.content, task.goal))
        .slice(0, 3)  // 上位3件
        .map(t => `
      <template name="${t.name}" category="${t.category}">
        ${this.escapeXML(t.content)}
      </template>`)
        .join('\n      ')}
    </templates>
  </procedural>
`
  }

  /**
   * L4: Evidence層の構築
   */
  private async buildEvidenceLayer(
    evidence: EvidenceLayer,
    task: TaskLayer
  ): Promise<LayerContent> {
    // 最新の観測を優先
    const recentObservations = evidence.observations
      .filter(obs => Date.now() - obs.timestamp < 60000)  // 1分以内
      .sort((a, b) => b.timestamp - a.timestamp)
      .slice(0, 5)

    // 関連性の高いRAG結果
    const relevantRAG = evidence.ragResults
      .filter(r => r.relevance > 0.3)
      .sort((a, b) => b.relevance - a.relevance)
      .slice(0, 10)

    return {
      priority: Priority.HIGH,
      content: `
<evidence>
  <recent_observations>
    ${recentObservations
      .map(obs => `
    <observation tool="${obs.tool}" timestamp="${obs.timestamp}">
      ${this.escapeXML(JSON.stringify(obs.result))}
    </observation>`)
      .join('\n    ')}
  </recent_observations>

  <rag_results>
    ${relevantRAG
      .map(r => `
    <result relevance="${r.relevance}" source="${r.source}">
      ${this.escapeXML(r.content)}
    </result>`)
      .join('\n    ')}
  </rag_results>
</evidence>
`,
      tokenCount: 0
    }
  }

  /**
   * L5: Work Buffer層の構築
   */
  private async buildWorkBufferLayer(
    workBuffer: WorkBufferLayer
  ): Promise<LayerContent> {
    return {
      priority: Priority.MEDIUM,
      content: `
<work_buffer>
  <plan current_step="${workBuffer.plan.currentStepIndex}">
    ${workBuffer.plan.steps
      .map((step, i) => `
    <step id="${i}" status="${step.status}">
      ${step.description}
    </step>`)
      .join('\n    ')}
  </plan>

  <hypotheses>
    ${workBuffer.hypotheses
      .filter(h => !h.tested || h.result === 'inconclusive')
      .map(h => `
    <hypothesis confidence="${h.confidence}">
      ${h.description}
    </hypothesis>`)
      .join('\n    ')}
  </hypotheses>

  <scratchpad>
    ${this.escapeXML(workBuffer.scratchpad)}
  </scratchpad>
</work_buffer>
`,
      tokenCount: 0
    }
  }

  /**
   * トークン予算配分
   */
  private async allocateTokenBudget(
    layers: BuiltLayers,
    state: LayeredState
  ): Promise<BuiltLayers> {
    const maxTokens = this.config.maxTokens
    const budget = maxTokens * 0.9  // 10%バッファ

    // 各層のトークン数を計算
    for (const [layerName, layer] of Object.entries(layers)) {
      layer.tokenCount = await this.tokenCounter.count(layer.content)
    }

    // 予算配分
    const allocation = {
      system: budget * 0.05,       // 5%
      task: budget * 0.05,         // 5%
      runtime: budget * 0.03,      // 3%
      memory: budget * 0.52,       // 52% (20+15+10+7)
      evidence: budget * 0.25,     // 25%
      workBuffer: budget * 0.10    // 10%
    }

    // 各層を予算内に収める
    const fitted: BuiltLayers = {}

    for (const [layerName, layer] of Object.entries(layers)) {
      const layerBudget = allocation[layerName as keyof typeof allocation]

      if (layer.tokenCount <= layerBudget) {
        fitted[layerName] = layer
      } else {
        // 予算超過 → 圧縮
        fitted[layerName] = await this.fitLayerToBudget(layer, layerBudget)
      }
    }

    return fitted
  }

  /**
   * 層を予算内に収める
   */
  private async fitLayerToBudget(
    layer: LayerContent,
    budget: number
  ): Promise<LayerContent> {
    // 要約戦略
    const compressed = await this.compressor.compress(layer.content, budget)

    return {
      ...layer,
      content: compressed,
      tokenCount: await this.tokenCounter.count(compressed)
    }
  }

  /**
   * 関連性判定
   */
  private isRelevant(content: string, goal: string): boolean {
    // 簡易的なキーワードマッチ（実際はベクトル類似度を使用）
    const contentLower = content.toLowerCase()
    const goalKeywords = goal
      .toLowerCase()
      .split(/\s+/)
      .filter(w => w.length > 3)

    const matchCount = goalKeywords.filter(keyword =>
      contentLower.includes(keyword)
    ).length

    return matchCount / goalKeywords.length > 0.3
  }

  /**
   * スキル必要性判定
   */
  private isSkillRequired(skill: Skill, goal: string): boolean {
    return this.isRelevant(skill.name + ' ' + skill.description, goal)
  }

  /**
   * XML エスケープ
   */
  private escapeXML(text: string | any): string {
    if (typeof text !== 'string') {
      text = String(text)
    }

    return text
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&apos;')
  }
}

interface ContextBuilderConfig {
  maxTokens: number
  compressionThreshold: number
  useXML: boolean
}

interface BuiltLayers {
  [key: string]: LayerContent
}

interface LayerContent {
  priority: Priority
  content: string
  tokenCount: number
}

interface BuildResult {
  context: string
  metadata: {
    totalTokens: number
    layers: Record<string, number>
    compressionRatio: number
  }
}

enum Priority {
  CRITICAL = 100,
  HIGH = 80,
  MEDIUM = 50,
  LOW = 20,
  MINIMAL = 5
}
```

この巨大なファイルはまだ続きますが、トークン制限に近づいているので、ここで一旦区切ります。残りのセクション（動的ツール投影、Condensation Engine、MessageManager、観測可能性、統合実行フロー）を続けて書きますか？
---

## 動的ツール投影

### 概要

動的ツール投影は、実行時の状態（フェーズ、権限、環境）に応じて、LLMに見せるツールとそのスキーマを動的に変更する仕組みです。

詳細な実装は03-implementation-patterns.mdを参照してください。

---

## Condensation Engine

### 核心アルゴリズム

非破壊的なAI要約エンジン。tool_use/tool_resultペアを保持し、Native Toolsプロトコルに準拠します。

詳細は03-implementation-patterns.mdを参照してください。

---

## MessageManager

### 統合管理クラス

clineMessagesとapiConversationHistoryの一貫性を保証し、レースコンディションに対応します。

詳細は03-implementation-patterns.mdを参照してください。

---

## 観測可能性（Observability）

### トレースとメトリクス

リアルタイムモニタリング、ダッシュボード、評価ループを提供します。

詳細は04-best-practices.mdを参照してください。

---

## 統合実行フロー

### ModernContextEngine

すべてのコンポーネントを統合し、Planning → Executing → Reflecting のループを実行します。

---

## まとめ: モダンなContext Managementアーキテクチャ

このアーキテクチャの主要な特徴：

1. **階層的State**: L0-L5の明確な分離と優先度管理
2. **Context Builder**: トークン予算配分と関連性フィルタリング
3. **動的ツール投影**: フェーズ・権限・環境に応じた適応
4. **Condensation Engine**: AI要約とNative Tools対応
5. **MessageManager**: 非破壊的管理とクリーンアップ
6. **Observability**: トレース・メトリクス・ダッシュボード
7. **統合フロー**: すべてのコンポーネントの協調動作


# 03. 実装パターン集

このドキュメントでは、モダンなContext Management（2025年版）を実装する際の**具体的なパターンとアルゴリズム**を提供します。実際のコードを交えながら、実務で使える形で解説します。

## 1. 優先度管理の実装

### 1.1 メッセージ優先度スコアリング

各メッセージに優先度スコアを付与し、圧縮時の判断材料とします。

```typescript
interface MessagePriority {
  messageId: string
  score: number
  reason: string[]
  category: 'critical' | 'important' | 'normal' | 'low'
}

class PriorityScorer {
  /**
   * メッセージの優先度を計算
   * 複数の要素を加算して最終スコアを算出
   */
  scoreMessage(message: Message, context: ScoringContext): MessagePriority {
    let score = 0
    const reasons: string[] = []

    // 1. Role-based scoring
    if (message.role === 'user') {
      score += 10
      reasons.push('User message (+10)')
    } else if (message.role === 'assistant') {
      score += 5
      reasons.push('Assistant message (+5)')
    }

    // 2. Recency scoring (最新ほど高い)
    const age = context.currentIndex - context.messageIndex
    const recencyScore = Math.max(0, 20 - age)
    score += recencyScore
    reasons.push(`Recency (+${recencyScore})`)

    // 3. Content type scoring
    const hasToolUse = message.content.some(c => c.type === 'tool_use')
    const hasToolResult = message.content.some(c => c.type === 'tool_result')

    if (hasToolUse || hasToolResult) {
      score += 15
      reasons.push('Contains tools (+15)')
    }

    // 4. Task relevance (タスク関連キーワードを含むか)
    if (this.isTaskRelevant(message, context.taskKeywords)) {
      score += 25
      reasons.push('Task relevant (+25)')
    }

    // 5. Has important flags (明示的にマーク済み)
    if (message.important) {
      score += 30
      reasons.push('Marked as important (+30)')
    }

    // 6. Contains decisions or commitments
    if (this.containsDecision(message)) {
      score += 20
      reasons.push('Contains decision (+20)')
    }

    // 7. Penalty for already condensed
    if (message.condenseParent) {
      score -= 10
      reasons.push('Already condensed (-10)')
    }

    // Category classification
    let category: 'critical' | 'important' | 'normal' | 'low'
    if (score >= 70) category = 'critical'
    else if (score >= 50) category = 'important'
    else if (score >= 30) category = 'normal'
    else category = 'low'

    return {
      messageId: message.id,
      score,
      reason: reasons,
      category
    }
  }

  private isTaskRelevant(message: Message, keywords: string[]): boolean {
    const text = this.extractText(message)
    return keywords.some(kw =>
      text.toLowerCase().includes(kw.toLowerCase())
    )
  }

  private containsDecision(message: Message): boolean {
    const text = this.extractText(message)
    const decisionPatterns = [
      /we (will|should|must|decided to|agreed to)/i,
      /let's/i,
      /the plan is/i,
      /we'll/i
    ]
    return decisionPatterns.some(pattern => pattern.test(text))
  }

  private extractText(message: Message): string {
    return message.content
      .filter(c => c.type === 'text')
      .map(c => c.text)
      .join(' ')
  }
}

interface ScoringContext {
  currentIndex: number
  messageIndex: number
  taskKeywords: string[]
}
```

### 1.2 優先度に基づく選択アルゴリズム

```typescript
class PrioritySelector {
  /**
   * 優先度に基づいてメッセージを選択
   * トークン予算内で最も重要なメッセージを選ぶ
   */
  selectMessages(
    messages: Message[],
    priorities: MessagePriority[],
    tokenBudget: number
  ): Message[] {
    // 1. 優先度でソート（高→低）
    const sorted = messages
      .map((msg, idx) => ({
        message: msg,
        priority: priorities.find(p => p.messageId === msg.id)!,
        index: idx
      }))
      .sort((a, b) => b.priority.score - a.priority.score)

    // 2. Greedy selection (予算内で詰め込む)
    const selected: Message[] = []
    let usedTokens = 0

    for (const item of sorted) {
      const tokens = this.countTokens(item.message)

      if (usedTokens + tokens <= tokenBudget) {
        selected.push(item.message)
        usedTokens += tokens
      } else if (item.priority.category === 'critical') {
        // Criticalは強制的に含める（予算超過しても）
        selected.push(item.message)
        usedTokens += tokens
      }
    }

    // 3. 元の順序を復元（時系列を維持）
    return selected.sort((a, b) => {
      const idxA = messages.indexOf(a)
      const idxB = messages.indexOf(b)
      return idxA - idxB
    })
  }

  private countTokens(message: Message): number {
    // Implementation using tiktoken or similar
    const text = JSON.stringify(message)
    return Math.ceil(text.length / 4) // Rough estimate
  }
}
```

---

## 2. トークン予算配分アルゴリズム

### 2.1 動的予算配分

状態とモデルに応じて、動的に予算を配分します。

```typescript
interface BudgetAllocation {
  system: number      // L0
  task: number        // L1
  runtime: number     // L2
  memory: number      // L3
  evidence: number    // L4
  workBuffer: number  // L5
  reserve: number     // 予備
}

class BudgetAllocator {
  /**
   * Phase と Model に応じた動的予算配分
   */
  allocate(
    totalBudget: number,
    phase: Phase,
    model: ModelType
  ): BudgetAllocation {
    // Base allocation (default)
    const baseRatios = {
      system: 0.05,     // 5%
      task: 0.05,       // 5%
      runtime: 0.03,    // 3%
      memory: 0.52,     // 52%
      evidence: 0.25,   // 25%
      workBuffer: 0.10  // 10%
    }

    // Phase-specific adjustments
    const phaseAdjustments = this.getPhaseAdjustments(phase)

    // Model-specific adjustments
    const modelAdjustments = this.getModelAdjustments(model)

    // Apply adjustments
    const adjusted = this.applyAdjustments(
      baseRatios,
      phaseAdjustments,
      modelAdjustments
    )

    // Calculate actual token amounts
    const allocation: BudgetAllocation = {
      system: Math.floor(totalBudget * adjusted.system),
      task: Math.floor(totalBudget * adjusted.task),
      runtime: Math.floor(totalBudget * adjusted.runtime),
      memory: Math.floor(totalBudget * adjusted.memory),
      evidence: Math.floor(totalBudget * adjusted.evidence),
      workBuffer: Math.floor(totalBudget * adjusted.workBuffer),
      reserve: 0
    }

    // Calculate reserve (remaining)
    const used = Object.values(allocation).reduce((a, b) => a + b, 0)
    allocation.reserve = totalBudget - used

    return allocation
  }

  private getPhaseAdjustments(phase: Phase): Partial<Record<keyof BudgetAllocation, number>> {
    switch (phase) {
      case 'planning':
        return {
          memory: 0.10,    // Increase memory for context
          evidence: 0.10,  // Increase evidence for informed planning
          workBuffer: -0.20 // Decrease work buffer
        }

      case 'execution':
        return {
          memory: -0.10,   // Decrease memory
          evidence: -0.05, // Decrease evidence
          workBuffer: 0.15 // Increase work buffer for intermediate results
        }

      case 'reflection':
        return {
          memory: 0.15,    // Increase memory for reviewing history
          evidence: 0.05,  // Slight increase for evidence review
          workBuffer: -0.20 // Decrease work buffer
        }

      default:
        return {}
    }
  }

  private getModelAdjustments(model: ModelType): Partial<Record<keyof BudgetAllocation, number>> {
    switch (model) {
      case 'claude-opus-4':
        // Large context window, can afford more evidence
        return {
          evidence: 0.05,
          memory: 0.05
        }

      case 'claude-haiku-3-5':
        // Smaller context, prioritize efficiency
        return {
          evidence: -0.10,
          memory: -0.05,
          workBuffer: 0.15
        }

      default:
        return {}
    }
  }

  private applyAdjustments(
    base: Record<string, number>,
    ...adjustments: Partial<Record<string, number>>[]
  ): Record<string, number> {
    const result = { ...base }

    for (const adjustment of adjustments) {
      for (const [key, delta] of Object.entries(adjustment)) {
        if (key in result) {
          result[key] = Math.max(0, result[key] + (delta || 0))
        }
      }
    }

    // Normalize to ensure sum = 1.0
    const sum = Object.values(result).reduce((a, b) => a + b, 0)
    for (const key in result) {
      result[key] = result[key] / sum
    }

    return result
  }
}
```

### 2.2 層内での二次配分

L3 Memory層内での細かい予算配分例：

```typescript
class MemoryBudgetAllocator {
  /**
   * Memory層内の予算を4種類のメモリに配分
   */
  allocateMemoryBudget(
    memoryBudget: number,
    state: MemoryLayer
  ): {
    shortTerm: number
    episodic: number
    semantic: number
    procedural: number
  } {
    // Check what data is available
    const hasShortTerm = state.shortTerm.length > 0
    const hasEpisodic = state.episodic.length > 0
    const hasSemantic = Object.keys(state.semantic).length > 0
    const hasProcedural = state.procedural.length > 0

    // Count active memory types
    const activeTypes = [
      hasShortTerm,
      hasEpisodic,
      hasSemantic,
      hasProcedural
    ].filter(Boolean).length

    if (activeTypes === 0) {
      return { shortTerm: 0, episodic: 0, semantic: 0, procedural: 0 }
    }

    // Base allocation (equal split among active types)
    const basePerType = memoryBudget / activeTypes

    // Priority adjustments
    const priorities = {
      shortTerm: 1.5,    // Higher priority (recent context)
      episodic: 1.0,     // Normal
      semantic: 0.8,     // Lower (can be looked up)
      procedural: 1.2    // Higher (action-oriented)
    }

    // Calculate weighted allocation
    let totalWeight = 0
    if (hasShortTerm) totalWeight += priorities.shortTerm
    if (hasEpisodic) totalWeight += priorities.episodic
    if (hasSemantic) totalWeight += priorities.semantic
    if (hasProcedural) totalWeight += priorities.procedural

    return {
      shortTerm: hasShortTerm
        ? Math.floor(memoryBudget * priorities.shortTerm / totalWeight)
        : 0,
      episodic: hasEpisodic
        ? Math.floor(memoryBudget * priorities.episodic / totalWeight)
        : 0,
      semantic: hasSemantic
        ? Math.floor(memoryBudget * priorities.semantic / totalWeight)
        : 0,
      procedural: hasProcedural
        ? Math.floor(memoryBudget * priorities.procedural / totalWeight)
        : 0
    }
  }
}
```

---

## 3. 圧縮戦略の実装

### 3.1 Condensation（AI要約）の実装

```typescript
class CondensationEngine {
  constructor(
    private llm: LLMClient,
    private tokenCounter: TokenCounter
  ) {}

  /**
   * メッセージ群をAIで要約して圧縮
   * 70-90%のトークン削減を目指す
   */
  async condense(
    messages: Message[],
    keepRecent: number = 10
  ): Promise<CondensationResult> {
    // 1. Split messages into "to condense" and "to keep"
    const { toCondense, toKeep, toolBlocks } = this.splitMessages(messages, keepRecent)

    if (toCondense.length === 0) {
      return { success: false, reason: 'Nothing to condense' }
    }

    // 2. Create condensation prompt
    const prompt = this.createCondensationPrompt(toCondense)

    // 3. Call LLM for summarization
    const summaryResponse = await this.llm.generate({
      messages: [
        {
          role: 'user',
          content: prompt
        }
      ],
      temperature: 0.3,  // Lower temperature for consistent summaries
      maxTokens: 4000
    })

    const summary = summaryResponse.content[0].text

    // 4. Extract reasoning blocks if present
    const reasoningBlocks = summaryResponse.content
      .filter(c => c.type === 'reasoning')

    // 5. Create condensed message
    const condensedMessage: Message = {
      id: crypto.randomUUID(),
      role: 'assistant',
      content: [
        {
          type: 'reasoning',
          text: `[Context Condensation: ${toCondense.length} messages → summary]`
        },
        ...reasoningBlocks,
        {
          type: 'text',
          text: `## Context Summary\n\n${summary}`
        },
        ...toolBlocks  // Preserve tool_use blocks
      ],
      condenseId: crypto.randomUUID(),
      timestamp: Date.now()
    }

    // 6. Tag original messages
    const tagged = toCondense.map(msg => ({
      ...msg,
      condenseParent: condensedMessage.condenseId
    }))

    // 7. Calculate compression ratio
    const originalTokens = this.tokenCounter.countMessages(toCondense)
    const condensedTokens = this.tokenCounter.countMessages([condensedMessage])
    const ratio = 1 - (condensedTokens / originalTokens)

    return {
      success: true,
      condensedMessage,
      taggedMessages: tagged,
      compressionRatio: ratio,
      originalTokens,
      condensedTokens
    }
  }

  private splitMessages(messages: Message[], keepRecent: number) {
    const toKeep = messages.slice(-keepRecent)
    const toCondense = messages.slice(0, -keepRecent)

    // Extract tool_use blocks to preserve (Native Tools requirement)
    const toolBlocks: ContentBlock[] = []
    for (const msg of toCondense) {
      const tools = msg.content.filter(c => c.type === 'tool_use')
      toolBlocks.push(...tools)
    }

    return { toCondense, toKeep, toolBlocks }
  }

  private createCondensationPrompt(messages: Message[]): string {
    return `You are tasked with creating a concise summary of the following conversation history.

## Requirements:
1. Capture key decisions, commitments, and outcomes
2. Preserve important facts and context
3. Maintain chronological flow where relevant
4. Use bullet points for clarity
5. Aim for 70-90% token reduction
6. Focus on information that will be useful for continuing the conversation

## Conversation to summarize:

${this.formatMessages(messages)}

## Your summary:
`
  }

  private formatMessages(messages: Message[]): string {
    return messages.map(msg => {
      const text = msg.content
        .filter(c => c.type === 'text')
        .map(c => c.text)
        .join('\n')
      return `[${msg.role}]: ${text}`
    }).join('\n\n---\n\n')
  }
}

interface CondensationResult {
  success: boolean
  reason?: string
  condensedMessage?: Message
  taggedMessages?: Message[]
  compressionRatio?: number
  originalTokens?: number
  condensedTokens?: number
}
```

### 3.2 Truncation（スライディングウィンドウ）の実装

```typescript
class TruncationEngine {
  /**
   * スライディングウィンドウで古いメッセージをタグ付け
   * Condensation失敗時のフォールバック
   */
  truncate(
    messages: Message[],
    ratio: number = 0.5  // 50% of messages to truncate
  ): TruncationResult {
    // 1. Calculate how many to truncate
    const truncateCount = Math.floor(messages.length * ratio)

    if (truncateCount === 0) {
      return { success: false, reason: 'Nothing to truncate' }
    }

    // 2. Identify candidates (oldest messages first)
    const candidates = messages.slice(0, truncateCount)
    const keeping = messages.slice(truncateCount)

    // 3. Preserve critical messages
    const { toTruncate, preserved } = this.preserveCritical(candidates)

    // 4. Tag messages for truncation
    const truncateId = crypto.randomUUID()
    const tagged = toTruncate.map(msg => ({
      ...msg,
      truncationParent: truncateId
    }))

    // 5. Calculate reduction
    const originalTokens = this.countTokens(messages)
    const newTokens = this.countTokens([...preserved, ...keeping])
    const reduction = originalTokens - newTokens

    return {
      success: true,
      taggedMessages: tagged,
      preservedMessages: preserved,
      truncationId: truncateId,
      tokensReduced: reduction,
      reductionRatio: reduction / originalTokens
    }
  }

  /**
   * Preserve messages that should not be truncated
   */
  private preserveCritical(candidates: Message[]): {
    toTruncate: Message[]
    preserved: Message[]
  } {
    const preserved: Message[] = []
    const toTruncate: Message[] = []

    for (const msg of candidates) {
      // Preserve if:
      // 1. Marked as important
      // 2. Contains tool_use/tool_result (Native Tools requirement)
      // 3. Contains decisions
      // 4. User message (higher priority)

      const hasTools = msg.content.some(c =>
        c.type === 'tool_use' || c.type === 'tool_result'
      )

      const isImportant = msg.important === true
      const isUser = msg.role === 'user'
      const hasDecision = this.containsDecision(msg)

      if (hasTools || isImportant || (isUser && hasDecision)) {
        preserved.push(msg)
      } else {
        toTruncate.push(msg)
      }
    }

    return { toTruncate, preserved }
  }

  private containsDecision(msg: Message): boolean {
    const text = msg.content
      .filter(c => c.type === 'text')
      .map(c => c.text)
      .join(' ')

    const patterns = [
      /decided/i,
      /agreed/i,
      /will do/i,
      /let's/i,
      /plan is/i
    ]

    return patterns.some(p => p.test(text))
  }

  private countTokens(messages: Message[]): number {
    // Use tiktoken or similar
    const text = JSON.stringify(messages)
    return Math.ceil(text.length / 4)
  }
}

interface TruncationResult {
  success: boolean
  reason?: string
  taggedMessages?: Message[]
  preservedMessages?: Message[]
  truncationId?: string
  tokensReduced?: number
  reductionRatio?: number
}
```

### 3.3 二段階圧縮戦略

Condensation → Truncation のフォールバック戦略：

```typescript
class CompressionOrchestrator {
  constructor(
    private condensation: CondensationEngine,
    private truncation: TruncationEngine
  ) {}

  /**
   * 二段階圧縮: まずCondensation、失敗したらTruncation
   */
  async compress(
    messages: Message[],
    targetTokens: number
  ): Promise<CompressionResult> {
    const currentTokens = this.countTokens(messages)

    if (currentTokens <= targetTokens) {
      return {
        method: 'none',
        success: true,
        messages: messages
      }
    }

    // Stage 1: Try Condensation (AI-powered)
    try {
      const condensed = await this.condensation.condense(messages, 10)

      if (condensed.success && condensed.condensedTokens! < targetTokens) {
        return {
          method: 'condensation',
          success: true,
          messages: this.applyCondensation(messages, condensed),
          metadata: {
            compressionRatio: condensed.compressionRatio,
            originalTokens: condensed.originalTokens,
            finalTokens: condensed.condensedTokens
          }
        }
      }
    } catch (error) {
      console.warn('Condensation failed, falling back to truncation', error)
    }

    // Stage 2: Fallback to Truncation
    const ratio = 1 - (targetTokens / currentTokens)
    const truncated = this.truncation.truncate(messages, ratio)

    if (truncated.success) {
      return {
        method: 'truncation',
        success: true,
        messages: this.applyTruncation(messages, truncated),
        metadata: {
          tokensReduced: truncated.tokensReduced,
          reductionRatio: truncated.reductionRatio
        }
      }
    }

    // Both failed
    return {
      method: 'failed',
      success: false,
      messages: messages
    }
  }

  private applyCondensation(
    messages: Message[],
    result: CondensationResult
  ): Message[] {
    const condensedIds = new Set(result.taggedMessages!.map(m => m.id))

    return [
      ...messages.filter(m => !condensedIds.has(m.id)),
      result.condensedMessage!
    ]
  }

  private applyTruncation(
    messages: Message[],
    result: TruncationResult
  ): Message[] {
    const truncatedIds = new Set(result.taggedMessages!.map(m => m.id))

    return messages.filter(m => !truncatedIds.has(m.id))
  }

  private countTokens(messages: Message[]): number {
    return Math.ceil(JSON.stringify(messages).length / 4)
  }
}

interface CompressionResult {
  method: 'none' | 'condensation' | 'truncation' | 'failed'
  success: boolean
  messages: Message[]
  metadata?: {
    compressionRatio?: number
    originalTokens?: number
    finalTokens?: number
    tokensReduced?: number
    reductionRatio?: number
  }
}
```

---

## 4. tool_use/tool_resultペア保持

Native Toolsプロトコルに準拠したペア保持の実装。

### 4.1 ペア検出

```typescript
class ToolPairDetector {
  /**
   * tool_use と tool_result のペアを検出
   */
  detectPairs(messages: Message[]): ToolPair[] {
    const pairs: ToolPair[] = []
    const toolUseMap = new Map<string, ToolUseInfo>()

    // First pass: collect all tool_use blocks
    for (let i = 0; i < messages.length; i++) {
      const msg = messages[i]

      for (const content of msg.content) {
        if (content.type === 'tool_use') {
          toolUseMap.set(content.id, {
            toolUseId: content.id,
            toolName: content.name,
            messageIndex: i,
            messageId: msg.id,
            content: content
          })
        }
      }
    }

    // Second pass: find matching tool_result blocks
    for (let i = 0; i < messages.length; i++) {
      const msg = messages[i]

      for (const content of msg.content) {
        if (content.type === 'tool_result') {
          const toolUse = toolUseMap.get(content.tool_use_id)

          if (toolUse) {
            pairs.push({
              toolUseId: content.tool_use_id,
              toolName: toolUse.toolName,
              toolUseMessageIndex: toolUse.messageIndex,
              toolResultMessageIndex: i,
              toolUseMessageId: toolUse.messageId,
              toolResultMessageId: msg.id,
              toolUseContent: toolUse.content,
              toolResultContent: content
            })
          }
        }
      }
    }

    return pairs
  }

  /**
   * メッセージが未完了のtool_useを含むかチェック
   */
  hasOpenToolUse(message: Message, pairs: ToolPair[]): boolean {
    const completedToolUseIds = new Set(pairs.map(p => p.toolUseId))

    return message.content.some(c =>
      c.type === 'tool_use' && !completedToolUseIds.has(c.id)
    )
  }

  /**
   * ペアの相方メッセージを取得
   */
  getPairPartner(
    messageId: string,
    messages: Message[],
    pairs: ToolPair[]
  ): Message | null {
    const relevantPairs = pairs.filter(p =>
      p.toolUseMessageId === messageId || p.toolResultMessageId === messageId
    )

    if (relevantPairs.length === 0) return null

    // Get partner message ID
    const partnerId = relevantPairs[0].toolUseMessageId === messageId
      ? relevantPairs[0].toolResultMessageId
      : relevantPairs[0].toolUseMessageId

    return messages.find(m => m.id === partnerId) || null
  }
}

interface ToolPair {
  toolUseId: string
  toolName: string
  toolUseMessageIndex: number
  toolResultMessageIndex: number
  toolUseMessageId: string
  toolResultMessageId: string
  toolUseContent: ContentBlock
  toolResultContent: ContentBlock
}

interface ToolUseInfo {
  toolUseId: string
  toolName: string
  messageIndex: number
  messageId: string
  content: ContentBlock
}
```

### 4.2 圧縮時のペア保護

```typescript
class PairPreservingCompressor {
  constructor(
    private pairDetector: ToolPairDetector,
    private compressor: CompressionOrchestrator
  ) {}

  /**
   * ペアを壊さずに圧縮
   */
  async compressPreservingPairs(
    messages: Message[],
    targetTokens: number
  ): Promise<Message[]> {
    // 1. Detect all pairs
    const pairs = this.pairDetector.detectPairs(messages)

    // 2. Mark messages as part of pairs
    const pairMessageIds = new Set<string>()
    for (const pair of pairs) {
      pairMessageIds.add(pair.toolUseMessageId)
      pairMessageIds.add(pair.toolResultMessageId)
    }

    // 3. Separate pair messages from others
    const pairMessages: Message[] = []
    const otherMessages: Message[] = []

    for (const msg of messages) {
      if (pairMessageIds.has(msg.id)) {
        pairMessages.push(msg)
      } else {
        otherMessages.push(msg)
      }
    }

    // 4. Calculate token budget
    const pairTokens = this.countTokens(pairMessages)
    const remainingBudget = targetTokens - pairTokens

    if (remainingBudget <= 0) {
      // Pairs alone exceed budget - keep all pairs anyway (critical)
      console.warn('Tool pairs exceed token budget, keeping anyway')
      return pairMessages
    }

    // 5. Compress other messages
    const compressed = await this.compressor.compress(
      otherMessages,
      remainingBudget
    )

    // 6. Merge and sort by original order
    const allMessages = [...pairMessages, ...compressed.messages]
    const originalOrder = new Map(messages.map((m, i) => [m.id, i]))

    allMessages.sort((a, b) => {
      const orderA = originalOrder.get(a.id) ?? Infinity
      const orderB = originalOrder.get(b.id) ?? Infinity
      return orderA - orderB
    })

    return allMessages
  }

  private countTokens(messages: Message[]): number {
    return Math.ceil(JSON.stringify(messages).length / 4)
  }
}
```

---

## 5. レースコンディション対策

### 5.1 メッセージロックメカニズム

```typescript
class MessageLock {
  private locks = new Map<string, Promise<void>>()
  private lockHolders = new Map<string, string>()

  /**
   * メッセージIDに対してロックを取得
   */
  async acquire(messageId: string, holderId: string): Promise<() => void> {
    // Wait for existing lock if any
    while (this.locks.has(messageId)) {
      await this.locks.get(messageId)
    }

    // Create new lock
    let releaseFn: () => void
    const lockPromise = new Promise<void>(resolve => {
      releaseFn = resolve
    })

    this.locks.set(messageId, lockPromise)
    this.lockHolders.set(messageId, holderId)

    // Return release function
    return () => {
      this.locks.delete(messageId)
      this.lockHolders.delete(messageId)
      releaseFn!()
    }
  }

  /**
   * ロックの強制解放（タイムアウト時など）
   */
  forceRelease(messageId: string): void {
    this.locks.delete(messageId)
    this.lockHolders.delete(messageId)
  }

  /**
   * ロック状態を確認
   */
  isLocked(messageId: string): boolean {
    return this.locks.has(messageId)
  }

  /**
   * ロックホルダーを取得
   */
  getHolder(messageId: string): string | undefined {
    return this.lockHolders.get(messageId)
  }
}
```

### 5.2 トランザクショナルなメッセージ操作

```typescript
class TransactionalMessageManager {
  constructor(
    private lock: MessageLock,
    private storage: MessageStorage
  ) {}

  /**
   * アトミックなメッセージ更新
   */
  async updateMessage<T>(
    messageId: string,
    updateFn: (msg: Message) => Message,
    operationId: string
  ): Promise<Message> {
    const release = await this.lock.acquire(messageId, operationId)

    try {
      // 1. Read current state
      const current = await this.storage.getMessage(messageId)

      if (!current) {
        throw new Error(`Message ${messageId} not found`)
      }

      // 2. Apply update
      const updated = updateFn(current)

      // 3. Validate
      this.validate(updated)

      // 4. Write back
      await this.storage.updateMessage(messageId, updated)

      return updated
    } finally {
      release()
    }
  }

  /**
   * バッチ更新（複数メッセージを一貫性を保って更新）
   */
  async batchUpdate(
    updates: Array<{ messageId: string; updateFn: (msg: Message) => Message }>,
    operationId: string
  ): Promise<Message[]> {
    // 1. Sort message IDs to prevent deadlock
    const sorted = updates.slice().sort((a, b) =>
      a.messageId.localeCompare(b.messageId)
    )

    // 2. Acquire all locks
    const releases: Array<() => void> = []
    for (const { messageId } of sorted) {
      const release = await this.lock.acquire(messageId, operationId)
      releases.push(release)
    }

    try {
      // 3. Apply all updates
      const results: Message[] = []

      for (const { messageId, updateFn } of sorted) {
        const current = await this.storage.getMessage(messageId)
        if (!current) {
          throw new Error(`Message ${messageId} not found`)
        }

        const updated = updateFn(current)
        this.validate(updated)

        await this.storage.updateMessage(messageId, updated)
        results.push(updated)
      }

      return results
    } finally {
      // 4. Release all locks (in reverse order)
      for (const release of releases.reverse()) {
        release()
      }
    }
  }

  private validate(message: Message): void {
    if (!message.id) throw new Error('Message must have ID')
    if (!message.role) throw new Error('Message must have role')
    if (!message.content) throw new Error('Message must have content')
  }
}
```

### 5.3 楽観的ロック（バージョニング）

```typescript
interface VersionedMessage extends Message {
  version: number
  lastModified: number
}

class OptimisticLockManager {
  /**
   * 楽観的ロックでメッセージを更新
   */
  async updateWithOptimisticLock(
    messageId: string,
    updateFn: (msg: VersionedMessage) => VersionedMessage,
    maxRetries: number = 3
  ): Promise<VersionedMessage> {
    let retries = 0

    while (retries < maxRetries) {
      // 1. Read current version
      const current = await this.storage.getMessage(messageId) as VersionedMessage

      if (!current) {
        throw new Error(`Message ${messageId} not found`)
      }

      // 2. Apply update
      const updated = updateFn({ ...current })
      updated.version = current.version + 1
      updated.lastModified = Date.now()

      // 3. Try to write with version check
      const success = await this.storage.updateIfVersionMatches(
        messageId,
        updated,
        current.version
      )

      if (success) {
        return updated
      }

      // Version mismatch - retry
      retries++
      await this.sleep(Math.pow(2, retries) * 100) // Exponential backoff
    }

    throw new Error(`Failed to update message ${messageId} after ${maxRetries} retries`)
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms))
  }
}
```

---

## 6. チェックポイント統合

### 6.1 チェックポイント保存

```typescript
class CheckpointManager {
  constructor(
    private storage: CheckpointStorage,
    private compressor: CompressionOrchestrator
  ) {}

  /**
   * 現在の状態をチェックポイントとして保存
   */
  async saveCheckpoint(
    conversationId: string,
    state: LayeredState,
    messages: Message[]
  ): Promise<Checkpoint> {
    const checkpoint: Checkpoint = {
      id: crypto.randomUUID(),
      conversationId,
      timestamp: Date.now(),
      state: this.cloneState(state),
      messages: this.cloneMessages(messages),
      metadata: {
        messageCount: messages.length,
        totalTokens: this.countTokens(messages),
        phase: state.l2_runtime.phase
      }
    }

    await this.storage.save(checkpoint)

    return checkpoint
  }

  /**
   * チェックポイントから復元
   */
  async restore(checkpointId: string): Promise<{
    state: LayeredState
    messages: Message[]
  }> {
    const checkpoint = await this.storage.load(checkpointId)

    if (!checkpoint) {
      throw new Error(`Checkpoint ${checkpointId} not found`)
    }

    return {
      state: this.cloneState(checkpoint.state),
      messages: this.cloneMessages(checkpoint.messages)
    }
  }

  /**
   * チェックポイント一覧を取得
   */
  async listCheckpoints(
    conversationId: string,
    limit: number = 10
  ): Promise<CheckpointMetadata[]> {
    const checkpoints = await this.storage.list(conversationId, limit)

    return checkpoints.map(cp => ({
      id: cp.id,
      timestamp: cp.timestamp,
      messageCount: cp.metadata.messageCount,
      totalTokens: cp.metadata.totalTokens,
      phase: cp.metadata.phase
    }))
  }

  /**
   * 古いチェックポイントを削除
   */
  async cleanup(
    conversationId: string,
    keepCount: number = 5
  ): Promise<number> {
    const checkpoints = await this.storage.list(conversationId)

    if (checkpoints.length <= keepCount) {
      return 0
    }

    // Sort by timestamp (newest first)
    checkpoints.sort((a, b) => b.timestamp - a.timestamp)

    // Delete old ones
    const toDelete = checkpoints.slice(keepCount)

    for (const cp of toDelete) {
      await this.storage.delete(cp.id)
    }

    return toDelete.length
  }

  private cloneState(state: LayeredState): LayeredState {
    return JSON.parse(JSON.stringify(state))
  }

  private cloneMessages(messages: Message[]): Message[] {
    return JSON.parse(JSON.stringify(messages))
  }

  private countTokens(messages: Message[]): number {
    return Math.ceil(JSON.stringify(messages).length / 4)
  }
}

interface Checkpoint {
  id: string
  conversationId: string
  timestamp: number
  state: LayeredState
  messages: Message[]
  metadata: {
    messageCount: number
    totalTokens: number
    phase: Phase
  }
}

interface CheckpointMetadata {
  id: string
  timestamp: number
  messageCount: number
  totalTokens: number
  phase: Phase
}
```

### 6.2 自動チェックポイント

```typescript
class AutoCheckpointManager {
  private lastCheckpoint: number = 0
  private checkpointInterval: number = 10 * 60 * 1000  // 10 minutes

  constructor(
    private checkpointManager: CheckpointManager
  ) {}

  /**
   * 条件に応じて自動的にチェックポイントを作成
   */
  async maybeCreateCheckpoint(
    conversationId: string,
    state: LayeredState,
    messages: Message[],
    force: boolean = false
  ): Promise<Checkpoint | null> {
    const now = Date.now()

    // Check if checkpoint is needed
    const shouldCheckpoint = force ||
      this.checkTimeBased(now) ||
      this.checkPhaseBased(state) ||
      this.checkMessageBased(messages) ||
      this.checkTokenBased(messages)

    if (!shouldCheckpoint) {
      return null
    }

    // Create checkpoint
    const checkpoint = await this.checkpointManager.saveCheckpoint(
      conversationId,
      state,
      messages
    )

    this.lastCheckpoint = now

    return checkpoint
  }

  private checkTimeBased(now: number): boolean {
    return now - this.lastCheckpoint >= this.checkpointInterval
  }

  private checkPhaseBased(state: LayeredState): boolean {
    // Checkpoint on phase transitions
    const previousPhase = this.previousPhase
    const currentPhase = state.l2_runtime.phase

    if (previousPhase && previousPhase !== currentPhase) {
      this.previousPhase = currentPhase
      return true
    }

    this.previousPhase = currentPhase
    return false
  }

  private checkMessageBased(messages: Message[]): boolean {
    // Checkpoint every N messages
    const messagesSinceCheckpoint = messages.length - this.lastMessageCount
    this.lastMessageCount = messages.length

    return messagesSinceCheckpoint >= 50
  }

  private checkTokenBased(messages: Message[]): boolean {
    // Checkpoint when tokens exceed threshold
    const tokens = this.countTokens(messages)
    return tokens >= 100000
  }

  private previousPhase: Phase | null = null
  private lastMessageCount: number = 0

  private countTokens(messages: Message[]): number {
    return Math.ceil(JSON.stringify(messages).length / 4)
  }
}
```

---

## 7. 実践的なコード例

### 7.1 完全な実行フロー

すべてのパターンを統合した実行例：

```typescript
class ModernContextEngine {
  private state: LayeredState
  private messages: Message[] = []
  private checkpointManager: CheckpointManager
  private compressor: CompressionOrchestrator
  private messageManager: TransactionalMessageManager
  private toolProjector: DynamicToolProjector
  private budgetAllocator: BudgetAllocator

  constructor(private config: EngineConfig) {
    this.state = this.initializeState()
    this.checkpointManager = new CheckpointManager(config.storage, compressor)
    this.compressor = new CompressionOrchestrator(condensation, truncation)
    this.messageManager = new TransactionalMessageManager(lock, storage)
    this.toolProjector = new DynamicToolProjector()
    this.budgetAllocator = new BudgetAllocator()
  }

  /**
   * メインの実行ループ
   */
  async executeTask(userInput: string): Promise<TaskResult> {
    // 1. Add user message
    await this.addUserMessage(userInput)

    // 2. Check if compression is needed
    if (this.needsCompression()) {
      await this.compressContext()
    }

    // 3. Build context for LLM
    const context = await this.buildContext()

    // 4. Project tools based on current state
    const tools = this.toolProjector.project(this.state)

    // 5. Call LLM
    const response = await this.llm.generate({
      system: context.system,
      messages: context.messages,
      tools: tools,
      maxTokens: this.config.maxTokens
    })

    // 6. Process response
    await this.processResponse(response)

    // 7. Update state
    this.updateState(response)

    // 8. Maybe create checkpoint
    await this.checkpointManager.maybeCreateCheckpoint(
      this.config.conversationId,
      this.state,
      this.messages
    )

    return {
      success: true,
      response: response.content
    }
  }

  private async addUserMessage(input: string): Promise<void> {
    const message: Message = {
      id: crypto.randomUUID(),
      role: 'user',
      content: [{ type: 'text', text: input }],
      timestamp: Date.now()
    }

    this.messages.push(message)
  }

  private needsCompression(): boolean {
    const currentTokens = this.countTokens(this.messages)
    const threshold = this.config.maxTokens * 0.75  // 75%

    return currentTokens >= threshold
  }

  private async compressContext(): Promise<void> {
    const targetTokens = this.config.maxTokens * 0.5  // Compress to 50%

    const result = await this.compressor.compress(
      this.messages,
      targetTokens
    )

    if (result.success) {
      this.messages = result.messages
    }
  }

  private async buildContext(): Promise<ContextOutput> {
    const budget = this.budgetAllocator.allocate(
      this.config.maxTokens,
      this.state.l2_runtime.phase,
      this.config.model
    )

    const builder = new ContextBuilder(this.state, this.messages, budget)
    return builder.buildAllLayers()
  }

  private async processResponse(response: LLMResponse): Promise<void> {
    const message: Message = {
      id: crypto.randomUUID(),
      role: 'assistant',
      content: response.content,
      timestamp: Date.now()
    }

    this.messages.push(message)

    // Handle tool calls if any
    for (const content of response.content) {
      if (content.type === 'tool_use') {
        await this.executeTool(content)
      }
    }
  }

  private async executeTool(toolUse: ToolUseBlock): Promise<void> {
    // Execute the tool
    const result = await this.toolExecutor.execute(
      toolUse.name,
      toolUse.input
    )

    // Add tool result message
    const message: Message = {
      id: crypto.randomUUID(),
      role: 'user',
      content: [{
        type: 'tool_result',
        tool_use_id: toolUse.id,
        content: result
      }],
      timestamp: Date.now()
    }

    this.messages.push(message)
  }

  private updateState(response: LLMResponse): void {
    // Update phase if needed
    // Update permissions if needed
    // Update memory if needed
    // etc.
  }

  private initializeState(): LayeredState {
    return {
      l0_system: { /* ... */ },
      l1_task: { /* ... */ },
      l2_runtime: { phase: 'planning', /* ... */ },
      l3_memory: { /* ... */ },
      l4_evidence: { /* ... */ },
      l5_workBuffer: { /* ... */ }
    }
  }

  private countTokens(messages: Message[]): number {
    return Math.ceil(JSON.stringify(messages).length / 4)
  }
}

interface EngineConfig {
  conversationId: string
  maxTokens: number
  model: ModelType
  storage: CheckpointStorage
}

interface TaskResult {
  success: boolean
  response: ContentBlock[]
  error?: string
}
```

### 7.2 エラーハンドリングとリカバリ

```typescript
class RobustContextEngine extends ModernContextEngine {
  /**
   * エラーハンドリング付き実行
   */
  async executeTaskSafely(userInput: string): Promise<TaskResult> {
    try {
      return await this.executeTask(userInput)
    } catch (error) {
      return await this.handleError(error as Error, userInput)
    }
  }

  private async handleError(
    error: Error,
    userInput: string
  ): Promise<TaskResult> {
    console.error('Task execution failed:', error)

    // Try to recover
    if (error.message.includes('token limit')) {
      // Force compression
      await this.forceCompression()
      return await this.executeTask(userInput)
    }

    if (error.message.includes('checkpoint')) {
      // Restore from last checkpoint
      await this.restoreFromCheckpoint()
      return await this.executeTask(userInput)
    }

    // Cannot recover
    return {
      success: false,
      response: [],
      error: error.message
    }
  }

  private async forceCompression(): Promise<void> {
    const targetTokens = this.config.maxTokens * 0.3  // Aggressive: 30%

    const result = await this.compressor.compress(
      this.messages,
      targetTokens
    )

    if (result.success) {
      this.messages = result.messages
    } else {
      // Truncate as last resort
      this.messages = this.messages.slice(-10)
    }
  }

  private async restoreFromCheckpoint(): Promise<void> {
    const checkpoints = await this.checkpointManager.listCheckpoints(
      this.config.conversationId,
      1
    )

    if (checkpoints.length > 0) {
      const restored = await this.checkpointManager.restore(checkpoints[0].id)
      this.state = restored.state
      this.messages = restored.messages
    }
  }
}
```

---

## まとめ:  実装パターン集

この実装パターン集では、以下を提供しました：

1. **優先度管理**: スコアリングと選択アルゴリズム
2. **トークン予算配分**: 動的配分と層内二次配分
3. **圧縮戦略**: Condensation、Truncation、二段階戦略
4. **ペア保持**: Native Toolsプロトコル準拠
5. **レースコンディション対策**: ロック、トランザクション、楽観的ロック
6. **チェックポイント**: 保存、復元、自動化
7. **実践例**: 完全な実行フローとエラーハンドリング

これらのパターンを組み合わせることで、堅牢で効率的なContext Management システムを構築できます。

# 04. ベストプラクティス

このドキュメントでは、モダンなContext Management（2025年版）を実装・運用する際の**ベストプラクティス、よくある落とし穴、設計原則**を解説します。


## 1. 設計原則

### 1.1 SOLID原則の適用

#### Single Responsibility Principle（単一責任の原則）

各コンポーネントは1つの責任のみを持つべきです。

```typescript
// ❌ BAD: 複数の責任が混在
class ContextManager {
  compressMessages() { /* ... */ }
  allocateBudget() { /* ... */ }
  executeTool() { /* ... */ }
  saveCheckpoint() { /* ... */ }
  projectTools() { /* ... */ }
}

// ✅ GOOD: 責任を分離
class CompressionEngine {
  compressMessages() { /* ... */ }
}

class BudgetAllocator {
  allocateBudget() { /* ... */ }
}

class ToolExecutor {
  executeTool() { /* ... */ }
}

class CheckpointManager {
  saveCheckpoint() { /* ... */ }
}

class ToolProjector {
  projectTools() { /* ... */ }
}
```

#### Interface Segregation（インターフェース分離の原則）

クライアントは使わないメソッドに依存すべきではありません。

```typescript
// ❌ BAD: 巨大なインターフェース
interface ContextEngine {
  compress(): Promise<void>
  expand(): Promise<void>
  allocate(): void
  project(): Tool[]
  save(): Promise<void>
  restore(): Promise<void>
  execute(): Promise<void>
}

// ✅ GOOD: 小さく分離されたインターフェース
interface Compressible {
  compress(): Promise<void>
}

interface Expandable {
  expand(): Promise<void>
}

interface BudgetManaged {
  allocate(): void
}

interface ToolProjecting {
  project(): Tool[]
}

interface Checkpointable {
  save(): Promise<void>
  restore(): Promise<void>
}
```

### 1.2 依存性注入（DI）

ハードコーディングではなく、依存性を注入します。

```typescript
// ❌ BAD: ハードコーディング
class ContextEngine {
  private compressor = new CondensationEngine()
  private storage = new FileStorage()

  async compress() {
    await this.compressor.condense(/* ... */)
  }
}

// ✅ GOOD: 依存性注入
class ContextEngine {
  constructor(
    private compressor: ICompressor,
    private storage: IStorage
  ) {}

  async compress() {
    await this.compressor.condense(/* ... */)
  }
}

// 使用例
const engine = new ContextEngine(
  new CondensationEngine(llm, tokenCounter),
  new FileStorage('/path/to/data')
)
```

### 1.3 不変性（Immutability）

可能な限り不変なデータ構造を使います。

```typescript
// ❌ BAD: 破壊的変更
function addMessage(messages: Message[], newMessage: Message): void {
  messages.push(newMessage)  // 元の配列を変更
}

// ✅ GOOD: 非破壊的
function addMessage(messages: Message[], newMessage: Message): Message[] {
  return [...messages, newMessage]  // 新しい配列を返す
}

// ❌ BAD: オブジェクトの直接変更
function updateState(state: LayeredState, phase: Phase): void {
  state.l2_runtime.phase = phase
}

// ✅ GOOD: 新しいオブジェクトを返す
function updateState(state: LayeredState, phase: Phase): LayeredState {
  return {
    ...state,
    l2_runtime: {
      ...state.l2_runtime,
      phase
    }
  }
}
```

### 1.4 非同期処理のベストプラクティス

```typescript
// ❌ BAD: Promise地獄
function processMessages() {
  return loadMessages().then(messages => {
    return compressMessages(messages).then(compressed => {
      return saveMessages(compressed).then(result => {
        return result
      })
    })
  })
}

// ✅ GOOD: async/await
async function processMessages() {
  const messages = await loadMessages()
  const compressed = await compressMessages(messages)
  const result = await saveMessages(compressed)
  return result
}

// ❌ BAD: 並列実行可能なのに直列
async function loadData() {
  const messages = await loadMessages()
  const state = await loadState()
  const checkpoints = await loadCheckpoints()
  return { messages, state, checkpoints }
}

// ✅ GOOD: 並列実行
async function loadData() {
  const [messages, state, checkpoints] = await Promise.all([
    loadMessages(),
    loadState(),
    loadCheckpoints()
  ])
  return { messages, state, checkpoints }
}
```

---

## 2. よくある落とし穴と回避方法

### 2.1 トークンカウントの不正確さ

**問題**: 簡易的な文字数カウントではトークン数が不正確

```typescript
// ❌ BAD: 文字数ベース（不正確）
function countTokens(text: string): number {
  return Math.ceil(text.length / 4)
}

// ✅ GOOD: tiktoken使用（正確）
import { encoding_for_model } from 'tiktoken'

function countTokens(text: string, model: string = 'gpt-4'): number {
  const encoding = encoding_for_model(model)
  const tokens = encoding.encode(text)
  encoding.free()
  return tokens.length
}
```

**ベストプラクティス**:
- 正確なトークンカウントには `tiktoken` を使用
- モデルごとに適切なエンコーディングを選択（`o200k_base` など）
- カウント結果はキャッシュする（同じテキストの再計算を避ける）

### 2.2 tool_use/tool_resultペアの破壊

**問題**: 圧縮時にペアが壊れてプロトコル違反

```typescript
// ❌ BAD: ペアを考慮せずに削除
function compress(messages: Message[]): Message[] {
  return messages.slice(-100)  // 単純に古いものを削除
}

// ✅ GOOD: ペアを保持
function compress(messages: Message[]): Message[] {
  const pairs = detectToolPairs(messages)
  const pairMessageIds = new Set(
    pairs.flatMap(p => [p.toolUseMessageId, p.toolResultMessageId])
  )

  // ペアメッセージは必ず保持
  const mustKeep = messages.filter(m => pairMessageIds.has(m.id))
  const canCompress = messages.filter(m => !pairMessageIds.has(m.id))

  const compressed = compressMessages(canCompress)

  return [...mustKeep, ...compressed].sort(byTimestamp)
}
```

**ベストプラクティス**:
- tool_use と tool_result は常にペアで保持
- 圧縮前に必ずペア検出を実行
- ペアメッセージは `critical` 扱いにする

### 2.3 レースコンディション

**問題**: 並行アクセスでメッセージが破損

```typescript
// ❌ BAD: 保護なし
async function addMessage(message: Message) {
  const messages = await loadMessages()
  messages.push(message)
  await saveMessages(messages)
}

// ✅ GOOD: ロック機構
async function addMessage(message: Message) {
  const release = await lock.acquire('messages')
  try {
    const messages = await loadMessages()
    messages.push(message)
    await saveMessages(messages)
  } finally {
    release()
  }
}
```

**ベストプラクティス**:
- 並行アクセスが予想される場合は必ずロック
- トランザクション境界を明確に
- デッドロックを避けるためロック順序を統一

### 2.4 メモリリーク

**問題**: 古いメッセージやチェックポイントが溜まり続ける

```typescript
// ❌ BAD: 無限に蓄積
class ContextEngine {
  private messages: Message[] = []
  private checkpoints: Checkpoint[] = []

  async addMessage(msg: Message) {
    this.messages.push(msg)  // 永遠に増え続ける
  }

  async saveCheckpoint() {
    this.checkpoints.push(createCheckpoint())  // これも増え続ける
  }
}

// ✅ GOOD: 定期的にクリーンアップ
class ContextEngine {
  private messages: Message[] = []
  private checkpoints: Checkpoint[] = []
  private maxCheckpoints = 10

  async addMessage(msg: Message) {
    this.messages.push(msg)

    // 定期的に圧縮
    if (this.needsCompression()) {
      await this.compress()
    }
  }

  async saveCheckpoint() {
    this.checkpoints.push(createCheckpoint())

    // 古いチェックポイントを削除
    if (this.checkpoints.length > this.maxCheckpoints) {
      this.checkpoints = this.checkpoints.slice(-this.maxCheckpoints)
    }
  }
}
```

**ベストプラクティス**:
- 最大サイズ/数を設定
- 定期的なクリーンアップ処理
- ガベージコレクション可能な設計

### 2.5 圧縮の過度な実行

**問題**: 頻繁に圧縮してコスト増大

```typescript
// ❌ BAD: 毎回圧縮
async function addMessage(msg: Message) {
  messages.push(msg)
  await compress()  // 毎回LLM呼び出し！
}

// ✅ GOOD: しきい値ベース
async function addMessage(msg: Message) {
  messages.push(msg)

  const tokens = countTokens(messages)
  const threshold = maxTokens * 0.75  // 75%で圧縮

  if (tokens >= threshold) {
    await compress()
  }
}
```

**ベストプラクティス**:
- しきい値ベースのトリガー（75-80%）
- 圧縮間隔の最小値を設定（例: 最低10分間隔）
- コスト追跡とアラート

### 2.6 State同期の失敗

**問題**: メッセージとStateが不整合

```typescript
// ❌ BAD: 個別に更新
async function processResponse(response: LLMResponse) {
  await saveMessage(response)
  await updateState(extractState(response))  // 失敗すると不整合
}

// ✅ GOOD: トランザクショナル
async function processResponse(response: LLMResponse) {
  await transaction(async (tx) => {
    await tx.saveMessage(response)
    await tx.updateState(extractState(response))
  })
}
```

**ベストプラクティス**:
- メッセージとStateは一緒に更新
- チェックポイントに両方含める
- 復元時も両方を復元

---

## 3. パフォーマンス最適化

### 3.1 トークンカウントのキャッシュ

```typescript
class CachedTokenCounter {
  private cache = new Map<string, number>()

  count(text: string): number {
    const hash = this.hash(text)

    if (this.cache.has(hash)) {
      return this.cache.get(hash)!
    }

    const tokens = this.actualCount(text)
    this.cache.set(hash, tokens)

    // キャッシュサイズ制限
    if (this.cache.size > 1000) {
      const firstKey = this.cache.keys().next().value
      this.cache.delete(firstKey)
    }

    return tokens
  }

  private hash(text: string): string {
    // Fast hash (e.g., FNV-1a)
    let hash = 2166136261
    for (let i = 0; i < text.length; i++) {
      hash ^= text.charCodeAt(i)
      hash = Math.imul(hash, 16777619)
    }
    return hash.toString(36)
  }

  private actualCount(text: string): number {
    const encoding = encoding_for_model('gpt-4')
    const tokens = encoding.encode(text)
    encoding.free()
    return tokens.length
  }
}
```

### 3.2 遅延読み込み（Lazy Loading）

```typescript
class LazyContextBuilder {
  private _messages: Message[] | null = null
  private _state: LayeredState | null = null

  // メッセージは必要になるまで読み込まない
  private async getMessages(): Promise<Message[]> {
    if (!this._messages) {
      this._messages = await this.storage.loadMessages()
    }
    return this._messages
  }

  // Evidence層だけ必要な場合、他の層は読み込まない
  async buildEvidenceLayer(): Promise<string> {
    const state = await this.getState()
    return this.formatEvidence(state.l4_evidence)
  }
}
```

### 3.3 並列処理

```typescript
class ParallelContextBuilder {
  async buildAllLayers(): Promise<ContextOutput> {
    // 独立した層は並列にビルド
    const [system, task, runtime, memory, evidence, workBuffer] =
      await Promise.all([
        this.buildSystemLayer(),
        this.buildTaskLayer(),
        this.buildRuntimeLayer(),
        this.buildMemoryLayer(),
        this.buildEvidenceLayer(),
        this.buildWorkBufferLayer()
      ])

    return {
      system: this.combineSystem(system, task),
      messages: this.combineMessages(
        runtime,
        memory,
        evidence,
        workBuffer
      )
    }
  }
}
```

### 3.4 インクリメンタルな更新

```typescript
class IncrementalCompressor {
  private lastCompressedIndex = 0

  async compress(messages: Message[]): Promise<Message[]> {
    // 前回圧縮済みの部分はスキップ
    const newMessages = messages.slice(this.lastCompressedIndex)

    if (newMessages.length < 10) {
      return messages  // 少量なら圧縮しない
    }

    const compressed = await this.condense(newMessages)
    this.lastCompressedIndex = messages.length

    return [...messages.slice(0, this.lastCompressedIndex), compressed]
  }
}
```

---

## 4. セキュリティとコンプライアンス

### 4.1 機密情報の保護

```typescript
class SecureContextManager {
  private sensitivePatterns = [
    /\b\d{3}-\d{2}-\d{4}\b/,  // SSN
    /\b\d{16}\b/,              // Credit card
    /sk-[a-zA-Z0-9]{48}/,      // API key
    /password[:\s]+\S+/i       // Password
  ]

  /**
   * メッセージから機密情報を検出・マスク
   */
  sanitize(message: Message): Message {
    return {
      ...message,
      content: message.content.map(c => {
        if (c.type === 'text') {
          return {
            ...c,
            text: this.maskSensitive(c.text)
          }
        }
        return c
      })
    }
  }

  private maskSensitive(text: string): string {
    let masked = text
    for (const pattern of this.sensitivePatterns) {
      masked = masked.replace(pattern, '[REDACTED]')
    }
    return masked
  }

  /**
   * チェックポイント保存前に必ずサニタイズ
   */
  async saveCheckpoint(checkpoint: Checkpoint): Promise<void> {
    const sanitized = {
      ...checkpoint,
      messages: checkpoint.messages.map(m => this.sanitize(m))
    }

    await this.storage.save(sanitized)
  }
}
```

### 4.2 権限チェック

```typescript
class PermissionAwareToolProjector {
  project(state: LayeredState, user: User): Tool[] {
    const allTools = this.getAllTools()

    return allTools.filter(tool => {
      // 1. Phase check
      if (!this.isAllowedInPhase(tool, state.l2_runtime.phase)) {
        return false
      }

      // 2. Permission check
      if (!this.hasPermission(user, tool)) {
        return false
      }

      // 3. Environment check
      if (!this.isAllowedInEnv(tool, state.l2_runtime.environment)) {
        return false
      }

      return true
    })
  }

  private hasPermission(user: User, tool: Tool): boolean {
    const requiredPermissions = tool.requiredPermissions || []
    return requiredPermissions.every(p => user.permissions.includes(p))
  }

  private isAllowedInEnv(tool: Tool, env: Environment): boolean {
    if (tool.destructive && env === 'production') {
      return false  // 本番環境で破壊的操作は禁止
    }
    return true
  }
}
```

### 4.3 監査ログ

```typescript
class AuditLogger {
  /**
   * すべてのツール呼び出しをログ
   */
  async logToolExecution(
    user: User,
    tool: Tool,
    input: any,
    output: any,
    timestamp: number
  ): Promise<void> {
    const entry: AuditEntry = {
      id: crypto.randomUUID(),
      timestamp,
      userId: user.id,
      toolName: tool.name,
      input: this.sanitize(input),
      output: this.sanitize(output),
      success: !output.error,
      ipAddress: user.ipAddress,
      sessionId: user.sessionId
    }

    await this.storage.append(entry)
  }

  /**
   * 機密情報をマスク
   */
  private sanitize(data: any): any {
    // Deep clone and mask sensitive fields
    const cloned = JSON.parse(JSON.stringify(data))
    return this.maskFields(cloned, ['password', 'apiKey', 'token'])
  }

  private maskFields(obj: any, fields: string[]): any {
    if (typeof obj !== 'object' || obj === null) {
      return obj
    }

    for (const key in obj) {
      if (fields.includes(key)) {
        obj[key] = '[REDACTED]'
      } else if (typeof obj[key] === 'object') {
        obj[key] = this.maskFields(obj[key], fields)
      }
    }

    return obj
  }
}
```

### 4.4 GDPR対応（削除の権利）

```typescript
class GDPRCompliantStorage {
  /**
   * ユーザーデータの完全削除
   */
  async deleteUserData(userId: string): Promise<void> {
    // 1. メッセージ削除
    await this.messageStorage.deleteByUser(userId)

    // 2. チェックポイント削除
    await this.checkpointStorage.deleteByUser(userId)

    // 3. 監査ログから個人情報をマスク（ログは残す）
    await this.auditLog.anonymizeUser(userId)

    // 4. キャッシュクリア
    await this.cache.clearUser(userId)

    // 5. 削除ログを記録
    await this.deletionLog.record({
      userId,
      timestamp: Date.now(),
      requestedBy: 'user'
    })
  }

  /**
   * データエクスポート（データポータビリティ）
   */
  async exportUserData(userId: string): Promise<UserDataExport> {
    const [messages, checkpoints, auditLogs] = await Promise.all([
      this.messageStorage.getByUser(userId),
      this.checkpointStorage.getByUser(userId),
      this.auditLog.getByUser(userId)
    ])

    return {
      userId,
      exportedAt: Date.now(),
      messages,
      checkpoints,
      auditLogs
    }
  }
}
```

---

## 5. テスト戦略

### 5.1 ユニットテスト

```typescript
describe('PriorityScorer', () => {
  it('should score user messages higher than assistant', () => {
    const scorer = new PriorityScorer()

    const userMsg: Message = {
      id: '1',
      role: 'user',
      content: [{ type: 'text', text: 'Hello' }]
    }

    const assistantMsg: Message = {
      id: '2',
      role: 'assistant',
      content: [{ type: 'text', text: 'Hi' }]
    }

    const userScore = scorer.scoreMessage(userMsg, createContext())
    const assistantScore = scorer.scoreMessage(assistantMsg, createContext())

    expect(userScore.score).toBeGreaterThan(assistantScore.score)
  })

  it('should preserve tool_use/tool_result pairs', () => {
    const compressor = new PairPreservingCompressor(detector, orchestrator)

    const messages = [
      createUserMessage('Use calculator'),
      createAssistantMessageWithToolUse('calculator', { expr: '1+1' }),
      createToolResultMessage('calculator', '2'),
      createAssistantMessage('The answer is 2')
    ]

    const compressed = await compressor.compressPreservingPairs(messages, 100)

    // ペアは必ず両方残っている
    const hasToolUse = compressed.some(m =>
      m.content.some(c => c.type === 'tool_use')
    )
    const hasToolResult = compressed.some(m =>
      m.content.some(c => c.type === 'tool_result')
    )

    expect(hasToolUse && hasToolResult).toBe(true)
  })
})
```

### 5.2 統合テスト

```typescript
describe('ContextEngine Integration', () => {
  it('should handle full execution flow', async () => {
    const engine = new ModernContextEngine({
      conversationId: 'test-123',
      maxTokens: 200000,
      model: 'claude-opus-4'
    })

    // 1. User input
    const result1 = await engine.executeTask('What is 2+2?')
    expect(result1.success).toBe(true)

    // 2. Follow-up
    const result2 = await engine.executeTask('What about 3+3?')
    expect(result2.success).toBe(true)

    // 3. Verify state
    const state = engine.getState()
    expect(state.l3_memory.shortTerm.length).toBeGreaterThan(0)

    // 4. Verify messages
    const messages = engine.getMessages()
    expect(messages.length).toBeGreaterThan(0)
  })

  it('should compress when threshold exceeded', async () => {
    const engine = new ModernContextEngine({
      conversationId: 'test-456',
      maxTokens: 1000,  // Very small
      model: 'claude-haiku-3-5'
    })

    // Fill with many messages
    for (let i = 0; i < 50; i++) {
      await engine.executeTask(`Message ${i}`)
    }

    // Verify compression occurred
    const messages = engine.getMessages()
    const hasSummary = messages.some(m =>
      m.content.some(c =>
        c.type === 'text' && c.text.includes('Context Summary')
      )
    )

    expect(hasSummary).toBe(true)
  })
})
```

### 5.3 パフォーマンステスト

```typescript
describe('Performance', () => {
  it('should handle 1000 messages efficiently', async () => {
    const messages = Array.from({ length: 1000 }, (_, i) =>
      createMessage(`Message ${i}`)
    )

    const start = Date.now()
    const scorer = new PriorityScorer()

    const scores = messages.map((msg, idx) =>
      scorer.scoreMessage(msg, { currentIndex: 1000, messageIndex: idx })
    )

    const elapsed = Date.now() - start

    expect(elapsed).toBeLessThan(1000)  // < 1 second
    expect(scores.length).toBe(1000)
  })

  it('should cache token counts', async () => {
    const counter = new CachedTokenCounter()
    const text = 'Hello world '.repeat(100)

    // First count (uncached)
    const start1 = Date.now()
    const count1 = counter.count(text)
    const time1 = Date.now() - start1

    // Second count (cached)
    const start2 = Date.now()
    const count2 = counter.count(text)
    const time2 = Date.now() - start2

    expect(count1).toBe(count2)
    expect(time2).toBeLessThan(time1 / 10)  // 10x faster
  })
})
```

### 5.4 Property-based Testing

```typescript
import fc from 'fast-check'

describe('Property-based tests', () => {
  it('compression should always reduce token count', () => {
    fc.assert(
      fc.asyncProperty(
        fc.array(fc.record({
          id: fc.uuid(),
          role: fc.constantFrom('user', 'assistant'),
          content: fc.array(fc.record({
            type: fc.constant('text'),
            text: fc.string({ minLength: 10, maxLength: 100 })
          }))
        }), { minLength: 20, maxLength: 100 }),
        async (messages) => {
          const originalTokens = countTokens(messages)
          const compressed = await compressor.compress(messages, originalTokens * 0.5)

          if (compressed.success) {
            const compressedTokens = countTokens(compressed.messages)
            expect(compressedTokens).toBeLessThanOrEqual(originalTokens)
          }
        }
      )
    )
  })
})
```

---

## 6. モニタリングとデバッグ

### 6.1 メトリクス収集

```typescript
class MetricsCollector {
  private metrics: Metrics = {
    tokenUsage: {
      total: 0,
      byLayer: {},
      utilization: 0
    },
    compression: {
      attempts: 0,
      successes: 0,
      failures: 0,
      avgRatio: 0
    },
    performance: {
      avgLatency: 0,
      p95Latency: 0,
      p99Latency: 0
    },
    errors: {
      total: 0,
      byType: {}
    }
  }

  recordTokenUsage(layer: string, tokens: number): void {
    this.metrics.tokenUsage.total += tokens
    this.metrics.tokenUsage.byLayer[layer] =
      (this.metrics.tokenUsage.byLayer[layer] || 0) + tokens
  }

  recordCompression(success: boolean, ratio: number): void {
    this.metrics.compression.attempts++

    if (success) {
      this.metrics.compression.successes++
      this.updateAvgRatio(ratio)
    } else {
      this.metrics.compression.failures++
    }
  }

  recordLatency(ms: number): void {
    this.latencies.push(ms)
    this.updateLatencyMetrics()
  }

  recordError(error: Error): void {
    this.metrics.errors.total++
    const type = error.constructor.name
    this.metrics.errors.byType[type] =
      (this.metrics.errors.byType[type] || 0) + 1
  }

  getMetrics(): Metrics {
    return { ...this.metrics }
  }

  private latencies: number[] = []

  private updateLatencyMetrics(): void {
    const sorted = this.latencies.slice().sort((a, b) => a - b)
    this.metrics.performance.avgLatency =
      sorted.reduce((a, b) => a + b, 0) / sorted.length
    this.metrics.performance.p95Latency =
      sorted[Math.floor(sorted.length * 0.95)]
    this.metrics.performance.p99Latency =
      sorted[Math.floor(sorted.length * 0.99)]
  }

  private updateAvgRatio(newRatio: number): void {
    const n = this.metrics.compression.successes
    const oldAvg = this.metrics.compression.avgRatio
    this.metrics.compression.avgRatio =
      (oldAvg * (n - 1) + newRatio) / n
  }
}
```

### 6.2 構造化ログ

```typescript
class StructuredLogger {
  log(level: LogLevel, message: string, context: Record<string, any>): void {
    const entry = {
      timestamp: new Date().toISOString(),
      level,
      message,
      ...context,
      // 自動的に追加される情報
      conversationId: this.conversationId,
      userId: this.userId,
      sessionId: this.sessionId
    }

    console.log(JSON.stringify(entry))
  }

  info(message: string, context?: Record<string, any>): void {
    this.log('INFO', message, context || {})
  }

  warn(message: string, context?: Record<string, any>): void {
    this.log('WARN', message, context || {})
  }

  error(message: string, error: Error, context?: Record<string, any>): void {
    this.log('ERROR', message, {
      ...context,
      error: {
        name: error.name,
        message: error.message,
        stack: error.stack
      }
    })
  }

  // 使用例
  logCompression(result: CompressionResult): void {
    this.info('Compression completed', {
      method: result.method,
      success: result.success,
      originalTokens: result.metadata?.originalTokens,
      finalTokens: result.metadata?.finalTokens,
      ratio: result.metadata?.compressionRatio
    })
  }
}
```

### 6.3 トレーシング

```typescript
class DistributedTracer {
  startSpan(operation: string, parent?: Span): Span {
    const span: Span = {
      id: crypto.randomUUID(),
      parentId: parent?.id,
      operation,
      startTime: Date.now(),
      attributes: {}
    }

    this.activeSpans.set(span.id, span)
    return span
  }

  endSpan(span: Span): void {
    span.endTime = Date.now()
    span.duration = span.endTime - span.startTime

    this.activeSpans.delete(span.id)
    this.completedSpans.push(span)

    // Send to tracing backend (e.g., Jaeger, Zipkin)
    this.backend.send(span)
  }

  addAttribute(span: Span, key: string, value: any): void {
    span.attributes[key] = value
  }

  // 使用例
  async executeWithTracing<T>(
    operation: string,
    fn: (span: Span) => Promise<T>
  ): Promise<T> {
    const span = this.startSpan(operation)

    try {
      const result = await fn(span)
      this.addAttribute(span, 'success', true)
      return result
    } catch (error) {
      this.addAttribute(span, 'success', false)
      this.addAttribute(span, 'error', (error as Error).message)
      throw error
    } finally {
      this.endSpan(span)
    }
  }

  private activeSpans = new Map<string, Span>()
  private completedSpans: Span[] = []
}

// 使用例
async function compressWithTracing(messages: Message[]): Promise<Message[]> {
  return tracer.executeWithTracing('compress', async (span) => {
    tracer.addAttribute(span, 'messageCount', messages.length)
    tracer.addAttribute(span, 'tokens', countTokens(messages))

    const result = await compressor.compress(messages, 100000)

    tracer.addAttribute(span, 'compressionRatio', result.metadata?.compressionRatio)

    return result.messages
  })
}
```

---

## 7. プロダクション運用チェックリスト

### 7.1 デプロイ前チェックリスト

#### セキュリティ

- [ ] 機密情報のマスキングが実装されている
- [ ] 権限チェックがすべてのツール呼び出しで実行される
- [ ] 監査ログが記録されている
- [ ] GDPR対応（削除、エクスポート）が実装されている
- [ ] 本番環境で破壊的操作が制限されている

#### パフォーマンス

- [ ] トークンカウントがキャッシュされている
- [ ] 圧縮しきい値が適切に設定されている（75-80%）
- [ ] 並列処理が可能な箇所で使われている
- [ ] メモリリーク対策がされている
- [ ] パフォーマンステストが通っている

#### 信頼性

- [ ] レースコンディション対策がされている
- [ ] tool_use/tool_resultペアが保持される
- [ ] エラーハンドリングとリカバリが実装されている
- [ ] チェックポイントからの復元が可能
- [ ] すべてのユニットテスト・統合テストが通っている

#### 観測可能性

- [ ] メトリクス収集が実装されている
- [ ] 構造化ログが出力されている
- [ ] トレーシングが有効になっている
- [ ] アラートが設定されている
- [ ] ダッシュボードが準備されている

### 7.2 本番監視項目

```typescript
const productionAlerts = {
  // トークン使用率
  tokenUtilization: {
    warning: 0.80,   // 80%で警告
    critical: 0.90   // 90%でクリティカル
  },

  // 圧縮成功率
  compressionSuccessRate: {
    warning: 0.90,   // 90%未満で警告
    critical: 0.80   // 80%未満でクリティカル
  },

  // レイテンシ
  latency: {
    p95Warning: 5000,    // 5秒
    p95Critical: 10000,  // 10秒
    p99Warning: 10000,   // 10秒
    p99Critical: 20000   // 20秒
  },

  // エラー率
  errorRate: {
    warning: 0.01,   // 1%
    critical: 0.05   // 5%
  },

  // メモリ使用量
  memoryUsage: {
    warning: 0.80,   // 80%
    critical: 0.90   // 90%
  }
}
```

### 7.3 インシデント対応手順

```typescript
class IncidentHandler {
  /**
   * トークン超過エラー
   */
  async handleTokenOverflow(): Promise<void> {
    // 1. アグレッシブに圧縮
    await this.forceCompress(0.3)  // 30%まで削減

    // 2. 古いチェックポイントを削除
    await this.cleanupCheckpoints()

    // 3. キャッシュクリア
    await this.clearCache()

    // 4. アラート通知
    await this.notify('Token overflow handled')
  }

  /**
   * 圧縮失敗
   */
  async handleCompressionFailure(): Promise<void> {
    // 1. Truncationにフォールバック
    await this.fallbackToTruncation()

    // 2. エラーログ記録
    this.logger.error('Compression failed, used truncation fallback')

    // 3. メトリクス記録
    this.metrics.recordCompressionFailure()
  }

  /**
   * データ破損検出
   */
  async handleDataCorruption(): Promise<void> {
    // 1. 最新のチェックポイントから復元
    const checkpoint = await this.getLatestCheckpoint()
    await this.restore(checkpoint)

    // 2. 破損データを隔離
    await this.quarantineCorruptedData()

    // 3. クリティカルアラート
    await this.alertCritical('Data corruption detected and recovered')
  }
}
```

### 7.4 段階的ロールアウト

```typescript
class FeatureFlags {
  private flags = {
    useCondensation: {
      enabled: false,
      rolloutPercentage: 0
    },
    useDynamicToolProjection: {
      enabled: false,
      rolloutPercentage: 0
    },
    useLayeredState: {
      enabled: false,
      rolloutPercentage: 0
    }
  }

  isEnabled(feature: string, userId: string): boolean {
    const flag = this.flags[feature]

    if (!flag || !flag.enabled) {
      return false
    }

    // Hash-based rollout
    const hash = this.hashUserId(userId)
    const bucket = hash % 100

    return bucket < flag.rolloutPercentage
  }

  // ロールアウト計画
  async rollout(feature: string): Promise<void> {
    // Phase 1: 5% (1 week)
    await this.setRollout(feature, 5)
    await this.monitor(7 * 24 * 60 * 60 * 1000)

    // Phase 2: 25% (1 week)
    await this.setRollout(feature, 25)
    await this.monitor(7 * 24 * 60 * 60 * 1000)

    // Phase 3: 50% (1 week)
    await this.setRollout(feature, 50)
    await this.monitor(7 * 24 * 60 * 60 * 1000)

    // Phase 4: 100%
    await this.setRollout(feature, 100)
  }

  private async monitor(duration: number): Promise<void> {
    // Monitor metrics and rollback if needed
    const metrics = await this.getMetrics()

    if (metrics.errorRate > 0.05) {
      throw new Error('Error rate too high, aborting rollout')
    }
  }

  private hashUserId(userId: string): number {
    let hash = 0
    for (let i = 0; i < userId.length; i++) {
      hash = ((hash << 5) - hash) + userId.charCodeAt(i)
      hash = hash & hash
    }
    return Math.abs(hash)
  }
}
```

---

## まとめ: ベストプラクティス

このベストプラクティスガイドでは、以下を提供しました：

1. **設計原則**: SOLID、DI、不変性、非同期処理
2. **よくある落とし穴**: トークンカウント、ペア破壊、レースコンディション、メモリリーク、過度な圧縮、State同期
3. **パフォーマンス最適化**: キャッシュ、遅延読み込み、並列処理、インクリメンタル更新
4. **セキュリティ**: 機密情報保護、権限チェック、監査ログ、GDPR対応
5. **テスト戦略**: ユニット、統合、パフォーマンス、Property-based
6. **モニタリング**: メトリクス、ログ、トレーシング
7. **プロダクション運用**: チェックリスト、監視項目、インシデント対応、段階的ロールアウト

これらのプラクティスを適用することで、堅牢で信頼性の高いContext Management システムを本番環境で運用できます。

# 05. 統合実装例

このドキュメントでは、モダンなContext Management（2025年版）を**実際のフレームワークやシステムと統合**する具体的な方法を示します。

---

## 目次

1. [LangGraph統合](#1-langgraph統合)
2. [GraphAI統合](#2-graphai統合)
3. [Mulmo統合](#3-mulmo統合)
4. [RAGシステム統合](#4-ragシステム統合)
5. [チェックポイントシステム統合](#5-チェックポイントシステム統合)
6. [既存システムからのマイグレーション](#6-既存システムからのマイグレーション)

---

## 1. LangGraph統合

LangGraphは状態ベースのワークフロー構築に適しています。Context ManagementをLangGraphの状態管理と統合します。

### 1.1 基本的な統合

```typescript
import { StateGraph, END } from '@langchain/langgraph'

// LangGraph用の状態定義
interface GraphState {
  // Context Management統合
  layeredState: LayeredState
  messages: Message[]
  contextBudget: BudgetAllocation

  // LangGraph固有
  currentNode: string
  nextAction: string | null
  toolResults: Record<string, any>
}

class LangGraphContextIntegration {
  private contextEngine: ModernContextEngine
  private graph: StateGraph<GraphState>

  constructor() {
    this.contextEngine = new ModernContextEngine({
      maxTokens: 200000,
      model: 'claude-opus-4'
    })

    this.graph = this.buildGraph()
  }

  private buildGraph(): StateGraph<GraphState> {
    const graph = new StateGraph<GraphState>({
      channels: {
        layeredState: null,
        messages: null,
        contextBudget: null,
        currentNode: null,
        nextAction: null,
        toolResults: null
      }
    })

    // ノード定義
    graph.addNode('initialize', this.initializeNode.bind(this))
    graph.addNode('plan', this.planNode.bind(this))
    graph.addNode('execute', this.executeNode.bind(this))
    graph.addNode('reflect', this.reflectNode.bind(this))
    graph.addNode('compress', this.compressNode.bind(this))

    // エッジ定義
    graph.addEdge('initialize', 'plan')
    graph.addConditionalEdges(
      'plan',
      this.shouldExecute.bind(this),
      {
        execute: 'execute',
        reflect: 'reflect',
        end: END
      }
    )
    graph.addConditionalEdges(
      'execute',
      this.needsCompression.bind(this),
      {
        compress: 'compress',
        reflect: 'reflect'
      }
    )
    graph.addEdge('compress', 'reflect')
    graph.addConditionalEdges(
      'reflect',
      this.shouldContinue.bind(this),
      {
        plan: 'plan',
        end: END
      }
    )

    graph.setEntryPoint('initialize')

    return graph
  }

  /**
   * 初期化ノード: Context Managementのセットアップ
   */
  private async initializeNode(state: GraphState): Promise<Partial<GraphState>> {
    // LayeredStateを初期化
    const layeredState = this.contextEngine.getState()

    // 予算配分
    const contextBudget = this.contextEngine.allocateBudget(
      200000,
      'planning',
      'claude-opus-4'
    )

    return {
      layeredState,
      contextBudget,
      currentNode: 'initialize',
      messages: []
    }
  }

  /**
   * プランニングノード: コンテキストを使って計画
   */
  private async planNode(state: GraphState): Promise<Partial<GraphState>> {
    // Phase更新
    const updatedState = {
      ...state.layeredState,
      l2_runtime: {
        ...state.layeredState.l2_runtime,
        phase: 'planning' as Phase
      }
    }

    // コンテキスト構築
    const context = await this.contextEngine.buildContext(
      updatedState,
      state.messages,
      state.contextBudget
    )

    // LLM呼び出し（プランニング用ツール投影）
    const tools = this.contextEngine.projectTools(updatedState, 'planning')

    const response = await this.callLLM({
      system: context.system,
      messages: context.messages,
      tools
    })

    // メッセージ追加
    const newMessages = [...state.messages, {
      id: crypto.randomUUID(),
      role: 'assistant' as const,
      content: response.content,
      timestamp: Date.now()
    }]

    return {
      layeredState: updatedState,
      messages: newMessages,
      currentNode: 'plan',
      nextAction: this.extractNextAction(response)
    }
  }

  /**
   * 実行ノード: ツールを実行
   */
  private async executeNode(state: GraphState): Promise<Partial<GraphState>> {
    // Phase更新
    const updatedState = {
      ...state.layeredState,
      l2_runtime: {
        ...state.layeredState.l2_runtime,
        phase: 'execution' as Phase
      }
    }

    // 実行用ツール投影（planningより多い）
    const tools = this.contextEngine.projectTools(updatedState, 'execution')

    // ツール実行
    const toolResults: Record<string, any> = {}
    const lastMessage = state.messages[state.messages.length - 1]

    for (const content of lastMessage.content) {
      if (content.type === 'tool_use') {
        const result = await this.executeTool(content.name, content.input)
        toolResults[content.id] = result

        // tool_result メッセージを追加
        state.messages.push({
          id: crypto.randomUUID(),
          role: 'user',
          content: [{
            type: 'tool_result',
            tool_use_id: content.id,
            content: JSON.stringify(result)
          }],
          timestamp: Date.now()
        })
      }
    }

    return {
      layeredState: updatedState,
      messages: state.messages,
      currentNode: 'execute',
      toolResults
    }
  }

  /**
   * 圧縮ノード: トークン数が多い場合に実行
   */
  private async compressNode(state: GraphState): Promise<Partial<GraphState>> {
    const compressed = await this.contextEngine.compress(
      state.messages,
      state.contextBudget.memory * 0.5  // 50%まで圧縮
    )

    return {
      messages: compressed.messages,
      currentNode: 'compress'
    }
  }

  /**
   * 振り返りノード: 結果を評価
   */
  private async reflectNode(state: GraphState): Promise<Partial<GraphState>> {
    const updatedState = {
      ...state.layeredState,
      l2_runtime: {
        ...state.layeredState.l2_runtime,
        phase: 'reflection' as Phase
      }
    }

    // Work Bufferを更新（実行結果を記録）
    const workBuffer = {
      ...state.layeredState.l5_workBuffer,
      executionResults: state.toolResults
    }

    const finalState = {
      ...updatedState,
      l5_workBuffer: workBuffer
    }

    return {
      layeredState: finalState,
      currentNode: 'reflect'
    }
  }

  // 条件分岐関数
  private shouldExecute(state: GraphState): string {
    if (state.nextAction === 'execute') return 'execute'
    if (state.nextAction === 'reflect') return 'reflect'
    return 'end'
  }

  private needsCompression(state: GraphState): string {
    const tokens = this.contextEngine.countTokens(state.messages)
    const threshold = state.contextBudget.memory * 0.75

    return tokens > threshold ? 'compress' : 'reflect'
  }

  private shouldContinue(state: GraphState): string {
    // タスク完了判定
    const isComplete = this.isTaskComplete(state)
    return isComplete ? 'end' : 'plan'
  }

  // ヘルパー
  private async callLLM(input: any): Promise<any> {
    // LLM呼び出しの実装
    return { content: [] }
  }

  private async executeTool(name: string, input: any): Promise<any> {
    // ツール実行の実装
    return {}
  }

  private extractNextAction(response: any): string {
    // レスポンスから次のアクションを抽出
    return 'execute'
  }

  private isTaskComplete(state: GraphState): boolean {
    // タスク完了判定
    return false
  }
}
```

### 1.2 LangGraphのCheckpointと統合

```typescript
import { MemorySaver, Checkpoint } from '@langchain/langgraph'

class LangGraphCheckpointIntegration {
  private checkpointSaver: MemorySaver
  private contextCheckpoint: CheckpointManager

  constructor() {
    this.checkpointSaver = new MemorySaver()
    this.contextCheckpoint = new CheckpointManager(storage)
  }

  /**
   * LangGraphのチェックポイント保存時に、Context Managementのチェックポイントも保存
   */
  async saveCheckpoint(
    threadId: string,
    checkpoint: Checkpoint,
    contextState: LayeredState,
    messages: Message[]
  ): Promise<void> {
    // 1. LangGraphのチェックポイント保存
    await this.checkpointSaver.put(threadId, checkpoint)

    // 2. Context Managementのチェックポイント保存
    await this.contextCheckpoint.saveCheckpoint(
      threadId,
      contextState,
      messages
    )
  }

  /**
   * 復元時は両方から復元
   */
  async restoreCheckpoint(threadId: string): Promise<{
    graphCheckpoint: Checkpoint
    contextState: LayeredState
    messages: Message[]
  }> {
    // 1. LangGraphのチェックポイント復元
    const graphCheckpoint = await this.checkpointSaver.get(threadId)

    // 2. Context Managementのチェックポイント復元
    const contextData = await this.contextCheckpoint.restore(threadId)

    return {
      graphCheckpoint,
      contextState: contextData.state,
      messages: contextData.messages
    }
  }
}
```

---

## 2. GraphAI統合

GraphAIは軽量なグラフベースのAIエージェントフレームワークです。

### 2.1 GraphAIノードとしてのContext Management

```typescript
import { GraphAI } from 'graphai'

// Context ManagementをGraphAIノードとして定義
const contextManagementNode = {
  name: 'contextManager',
  processor: async (context: any, inputs: any[]) => {
    const { userInput, state } = inputs[0]

    // Context Engine初期化
    const engine = new ModernContextEngine({
      conversationId: context.threadId,
      maxTokens: 200000,
      model: 'claude-opus-4'
    })

    // メッセージ追加
    await engine.addUserMessage(userInput)

    // 圧縮判定
    if (engine.needsCompression()) {
      await engine.compress()
    }

    // コンテキスト構築
    const contextOutput = await engine.buildContext()

    // ツール投影
    const tools = engine.projectTools()

    return {
      context: contextOutput,
      tools,
      messages: engine.getMessages(),
      state: engine.getState()
    }
  }
}

// GraphAIグラフ定義
const graph = {
  version: 0.5,
  nodes: {
    // ユーザー入力
    userInput: {
      value: 'What is the weather today?'
    },

    // Context Management
    context: {
      processor: contextManagementNode.processor,
      inputs: [
        { userInput: ':userInput', state: ':previousState' }
      ]
    },

    // LLM呼び出し
    llm: {
      processor: async (context: any, inputs: any[]) => {
        const { context: ctx, tools } = inputs[0]

        const response = await callClaude({
          system: ctx.system,
          messages: ctx.messages,
          tools
        })

        return response
      },
      inputs: [':context']
    },

    // レスポンス処理
    processResponse: {
      processor: async (context: any, inputs: any[]) => {
        const [ctxData, llmResponse] = inputs

        // メッセージ追加
        const updatedMessages = [
          ...ctxData.messages,
          {
            id: crypto.randomUUID(),
            role: 'assistant',
            content: llmResponse.content,
            timestamp: Date.now()
          }
        ]

        // Stateを更新
        const updatedState = updateStateFromResponse(
          ctxData.state,
          llmResponse
        )

        return {
          messages: updatedMessages,
          state: updatedState,
          response: llmResponse
        }
      },
      inputs: [':context', ':llm']
    }
  }
}

// GraphAI実行
const graphAI = new GraphAI(graph)
const result = await graphAI.run()
```

### 2.2 GraphAIのストリーミングと統合

```typescript
class GraphAIStreamingIntegration {
  /**
   * ストリーミングレスポンスを処理しながら、リアルタイムでContext更新
   */
  async runWithStreaming(userInput: string): Promise<void> {
    const engine = new ModernContextEngine({
      conversationId: 'stream-123',
      maxTokens: 200000,
      model: 'claude-opus-4'
    })

    // ユーザーメッセージ追加
    await engine.addUserMessage(userInput)

    // コンテキスト構築
    const context = await engine.buildContext()
    const tools = engine.projectTools()

    // ストリーミングLLM呼び出し
    const stream = await callClaudeStreaming({
      system: context.system,
      messages: context.messages,
      tools
    })

    // ストリーミング受信
    let accumulatedContent: ContentBlock[] = []

    for await (const chunk of stream) {
      if (chunk.type === 'content_block_delta') {
        // チャンク蓄積
        accumulatedContent.push(chunk.delta)

        // ユーザーに表示（リアルタイム）
        this.displayToUser(chunk.delta)
      }

      if (chunk.type === 'message_stop') {
        // 完了時にメッセージ追加
        await engine.addAssistantMessage(accumulatedContent)

        // Stateを更新
        engine.updateStateFromResponse(accumulatedContent)
      }
    }
  }

  private displayToUser(delta: any): void {
    // ユーザーに表示する実装
    process.stdout.write(delta.text || '')
  }
}
```

---

## 3. Mulmo統合

Mulmoはマルチモーダル対応のエージェントフレームワークです。

### 3.3 Mulmoのマルチモーダルコンテンツとの統合

```typescript
interface MultimodalMessage extends Message {
  content: Array<
    | TextBlock
    | ImageBlock
    | AudioBlock
    | VideoBlock
    | ToolUseBlock
    | ToolResultBlock
  >
}

interface ImageBlock {
  type: 'image'
  source: {
    type: 'base64' | 'url'
    media_type: string
    data?: string
    url?: string
  }
}

class MulmoContextIntegration {
  private engine: ModernContextEngine

  /**
   * マルチモーダルメッセージの追加
   */
  async addMultimodalMessage(
    text: string,
    images?: ImageBlock[],
    audio?: AudioBlock[]
  ): Promise<void> {
    const content: ContentBlock[] = [
      { type: 'text', text }
    ]

    if (images) {
      content.push(...images)
    }

    if (audio) {
      content.push(...audio)
    }

    const message: MultimodalMessage = {
      id: crypto.randomUUID(),
      role: 'user',
      content,
      timestamp: Date.now()
    }

    await this.engine.addMessage(message)
  }

  /**
   * 画像を含むメッセージの圧縮
   * 画像は要約テキストに置き換え
   */
  async compressMultimodal(
    messages: MultimodalMessage[]
  ): Promise<MultimodalMessage[]> {
    const toCondense = messages.slice(0, -10)
    const toKeep = messages.slice(-10)

    // 画像を含むメッセージを検出
    const imageMessages = toCondense.filter(m =>
      m.content.some(c => c.type === 'image')
    )

    // 画像を要約
    const imageSummaries = await Promise.all(
      imageMessages.map(async (msg) => {
        const imageBlocks = msg.content.filter(
          c => c.type === 'image'
        ) as ImageBlock[]

        const summaries = await Promise.all(
          imageBlocks.map(img => this.summarizeImage(img))
        )

        return {
          messageId: msg.id,
          summaries
        }
      })
    )

    // テキスト要約を作成
    const textSummary = await this.summarizeText(toCondense)

    // 画像要約を追加
    const imageSummaryText = imageSummaries
      .map(s => `Images: ${s.summaries.join(', ')}`)
      .join('\n')

    // 圧縮メッセージを作成
    const condensedMessage: MultimodalMessage = {
      id: crypto.randomUUID(),
      role: 'assistant',
      content: [
        {
          type: 'text',
          text: `## Context Summary\n\n${textSummary}\n\n${imageSummaryText}`
        }
      ],
      condenseId: crypto.randomUUID(),
      timestamp: Date.now()
    }

    return [condensedMessage, ...toKeep]
  }

  private async summarizeImage(image: ImageBlock): Promise<string> {
    // 画像をLLMで要約（Claude Vision等）
    const response = await callClaudeVision({
      image: image.source,
      prompt: 'Describe this image briefly in one sentence.'
    })

    return response.text
  }

  private async summarizeText(messages: Message[]): Promise<string> {
    // テキスト要約（通常のCondensation）
    const result = await this.engine.condense(messages)
    return result.condensedMessage?.content[0].text || ''
  }
}
```

### 3.2 Mulmoのツールチェーン統合

```typescript
class MulmoToolChainIntegration {
  /**
   * Mulmoのツールチェーンと動的ツール投影を統合
   */
  async executeToolChain(
    state: LayeredState,
    toolChain: string[]
  ): Promise<any> {
    const results: any[] = []

    for (const toolName of toolChain) {
      // 現在のphaseでツールが許可されているかチェック
      const allowedTools = this.projectTools(state)
      const tool = allowedTools.find(t => t.name === toolName)

      if (!tool) {
        throw new Error(
          `Tool ${toolName} not allowed in phase ${state.l2_runtime.phase}`
        )
      }

      // ツール実行
      const result = await this.executeTool(tool, state)
      results.push(result)

      // Stateを更新（次のツール実行に影響）
      state = this.updateStateFromToolResult(state, result)
    }

    return results
  }

  private projectTools(state: LayeredState): Tool[] {
    const projector = new DynamicToolProjector()
    return projector.project(state)
  }

  private async executeTool(tool: Tool, state: LayeredState): Promise<any> {
    // ツール実行の実装
    return {}
  }

  private updateStateFromToolResult(
    state: LayeredState,
    result: any
  ): LayeredState {
    // ツール結果からStateを更新
    return {
      ...state,
      l4_evidence: {
        ...state.l4_evidence,
        observations: [
          ...state.l4_evidence.observations,
          {
            timestamp: Date.now(),
            source: 'tool_execution',
            content: JSON.stringify(result)
          }
        ]
      }
    }
  }
}
```

---

## 4. RAGシステム統合

Retrieval-Augmented Generation（RAG）とContext Managementを統合します。

### 4.1 Evidence層へのRAG結果の統合

```typescript
interface RAGResult {
  query: string
  results: Array<{
    id: string
    content: string
    metadata: {
      source: string
      relevanceScore: number
      timestamp: number
    }
  }>
  totalResults: number
}

class RAGContextIntegration {
  private vectorStore: VectorStore
  private contextEngine: ModernContextEngine

  /**
   * ユーザークエリからRAGを実行し、Evidence層に統合
   */
  async queryWithRAG(
    userQuery: string,
    state: LayeredState
  ): Promise<LayeredState> {
    // 1. クエリの再構成（Stateを考慮）
    const rewrittenQuery = this.rewriteQuery(userQuery, state)

    // 2. ベクトル検索
    const ragResults = await this.vectorStore.search(
      rewrittenQuery,
      {
        topK: 5,
        filter: this.buildFilter(state)
      }
    )

    // 3. Evidence層に追加
    const updatedState = {
      ...state,
      l4_evidence: {
        ...state.l4_evidence,
        ragResults: ragResults.results.map(r => ({
          id: r.id,
          content: r.content,
          source: r.metadata.source,
          relevanceScore: r.metadata.relevanceScore,
          timestamp: r.metadata.timestamp
        }))
      }
    }

    return updatedState
  }

  /**
   * クエリ再構成: Stateから関連情報を抽出してクエリを改善
   */
  private rewriteQuery(query: string, state: LayeredState): string {
    // タスクコンテキストを追加
    const taskContext = state.l1_task.goal

    // 最近の会話内容から関連キーワードを抽出
    const recentContext = state.l3_memory.shortTerm
      .slice(-3)
      .join(' ')

    // LLMでクエリを再構成
    return `${query} (Context: ${taskContext}, Recent: ${recentContext})`
  }

  /**
   * フィルタ構築: 権限・環境に応じてフィルタリング
   */
  private buildFilter(state: LayeredState): Record<string, any> {
    const filter: Record<string, any> = {}

    // 権限に応じたフィルタ
    const permissions = state.l2_runtime.permissions
    if (!permissions.includes('admin')) {
      filter.accessLevel = { $in: ['public', 'user'] }
    }

    // 環境に応じたフィルタ
    if (state.l2_runtime.environment === 'production') {
      filter.environment = { $in: ['production', 'all'] }
    }

    return filter
  }

  /**
   * Evidence予算内での最適化
   */
  async optimizeEvidenceForBudget(
    ragResults: RAGResult[],
    budget: number
  ): Promise<RAGResult[]> {
    // 1. 関連度でソート
    const sorted = [...ragResults].sort(
      (a, b) => b.metadata.relevanceScore - a.metadata.relevanceScore
    )

    // 2. 予算内で詰め込む
    const selected: RAGResult[] = []
    let usedTokens = 0

    for (const result of sorted) {
      const tokens = this.countTokens(result.content)

      if (usedTokens + tokens <= budget) {
        selected.push(result)
        usedTokens += tokens
      } else {
        break
      }
    }

    return selected
  }

  private countTokens(text: string): number {
    return Math.ceil(text.length / 4)
  }
}
```

### 4.2 Hybrid Search（ベクトル + キーワード）

```typescript
class HybridSearchIntegration {
  /**
   * ベクトル検索とキーワード検索を組み合わせ
   */
  async hybridSearch(
    query: string,
    state: LayeredState,
    options: {
      vectorWeight: number  // 0.0 - 1.0
      keywordWeight: number // 0.0 - 1.0
    }
  ): Promise<RAGResult[]> {
    // 1. 並列に両方実行
    const [vectorResults, keywordResults] = await Promise.all([
      this.vectorSearch(query),
      this.keywordSearch(query)
    ])

    // 2. スコアを正規化して統合
    const combined = this.combineResults(
      vectorResults,
      keywordResults,
      options
    )

    // 3. 再ランキング（Stateを考慮）
    const reranked = this.rerankWithContext(combined, state)

    return reranked
  }

  private async vectorSearch(query: string): Promise<RAGResult[]> {
    // ベクトル検索の実装
    return []
  }

  private async keywordSearch(query: string): Promise<RAGResult[]> {
    // キーワード検索の実装（BM25等）
    return []
  }

  private combineResults(
    vectorResults: RAGResult[],
    keywordResults: RAGResult[],
    weights: { vectorWeight: number; keywordWeight: number }
  ): RAGResult[] {
    // Reciprocal Rank Fusion (RRF)
    const k = 60
    const scores = new Map<string, number>()

    // Vector results
    vectorResults.forEach((result, rank) => {
      const score = weights.vectorWeight / (k + rank + 1)
      scores.set(result.id, (scores.get(result.id) || 0) + score)
    })

    // Keyword results
    keywordResults.forEach((result, rank) => {
      const score = weights.keywordWeight / (k + rank + 1)
      scores.set(result.id, (scores.get(result.id) || 0) + score)
    })

    // Merge and sort
    const allResults = [...vectorResults, ...keywordResults]
    const unique = Array.from(
      new Map(allResults.map(r => [r.id, r])).values()
    )

    return unique.sort((a, b) => {
      const scoreA = scores.get(a.id) || 0
      const scoreB = scores.get(b.id) || 0
      return scoreB - scoreA
    })
  }

  private rerankWithContext(
    results: RAGResult[],
    state: LayeredState
  ): RAGResult[] {
    // Stateのタスク・権限・環境を考慮して再ランキング
    return results.map(result => {
      let bonus = 0

      // タスク関連性
      if (this.isTaskRelevant(result, state.l1_task)) {
        bonus += 0.2
      }

      // 新鮮さ
      const age = Date.now() - result.metadata.timestamp
      const daysSinceUpdate = age / (1000 * 60 * 60 * 24)
      if (daysSinceUpdate < 7) {
        bonus += 0.1
      }

      return {
        ...result,
        metadata: {
          ...result.metadata,
          relevanceScore: result.metadata.relevanceScore + bonus
        }
      }
    }).sort((a, b) =>
      b.metadata.relevanceScore - a.metadata.relevanceScore
    )
  }

  private isTaskRelevant(result: RAGResult, task: TaskLayer): boolean {
    // タスクのgoalと結果の関連性をチェック
    const taskKeywords = task.goal.toLowerCase().split(/\s+/)
    const resultText = result.content.toLowerCase()

    return taskKeywords.some(kw => resultText.includes(kw))
  }
}
```

---

## 5. チェックポイントシステム統合

### 5.1 Shadow Git統合

Roo Codeで使われているShadow Git方式との統合：

```typescript
import { simpleGit, SimpleGit } from 'simple-git'

class ShadowGitIntegration {
  private git: SimpleGit
  private shadowDir: string

  constructor(shadowDir: string) {
    this.shadowDir = shadowDir
    this.git = simpleGit(shadowDir)
  }

  /**
   * メッセージとStateをGitコミットとして保存
   */
  async saveCheckpoint(
    conversationId: string,
    state: LayeredState,
    messages: Message[],
    metadata: CheckpointMetadata
  ): Promise<string> {
    // 1. JSONファイルを作成
    const stateFile = `${this.shadowDir}/state.json`
    const messagesFile = `${this.shadowDir}/messages.json`
    const metadataFile = `${this.shadowDir}/metadata.json`

    await Promise.all([
      fs.writeFile(stateFile, JSON.stringify(state, null, 2)),
      fs.writeFile(messagesFile, JSON.stringify(messages, null, 2)),
      fs.writeFile(metadataFile, JSON.stringify(metadata, null, 2))
    ])

    // 2. Git add
    await this.git.add([stateFile, messagesFile, metadataFile])

    // 3. Git commit
    const commitMessage = `Checkpoint: ${metadata.phase} (${messages.length} messages, ${metadata.totalTokens} tokens)`

    await this.git.commit(commitMessage)

    // 4. コミットハッシュを取得
    const log = await this.git.log({ maxCount: 1 })
    return log.latest!.hash
  }

  /**
   * 特定のコミットから復元
   */
  async restoreCheckpoint(commitHash: string): Promise<{
    state: LayeredState
    messages: Message[]
    metadata: CheckpointMetadata
  }> {
    // 1. コミットをチェックアウト
    await this.git.checkout(commitHash)

    // 2. ファイルを読み込み
    const [state, messages, metadata] = await Promise.all([
      fs.readFile(`${this.shadowDir}/state.json`, 'utf-8').then(JSON.parse),
      fs.readFile(`${this.shadowDir}/messages.json`, 'utf-8').then(JSON.parse),
      fs.readFile(`${this.shadowDir}/metadata.json`, 'utf-8').then(JSON.parse)
    ])

    return { state, messages, metadata }
  }

  /**
   * チェックポイント一覧を取得
   */
  async listCheckpoints(limit: number = 10): Promise<CheckpointInfo[]> {
    const log = await this.git.log({ maxCount: limit })

    return log.all.map(commit => ({
      hash: commit.hash,
      message: commit.message,
      date: new Date(commit.date),
      author: commit.author_name
    }))
  }

  /**
   * ブランチを作成（実験的なパスを試す）
   */
  async createExperimentalBranch(name: string): Promise<void> {
    await this.git.checkoutLocalBranch(name)
  }

  /**
   * ブランチをマージ（実験が成功したら）
   */
  async mergeExperimentalBranch(branchName: string): Promise<void> {
    await this.git.checkout('main')
    await this.git.merge([branchName])
  }

  /**
   * 差分を確認
   */
  async diffCheckpoints(
    fromHash: string,
    toHash: string
  ): Promise<string> {
    return await this.git.diff([fromHash, toHash])
  }
}

interface CheckpointInfo {
  hash: string
  message: string
  date: Date
  author: string
}
```

### 5.2 Time-travel debugging

```typescript
class TimeTravelDebugger {
  private shadowGit: ShadowGitIntegration
  private contextEngine: ModernContextEngine

  /**
   * 特定の時点に戻って再実行
   */
  async replayFromCheckpoint(
    checkpointHash: string,
    newUserInput: string
  ): Promise<any> {
    // 1. チェックポイントから復元
    const { state, messages } = await this.shadowGit.restoreCheckpoint(
      checkpointHash
    )

    // 2. エンジンを復元状態で初期化
    this.contextEngine.setState(state)
    this.contextEngine.setMessages(messages)

    // 3. 新しい入力で再実行
    const result = await this.contextEngine.executeTask(newUserInput)

    return result
  }

  /**
   * 複数のパスを並列に試す（A/Bテスト）
   */
  async compareAlternativePaths(
    checkpointHash: string,
    alternatives: string[]
  ): Promise<ComparisonResult[]> {
    const results = await Promise.all(
      alternatives.map(async (input, idx) => {
        // 各代替パスで実験的ブランチを作成
        await this.shadowGit.createExperimentalBranch(`experiment-${idx}`)

        const result = await this.replayFromCheckpoint(checkpointHash, input)

        return {
          input,
          result,
          branch: `experiment-${idx}`
        }
      })
    )

    return results
  }

  /**
   * 最良のパスをメインブランチにマージ
   */
  async chooseBestPath(branchName: string): Promise<void> {
    await this.shadowGit.mergeExperimentalBranch(branchName)
  }
}

interface ComparisonResult {
  input: string
  result: any
  branch: string
}
```

---

## 6. 既存システムからのマイグレーション

### 6.1 シンプルな履歴管理からの移行

```typescript
class MigrationHelper {
  /**
   * 古い形式（単純な配列）から新しい形式へ
   */
  async migrateFromSimpleHistory(
    oldMessages: Array<{ role: string; content: string }>
  ): Promise<{
    state: LayeredState
    messages: Message[]
  }> {
    // 1. メッセージを新形式に変換
    const messages: Message[] = oldMessages.map((old, idx) => ({
      id: crypto.randomUUID(),
      role: old.role as 'user' | 'assistant',
      content: [
        {
          type: 'text',
          text: old.content
        }
      ],
      timestamp: Date.now() - (oldMessages.length - idx) * 60000 // 推定
    }))

    // 2. Stateを推定
    const state = this.inferStateFromMessages(messages)

    return { state, messages }
  }

  /**
   * メッセージからStateを推定
   */
  private inferStateFromMessages(messages: Message[]): LayeredState {
    // 簡易的な推定ロジック
    const recentMessages = messages.slice(-10)
    const allText = recentMessages
      .map(m => m.content.filter(c => c.type === 'text').map(c => c.text))
      .flat()
      .join(' ')

    // タスクを推定
    const goal = this.extractGoal(allText)

    // フェーズを推定
    const phase = this.inferPhase(allText)

    return {
      l0_system: {
        policies: [],
        constraints: [],
        safetyGuidelines: []
      },
      l1_task: {
        goal,
        successCriteria: [],
        constraints: [],
        outputFormat: 'text'
      },
      l2_runtime: {
        phase,
        permissions: ['read', 'write'],
        environment: 'development',
        resourceLimits: {}
      },
      l3_memory: {
        shortTerm: recentMessages.map(m =>
          m.content.filter(c => c.type === 'text').map(c => c.text).join(' ')
        ),
        episodic: [],
        semantic: {},
        procedural: []
      },
      l4_evidence: {
        observations: [],
        ragResults: [],
        citations: []
      },
      l5_workBuffer: {
        plan: [],
        currentStep: null,
        diff: [],
        hypotheses: []
      }
    }
  }

  private extractGoal(text: string): string {
    // 簡易的なゴール抽出（最初のユーザーメッセージから）
    const sentences = text.split(/[.!?]/)
    return sentences[0] || 'Assist user with their request'
  }

  private inferPhase(text: string): Phase {
    // キーワードからフェーズを推定
    if (text.includes('plan') || text.includes('how should')) {
      return 'planning'
    }
    if (text.includes('execute') || text.includes('run')) {
      return 'execution'
    }
    if (text.includes('review') || text.includes('evaluate')) {
      return 'reflection'
    }
    return 'planning'
  }

  /**
   * 段階的移行（既存システムと並行稼働）
   */
  async gradualMigration(
    conversationId: string,
    useNewSystem: boolean
  ): Promise<void> {
    if (useNewSystem) {
      // 新システムを使用
      const engine = new ModernContextEngine({
        conversationId,
        maxTokens: 200000,
        model: 'claude-opus-4'
      })

      // ... 新システムのロジック
    } else {
      // 旧システムを使用（後方互換性）
      const oldEngine = new LegacyContextManager()

      // ... 旧システムのロジック
    }
  }

  /**
   * データ整合性チェック
   */
  async validateMigration(
    oldData: any,
    newData: { state: LayeredState; messages: Message[] }
  ): Promise<ValidationResult> {
    const errors: string[] = []

    // メッセージ数の一致
    if (oldData.messages.length !== newData.messages.length) {
      errors.push('Message count mismatch')
    }

    // 内容の一致（テキストのみ）
    for (let i = 0; i < oldData.messages.length; i++) {
      const oldText = oldData.messages[i].content
      const newText = newData.messages[i].content
        .filter(c => c.type === 'text')
        .map(c => c.text)
        .join('')

      if (oldText !== newText) {
        errors.push(`Message ${i} content mismatch`)
      }
    }

    return {
      valid: errors.length === 0,
      errors
    }
  }
}

interface ValidationResult {
  valid: boolean
  errors: string[]
}
```

### 6.2 ロールバック計画

```typescript
class RollbackManager {
  /**
   * 新システムで問題が発生した場合のロールバック
   */
  async rollbackToOldSystem(
    conversationId: string,
    state: LayeredState,
    messages: Message[]
  ): Promise<any> {
    // 1. 新形式を旧形式に変換
    const oldMessages = messages.map(msg => ({
      role: msg.role,
      content: msg.content
        .filter(c => c.type === 'text')
        .map(c => c.text)
        .join('\n')
    }))

    // 2. 旧システムで保存
    await this.legacyStorage.save(conversationId, oldMessages)

    // 3. 新システムのデータをバックアップ
    await this.backupNewData(conversationId, state, messages)

    return oldMessages
  }

  private async backupNewData(
    conversationId: string,
    state: LayeredState,
    messages: Message[]
  ): Promise<void> {
    const backupPath = `./backups/${conversationId}-${Date.now()}.json`

    await fs.writeFile(
      backupPath,
      JSON.stringify({ state, messages }, null, 2)
    )
  }
}
```

---

## まとめ: 統合実装例

この統合実装例では、以下を提供しました：

1. **LangGraph統合**: 状態ベースワークフローとContext Managementの統合、チェックポイント連携
2. **GraphAI統合**: ノードベース実装、ストリーミング対応
3. **Mulmo統合**: マルチモーダルコンテンツの圧縮、ツールチェーン統合
4. **RAGシステム統合**: Evidence層への統合、Hybrid Search、予算最適化
5. **チェックポイント統合**: Shadow Git方式、Time-travel debugging
6. **マイグレーション**: 既存システムからの移行、段階的移行、ロールバック

これらの統合パターンを参考に、あなたのシステムに最適な形でContext Managementを組み込むことができます。

