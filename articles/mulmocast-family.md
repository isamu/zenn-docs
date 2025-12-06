---
title: "mulmocastファミリー - AI時代のマルチモーダルコンテンツ生成ツール"
emoji: "🎬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [mulmocast, MCP, AI, LLM, multimodal]
published: false
publication_name: "singularity"
---

# mulmocastファミリーとは

mulmocastファミリーは、**AIとの協働でプレゼンテーション動画、ポッドキャスト、スライド資料を生成する次世代のコンテンツ制作プラットフォーム**です。

中心となるのは**mulmoScript**と呼ばれるJSON/YAML形式のスクリプト言語で、これは映画の脚本のようにマルチモーダルコンテンツの構成を記述します。LLM（大規模言語モデル）がmulmoScriptを生成し、それを元に動画、音声、画像、PDFなど様々な形式のコンテンツが自動生成されます。

**主な用途**:
- プレゼンテーション動画の自動生成
- ビジネス資料（提案書・戦略書）の作成
- ポッドキャスト音声の制作
- 教育コンテンツ（絵本・マンガ形式）の作成

MCPサーバとしての機能も提供しており、Claude Desktop等のMCP対応クライアントから簡単に利用できます。

## ファミリー構成

mulmocastファミリーは主に以下のコンポーネントで構成されています:

### 1. mulmocast-cli

**役割**: AIパワード・プレゼンテーション生成エンジン

