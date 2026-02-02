---
title: "MulmoScriptの作成方法まとめ - CLI、AI、変換ツール、GUIアプリ"
emoji: "🎬"
type: "tech"
topics: ["mulmocast", "ai", "presentation", "typescript"]
published: false
---

## はじめに

MulmoCastは、スライドとナレーションを組み合わせた動画やWebコンテンツを生成するツールです。その核となるのが**MulmoScript**というJSON形式のスクリプトファイルです。

この記事では、MulmoScriptを作成する5つの方法を紹介します。

## CLIツールのインストール

この記事で使用するCLIツールは、npmでグローバルインストールできます。

```bash
# mulmoコマンド（MulmoScript生成・動画/音声出力）
npm install -g mulmocast

# mulmo-slideコマンド（既存ファイルの変換）
npm install -g @mulmocast/slide
```

## MulmoScriptとは

MulmoScriptは以下の構造を持つJSONファイルです：

```json
{
  "$mulmocast": { "version": "1.1", "credit": "closing" },
  "lang": "ja",
  "title": "プレゼンテーションタイトル",
  "beats": [
    {
      "text": "このスライドで話す内容",
      "image": {
        "type": "markdown",
        "markdown": ["# タイトル", "- ポイント1", "- ポイント2"]
      }
    }
  ]
}
```

## 1. CLIで作る（mulmo tool scripting）

最も手軽な方法。対話的にテーマを指定するか、URLから自動生成します。

### 対話モード

```bash
mulmo tool scripting -i -t business
```

LLMと対話しながら、スライドの構成を決めていきます。

### URLから生成

```bash
mulmo tool scripting -t coding -u https://example.com/article
```

Webページの内容を解析し、自動的にMulmoScriptを生成します。

### テンプレート指定

利用可能なテンプレート：`business`, `coding`, `documentary`, `vision`, `shorts` など

```bash
# テンプレート一覧を確認
mulmo tool info templates
```

## 2. 自前でAI（LLM）で作る

ChatGPTやClaudeなどのLLMを使って、自分でMulmoScriptを生成する方法です。2つのアプローチがあります。

### 方法A: 完全なMulmoScriptを一括生成

プロンプトテンプレートやスキーマをLLMに渡して、完全なMulmoScriptを一度に生成してもらう方法です。

#### Step 1: プロンプトまたはスキーマを取得

```bash
# プロンプトテンプレートを取得（テンプレート指定）
mulmo tool prompt -t business

# または、JSON Schemaを取得
mulmo tool schema
```

#### Step 2: LLMに依頼

取得したプロンプト（またはスキーマ）と、作りたい内容をChatGPTやClaudeのチャットに投げます。

**例：ChatGPTへの依頼**
```
以下のプロンプトテンプレートに従って、「GraphAIの概要説明」についての
MulmoScriptをJSON形式で作成してください。

[ここにmulmo tool promptの出力を貼り付け]
```

LLMが完全なMulmoScript（`$mulmocast`、`lang`、`beats`など全て含む）を生成してくれます。

> **注意**: AIの出力は不完全な場合があります。生成されたJSONの構文エラーや必須フィールドの欠落がないか確認し、必要に応じて修正してください。

---

### 方法B: beatsだけ生成して補完

beatsの配列だけを作成し、残りのメタデータは`mulmo tool complete`で自動補完する方法です。より軽量で、細かい調整がしやすいアプローチです。

#### beatsの構造

beatsは以下の構造を持つ配列です。用途に応じて様々なimage typeを使い分けられます。以下のサンプルをコピーしてLLMへの依頼やプログラムでの生成に活用してください。

**基本形（Markdownスライド）：**
```json
{
  "text": "ナレーション（読み上げる内容）",
  "image": {
    "type": "markdown",
    "markdown": ["# スライドタイトル", "", "- ポイント1", "- ポイント2"]
  }
}
```

**画像URL指定：**
```json
{
  "text": "この画像について説明します。",
  "image": {
    "type": "image",
    "source": {
      "kind": "url",
      "url": "https://example.com/image.png"
    }
  }
}
```

**動画URL指定：**
```json
{
  "text": "動画をご覧ください。",
  "image": {
    "type": "movie",
    "source": {
      "kind": "url",
      "url": "https://example.com/video.mp4"
    }
  }
}
```

**テキストスライド（シンプルなテキスト表示）：**
```json
{
  "text": "重要なポイントをお伝えします。",
  "image": {
    "type": "textSlide",
    "text": "ここに大きく表示したいテキスト"
  }
}
```

