---
title: "Mac で無料・正確な日本語読み上げ ── Kokoro と yomiscript の仕組みと使い方"
emoji: "🗣️"
type: "tech"
topics: ["TTS", "Kokoro", "MCP", "ClaudeCode", "LLM"]
published: true
publication_name: "singularity"
---

日本語の読み上げ（TTS）でいちばん困るのは、声の質より**読み間違い**です。「一日から三日まで休みです。」の「一日」は「ついたち」なのか「いちにち」なのか。「上手」は「じょうず」か「うわて」か。日本語では、同じ書き方に読みが何通りもある言葉がたくさんあります。

[yomiscript](https://github.com/receptron/yomiscript) は、この問題を「**文を書いた LLM に、読み方も書かせる**」ことで解決します。音を出すのは、無料で公開されている音声合成モデル **Kokoro-82M** で、手元の Mac の中で動きます。音声合成のための API キーも利用料もいりません。読み方を書く LLM（Claude Code など）は、ふだん使っているものをそのまま使います。その利用料は別にかかります。

この記事は、次の一次資料と、手元の Mac で実際に動かした結果だけをもとに書いています。

- [receptron/yomiscript の README](https://github.com/receptron/yomiscript) ── 仕組みと、パッケージの一覧
- [CLI の README](https://github.com/receptron/yomiscript/tree/main/packages/cli) と [MCP サーバーの README](https://github.com/receptron/yomiscript/tree/main/packages/mcp) ── 使い方と設定
- [eval/browser_tts の README](https://github.com/receptron/yomiscript/tree/main/eval/browser_tts) ── 読みの正確さを測った実験の結果
- npm の `yomiscript` 0.1.5 と `@yomiscript/mcp` 0.1.5 ── 手元で入れて動かした版

---

## 何が新しいのか

ふつうの TTS は、渡された文を**エンジン自身が解析して**読みを決めます。辞書と規則で推測するので、文脈が必要な言葉で間違えます。

yomiscript は、この役割を分けています。README の図を日本語にすると、こうなります。

```
LLM（文と、その読み方を書く）        YomiScript（発音の記述）              Katarite（発音 → 音）
──────────────────────────          ───────────────────────            ────────────────────
「一日から三日まで休みです。」  ──►  ja /tsɯitatɕi<t:一日> kaɾa<t:から>  ──►  Kokoro-82M
                                      mikka<t:三日> made<t:まで> ...↘/
```

登場するのは 3 つです。

- **Yomite（読み手）** ── 文を YomiScript に変える役。ここでは、呼び出している LLM 自身です。文を書いた本人なので、「一日」がどちらの意味かを知っています
- **YomiScript** ── どう発音するかを書いた記述。発音は国際音声記号（IPA）で書き、`<t:…>` に元の文字を添えます。`↘` は文末で下がる、`↗` は上がる（疑問）を表します
- **Katarite（語り手）** ── YomiScript を音にする役。ここでは Kokoro-82M です。**自分では文を解析しません**。書かれた発音のとおりに読むだけです

README には、これは「TTS ではない」と書かれています。読み方を決めるのはエンジンではなく LLM だから、という意味です。

## どれくらい正確なのか

[eval/browser_tts](https://github.com/receptron/yomiscript/tree/main/eval/browser_tts) に実験結果があります。同音異字語、数字、日付、固有名詞などを集めた、読み間違えやすい 70 文（hard set）での結果です。

| 読みを決めたもの | 正しく読めた文の割合 |
|---|---|
| Codex（LLM）が書いた YomiScript | 100% |
| Claude（LLM）が書いた YomiScript | 98.6% |
| pyopenjtalk（辞書ベースの読み） | 78.6% |
| Kokoro に元から付いている日本語の読み | 32.9% |

音声を Whisper（音声認識）で書き起こして、かなの誤り率も測っています。

| 音声 | かな誤り率 |
|---|---|
| LLM の YomiScript ＋ Kokoro | 3.7〜3.9% |
| macOS の音声 | 5.5% |
| Kokoro（元のまま） | 10.3% |

注意点も README に書かれています。どれも **1 回ずつの実行**で、正解の読みは**プロジェクトのネイティブ話者 1 人が確認した暫定版**です。Whisper 自身も間違えるので、誤り率は「システム同士の比較」にしか使えない、とも書かれています。

Claude の 1 問の失敗は、読みではなく表記の問題でした（「経済」を仕様と違う綴りで書いた）。

聞き比べは [yomiscript.pages.dev](https://yomiscript.pages.dev/) でできます。元の Kokoro が読み間違える文を、元のままの読みと、LLM が書いた YomiScript の読みで並べて再生できます。音声はブラウザの中で作られます。

## なぜ無料なのか

- **Kokoro-82M** は Apache 2.0 で公開されているモデルで、Mac の CPU で動きます
- yomiscript のコードも Apache 2.0 です。日本語のピッチ予測モデルは CC BY-SA 4.0 です
- 読み方を書く LLM は、ふだん使っている Claude Code などがそのまま兼ねます。TTS 側に LLM はありません

ただし、LLM の出力は増えます。README によると、YomiScript は元の日本語の約 3 倍の文字数で、`<t:…>` を付けると約 6 倍です。従量課金の API で使う場合は、この分の料金がかかります。

## 使い方 1: Claude Code から話させる（MCP）

いちばん手軽なのは、MCP サーバーとして Claude Code に登録する方法です。Node.js が入っていれば、1 行で終わります。

```bash
claude mcp add yomiscript -- npx -y @yomiscript/mcp
```

これで Claude に `speak` というツールが 1 つ増えます。あとは「〇〇を読み上げて」と頼むだけです。Claude が YomiScript を書いてツールを呼び、Mac のスピーカーから声が出ます。

仕組みのポイントは、**ツールの説明文そのものが YomiScript の書き方になっている**ことです。音素の表、`<t:…>` の付け方、例文が説明文に入っているので、Claude は何も教わらなくても YomiScript を書けます。書き間違えると、サーバーがパーサーのエラーを返すので、Claude はそれを見て直し、呼び直せます。その場合、音は出ません。

Claude Desktop なら `claude_desktop_config.json` に次を足します。

```json
{
  "mcpServers": {
    "yomiscript": {
      "command": "npx",
      "args": ["-y", "@yomiscript/mcp"]
    }
  }
}
```

### 初回だけ時間がかかる

モデルは npm パッケージに入っていません。最初に話したとき、Hugging Face から `~/.cache/yomiscript/` にダウンロードされます。

- Kokoro（fp32）: 325 MB
- 日本語のアクセント辞書: 28 MB（最初に日本語を話したとき）
- 日本語のピッチ予測モデル: 8 MB（同上）

2 回目からはキャッシュを読むだけです。README には、Mac で最初の日本語の文が 1.1 秒、2 文目が 0.3 秒だったと書かれています。

### 主な設定

環境変数で変えられます。`claude mcp add` なら `-e` で渡します。

| 変数 | 既定 | 意味 |
|---|---|---|
| `YOMISCRIPT_PLAY` | `1` | `0` にすると、この Mac では鳴らさない |
| `YOMISCRIPT_RETURN_AUDIO` | `0` | `1` にすると、音声（WAV）をクライアントにも返す |
| `YOMISCRIPT_MODEL` | `fp32` | `w16` にすると、軽い fp16 の重みを使う |
| `YOMISCRIPT_KATARITE` | `kokoro` | `supertonic` で別の音声エンジンを使う |

Supertonic 3 の重みは OpenRAIL-M で、用途に制限があります。無料で気にせず使うなら、既定の Kokoro のままが安全です。

## 使い方 2: コマンドラインで鳴らす（CLI）

MCP を使わず、コマンドで直接鳴らすこともできます。

```bash
npm install -g yomiscript
# 入れずに使うなら、各コマンドの先頭を npx yomiscript に
```

```bash
# 再生する
yomiscript speak "ja /tsɯitatɕi<t:一日> kaɾa<t:から> mikka<t:三日> made<t:まで> jasɯmi<t:休み> desɯ<t:です。> ↘/"

# WAV に書き出す
yomiscript speak --out hello.wav "ja /kʲoː<t:今日> wa<t:は> ‖<t:、> ame<t:雨> desɯ<t:です。> ↘/"

# ファイルや標準入力から
yomiscript speak --file utterance.ys
echo "ja /ame ↘/" | yomiscript speak
```

Mac では `afplay` で再生されます。Linux では `paplay` か `aplay`、それ以外の環境では一時 WAV のパスが表示されます。

### YomiScript を自分で書かなくていい

YomiScript を手で書く必要はありません。次のコマンドで、**LLM に渡すための書き方の説明**が出力されます。

```bash
yomiscript instructions
```

これを ChatGPT や Claude などのプロンプトに貼り、「次の文を YomiScript にして」と頼めば、YomiScript が返ってきます。説明文やコードの囲み（```）が付いていたら取り除いて、`yomiscript speak` に渡します。MCP に対応していない LLM でも使える方法です。

## 手元で動かした結果

2026 年 11 月 7 日に、Mac（Apple Silicon、Node.js 26）で `yomiscript` 0.1.5 を入れて試しました。

```
$ yomiscript speak --out ichi.wav "ja /tsɯitatɕi<t:一日> kaɾa<t:から> mikka<t:三日> made<t:まで> jasɯmi<t:休み> desɯ<t:です。> ↘/"
wrote ichi.wav
一日から三日まで休みです。 (2.92 s of audio in 2.02 s, kokoro)
```

約 2.9 秒の音声が、約 2.0 秒で作られました。ほかの処理で Mac がかなり重い状態での値です。stderr には、`<t:…>` をつなげた元の文と所要時間が出ます。

MCP サーバーも、標準入出力から `speak` を直接呼んで確かめました（`YOMISCRIPT_PLAY=0` で、鳴らさない設定）。

```
{"result":{"content":[{"type":"text","text":"Synthesized \"今日は雨です。\" (1.82 s)."}]}, ...}
```

## いまの制約

まだ初期段階です。README には「Status: early」とあり、YomiScript の仕様は v0.2 の草案です。

- **長い文は一度に話せない** ── Kokoro は 1 回に 510 音素までしか扱えません。リリース前の課題として、文の区切り（`↘` / `↗`）で分けて読む機能が [docs/plan.md](https://github.com/receptron/yomiscript/blob/main/docs/plan.md) に挙がっています。現状は、長い文章を 1 回で渡すと失敗します
- **LLM の出力が増える** ── 前に書いたとおり、元の文の 3〜6 倍の文字数になります
- **評価はまだ小規模** ── 正解の読みは暫定版で、実験も 1 回ずつです

## まとめ

- yomiscript は、**読み方を LLM が決め、Kokoro はそのとおりに読むだけ**、という分担で日本語の読み間違いを減らします
- 公開されている実験では、読み間違えやすい 70 文で、LLM が書いた読みは 98.6〜100% 正しく、Kokoro に元から付いている読みは 32.9% でした
- Mac なら `claude mcp add yomiscript -- npx -y @yomiscript/mcp` の 1 行で、Claude Code に話させられます。音声合成はローカルで動き、その料金はかかりません。かかるのは、読み方を書く LLM の利用料だけです