[mulmocast-cli](https://github.com/receptron/mulmocast-cli)は、mulmocastの中核となるコマンドラインツールです。mulmoScriptから動画、音声、画像、PDFなど複数の形式のコンテンツを自動生成します。

**主な機能**:
- **マルチフォーマット出力**: 動画、ポッドキャスト、スライドショー、PDF、マンガ形式に対応
- **AI駆動のスクリプト生成**: 対話形式でmulmoScriptを作成
- **自動音声生成**: テキストから自然な音声を生成（複数のTTSサービス対応）
- **画像自動生成**: 指定されたビジュアルスタイルに合わせた画像合成
- **多言語対応**: 英語・日本語の翻訳・字幕機能

**インストール**:
```bash
npm install -g mulmocast
brew install ffmpeg  # macOS (動画生成に必要)
```

**基本的な使い方**:
```bash
# 1. 対話形式でスクリプトを生成（絵本テンプレート）
mulmo tool scripting -i -t children_book -o ./ -s story

# 2. 生成されたスクリプトから動画を作成（日本語字幕付き）
mulmo movie story-1746600802426.json -c ja
```

**必要な環境変数**:
- `OPENAI_API_KEY`: スクリプト生成と画像作成に必須
- `GEMINI_API_KEY`: Google画像・音声合成（オプション）
- `ANTHROPIC_API_TOKEN`: Claude統合（オプション）
- `REPLICATE_API_TOKEN`: 動画生成（オプション）
- `ELEVENLABS_API_KEY` / `NIJIVOICE_API_KEY`: 音声合成（オプション）

### 2. mulmocast-mcp

**役割**: mulmocastのMCPサーバ

[mulmocast-mcp](https://github.com/receptron/mulmocast-mcp)は、mulmocast-cliの機能をMCP (Model Context Protocol)サーバとして提供します。Claude Desktop等のMCP対応クライアントから、mulmoScriptを使った動画・音声・画像生成機能を利用できます。

**主な機能**:
- mulmocast-cliのMCP化
- Claude Desktopからの直接呼び出し
- マルチモーダルコンテンツの生成

**セットアップ** (Claude Desktop):
```json
{
  "mcpServers": {
    "mulmocast": {
      "command": "npx",
      "args": ["mulmocast-mcp@latest"],
      "env": {
        "OPENAI_API_KEY": "sk-xxx",
        "REPLICATE_API_TOKEN": "r8_xxx",
        "ANTHROPIC_API_KEY": "sk-ant-xxx"
      },
      "transport": { "stdio": true }
    }
  }
}
```

**利用シーン**:
- Claude Desktopでプレゼンテーション動画を作成
- チャット形式でインタラクティブにコンテンツを生成
- AIアシスタントと連携したコンテンツ制作

### 3. mulmocast-vision

**役割**: ビジネススライド自動生成MCPサーバ

[mulmocast-vision](https://github.com/receptron/mulmocast-vision)は、LLMを使ってプロフェッショナルなプレゼンテーションスライドを自動生成するMCPサーバです。**80種類以上のビジネス向けスライドテンプレート**を提供し、提案書・戦略書・企業分析資料などを数秒で作成できます。

**主な機能**:
- **豊富なテンプレート**: 80種類以上のビジネススライドテンプレート
- **高速生成**: LLM統合により数秒でプロフェッショナルなスライドを作成
- **PDF出力**: 自動的にPDF形式で保存
- **カスタマイズ可能**: HTMLベースのテンプレートで完全なデザイン変更が可能

**セットアップ** (Claude Desktop):
```json
{
  "mcpServers": {
    "mulmocast-vision": {
      "command": "npx",
      "args": ["mulmocast-vision@latest"],
      "transport": {"stdio": true}
    }
  }
}
```

**使い方**:
1. Claude Desktopを起動
2. 「複数企業の比較分析スライドを作成して」と指示
3. 自動的に`~/Documents/mulmocast-vision/`にPDFが保存される

**対応スライド種類**:
- 問題・解決策・価値フレームワーク
- ビジネスモデルキャンバス
- SWOT/PEST/3C分析
- アジェンダ・サマリースライド

**ログ機能**:
- `/tmp/mulmocast-vision-mcp/`に動作ログを記録（JSON Lines形式）
- 日次ローテーション対応

### 4. mulmocast-viewer

**役割**: Web用マルチメディアプレイヤーコンポーネント

[mulmocast-viewer](https://github.com/receptron/mulmocast-viewer)は、mulmocast-cliで生成したマルチメディアバンドル（JSONとメディアファイル）をWebブラウザで表示するためのVue 3コンポーネントライブラリです。

**主な機能**:
- **MulmoViewerコンポーネント**: ナビゲーション機能付きメインプレイヤー
- **カスタムUI対応**: スロットベースで柔軟にレイアウトをカスタマイズ
- **言語選択**: 音声とテキストの言語を個別に制御
- **BeatGridView & BeatListView**: ビートカタログをグリッド/新聞レイアウトで表示
- **Vue Router統合**: リンク付きナビゲーションに対応
- **マルチフォーマット対応**: JSONメタデータと関連メディアファイルを扱う

**インストール**:
```bash
npm install mulmocast-viewer
# or
yarn add mulmocast-viewer
```

**基本的な使い方**:
1. **コンテンツ生成**: mulmocast-cliでバンドルを作成
2. **配置**: 生成されたフォルダをプロジェクトのpublicディレクトリに配置
3. **JSON読み込み**: `mulmo_view.json`をインポートまたはフェッチ
4. **パス設定**: メディアファイルの場所を指す`basePath`を設定
5. **統合**: MulmoViewerコンポーネントを適切なpropsと言語バインディングで使用

**統合例**:
```vue
<template>
  <MulmoViewer
    :data="mulmoData"
    :basePath="'/content/'"
    v-model:audioLang="audioLang"
    v-model:textLang="textLang"
  />
</template>

<script setup>
import { MulmoViewer } from 'mulmocast-viewer';
import { ref } from 'vue';

const mulmoData = ref(/* 読み込んだJSON */);
const audioLang = ref('ja');
const textLang = ref('ja');
</script>
```

## 実践的なワークフロー例

### パターン1: プレゼンテーション動画を作る（CLIで完結）

```bash
# 1. 対話形式でスクリプト生成
mulmo tool scripting -i -t business_pitch -o ./ -s my_presentation

# 2. 生成されたスクリプトから動画を作成
mulmo movie my_presentation-1234567890.json -c ja

# 結果: 音声、画像、動画ファイルが自動生成される
```

### パターン2: ビジネススライドを作る（MCPで簡単）

1. **Claude Desktop設定**で`mulmocast-vision`を追加
2. Claude Desktopで「当社の新製品3つの比較分析スライドを作成して」と指示
3. `~/Documents/mulmocast-vision/`にPDFが自動保存される

### パターン3: Web公開用コンテンツを作る（フルスタック）

1. **スクリプト生成** (mulmocast-cli)
   ```bash
   mulmo tool scripting -i -t children_book -o ./content -s story
   ```

2. **動画・音声生成** (mulmocast-cli)
   ```bash
   mulmo movie ./content/story-xxx.json
   ```

3. **Web表示** (mulmocast-viewer)
   ```bash
   npm install mulmocast-viewer
   # Vue 3プロジェクトにMulmoViewerコンポーネントを統合
   ```

4. **公開**: 生成されたバンドルをWebサーバにデプロイ

### パターン4: Claude Desktopでインタラクティブに作成（最も簡単）

1. **Claude Desktop設定**で`mulmocast-mcp`を追加
2. Claudeと会話しながらコンテンツを作成
   - 「絵本のストーリーを考えて、動画にして」
   - 「もっと明るい雰囲気にして」
   - 「BGMを追加して」
3. 自動的に動画・音声・画像が生成される

## mulmoScriptとは

mulmoScriptは、**マルチモーダルコンテンツを記述するためのJSON/YAML形式のスクリプト言語**です。映画の脚本のように、物語の構成、ビジュアル、音声などを統合的に記述します。

**mulmoScriptの特徴**:
- **LLMフレンドリー**: AIが生成しやすい構造化されたフォーマット
- **人間が読める**: JSONなので、プログラマでなくても理解・編集可能
- **マルチモーダル**: テキスト、画像、音声、レイアウトを統合記述
- **バージョン管理**: Git等で管理しやすいテキストベース
- **再利用可能**: テンプレートとして共有・再利用が容易

**mulmoScriptの構成例**:
```json
{
  "title": "AI時代のプレゼンテーション",
  "beats": [
    {
      "text": "これからのプレゼンテーションは、AIと人間の協働で作られます。",
      "image": {
        "prompt": "futuristic presentation with AI",
        "style": "professional"
      },
      "audio": {
        "voice": "ja-JP-Neural"
      }
    }
  ]
}
```

## MCPとの統合

mulmocastファミリーの主要コンポーネントは**MCP (Model Context Protocol)**に対応しており、Claude Desktop等のMCP対応クライアントから直接利用できます。

**MCP対応コンポーネント**:
- **mulmocast-mcp**: 動画・音声・画像の包括的な生成
- **mulmocast-vision**: ビジネススライド（PDF）の自動生成

これにより、**AIアシスタントと会話するだけでプロフェッショナルなコンテンツを作成**できます。

## まとめ

mulmocastファミリーは、**AI時代のマルチモーダルコンテンツ制作を革新する統合ツールセット**です。LLMとの協働により、誰でも簡単にプロフェッショナルな動画、スライド、ポッドキャストを作成できます。

### 各コンポーネントの役割まとめ

| コンポーネント | 役割 | MCP対応 | レポジトリ |
|--------------|------|---------|-----------|
| **mulmocast-cli** | AIパワード動画・音声生成エンジン | - | [GitHub](https://github.com/receptron/mulmocast-cli) |
| **mulmocast-mcp** | 動画・音声生成MCPサーバ | ✓ | [GitHub](https://github.com/receptron/mulmocast-mcp) |
| **mulmocast-vision** | ビジネススライド自動生成MCP | ✓ | [GitHub](https://github.com/receptron/mulmocast-vision) |
| **mulmocast-viewer** | Web用マルチメディアプレイヤー | - | [GitHub](https://github.com/receptron/mulmocast-viewer) |

### こんな人におすすめ

- **ビジネスパーソン**: 提案書や戦略資料を短時間で作成したい
- **教育者**: 教材やプレゼンテーション動画を効率的に制作したい
- **コンテンツクリエイター**: ポッドキャストや動画コンテンツを量産したい
- **開発者**: マルチモーダルコンテンツをアプリに統合したい

### 始め方

#### 最も簡単な方法（MCPサーバ）
1. Claude Desktopに`mulmocast-vision`を設定
2. 「会社紹介のスライドを作って」と話しかける
3. PDFが自動生成される

#### 本格的に使う（CLI）
1. `npm install -g mulmocast`
2. `OPENAI_API_KEY`を設定
3. `mulmo tool scripting -i`でスクリプト生成
4. `mulmo movie <script.json>`で動画作成

### リソース

- **組織**: [receptron on GitHub](https://github.com/receptron)
- **メインレポジトリ**: [mulmocast-cli](https://github.com/receptron/mulmocast-cli)
- **関連記事**: [MulmoCast Vision - mcpでスライドをつくる](https://note.com/singsoc/n/nebf944301311)

mulmocastファミリーを使って、AI時代のコンテンツ制作を体験してください！