**Mermaid図：**
```json
{
  "text": "フローチャートで処理の流れを説明します。",
  "image": {
    "type": "mermaid",
    "mermaid": "graph TD\\n    A[開始] --> B[処理]\\n    B --> C[終了]"
  }
}
```

**AI画像生成（imagePrompt）：**
```json
{
  "text": "AIが生成した画像をご覧ください。",
  "imagePrompt": "夜の渋谷スクランブル交差点、ネオンが輝く都市の風景"
}
```

**AI動画生成（moviePrompt）：**
```json
{
  "text": "AIが生成した動画です。",
  "moviePrompt": "美しい日本庭園を散歩するカメラワーク",
  "duration": 5
}
```

**画像から動画生成（imagePrompt + moviePrompt）：**
```json
{
  "text": "画像を元に動画を生成しています。",
  "imagePrompt": "東京の夜景、高層ビル群",
  "moviePrompt": "カメラがゆっくりとパンする",
  "duration": 5
}
```

**beatの主なフィールド：**
- `text`: ナレーション（話す内容）
- `image`: 画像/スライドを直接指定
  - `image.type`: `"markdown"` / `"image"` / `"movie"` / `"textSlide"` / `"mermaid"` など
  - `image.source`: 画像・動画の場合、`kind`（`"url"` or `"path"`）と`url`/`path`を指定
- `imagePrompt`: AIに画像を生成させるプロンプト
- `moviePrompt`: AIに動画を生成させるプロンプト
- `duration`: 動画の長さ（秒）

#### 作成方法1: LLMで生成

上記のサンプルをChatGPTやClaudeに渡して、内容を生成してもらいます。

**例：ChatGPTへの依頼**
```
以下のJSON形式で「GraphAIの概要説明」のスライド5枚分を作成してください。

{
  "beats": [
    {
      "text": "ナレーション内容",
      "image": {
        "type": "markdown",
        "markdown": ["# タイトル", "- ポイント1"]
      }
    }
  ]
}
```

**LLMの出力例：**
```json
{
  "beats": [
    {
      "text": "GraphAIは、複雑なワークフローを簡単に構築できるフレームワークです。",
      "image": {
        "type": "markdown",
        "markdown": ["# GraphAIとは", "", "ワークフロー構築フレームワーク"]
      }
    },
    {
      "text": "主な特徴を見ていきましょう。",
      "image": {
        "type": "markdown",
        "markdown": ["## 特徴", "- 宣言的なワークフロー定義", "- 並列処理の自動化", "- LLM統合"]
      }
    }
  ]
}
```

#### 作成方法2: プログラムで生成

自作のプログラムでbeatsを生成することもできます。データ変換やバッチ処理に便利です。

```typescript
const beats = data.map(item => ({
  text: item.description,
  image: {
    type: "markdown",
    markdown: [`# ${item.title}`, "", ...item.points]
  }
}));

fs.writeFileSync("beats.json", JSON.stringify({ beats }, null, 2));
```

#### completeで補完

生成したbeatsをファイルに保存し、`complete`コマンドで完全なMulmoScriptに変換します。

```bash
mulmo tool complete beats.json -o mulmo_script.json
```

これにより、`$mulmocast`、`speechParams`、`imageParams`などの必要なメタデータが自動的に補完されます。

---

### 方法A vs 方法B

| 項目 | 方法A（一括生成） | 方法B（beats + complete） |
|------|------------------|--------------------------|
| 手軽さ | ★★★ | ★★☆ |
| 細かい制御 | ★★☆ | ★★★ |
| LLMトークン消費 | 多い | 少ない |
| 向いているケース | 素早く作りたい | 段階的に調整したい |

## 3. mulmo-slideで変換

既存のドキュメントやプレゼンテーションをMulmoScriptに変換します。

**なぜ変換するのか？**

既存のスライドやドキュメントをMulmoScriptに変換することで、以下が可能になります：

- **動画化** - スライドをナレーション付きの動画に変換
- **音声ナレーション追加** - スライド内容の説明をAI音声で自動生成
- **Webコンテンツ化** - インタラクティブなWebプレゼンテーションとして公開

単なるフォーマット変換ではなく、静的なスライドを「話すプレゼンテーション」に拡張するための変換です。

### Markdown

```bash
# プレーンなMarkdown（画像生成なし）
mulmo-slide markdown document.md -l ja

# Marp形式（画像生成あり）
mulmo-slide marp slides.md -l ja

