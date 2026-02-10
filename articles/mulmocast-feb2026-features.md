---
title: "MulmoCast 2月の新機能 - Markdownから動画生成、AIで対話・要約"
emoji: "🎬"
type: "tech"
topics: [AI, mulmocast, GraphAI, Markdown, LLM]
published: true
publication_name: "singularity"
---

# MulmoCast 2月初旬の新機能まとめ

2026年2月4日〜10日の1週間で、MulmoCast-Slides（`@mulmocast/slide`）と mulmocast-preprocessor に大きな機能追加を行いました。この記事では、追加した機能を「なぜ必要だったのか」「どう使うのか」をセットで紹介します。

## 全体像

追加した機能は大きく3つのテーマに分かれます。

| テーマ | 何ができるようになったか |
|-------|----------------------|
| **ExtendedScript** | MulmoScript にメタデータ・バリアント・出力プロファイルを追加する拡張フォーマット |
| **Markdown → 動画パイプライン** | 普通の Markdown ドキュメントからナレーション付き動画を自動生成 |
| **AI によるコンテンツ操作** | 作成した動画スクリプトに対して要約・質問・参照取得 |

## インストール・アップデート

```bash
# MulmoCast-Slides
npm install -g @mulmocast/slide

# mulmocast-preprocessor
npm install -g mulmocast-preprocessor
```

バージョン確認：

```bash
mulmo-slide --version    # 0.5.0 以降
npx mulmocast-preprocessor --version  # 0.4.1 以降
```

---

## 1. ExtendedScript — MulmoScript の拡張フォーマット

### なぜ必要だったのか

MulmoScript は「スライド画像 + ナレーションテキスト」のシンプルなフォーマットです。動画生成にはこれで十分ですが、長い論文やドキュメントを扱うときに次の課題がありました。

- **メタデータがない**: キーワード、セクション情報、文脈がスクリプトに残らない
- **バリアント切り替えができない**: 同じスライドの「詳細版」「要約版」を1ファイルで管理できない
- **AI が活用しにくい**: 質問応答や要約に必要な構造化情報が欠けている

そこで **ExtendedScript** という拡張フォーマットを導入しました。既存の MulmoScript と完全に互換性があり、必要なときだけメタデータを追加できます。

### ExtendedScript の構造

```json
{
  "$mulmocast": { "version": "1.1" },
  "title": "ドキュメントタイトル",
  "scriptMeta": {
    "audience": "エンジニア",
    "keywords": ["API設計", "REST", "GraphQL"],
    "references": [
      { "title": "RFC 7231", "url": "https://..." }
    ]
  },
  "outputProfiles": {
    "detailed": {
      "description": "全ビートを含む完全版",
      "includeTags": ["core", "detail", "example"]
    },
    "short": {
      "description": "コアビートのみの要約版",
      "includeTags": ["core"]
    }
  },
  "beats": [
    {
      "text": "ナレーションテキスト",
      "image": { "type": "markdown", "markdown": ["# スライド"] },
      "meta": {
        "section": "introduction",
        "tags": ["core"],
        "keywords": ["概要"],
        "context": "このスライドでは全体の流れを説明する"
      },
      "variants": {
        "short": { "text": "短縮版のナレーション" }
      }
    }
  ]
}
```

通常の MulmoScript と比べて `scriptMeta`、`outputProfiles`、各ビートの `meta`・`variants` が追加されています。

### ExtendedScript を作る方法

#### 方法 1: `/extend` スキル（Claude Code）

既存の MulmoScript に AI でメタデータを付与します。

```bash
# スキルをインストール（初回のみ）
mulmo-slide extend init

# Claude Code 内で実行
/extend scripts/my-slides/my-slides.json
```

#### 方法 2: `scaffold` コマンド（LLM 不要）

LLM を使わずに、スケルトンの ExtendedScript を生成します。

```bash
mulmo-slide extend scaffold scripts/my-slides/my-slides.json
```

これで `extended_script.json` が生成されます。メタデータは空ですが、あとから手動で埋めたり `/extend` で AI に任せたりできます。

#### 方法 3: `validate` でスキーマ検証

ExtendedScript が正しいフォーマットか検証できます。

```bash
mulmo-slide extend validate scripts/my-slides/extended_script.json
```

---

## 2. `/narrate` — PDF・PPTX からナレーション付き動画

### なぜ必要だったのか

PDF の論文やスライドを動画にしたいとき、これまでは以下のステップが必要でした。

