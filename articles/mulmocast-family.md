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

# 3. Web公開用バンドル（ZIP）を作成
mulmo bundle story-1746600802426.json
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

### 3. mulmoscript-mcp

**役割**: 対話式mulmoScript生成MCPサーバ

[mulmoscript-mcp](https://github.com/receptron/mulmoscript-mcp)は、Claude Desktopなどと**対話しながらmulmoScriptを作成**するためのMCPサーバです。自然言語での会話を通じて、AIがユーザーの意図を理解し、適切なmulmoScriptを生成します。

**主な機能**:
- **対話式スクリプト生成**: Claudeと会話しながらmulmoScriptを作成
- **インタラクティブな編集**: 「もっと明るい雰囲気にして」などの要望に応じて修正
- **テンプレート対応**: 様々なコンテンツタイプに対応したスクリプト生成

**セットアップ** (Claude Desktop):
```json
{
  "mcpServers": {
    "mulmoscript": {
      "command": "npx",
      "args": ["mulmoscript-mcp"],
      "transport": {"stdio": true}
    }
  }
}
```

**使い方**:
1. Claude Desktopで「子供向けの絵本ストーリーのスクリプトを作って」と依頼
2. Claudeがいくつか質問して、内容を詰める
3. mulmoScriptが自動生成される
4. 生成されたスクリプトを`mulmo movie`で動画化

### 4. mulmocast-vision

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

### 5. mulmocast-viewer

**役割**: Web用マルチメディアプレイヤーコンポーネント

[mulmocast-viewer](https://github.com/receptron/mulmocast-viewer)は、mulmocast-cliの**`mulmo bundle`コマンドで生成したバンドル（ZIP）を展開したデータ**をWebブラウザで表示するためのVue 3コンポーネントライブラリです。

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
1. **バンドル作成**: `mulmo bundle script.json`でZIPファイルを生成
2. **展開**: 生成されたZIPをプロジェクトのpublicディレクトリに展開
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

### 6. mulmo-movie (movie-separate)

**役割**: 動画からmulmoViewer用データを自動生成

[mulmo-movie](https://github.com/isamu/movie-separate)は、既存の動画ファイルを解析して**mulmoViewer用のデータ（`mulmo_view.json`）を自動生成**するAI駆動の動画処理ツールです。対話動画を自動的にセグメント分割、文字起こし、翻訳、話者識別まで行います。

**主な機能**:
- **スマート動画分割**: 無音区間を検出して、会話の途中で切らずにセグメント分割
- **音声抽出**: 各セグメントからMP3ファイルを自動生成
- **多言語対応**: 英語・日本語の音声を認識し、双方向翻訳
- **文字起こし**: OpenAI Whisper APIで高精度な音声認識
- **自動翻訳**: GPT-4o-miniで日英・英日翻訳
- **日本語音声生成**: 翻訳されたテキストからOpenAI TTSで音声を生成
- **並列処理**: p-limitで複数APIを並列呼び出し、高速処理
- **翻訳キャッシュ**: 既存の翻訳を再利用してAPIコストを削減
- **話者識別**: GPT-4oが自動的に話者を識別・ラベリング
- **重要度スコアリング**: 各セグメントに0-10の重要度を付与、カテゴリ分類
- **ダイジェスト作成**: 重要度の高いセグメントだけを抽出
- **JSON出力**: mulmoViewer用の構造化されたJSON形式で出力

**必要な環境**:
- Node.js 18以上
- ffmpeg
- OpenAI APIキー

**インストール**:
```bash
# グローバルインストール（推奨）
npm install -g mulmo-movie

# ローカルインストール
npm install mulmo-movie
```

**セットアップ**:
```bash
# 環境変数ファイルを作成
cp .env.example .env

# .envファイルを編集してAPIキーを設定
OPENAI_API_KEY=sk-your-api-key-here
```

**基本的な使い方**:
```bash
# 基本的な処理
mulmo-movie video.mp4

# 日本語音声として処理
mulmo-movie video.mp4 --lang ja

# テストモード
mulmo-movie --test video.mp4

# カスタム出力ディレクトリ
mulmo-movie video.mp4 --output ./custom-directory
```

**出力構造**:
`output/`ディレクトリに以下が生成されます:
- `mulmo_view.json`: メタデータと文字起こし結果の完全版
- セグメント動画: `1.mp4`, `2.mp4`, ...
- サムネイル画像: 各セグメント用
- 元言語の音声ファイル
- 生成された日本語音声ファイル

JSONには、テキスト、翻訳、話者名、タイムスタンプ、重要度スコア、カテゴリ、要約が含まれます。

**活用シーン**:
- **講演・セミナー動画**: 自動的にセグメント化して検索可能に
- **インタビュー動画**: 話者を自動識別して文字起こし
- **教育コンテンツ**: 多言語対応で世界中に配信
- **動画アーカイブ**: 既存の動画をmulmoViewer形式に変換して再活用

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

3. **バンドル作成** (mulmocast-cli)
   ```bash
   mulmo bundle ./content/story-xxx.json
   # story-bundle.zipが生成される
   ```

4. **Web表示** (mulmocast-viewer)
   ```bash
   npm install mulmocast-viewer
   # ZIPを展開してpublic/に配置
   # Vue 3プロジェクトにMulmoViewerコンポーネントを統合
   ```

5. **公開**: Webサーバにデプロイ

### パターン4: Claude Desktopでインタラクティブに作成（最も簡単）

1. **Claude Desktop設定**で`mulmoscript-mcp`と`mulmocast-mcp`を追加
2. Claudeと会話しながらスクリプトを作成
   - 「絵本のストーリーを考えて」（mulmoscript-mcp）
   - 「もっと明るい雰囲気にして」
   - スクリプトが生成される
3. 「このスクリプトから動画を作って」（mulmocast-mcp）
4. 自動的に動画・音声・画像が生成される

### パターン5: 既存の動画をmulmoViewer化（動画アーカイブ活用）

1. **既存動画を処理** (mulmo-movie)
   ```bash
   mulmo-movie interview.mp4 --lang ja
   ```

2. **自動生成される内容**:
   - セグメント分割された動画
   - 文字起こし（日英両言語）
   - 話者識別
   - 重要度スコア
   - `mulmo_view.json`

3. **Web表示** (mulmocast-viewer)
   ```bash
   npm install mulmocast-viewer
   # output/フォルダをpublic/に配置
   # MulmoViewerコンポーネントで表示
   ```

4. **活用**: インタラクティブな動画検索・視聴システムが完成

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
- **mulmoscript-mcp**: 対話式でmulmoScriptを生成
- **mulmocast-mcp**: 動画・音声・画像の包括的な生成
- **mulmocast-vision**: ビジネススライド（PDF）の自動生成

これにより、**AIアシスタントと会話するだけでプロフェッショナルなコンテンツを作成**できます。

**Claude Desktop設定例**（3つ全部セットアップ）:
```json
{
  "mcpServers": {
    "mulmoscript": {
      "command": "npx",
      "args": ["mulmoscript-mcp"],
      "transport": {"stdio": true}
    },
    "mulmocast": {
      "command": "npx",
      "args": ["mulmocast-mcp@latest"],
      "env": {
        "OPENAI_API_KEY": "sk-xxx",
        "REPLICATE_API_TOKEN": "r8_xxx"
      },
      "transport": {"stdio": true}
    },
    "mulmocast-vision": {
      "command": "npx",
      "args": ["mulmocast-vision@latest"],
      "transport": {"stdio": true}
    }
  }
}
```

## まとめ

mulmocastファミリーは、**AI時代のマルチモーダルコンテンツ制作を革新する統合ツールセット**です。LLMとの協働により、誰でも簡単にプロフェッショナルな動画、スライド、ポッドキャストを作成できます。

### 各コンポーネントの役割まとめ

| コンポーネント | 役割 | MCP対応 | レポジトリ |
|--------------|------|---------|-----------|
| **mulmocast-cli** | AIパワード動画・音声生成エンジン | - | [GitHub](https://github.com/receptron/mulmocast-cli) |
| **mulmoscript-mcp** | 対話式mulmoScript生成MCPサーバ | ✓ | [GitHub](https://github.com/receptron/mulmoscript-mcp) |
| **mulmocast-mcp** | 動画・音声生成MCPサーバ | ✓ | [GitHub](https://github.com/receptron/mulmocast-mcp) |
| **mulmocast-vision** | ビジネススライド自動生成MCP | ✓ | [GitHub](https://github.com/receptron/mulmocast-vision) |
| **mulmocast-viewer** | Web用マルチメディアプレイヤー | - | [GitHub](https://github.com/receptron/mulmocast-viewer) |
| **mulmo-movie** | 動画からmulmoViewer用データ生成 | - | [GitHub](https://github.com/isamu/movie-separate) |

### こんな人におすすめ

- **ビジネスパーソン**: 提案書や戦略資料を短時間で作成したい
- **教育者**: 教材やプレゼンテーション動画を効率的に制作したい
- **コンテンツクリエイター**: ポッドキャストや動画コンテンツを量産したい
- **開発者**: マルチモーダルコンテンツをアプリに統合したい
- **動画アーカイブ管理者**: 既存の動画を検索可能・インタラクティブに変換したい

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
