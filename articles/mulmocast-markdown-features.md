---
title: "MulmoCast Markdown新機能 - スタイル・レイアウト・Mermaid埋め込み"
emoji: "📊"
type: "tech"
topics: [AI, mulmocast, GraphAI, Markdown, Mermaid]
published: false
publication_name: "singularity"
---

# MulmoCast Markdownの新機能

MulmoCastに4つの強力なMarkdown機能が追加されました。

1. **100種類のプリセットスタイル** - 美しいデザインを簡単に適用
2. **レイアウト機能** - 2列、4分割、ヘッダー、サイドバーに対応
3. **Mermaid埋め込み** - 図とテキストを並べて表示
4. **画像埋め込み** - 外部画像をレイアウト内に配置

この記事では、それぞれの機能を初心者にもわかりやすく解説します。

## 必要なバージョン

これらの機能を使用するには **MulmoCast 2.1.30** 以降が必要です。

### インストール

```bash
npm install -g mulmocast
```

### アップデート

既にインストール済みの場合は、以下のコマンドで最新版に更新できます：

```bash
npm update -g mulmocast
```

バージョン確認：

```bash
npx mulmocast --version
```

## 1. スタイル機能

### スタイルとは？

`style`プロパティを指定するだけで、markdownやtextSlideに美しいデザインを適用できます。100種類のプリセットスタイルが用意されており、ビジネス、テック、日本風など様々なカテゴリから選べます。

### 使い方

```json
{
  "type": "markdown",
  "markdown": [
    "# タイトル",
    "コンテンツ"
  ],
  "style": "corporate-blue"
}
```

たったこれだけで、青いグラデーション背景のビジネス向けスライドが作成されます。

### スタイル適用例

`corporate-blue`スタイルを適用した例：

![corporate-blueスタイル](/images/mulmocast-markdown/style_corporate.png)

### 主なスタイルカテゴリ

| カテゴリ | 代表的なスタイル | 特徴 |
|---------|-----------------|------|
| business | corporate-blue, executive-gray | ビジネス向け、落ち着いた配色 |
| tech | cyber-neon, terminal-dark | テック感、ネオンカラー |
| japanese | washi-paper, sakura-pink, zen-garden | 和風デザイン |
| dark | charcoal-elegant, midnight-blue | ダークモード対応 |
| minimalist | clean-white, nordic-light | シンプル、読みやすい |
| creative | artistic-splash, neon-glow | 派手、印象的 |

### textSlideでも使える

textSlideでも同様にスタイルを適用できます：

```json
{
  "type": "textSlide",
  "slide": {
    "title": "タイトル",
    "subtitle": "サブタイトル",
    "bullets": ["項目1", "項目2", "項目3"]
  },
  "style": "cyber-neon"
}
```

`cyber-neon`スタイル適用例：

![cyber-neonスタイル](/images/mulmocast-markdown/style_cyber.png)

### スタイル一覧の確認方法

利用可能な全スタイル名を確認するには：

```bash
npx mulmocast tool info --category markdown-styles
```

## 2. レイアウト機能

### レイアウトとは？

通常のmarkdownは単一カラムですが、レイアウト機能を使うと2列表示や4分割表示ができます。

### 2列レイアウト（row-2）

左右に異なるコンテンツを配置できます：

```json
{
  "type": "markdown",
  "markdown": {
    "row-2": [
      ["# 左カラム", "左側のコンテンツ"],
      ["# 右カラム", "右側のコンテンツ"]
    ]
  },
  "style": "soft-gray"
}
```

![2列レイアウト](/images/mulmocast-markdown/layout_row2.png)

### 4分割レイアウト（2x2）

4つのセクションに分割して表示：

```json
{
  "type": "markdown",
  "markdown": {
    "2x2": [
      "# 左上",
      "# 右上",
      "# 左下",
      "# 右下"
    ]
  },
  "style": "soft-gray"
}
```

![4分割レイアウト](/images/mulmocast-markdown/layout_2x2.png)