1. PDF → MulmoScript に変換
2. 各スライドのナレーションを手動で書く
3. 動画を生成

特にステップ 2 が大変です。30ページの論文なら30個のナレーションを書かなければいけません。**AI にナレーションを自動生成してもらえれば**、数分で完了します。

### 使い方

```bash
# スキルをインストール（初回のみ）
mulmo-slide extend init

# Claude Code 内で実行
/narrate your-paper.pdf
```

これだけで `/narrate` スキルが自動的に：

1. PDF をスライド画像に変換
2. 各ページからテキストを抽出
3. AI がナレーションを生成
4. メタデータ（キーワード、セクション、コンテキスト）を追加
5. スキーマを検証

生成されるファイル：

```
scripts/your-paper/
  your-paper.json        # 基本の MulmoScript
  extracted_texts.json    # PDF から抽出した各ページのテキスト
  extended_script.json   # ExtendedScript（ナレーション + メタデータ）
  images/
    your-paper-0.png     # 各ページの画像
    your-paper-1.png
    ...
```

### CLI でも実行可能

Claude Code を使わずに、CLI だけでも実行できます。

```bash
# フルパイプライン（OpenAI GPT を使用）
mulmo-slide narrate your-paper.pdf -l ja

# スキャフォールドのみ（LLM 不要、あとから /extend で補完）
mulmo-slide narrate your-paper.pdf --scaffold-only
```

### 対応フォーマット

```bash
/narrate your-paper.pdf       # PDF（論文、ドキュメント）
/narrate your-slides.pptx     # PowerPoint
/narrate your-slides.md       # Marp マークダウン
/narrate your-slides.key      # Keynote（macOS のみ）
```

---

## 3. `/md-to-mulmo` — Markdown ドキュメントから動画を自動生成

### なぜ必要だったのか

`/narrate` は「既にスライドがある」ことが前提です。しかし、技術ブログや仕様書のような**普通の Markdown ドキュメント**からも動画を作りたいケースがあります。

Markdown にはスライドの区切りがなく、見出しの階層構造や段落の長さもバラバラです。これを「どのスライドに何を入れるか」自動で設計するには、LLM の力が必要でした。

### パイプラインの概要

```
Markdown → parse-md → LLM がプラン設計 → assemble-extended → ExtendedScript
```

1. **parse-md**: Markdown の構造（見出し、段落、コードブロック、リスト等）を JSON に解析
2. **LLM 設計**: 解析結果をもとに「どのセクションをどのスライドに割り当てるか」プランを作成
3. **assemble-extended**: プランから ExtendedScript を自動組み立て

### 使い方（クイックスタート）

```bash
# スキルをインストール（初回のみ）
mulmo-slide extend init

# Claude Code 内で実行
/md-to-mulmo your-document.md
```

スキルが自動でパイプライン全体を実行し、`scripts/your-document/extended_script.json` が生成されます。

### 手動で各ステップを実行する場合

#### Step 1: Markdown を解析

```bash
mulmo-slide parse-md your-document.md
```

生成されるファイル：

```
scripts/your-document/
  parsed_structure.json           # 構造解析結果
  extended-script.schema.json     # ExtendedScript の JSON Schema
  presentation-plan.schema.json   # プレゼンプランの JSON Schema
```

`parsed_structure.json` には、Markdown のセクション構造、各要素（見出し、段落、コードブロック、リスト、テーブル等）が階層的に記録されます。

#### Step 2: プレゼンテーションプランを作成

```bash
# Claude Code 内で実行
/md-to-mulmo your-document.md
```

LLM が `parsed_structure.json` と `presentation-plan.schema.json` を読み込み、「どのセクションをどのスライドに配置するか」を設計します。結果は `presentation_plan.json` に保存されます。

#### Step 3: ExtendedScript を組み立て

```bash
mulmo-slide assemble-extended scripts/your-document/presentation_plan.json
```

プランから ExtendedScript を生成します。`detailed`（全ビート）と `short`（コアのみ）の2つの出力プロファイルが自動的に含まれます。

### 動画を生成する

ExtendedScript から動画を作るには、まず preprocessor で通常の MulmoScript に変換します。

```bash
# ExtendedScript → MulmoScript
npx mulmocast-preprocessor scripts/your-document/extended_script.json \
  -o scripts/your-document/your-document.json

# MulmoScript → 動画
npx mulmo movie scripts/your-document/your-document.json
```

