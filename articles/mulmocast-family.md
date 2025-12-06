---
title: "MulmoCast - AI時代のマルチモーダルコンテンツ生成ツール"
emoji: "🎬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [mulmocast, MCP, AI, LLM, multimodal]
published: true
publication_name: "singularity"
---

# MulmoCastとは

**MulmoCast**は、**AIとの協働でプレゼンテーション動画、ポッドキャスト、スライド資料を生成する次世代のコンテンツ制作プラットフォーム**です。

中心となるのは**mulmoScript**と呼ばれるJSON/YAML形式のスクリプト言語で、これは映画の脚本のようにマルチモーダルコンテンツの構成を記述します。LLM（大規模言語モデル）がmulmoScriptを生成し、それを元に動画、音声、画像、PDFなど様々な形式のコンテンツが自動生成されます。

**主な用途**:
- プレゼンテーション動画の自動生成
- ビジネス資料（提案書・戦略書）の作成
- ポッドキャスト音声の制作
- 教育コンテンツ（絵本・マンガ形式）の作成

## mulmocast-cli - MulmoCast本体

[mulmocast-cli](https://github.com/receptron/mulmocast-cli)は、MulmoCastの本体です。**コマンドラインツールとしても、ソフトウェアライブラリとしても利用できる**設計になっており、mulmoScriptから動画、音声、画像、PDFなど複数の形式のコンテンツを自動生成します。

### 主な機能

- **マルチフォーマット出力**: 動画、ポッドキャスト、スライドショー、PDF、マンガ形式に対応
- **AI駆動のスクリプト生成**: 対話形式でmulmoScriptを作成
- **自動音声生成**: テキストから自然な音声を生成（複数のTTSサービス対応）
- **画像自動生成**: 指定されたビジュアルスタイルに合わせた画像合成
- **多言語対応**: 英語・日本語の翻訳・字幕機能
- **ライブラリとしての利用**: 他のプログラムから組み込んで使用可能

### インストール

```bash
# CLIツールとしてグローバルインストール
npm install -g mulmocast

# ライブラリとして利用する場合
npm install mulmocast

# 動画生成に必要（macOS）
brew install ffmpeg
```

### CLIとしての使い方
```bash
# 1. 対話形式でスクリプトを生成（絵本テンプレート）
mulmo tool scripting -i -t children_book -o ./ -s story

# 2. 生成されたスクリプトから動画を作成（日本語字幕付き）
mulmo movie story-1746600802426.json -c ja

# 3. Web公開用バンドル（ZIP）を作成
mulmo bundle story-1746600802426.json
```

### ライブラリとしての使い方

electron版のmulmocast-appが実装例になっています。参考にしてください

https://github.com/receptron/mulmocast-app/blob/main/src/main/mulmo/handler_generator.ts

### 必要な環境変数
- `OPENAI_API_KEY`: スクリプト生成と画像作成に必須
- `GEMINI_API_KEY`: Google画像・音声合成（オプション）
- `ANTHROPIC_API_TOKEN`: Claude統合（オプション）
- `REPLICATE_API_TOKEN`: 動画生成（オプション）
- `ELEVENLABS_API_KEY` / `NIJIVOICE_API_KEY`: 音声合成（オプション）

## 派生ツール - MulmoCastファミリー

mulmocast-cliをより便利に使うための派生ツール群です。MCPサーバ、Webビューア、動画変換ツールなど、様々な用途に応じたツールが提供されています。

### mulmocast-mcp

[mulmocast-mcp](https://github.com/receptron/mulmocast-mcp)は、**mulmocast-cliをMCPサーバ化**したツールです。Claude Desktop等のMCP対応クライアントから、MulmoCastの動画・音声・画像生成機能を直接利用できます。

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

### mulmoscript-mcp

[mulmoscript-mcp](https://github.com/receptron/mulmoscript-mcp)は、Claude Desktopなどと**対話しながらmulmoScriptを作成**するためのMCPサーバです。自然言語での会話を通じて、AIがユーザーの意図を理解し、適切なmulmoScriptを生成します。

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

**使い方**: Claude Desktopで「子供向けの絵本ストーリーのスクリプトを作って」と依頼すると、対話を通じてmulmoScriptが生成されます。

### mulmocast-vision

[mulmocast-vision](https://github.com/receptron/mulmocast-vision)は、元々**mulmoScriptの`image.type = vision`のデータを扱うためのライブラリ**です。これを独立させて単体でも使えるようにし、さらにMCPサーバ化することで、LLMを使ってプロフェッショナルなプレゼンテーションスライドを自動生成できるようになりました。**80種類以上のビジネス向けスライドテンプレート**を提供し、提案書・戦略書・企業分析資料などを数秒で作成できます。

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

**使い方**: Claude Desktopで「複数企業の比較分析スライドを作成して」と指示すると、自動的に`~/Documents/mulmocast-vision/`にPDFが保存されます。

### mulmocast-app

[mulmocast-app](https://github.com/receptron/mulmocast-app)は、MulmoCastの**Electronデスクトップアプリケーション**です。Vue 3とTailwind CSSで構築されており、MulmoCastの機能をGUIで利用できます。

**公式サイト**: [https://mulmocast.com/](https://mulmocast.com/)
**Discord**: [discord.gg/XqmAYxm2Xf](https://discord.gg/XqmAYxm2Xf)

### mulmocast-viewer

[mulmocast-viewer](https://github.com/receptron/mulmocast-viewer)は、MulmoCastで生成したコンテンツ（**`mulmo bundle`コマンドで作成したバンドルを展開したデータ**）をWebブラウザで表示するためのVue 3コンポーネントライブラリです。

**デモサイト**:
- [viewer.mulmocast.com/contents](https://viewer.mulmocast.com/contents)
- [mulmocast.com/test](https://mulmocast.com/test)

**使い方**: `mulmo bundle`で作成したZIPを展開し、Vue 3プロジェクトに組み込みます。

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

### mulmo-movie (movie-separate)

[mulmo-movie](https://github.com/isamu/movie-separate)は、既存の動画ファイルを解析して**mulmoViewer用のデータ（`mulmo_view.json`）を自動生成**するツールです。講演動画やインタビュー動画などを、セグメント分割・文字起こし・翻訳・話者識別まで自動処理します。

**使い方**:
```bash
# グローバルインストール
npm install -g mulmo-movie

# 動画を処理
mulmo-movie video.mp4 --lang ja
```

**出力**: `output/`ディレクトリに`mulmo_view.json`と各種メディアファイルが生成され、mulmocast-viewerで表示できます。

## 使い方の例

### 1. CLIで動画を作る

```bash
# 対話形式でスクリプト生成
mulmo tool scripting -i -t business_pitch -o ./ -s presentation

# 動画を作成
mulmo movie presentation-xxx.json -c ja
```

### 2. MCPでスライドを作る

Claude Desktopに`mulmocast-vision`を設定し、「新製品3つの比較分析スライドを作って」と指示するだけで、PDFが自動生成されます。

### 3. Web公開する

```bash
# バンドル作成
mulmo bundle script.json

# ZIPを展開してVue 3プロジェクトに配置
# mulmocast-viewerで表示
```

### 4. 既存動画を活用する

```bash
# 動画を処理
mulmo-movie interview.mp4 --lang ja

# mulmocast-viewerで表示可能なデータが生成される
```

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

**MulmoCast（mulmocast-cli）**は、AI時代のマルチモーダルコンテンツ制作プラットフォームです。CLIツールとしてもライブラリとしても利用でき、動画、音声、画像、PDFなど様々な形式のコンテンツを自動生成します。

**派生ツール（MulmoCastファミリー）**により、MCPサーバとしての利用、Web表示、既存動画の変換など、様々な用途に対応できます。

### ツール一覧

| ツール | 説明 | レポジトリ |
|--------|------|-----------|
| **mulmocast-cli** | MulmoCast本体（CLI & ライブラリ） | [GitHub](https://github.com/receptron/mulmocast-cli) |
| mulmocast-app | デスクトップアプリ（Electron） | [GitHub](https://github.com/receptron/mulmocast-app) |
| mulmoscript-mcp | 対話式スクリプト生成MCP | [GitHub](https://github.com/receptron/mulmoscript-mcp) |
| mulmocast-mcp | MulmoCastのMCPサーバ化 | [GitHub](https://github.com/receptron/mulmocast-mcp) |
| mulmocast-vision | ビジネススライド生成MCP | [GitHub](https://github.com/receptron/mulmocast-vision) |
| mulmocast-viewer | Web用プレイヤー（Vue 3） | [GitHub](https://github.com/receptron/mulmocast-viewer) |
| mulmo-movie | 動画からデータ生成 | [GitHub](https://github.com/isamu/movie-separate) |

### 始め方

**GUI版（最も簡単）**: [mulmocast.com](https://mulmocast.com/)からデスクトップアプリをダウンロード

**MCP版**: Claude Desktopに設定して、会話で作成

**CLI版**:
```bash
npm install -g mulmocast
mulmo tool scripting -i
mulmo movie <script.json>
```

**ライブラリ版**: プログラムに組み込む
```bash
npm install mulmocast
```

### リソース

- **公式サイト**: [mulmocast.com](https://mulmocast.com/)
- **Discord**: [discord.gg/XqmAYxm2Xf](https://discord.gg/XqmAYxm2Xf)
- **GitHub**: [receptron organization](https://github.com/receptron)
- **メインレポジトリ**: [mulmocast-cli](https://github.com/receptron/mulmocast-cli)
- **関連記事**: [MulmoCast Vision - mcpでスライドをつくる](https://note.com/singsoc/n/nebf944301311)
