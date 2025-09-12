---
title: "MulmoCast Vision - mcpでスライドをつくる"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ AI, LLM, Tech, mulmocast, GraphAI]
published: true
publication_name: "singularity"
---

# MulmoCast Visionとは

MulmoCast Visionは、mcpでスライドを作成するツールです。

- 簡単なセットアップ
- すぐにPDF形式のスライドが作成できる
- 企画書や提案書、分析資料など、ビジネスでよく使うスライドを「指示するだけ」で自動生成
- ビジネスシーンで役立つ80種類以上のテンプレートを用意
- PDF化も自動、メール添付や共有もスムーズ
- HTMLベースなので、後から細かい調整が可能
- HTMLテンプレートを変更するだけで、様々なデザインを適用可能

という特徴があります。

このような資料が数分で作成できます。

[AI企業分析スライド例（PDF）](https://github.com/isamu/slide_example/blob/master/pdf/AI_Companies_Corporate_Analysis_2025.pdf)

## 設定方法

Claude desktopの例で説明します。`claude_desktop_config.json`に以下を追加してください。他のMCPでも同様の設定が可能です。

```json
// claude_desktop_config.json
"mulmocast-vision": {
  "command": "npx",
  "args": [
    "mulmocast-vision@latest"
  ],
  "transport": {
    "stdio": true
  }
}
```

設定は以上です。  
`npx`のパスが通っていない場合はフルパスを指定してください。`npx`がインストールされていない場合は、事前にインストールしてください。

## 使い方

「ソフトバンクの300年の戦略について20枚程度でスライド化」といった指示をするだけで、スライドが作成されます。  
使い方はこれだけです。

![](https://storage.googleapis.com/zenn-user-upload/b179cd0d99ec-20250913.png)

作成されたPDFの例：

[ソフトバンク300年戦略スライド（PDF）](https://github.com/isamu/slide_example/blob/master/pdf/softbank_300year_strategy.pdf)

PDFは「書類/mulmocast-vision/{日付}」以下に作成されます。

現在利用できる機能は、各ページのスライド作成、スライドの指定更新、スライド全体のPDF化、スライド指定でのPDF化です。  
プロンプトでそれぞれ指示可能です。

## 開発者向け情報

オープンソースなので、HTMLを変更することで様々なデザインを適用できます。  
スタイルの追加については[こちら](https://github.com/receptron/mulmocast-vision/blob/main/Style.ja.md)を参考にしてください。

---

### 公式リポジトリ・パッケージ

- [GitHub: receptron/mulmocast-vision](https://github.com/receptron/mulmocast-vision)
- [npm: mulmocast-vision](https://www.npmjs.com/package/mulmocast-vision)