出力: `output/your-document_ja.mp4`

### 出力プロファイルの切り替え

```bash
# 要約版で動画を作る
npx mulmocast-preprocessor scripts/your-document/extended_script.json \
  -o scripts/your-document/your-document.json -p short
```

---

## 4. mulmocast-preprocessor — AI でコンテンツを操作する

### なぜ必要だったのか

ExtendedScript を作ったら終わり、ではもったいない。スクリプトには元のドキュメントの内容がすべて含まれています。これを活用して：

- **内容を要約したい** — 長い論文の要点を素早く把握
- **質問したい** — 特定のトピックについて深掘り
- **対話的に探索したい** — チャットのように連続で質問

こうしたニーズに応えるため、preprocessor に AI コマンドを追加しました。

### summarize — 要約を生成する

```bash
# 日本語で要約を生成
npx mulmocast-preprocessor summarize scripts/your-paper/extended_script.json -l ja

# 英語で要約
npx mulmocast-preprocessor summarize scripts/your-paper/extended_script.json -l en
```

LLM がスクリプト全体を読み取り、人間が読みやすい形式で要約を出力します。

### query — 質問する

```bash
# 一つの質問を投げる
npx mulmocast-preprocessor query scripts/your-paper/extended_script.json "HBF とは何ですか？"
```

スクリプトの内容をコンテキストとして LLM に渡し、質問に回答します。

### 対話モード — チャットのように使う

```bash
npx mulmocast-preprocessor query scripts/your-paper/extended_script.json -i
```

`-i` フラグで対話モードに入ります。

```
> この論文の主題は何ですか？
LLM推論ハードウェアの課題について議論しています...

> 4つの研究方向とは？
1. High Bandwidth Flash (HBF)
2. Processing-Near-Memory (PNM)
3. 3D Compute-Logic Stacking
4. 低遅延インターコネクト

> /refs
参考文献一覧:
1. RFC 7231 - https://...
2. ...

> /fetch
参考URL の内容を取得中...

> exit
```

対話モードでは会話の履歴が保持されるので、前の質問を踏まえた深掘りができます。

#### 対話モードの特殊コマンド

| コマンド | 説明 |
|---------|------|
| `/refs` | 参考文献一覧を表示 |
| `/fetch` | 参考 URL の内容を取得して LLM のコンテキストに追加 |
| `/clear` | 会話履歴をクリア |
| `/history` | 会話履歴を表示 |
| `exit` | 対話モードを終了 |

### scriptMeta — ドキュメント全体のメタデータ

ExtendedScript の `scriptMeta` フィールドには、ドキュメント全体のメタデータを格納できます。

```json
{
  "scriptMeta": {
    "audience": "バックエンドエンジニア",
    "prerequisites": ["REST API の基礎知識"],
    "goals": ["GraphQL の利点を理解する"],
    "keywords": ["GraphQL", "API設計"],
    "references": [
      { "title": "GraphQL 公式ドキュメント", "url": "https://graphql.org/" }
    ]
  }
}
```

`summarize` や `query` コマンドは、このメタデータを自動的に LLM プロンプトに含めるため、より的確な回答が得られます。

---

## 5. フィルタリングとプロファイル — セクション・タグで絞り込み

### なぜ必要だったのか

長いドキュメントから動画を作ると、ビートが30個、50個と増えていきます。用途に応じて「この部分だけ」を取り出したいことがあります。

### プロファイルで切り替え

```bash
# 利用可能なプロファイルを一覧
npx mulmocast-preprocessor process scripts/your-doc/extended_script.json --list-profiles

# detailed プロファイル（全ビート）
npx mulmocast-preprocessor process scripts/your-doc/extended_script.json -p detailed

# short プロファイル（要約版）
npx mulmocast-preprocessor process scripts/your-doc/extended_script.json -p short
```

### セクションで絞り込み

```bash
# introduction セクションだけ
npx mulmocast-preprocessor process scripts/your-doc/extended_script.json --section introduction
```

### タグで絞り込み

```bash
# "example" タグがついたビートだけ
npx mulmocast-preprocessor process scripts/your-doc/extended_script.json --tags example
```

---

## 6. レイアウト自動検出 — Markdown の内容に応じたスライドレイアウト

### なぜ必要だったのか

Markdown をスライドに変換するとき、すべてのスライドが単一カラムだと単調です。コードブロックがあるスライドは左にコード・右に説明を並べたい。リストが多いスライドは2カラムにしたい。これを**手動で指定するのは面倒**なので、コンテンツに応じて自動判定するようにしました。

