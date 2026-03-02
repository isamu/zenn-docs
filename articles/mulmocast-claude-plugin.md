---
title: "MulmoCast Claude Plugin - Claude Codeから動画プレゼンテーションを作成"
emoji: "🔌"
type: "tech"
topics: [mulmocast, ClaudeCode, AI, presentation, plugin]
published: true
publication_name: "singularity"
---

# MulmoCast Claude Plugin とは

**MulmoCast Claude Plugin**は、[Claude Code](https://claude.com/claude-code)から直接動画プレゼンテーションを作成できるプラグインです。

URLやトピック、ドキュメントを入力するだけで、リサーチからナレーション作成、スライドデザイン、動画生成まで自動で行います。

https://github.com/receptron/mulmocast-claude-plugin

## できること

- **URLから動画を作成**: 記事を取得・分析し、プロフェッショナルなプレゼンテーション動画を生成
- **トピックから動画を作成**: Web検索でリサーチし、データリッチなスライドで動画を生成
- **ドキュメントから動画を作成**: PDFやテキストファイルを読み込み、動画にまとめる
- **多言語対応**: 日本語・英語に対応

## インストール

```bash
# Step 1: マーケットプレイスを追加
claude plugin marketplace add receptron/mulmocast-claude-plugin

# Step 2: プラグインをインストール
claude plugin install mulmocast@mulmocast-plugins
```

## 必要条件

- **Node.js** 22+
- **ffmpeg** — 動画・音声の結合に使用
- **OPENAI_API_KEY** — テキスト読み上げ（デフォルトTTSプロバイダー）

オプションで `GEMINI_API_KEY`（画像生成・TTS）、`REPLICATE_API_TOKEN`（動画生成）、`ELEVENLABS_API_KEY`（TTS）も利用可能です。

## 使い方

Claude Code で `/mulmocast:story` コマンドを実行します。

```
/mulmocast:story https://example.com/article 日本語でmovie
```

```
/mulmocast:story AI trends in 2026, 5 slides, English
```

```
/mulmocast:story path/to/quarterly-report.pdf
```

## 5フェーズの構造化プロセス

プラグインは対話的な5フェーズのプロセスで動画を作成します。各フェーズでユーザーの承認を得てから次に進みます。

### Phase 1: Research（リサーチ）

入力ソースに応じてコンテンツを収集・分析します。

- **URL**: WebFetchまたはPlaywrightでページを取得・解析
- **トピック**: WebSearchで3〜5のソースを検索
- **ファイル**: ドキュメントを読み込み、テーマを特定

リサーチ結果を**トピックブリーフ**として提示し、テーマ・トーン・キーインサイトを確認します。

### Phase 2: Structure（構成）

コンテンツ量に応じたビート数を決定し、**ビート構成**を提示します。

| ソースの長さ | ビート数 | 構成 |
|---|---|---|
| 短い（1記事） | 3〜8 | HOOK → SECTIONS → CLOSE |
| 中程度（長い記事） | 8〜15 | HOOK → (SECTION × N) → CLOSE |
| 長い（レポート） | 15〜25 | HOOK → (CHAPTER × N) → CLOSE |

### Phase 3: Narration（ナレーション）

各ビートの話し言葉ナレーションを作成します。

- 1ビートあたり2〜4文（30〜60語）
- 自然な話し言葉のリズム
- 具体的な数字や名前を含む

### Phase 4: Visual Design（ビジュアルデザイン）

11種類のレイアウトと12種類のコンテンツブロックを使ってスライドをデザインします。

**レイアウト**: title, columns, comparison, grid, bigQuote, stats, timeline, split, matrix, table, funnel

**コンテンツブロック**: text, bullets, code, callout, metric, divider, image, imageRef, chart, mermaid, table, section

ビート数に応じてスライドの密度も自動調整します。

| ビート数 | 密度 | アプローチ |
|---|---|---|
| 3〜5 | 最大 | チートシートのように詰め込む |
| 6〜10 | 標準 | 3〜5ポイント/スライド + 画像 |
| 11+ | ゆったり | 1ポイント/スライド、余白活用 |

### Phase 5: Assembly（組み立て）

ナレーションとビジュアルをMulmoScript JSONに組み立て、動画を生成します。

出力: `output/video/<basename>.mp4`

## スライドテーマ

6種類のプリセットテーマから、コンテンツに合ったテーマを自動選択します。

| コンテンツタイプ | テーマ |
|---|---|
| ビジネスニュース、金融データ | corporate（デフォルト） |
| ポップカルチャー、エンタメ | pop |
| 教育、チュートリアル | warm |
| 学術、研究 | minimal |
| スタートアップ、デザイン | creative |
| テック、開発者向け | dark |

## シネマティック・アニメーション

スライドベースのプレゼンテーションに加えて、**html_tailwind アニメーション**で映画風の演出を作成できます。CSSアニメーションとJavaScriptで、映画やアニメのHUDオーバーレイ・シーン背景をプログラマブルに生成します。

14種類のシネマティックテーマが用意されています：

| # | テーマ | 特徴 |
|---|---|---|
| 1 | Space Opera (Star Wars) | 3Dオープニングクロール、星空生成 |
| 2 | Cyberpunk Terminal (攻殻機動隊) | ネオングリーン端末、CRTスキャンライン |
| 3 | Mecha Anime (エヴァンゲリオン) | 赤警告画面、タイプライター、ステータスカウンター |
| 4 | Film Noir | モノクロ、ベネチアンブラインド、煙エフェクト |
| 5 | Retro Synthwave / 80s | ネオングリッド、サンセットグラデーション |
| 6 | Matrix / Digital Rain | デジタル雨、緑のコード |
| 7 | Documentary / Nature | ケンバーンズ効果、字幕オーバーレイ |
| 8 | Anime Opening | 速度線、ダイナミックテキスト |
| 9 | Horror / Thriller | グリッチ、赤フラッシュ、ノイズ |
| 10 | Terminator T-800 Vision | 都市シルエット生成、ターゲティングブラケット |
| 11 | Dragon Ball Scouter | 戦闘力カウンター、戦士シルエット、レンズフレーム |
| 12 | Blade Runner | 3層シティスケープ、ネオンサイン、雨 |
| 13 | Total Recall | 火星グラデーション、ドーム構造、脳スキャン |
| 14 | Iron Man JARVIS HUD | ヘキサゴナルグリッド、3Dホログラムパネル |

### サンプル動画

14テーマすべてを使ったショーケース動画です：

@[youtube](oLxfhzx5nnQ)

「Star Wars風に紹介して」「ターミネーターのHUDで」のように指示するだけで、テーマに合ったアニメーションが自動生成されます。

## リンク

- [ドキュメント](https://mulmocast.com/docs/claude-plugin)
- [GitHub - mulmocast-claude-plugin](https://github.com/receptron/mulmocast-claude-plugin)
- [GitHub - mulmocast-cli](https://github.com/receptron/mulmocast-cli)
- [Claude Code](https://claude.com/claude-code)
