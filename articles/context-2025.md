---
title: "Context Management 2025 - 完全ガイド 01"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech]
published: true
publication_name: "singularity"
---

**2024-2025年におけるContext Management（コンテキスト管理）の進化**と、それに基づく**モダンなアーキテクチャ設計**に関する包括的なドキュメントが含まれています。

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
v    const { includeTargetMessage = false } = options

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