### ヘッダー付きレイアウト

上部にヘッダーを追加できます：

```json
{
  "type": "markdown",
  "markdown": {
    "header": "# ページタイトル",
    "row-2": [
      "左コンテンツ",
      "右コンテンツ"
    ]
  },
  "style": "soft-gray"
}
```

![ヘッダー付きレイアウト](/images/mulmocast-markdown/layout_header.png)

### サイドバー付きレイアウト

左側にサイドバー（メニューなど）を配置：

```json
{
  "type": "markdown",
  "markdown": {
    "sidebar-left": ["## メニュー", "- 項目1", "- 項目2"],
    "content": "メインコンテンツ"
  },
  "style": "soft-gray"
}
```

![サイドバー付きレイアウト](/images/mulmocast-markdown/layout_sidebar.png)

## 3. Mermaid埋め込み

### Mermaidとは？

Mermaidはテキストでフローチャートやシーケンス図を描くための記法です。MulmoCastでは、markdown内に直接Mermaid記法を埋め込めます。

### 使い方

markdownの中でMermaidコードブロックを書くだけ：

```json
{
  "type": "markdown",
  "markdown": {
    "row-2": [
      ["```mermaid", "graph TD", "    A[開始] --> B[処理]", "    B --> C[終了]", "```"],
      ["# 説明", "このフローは..."]
    ]
  }
}
```

レイアウト機能と組み合わせることで、左に図、右に説明というスライドが作れます。

![Mermaid埋め込み](/images/mulmocast-markdown/layout_mermaid.png)

### 対応するMermaid記法

- フローチャート（graph TD, graph LR）
- シーケンス図
- クラス図
- ガントチャート
- その他Mermaidがサポートする全ての記法

## 4. 画像埋め込み

### 外部画像の埋め込み

Markdown記法で外部画像をレイアウト内に埋め込めます：

```json
{
  "type": "markdown",
  "markdown": {
    "row-2": [
      ["# 画像サンプル", "", "![画像](https://example.com/image.png)"],
      ["# 説明", "", "左側に画像、右側にテキストを配置できます。"]
    ]
  },
  "style": "soft-gray"
}
```

![画像埋め込み](/images/mulmocast-markdown/layout_image.png)

外部URLの画像は自動的に読み込まれます。

## 実践例：4つの機能を組み合わせる

スタイル、レイアウト、Mermaidを組み合わせた実践例：

![実践例](/images/mulmocast-markdown/combined_example.png)

```json
{
  "$mulmocast": { "version": "1.1" },
  "lang": "ja",
  "title": "プロジェクト概要",
  "beats": [
    {
      "speaker": "Presenter",
      "text": "プロジェクトのフロー図と説明です。",
      "image": {
        "type": "markdown",
        "markdown": {
          "header": "# プロジェクトフロー",
          "row-2": [
            ["```mermaid", "graph TD", "    A[企画] --> B[設計]", "    B --> C[開発]", "    C --> D[テスト]", "    D --> E[リリース]", "```"],
            ["## フェーズ説明", "", "1. **企画**: 要件定義", "2. **設計**: アーキテクチャ設計", "3. **開発**: 実装", "4. **テスト**: 品質保証", "5. **リリース**: 本番デプロイ"]
          ]
        },
        "style": "corporate-blue"
      }
    }
  ]
}
```

## まとめ

MulmoCastのMarkdown新機能により、より表現力豊かなプレゼンテーションが作成できるようになりました。

| 機能 | 用途 |
|-----|------|
| スタイル | デザインを簡単に適用 |
| レイアウト | 複数カラム、ヘッダー、サイドバー |
| Mermaid | 図とテキストを並べて表示 |
| 画像埋め込み | 外部画像をレイアウト内に配置 |

ぜひお試しください！

## 参考リンク

- [MulmoCast GitHub](https://github.com/receptron/mulmocast-cli)
- [MulmoCast ドキュメント](https://github.com/receptron/mulmocast-cli/blob/main/docs/image.md)