# LLMでナレーション生成
mulmo-slide markdown document.md -g -l ja
```

**共通オプション：**
- `-l, --lang` - 言語指定（ja, en, fr, de）
- `-g, --generate-text` - LLMでナレーション自動生成

**markdownコマンド固有オプション：**
- `-s, --separator` - スライド区切り方式（horizontal-rule, heading, heading-1, heading-2, heading-3, blank-lines, comment, page-break）
- `--mermaid` - Mermaidコードブロックをmermaid beatに変換
- `--directive` - Marpスタイルのディレクティブを削除
- `--style` - スライドスタイル指定（例：corporate-blue, finance-green）

**marpコマンド固有オプション：**
- `--theme` - カスタムテーマCSSファイルのパス
- `--allow-local-files` - ローカルファイルへのアクセスを許可

### PowerPoint

```bash
mulmo-slide pptx presentation.pptx -l ja
```

スピーカーノートがあれば、それをナレーションとして使用します。

### PDF

```bash
mulmo-slide pdf document.pdf -l ja
```

各ページを画像として取り込みます。

### Keynote（macOSのみ）

```bash
mulmo-slide keynote presentation.key
```

## 4. 手書きで作る

JSONを直接記述します。最も自由度が高いですが、手間がかかります。

```json
{
  "$mulmocast": {
    "version": "1.1",
    "credit": "closing"
  },
  "lang": "ja",
  "title": "手書きプレゼン",
  "description": "JSONを直接書いて作成",
  "beats": [
    {
      "text": "こんにちは。今日のプレゼンテーションへようこそ。",
      "image": {
        "type": "markdown",
        "markdown": [
          "# ようこそ",
          "",
          "MulmoCastで作る",
          "インタラクティブなプレゼン"
        ]
      }
    },
    {
      "text": "それでは始めましょう。",
      "image": {
        "type": "markdown",
        "markdown": [
          "## 目次",
          "1. はじめに",
          "2. 本題",
          "3. まとめ"
        ]
      }
    }
  ]
}
```

## 5. GUIアプリで作る

[https://mulmocast.com/](https://mulmocast.com/) からデスクトップアプリをダウンロードできます。

MulmoCastはAIネイティブなプレゼンテーションプラットフォームです。PowerPointやKeynoteがAI以前の時代に設計されたのに対し、MulmoCastは生成AIとの協調を前提にゼロから設計されています。

**主な機能：**
- MulmoScriptの視覚的な編集
- リアルタイムプレビュー
- 動画、スライドショー、PDF、ポッドキャストなど多様な出力形式に対応
- ChatGPTやClaudeなどのAIモデルとの連携

## 作成方法の比較

| 方法 | 難易度 | 自由度 | 向いているケース |
|------|--------|--------|------------------|
| CLI scripting | ★☆☆ | ★★☆ | 素早く作りたい |
| LLM + complete | ★★☆ | ★★★ | 細かく制御したい |
| mulmo-slide変換 | ★☆☆ | ★★☆ | 既存資料を活用 |
| 手書き | ★★★ | ★★★ | 完全にカスタム |
| GUIアプリ | ★☆☆ | ★★★ | 視覚的に編集 |

## MulmoScriptから出力を生成

作成したMulmoScriptから、様々な出力を生成できます：

```bash
# 動画を生成
mulmo movie mulmo_script.json

# PDFスライドを生成
mulmo pdf mulmo_script.json

# Webバンドルを生成（MulmoViewer用）
mulmo bundle mulmo_script.json

# 画像を生成
mulmo images mulmo_script.json

# 音声を生成
mulmo audio mulmo_script.json
```

## まとめ

MulmoScriptは複数の方法で作成できます：

1. **CLI** - 最も手軽、対話的またはURL指定
2. **LLM** - 柔軟性が高い、独自のワークフローに組み込み可能
3. **変換ツール** - 既存のMarkdown/PPTX/PDFを活用
4. **手書き** - 完全な制御、学習目的にも
5. **GUIアプリ** - 視覚的に編集（[mulmocast.com](https://mulmocast.com/)）

用途に応じて最適な方法を選んでください。

## 参考リンク

- [MulmoCast GitHub](https://github.com/receptron/mulmocast)
- [MulmoCast-Slides GitHub](https://github.com/receptron/MulmoCast-Slides)
- [npm: mulmocast](https://www.npmjs.com/package/mulmocast)
- [npm: @mulmocast/slide](https://www.npmjs.com/package/@mulmocast/slide)