### 自動検出されるレイアウト

| コンテンツ | レイアウト |
|-----------|----------|
| コードブロック + テキスト | `row-2`（2列：左にコード、右に説明） |
| Mermaid 図 + テキスト | `row-2`（2列：左に図、右に説明） |
| リスト4項目以上 | `2x2`（4分割グリッド） |
| H1見出し + コンテンツ | `header` + `content` / `row-2` / `2x2` |
| シンプルなテキスト | `default`（単一カラム） |

### Markdown 変換コマンドで使う

```bash
# レイアウト自動検出は常に有効
mulmo-slide markdown your-slides.md

# Mermaid 対応を追加
mulmo-slide markdown your-slides.md --mermaid
```

レイアウト検出はプラグインシステムとして実装されており、手動で特定のレイアウトを指定することもできます。

---

## 7. ブラウザ対応エクスポート

### なぜ必要だったのか

MulmoCast-Slides は Node.js 向けのツールですが、Web アプリケーション内で Markdown → MulmoScript の変換を行いたいケースがあります。Node.js 固有の依存（ファイルシステム、子プロセス等）を除外した**ブラウザ対応バンドル**を提供することで、ブラウザ環境でも利用できるようにしました。

### 使い方

```typescript
// ブラウザ用のインポート
import { transformMarkdownToMulmoScript } from "@mulmocast/slide/browser";

const mulmoScript = transformMarkdownToMulmoScript(markdownText, {
  mermaid: true,
  separator: "heading",
});
```

`@mulmocast/slide/browser` からインポートすると、Node.js 固有の依存がバンドルされないため、Vite や webpack などのバンドラーで問題なく利用できます。

---

## ワークフロー早見表

用途に応じたワークフローをまとめます。

### Markdown ドキュメント → 動画

```bash
# Claude Code で
/md-to-mulmo your-document.md

# ExtendedScript → MulmoScript
npx mulmocast-preprocessor scripts/your-document/extended_script.json \
  -o scripts/your-document/your-document.json

# 動画生成
npx mulmo movie scripts/your-document/your-document.json
```

### PDF / PPTX → 動画

```bash
# Claude Code で
/narrate your-paper.pdf

# ExtendedScript → MulmoScript
npx mulmocast-preprocessor scripts/your-paper/extended_script.json \
  -o scripts/your-paper/your-paper.json

# 動画生成
npx mulmo movie scripts/your-paper/your-paper.json
```

### 動画のスクリプトで遊ぶ

```bash
# 要約
npx mulmocast-preprocessor summarize scripts/your-doc/extended_script.json -l ja

# 対話的に質問
npx mulmocast-preprocessor query scripts/your-doc/extended_script.json -i

# セクション絞り込みで動画生成
npx mulmocast-preprocessor scripts/your-doc/extended_script.json \
  -o scripts/your-doc/your-doc.json --section introduction
npx mulmo movie scripts/your-doc/your-doc.json
```

## まとめ

2月4日〜10日の1週間で追加した機能をまとめます。

| 機能 | ツール | 何ができるか |
|------|-------|-------------|
| ExtendedScript | mulmo-slide | メタデータ・バリアント付きのリッチなスクリプト |
| `/narrate` | mulmo-slide | PDF/PPTX/MD から AI ナレーション付きスクリプト生成 |
| `/md-to-mulmo` | mulmo-slide | Markdown から AI がプレゼン構成を自動設計 |
| レイアウト自動検出 | mulmo-slide | コンテンツに応じて row-2 / 2x2 / header を自動選択 |
| ブラウザ対応 | mulmo-slide | `@mulmocast/slide/browser` でブラウザ環境に対応 |
| summarize | preprocessor | LLM でスクリプトを要約 |
| query | preprocessor | LLM でスクリプトに質問（対話モード対応） |
| フィルタリング | preprocessor | セクション・タグ・プロファイルで絞り込み |

すべてのツールは npm で公開されています。

```bash
npm install -g @mulmocast/slide
npm install -g mulmocast-preprocessor
```

## 参考リンク

- [MulmoCast-Slides GitHub](https://github.com/receptron/MulmoCast-Slides)
- [mulmocast-preprocessor (mulmocast-plus monorepo)](https://github.com/receptron/mulmocast-plus)
- [MulmoCast 本体](https://github.com/receptron/mulmocast-cli)
