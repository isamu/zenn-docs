---
title: "もうコードレビューをしていない。代わりに、壊れたら赤くなるものを積み上げた"
emoji: "🚦"
type: "tech"
topics: ["ClaudeCode", "AI", "テスト", "CI", "vibecoding"]
published: false
publication_name: "singularity"
---

最近、マージボタンを押すとき、僕はその diff を読んでいないことのほうが多くなりました。

**コードレビューをしていません。** 場合によっては、動作確認もしていません。

こう書くと無責任に聞こえると思います。実際、少し前の僕がこれを読んだら「そんなの事故るに決まってる」と言ったはずです。

でも、事故っていないのです。簡単なバグは、ほぼ出ません。

なぜかというと、**レビューでしか捕まらないものを、先に減らしたから**です。この記事は、その「先に減らす」ために何を整えたかの話です。

先に断っておくと、これは「たくさん書けるようになった」という話ではありません。むしろ逆で、**読む時間が足りなくなった人間が、読まずに済ませるために何をしたか**という、わりと後ろ向きな動機から始まっています。

## 前提：人間のレビューはスケールしない

少し前、[ターミナルを自作したらコミットが1日500を超えた](https://zenn.dev/singularity/articles/diy-terminal-500-commits)という記事を書きました。並列でエージェントを走らせると、書かれるコードの量が人間の読める速度を軽く超えます。

ちなみにあの「1日500」は、数え方でどうとでも変わる数字です。GitHub がデフォルトブランチへのコミットとして数えるぶんだけを平均すれば、1日60そこそこにしかなりません。**コミット数は生産性の指標として使えない**——これは前回も書きましたが、今回も同じ前提で読んでください。数を誇る話ではないです。

問題は量そのものではなく、**量に対して自分の読む速度が追いつかなくなった**ことでした。

このとき、選択肢は2つしかありません。

1. 生成を減らして、読める量に合わせる
2. **読まなくても壊れないようにする**

僕は2を選びました。そして2を選ぶというのは、精神論ではなく、**「壊れたら赤くなって止まる」を積み上げる作業**でした。

以下、実際に mulmoterminal / mulmoclaude で動いているものを、設定ファイルごと紹介します。

---

## ① CLAUDE.md — 規約を人の記憶ではなくファイルに置く

まず、エージェントに渡す規約です。ここは**2層**に分けています。

- **グローバル**（`~/.claude/CLAUDE.md`）— 全プロジェクトに効く普遍的なルール
- **リポジトリ固有**（各リポジトリの `CLAUDE.md`）— そのリポでしか通用しない事実

グローバル側は公開してあります。ここから先に出てくる規約は、だいたいこの中にあります。

https://github.com/isamu/claude

意外に思われるかもしれませんが、リポジトリ側——mulmoterminal の `CLAUDE.md` は **46行しかありません**。

長い規約は読まれません。これは人間の話ではなく、**AI についても同じ**です。数百行の憲法を書いても、実際に効くのは「毎回必ず踏む数行」だけでした。

だから、リポジトリ側に書くのは3種類に絞っています。

**(a) スタックとパッケージマネージャ**

```markdown
- TypeScript. Web UI: Vue 3 (Composition API) + Vite (`src/`).
  Backend: Express + node-pty, run via tsx (`server/`). Shared code in `common/`.
- Package manager: yarn (yarn.lock). Use `yarn add`; don't hand-edit package.json.
```

**(b) 変更後に必ず走らせるコマンド**

ここが一番効きます。そして一番、事故から生まれています。

```markdown
- `yarn typecheck` — vue-tsc -b. **App code only — it does NOT compile the specs.**
- `yarn typecheck:server` / `yarn typecheck:test` — CI runs these too.
  Change a shared type or a wire shape and run **all three**:
  `yarn typecheck` alone passes while CI fails.
```

📎 [`CLAUDE.md#L15-L19`](https://github.com/receptron/mulmoterminal/blob/5e8252440ddb7cb5e4df94e9793935fada0d5d9a/CLAUDE.md#L15-L19)

太字の部分は、実際に何度もやられたやつです。`yarn typecheck` がローカルで通ったのに CI が落ちる。理由は、テストが型チェックの対象に入っていないから。

これは「気をつける」で解決しません。**ファイルに書いて、毎回読ませる**しかない。

**(c) どこに何を置くかの「理由」**

```markdown
- `common/` — code shared by server and UI. Both tsconfig include it, so a value
  or wire type that BOTH sides decide from belongs here — never mirrored into
  `server/` and `src/` with a "keep the two copies in sync" comment.
```

📎 [`CLAUDE.md#L27-L32`](https://github.com/receptron/mulmoterminal/blob/5e8252440ddb7cb5e4df94e9793935fada0d5d9a/CLAUDE.md#L27-L32)

「同期を保つこと」というコメント付きのコピーを**禁止**しています。あれは、いつか必ず片方だけ直されます。

### グローバル側は「短い本体＋深いドキュメント」

全プロジェクト共通のほう（`~/.claude/CLAUDE.md`）も、同じ問題を抱えます。共通ルールは書きたいことが多く、放っておくと膨らむ。

なので、**本体には1行のポインタだけ置いて、詳細は `docs/` に逃がしています**。

```markdown
- Full unit-test pattern checklist, golden tests, and the designing-for-testability
  rules → `docs/testing.md`. Read before writing or refactoring tests.
- Windows-only traps (`fs.watch`, `path.resolve`) → `docs/windows-gotchas.md`
  — MUST read before debugging a Windows failure.
```

📎 [`isamu/claude CLAUDE.md#L168-L169`](https://github.com/isamu/claude/blob/273144a8bf83ff5ea2b92d78ae908cdb2ab4b541/CLAUDE.md#L168-L169)

いま置いてあるのはこの5つです。

| ファイル | 中身 |
|---|---|
| [`docs/testing.md`](https://github.com/isamu/claude/blob/273144a8bf83ff5ea2b92d78ae908cdb2ab4b541/docs/testing.md) | テストパターンの網羅リスト、テスタビリティのための設計 |
| [`docs/debugging-methodology.md`](https://github.com/isamu/claude/blob/273144a8bf83ff5ea2b92d78ae908cdb2ab4b541/docs/debugging-methodology.md) | バグの「家系図」マトリクス、ルールの横断掃除 |
| [`docs/windows-gotchas.md`](https://github.com/isamu/claude/blob/273144a8bf83ff5ea2b92d78ae908cdb2ab4b541/docs/windows-gotchas.md) | Windows 固有の罠 |
| [`docs/cross-platform-ci.md`](https://github.com/isamu/claude/blob/273144a8bf83ff5ea2b92d78ae908cdb2ab4b541/docs/cross-platform-ci.md) | 3 OS 対応の書き方 |
| [`docs/web-debugging.md`](https://github.com/isamu/claude/blob/273144a8bf83ff5ea2b92d78ae908cdb2ab4b541/docs/web-debugging.md) | ブラウザを手で操作するときの手順 |

ポイントは、ポインタに**「いつ読むか」を添える**ことです。`Read before writing tests` `MUST read before debugging a Windows failure`。こう書いておくと、エージェントは必要なときだけ深いほうを開きます。

常に全部を読ませようとすると、結局どれも効かなくなります。**規約は、量ではなく「踏む確率」で効きます。**

---

## ② ESLint — 「大きすぎる」「複雑すぎる」を機械が拒否する

これが、レビューを手放せた最大の理由かもしれません。

考えてみると、**コードレビューで僕が言いたくなることの大半は、機械が言えること**でした。長い。ネストが深い。引数が多い。分岐が多すぎる。

なので、全部 error にしました。

```js
{
  rules: {
    "max-lines-per-function": ["error", { max: 60, skipBlankLines: true, skipComments: true }],
    complexity: ["error", 20],
    "max-depth": ["error", 4],
    "max-params": ["warn", 6],
    "max-nested-callbacks": ["error", 4],
  },
}
```

📎 [`eslint.config.js#L77-L87`](https://github.com/receptron/mulmoterminal/blob/5e8252440ddb7cb5e4df94e9793935fada0d5d9a/eslint.config.js#L77-L87)

加えて `eslint-plugin-sonarjs` の **cognitive-complexity を error**（閾値15）にしています。循環的複雑度より、人間の読みにくさに近い指標です。

mulmoclaude 側はもう少し厳しく、`max-lines-per-function` が **50**、`complexity` が **15** です。

📎 [`mulmoclaude eslint.config.mjs#L270-L282`](https://github.com/receptron/mulmoclaude/blob/7773cd70324c3c9e4be3e143fbe4c2bd83b7c46a/eslint.config.mjs#L270-L282)

### 大事なのは「例外の作法」

厳しくすると、必ず正当な例外が出ます。ここで `// eslint-disable` を許すと、規約は死にます。

なので、**例外はインラインで消さず、設定ファイルに理由付きで置く**ようにしています。

mulmoterminal では、Vue コンポーネントの `<style>` ブロックを ESLint で禁止しています（スタイルは Tailwind ユーティリティで書く）。その上で、どうしても書けないものだけを許可リストに入れています。

```js
{
  // Scoped-CSS allowlist. Each entry is something Tailwind utilities cannot express;
  // keep the reason current, and delete the entry when the reason goes away.
  files: [
    "src/components/Sidebar.vue",       // @keyframes — the "thinking" spinner ring
    "src/components/GuiPanel.vue",      // `.frame + .frame` sibling-combinator spacing
    "src/components/FilesOverlay.vue",  // :deep into CodeMirror's injected root
    // ...
  ],
  rules: { "vue/no-restricted-block": "off" },
}
```

📎 [`eslint.config.js#L43-L57`](https://github.com/receptron/mulmoterminal/blob/5e8252440ddb7cb5e4df94e9793935fada0d5d9a/eslint.config.js#L43-L57)

**1行につき1つの理由**が書いてあります。そして「理由が消えたらエントリも消せ」と書いてある。

こうしておくと、半年後に見返したときに「これはまだ必要か？」を判断できます。インラインの `eslint-disable` には、これができません。

同じ思想で、`max-params` だけは `warn` のままにしてあります。設定ファイルにその理由が書いてあります。

```js
// All ERRORS except max-params, which stays WARN for its one intentional offender:
// spawnClaudePty's 7 params (hot path, not worth churning 5 call sites into an
// options object) — flip it to error once resolved.
```

📎 [`eslint.config.js#L77-L80`](https://github.com/receptron/mulmoterminal/blob/5e8252440ddb7cb5e4df94e9793935fada0d5d9a/eslint.config.js#L77-L80)

「今は妥協している。理由はこれ。解消したら error にする」——**妥協にも期限と条件を書く**。

---

## ③ テスト — できる限り pure 関数にして、エッジケースを潰す

数字から出します。

| | mulmoterminal | mulmoclaude |
|---|---|---|
| spec ファイル | **294** | **799** |
| テストケース | **3,373** | — |
| ソースファイル | 407 | — |

ソース1ファイルあたり、だいたい8ケース。多いように見えますが、**エージェントが書いているので、書くコスト自体は安い**のです。高いのは「壊れたことに気づかないコスト」のほうです。

### 設計のほうが本質

テストを増やすより先にやったのは、**テストできる形にする**ことでした。

ルール（フィルタ、並び順、上限、検証、整形、保持期間）は、**それ専用のファイルの pure 関数**に切り出す。ファイル読み書き、プロセス起動、ソケット、HTTP は呼び出し側に残す。

なぜなら、**アプリを起動しないと到達できないルールは、テストされない**からです。

純度を保てない境界では、依存を引数で渡します。`now()`、`isValidId`、`hasTmux`、ホームディレクトリのパス。import 時に時計やプロセスを掴むモジュールは、開発者のマシンに触らずにテストできません。これは「テストしにくい」ではなく**設計の欠陥**だと考えるようにしました。

### カバーするケース

エージェントに書かせるとき、パターンを明示しています。

- 正常系
- エッジケース（変だが妥当な入力）
- コーナーケース（エッジが複数重なる）
- 境界値（最小/最大、off-by-one）
- 空（空文字列、空配列、空オブジェクト）
- null / undefined
- 不正入力（型違い、壊れたデータ）
- エラー（例外が飛ぶべきところ）
- 否定（起きてはいけないこと）
- リグレッション（過去に出たバグ）

### そして、テストが本当に落ちるか確かめる

これは最近やっと習慣になったことですが、**新しいテストは、対象を壊したときに赤くなるかを確認**しています。条件を反転させる、ガードを消す、修正を revert する。赤くなるのを見て、戻す。

壊れたコードでも通るテストは、何か別のものをテストしています。そしてこれは、確認しない限り**永遠に見えません**。

### 何を先にテストするか

優先順位は「**どれだけ静かに失敗するか**」で決めています。

例外を投げる関数は、自分で報告してくれます。怖いのは、**それっぽい間違った値を返す**もののほうです。1つずれたインデックス。バイト数と文字数の取り違え。ずれた日付。大文字小文字が違う拡張子。ゆるすぎるバリデータ。プロトタイプチェーンを読んでしまう検索。

これらはユーザーが気づくまで見えません。だから先に潰します。

---

## ④ DRY — 重複は「気をつける」ではなく機械が数える

重複はメンテコストに直結します。同じロジックが3箇所にあると、修正が3箇所必要で、たいてい2箇所しか直りません。

なので、CI に2本のスキャンを足しています。

- **duplication-scan**（[jscpd](https://github.com/receptron/mulmoterminal/blob/5e8252440ddb7cb5e4df94e9793935fada0d5d9a/.github/workflows/duplication-scan.yaml#L45-L58)）— コピペの検出
- **dead-code-scan**（[knip](https://github.com/receptron/mulmoterminal/blob/5e8252440ddb7cb5e4df94e9793935fada0d5d9a/.github/workflows/dead-code-scan.yaml)）— どこからも import されていない export、誰も呼ばなくなったヘルパー

面白いのは、**どちらもビルドを落とさない**ようにしてあることです。しかも、方法が違います。

- **dead-code-scan** は明示的に `continue-on-error: true`
- **duplication-scan** は閾値を設定せずに走らせ、結果を SARIF で **Code Scanning に上げる**（＝アラートとして残るが、ジョブは失敗しない）

理由は workflow のコメントに書いてあります。

```yaml
# Reports only — this workflow NEVER fails the build (`continue-on-error`), mirroring
# duplication-scan.yaml:
#   1. knip can't infer every entry point (CLI subcommands spawned via tsx, codegen),
#      so a few false positives are expected. Blocking on day one would just train
#      people to add `// knip-ignore` reflexively.
#   2. knip has no base-branch diffing, so the report is the FULL current inventory,
#      not this PR's delta. It's a review aid, not a gate.
```

📎 [`dead-code-scan.yaml#L9-L19`](https://github.com/receptron/mulmoterminal/blob/5e8252440ddb7cb5e4df94e9793935fada0d5d9a/.github/workflows/dead-code-scan.yaml#L9-L19) ／ [`continue-on-error は L48`](https://github.com/receptron/mulmoterminal/blob/5e8252440ddb7cb5e4df94e9793935fada0d5d9a/.github/workflows/dead-code-scan.yaml#L48)

**初日からブロックすると、無視の作法が育つ**。これは ESLint の例外の話とまったく同じ構造です。

ゲートにするか、レポートにするか。この判断を間違えると、せっかく入れた仕組みが「みんなが黙って回避するもの」になります。

ちなみに、ESLint の `no-unused-vars` はファイル**内**しか見ません。重複スキャンはコピペしか見ません。**どこからも呼ばれなくなった export** は、そのどちらにも引っかからない——リファクタが最後の呼び出し元を消したときに残る、典型的な孤児です。knip はその隙間を埋めるために入れました。

---

## ⑤ CI を mac / Linux / Windows で回す（Windows は daily）

構成はこうなっています。

- **PR ごと**: ubuntu-latest + macOS-latest → [`ci.yml#L10-L18`](https://github.com/receptron/mulmoterminal/blob/5e8252440ddb7cb5e4df94e9793935fada0d5d9a/.github/workflows/ci.yml#L10-L18)
- **Windows**: 毎日 03:00 JST（＋ main への push）→ [`windows-daily.yaml#L14-L21`](https://github.com/receptron/mulmoterminal/blob/5e8252440ddb7cb5e4df94e9793935fada0d5d9a/.github/workflows/windows-daily.yaml#L14-L21)

### なぜ macOS を PR 毎に回すのか

冗長性のためではありません。**Docker サンドボックスが darwin 限定の機能**なので、Keychain からの認証情報のエクスポート、マウント構築といったコードパスが本当に走るのは macOS ランナーだけだからです。

その理由も、matrix のすぐ上にコメントとして置いてあります。

```yaml
# ubuntu + macOS on every PR. macOS matters here beyond redundancy: the Docker sandbox
# is darwin-gated (sandboxPlatformSupported), so its code paths — Keychain credential
# export/refresh, mount building — only run for real on a macOS runner.
```

📎 [`ci.yml#L10-L18`](https://github.com/receptron/mulmoterminal/blob/5e8252440ddb7cb5e4df94e9793935fada0d5d9a/.github/workflows/ci.yml#L10-L18)

### なぜ Windows を PR から外したのか

正直に言うと、**遅いからです**。NTFS の tar 展開、Defender、2回目の tsc。全 PR がその時間を払っても、得られるものが釣り合わない。

その代わり、**Windows が何を守っているかを workflow の先頭に書いてあります**。

```yaml
# What it actually guards: server/infra/pty-env.ts branches on PATH casing ("Path"),
# the ";" delimiter and "\" separators; server/infra/resolve-bin.ts resolves a bare
# command name to an .exe on PATH before node-pty sees it (#794), and its spec spawns
# a real PTY here; server/index.ts spawns powershell.exe; and the postinstall runs on
# every install. None of that had ever executed on Windows before this workflow.
```

📎 [`windows-daily.yaml#L1-L16`](https://github.com/receptron/mulmoterminal/blob/5e8252440ddb7cb5e4df94e9793935fada0d5d9a/.github/workflows/windows-daily.yaml#L1-L16)

「何を守っているか書けるなら、頻度は落としていい」。逆に、**書けないゲートは、たぶん要らないか、要るのに守れていないか、どちらかです**。

Windows は本当に固有の罠が多くて、`fs.watch` が 8.3 短縮パス（`C:\Users\RUNNER~1\…`）でプロセスを丸ごと abort させる（catch できない）とか、`path.resolve("/etc")` が `C:\etc` になって POSIX のパス一覧が静かに全部マッチしなくなるとか、そういう類です。これは daily でも回しておかないと、リリース直前に発見することになります。

---

## ⑥ Claude Code で実装 → Codex がレビュー → OK が出るまで繰り返す

そして、レビュー。

**実装したのと別の AI にレビューさせています。**

mulmoterminal / mulmoclaude では、GitHub Actions で全 PR に Codex の自動レビューが走ります。判定は決まったマーカーで返ってきます。

```
CODEX VERDICT: LGTM
CODEX VERDICT: CHANGES REQUESTED
```

📎 [`codex_review.yaml#L215-L220`](https://github.com/receptron/mulmoterminal/blob/5e8252440ddb7cb5e4df94e9793935fada0d5d9a/.github/workflows/codex_review.yaml#L215-L220)

さらに **CodeRabbit** と **Sourcery** も並走しているので、1つの PR に**3体のレビュアー**がぶら下がります。

設定で意図的にゆるくしているところがあります。

```yaml
# Auto-fire on EVERY PR — including drafts and docs-only diffs.
# Cost is bounded by concurrency.cancel-in-progress (a push burst collapses to one
# review against the final commit) and the Azure deployment SKU's TPM cap.
```

📎 [`codex_review.yaml#L34-L40`](https://github.com/receptron/mulmoterminal/blob/5e8252440ddb7cb5e4df94e9793935fada0d5d9a/.github/workflows/codex_review.yaml#L34-L40)

draft でも docs だけの diff でも走らせています。「レビューするかどうかを判断するコスト」のほうが、レビュー自体のコストより高いからです。連続 push は `concurrency` で最新コミットの1回に畳まれます。

### ここで一番大事なこと

**ボットの指摘を機械的に全部適用してはいけません。**

3体は普通に矛盾します。CodeRabbit が「こう書け」と言い、Sourcery が逆を言う。両方を満たそうとすると、誰も読めないコードができあがります。

なので運用としては、指摘を分類します。

- **本物の修正** → 直す（＋テストを足す）
- **妥当な nitpick** → 安ければ直す、そうでなければ「意図的」と返す
- **誤検知 / 古い情報** → 確認して、理由を書いて飛ばす

そして最後に、**何を直して何を意図的に飛ばしたかを PR にコメントする**。こうしておくと、人間が後からボットのスレッドを全部読み直さずに済みます。

「OK が出るまで繰り返す」というのは、つまりこの分類を繰り返すということです。

### そのループ自体をスキルにする

とはいえ、この繰り返しを手でやると疲れます。なので**ループそのものをスキルとして書いて**あります（[公開リポジトリ](https://github.com/isamu/claude)の `skills/` 配下です）。

- **[`codex-cross-review`](https://github.com/isamu/claude/blob/273144a8bf83ff5ea2b92d78ae908cdb2ab4b541/skills/codex-cross-review/SKILL.md)** — ローカルで `codex exec` を回して指摘を受け取り、こちらが評価して直し、再レビューを要求する。**最大5イテレーションの安全キャップ**付きで、イテレーションごとに状態ファイルを残します。ループが空回りしたとき、後から「何を言われて何を直したか」を追えるようにするためです
- **[`gh-review-loop`](https://github.com/isamu/claude/blob/273144a8bf83ff5ea2b92d78ae908cdb2ab4b541/skills/gh-review-loop/SKILL.md)** — GitHub 側のボット（Codex の Actions / CodeRabbit / Sourcery）が**最新コミットに**出したものを読み、直し、push し、再レビューを待つ。`gh pr view` では出てこない**インラインのスレッドまで読む**のがポイントです

そして、マージ条件を明示してあります。

> **全ボットが sign off ＋ CI がグリーン ＋ 人間が確認**

3つ目を残しているのが大事だと思っています。ここまで自動化しておいて何ですが、**最後のボタンまで機械に渡すと、本当に誰も見ていない状態**になります。僕が読まなくなったのは diff であって、マージの責任ではありません。

---

## 人間が残った場所

ここまで自動化して、僕が手を動かしているのはここだけになりました。

- **UI**（見ないと分からない）
- **複雑なもの**（仕様そのものが怪しいとき）
- **人間がテストするしかないもの**（体験、手触り、通知のタイミング）

言い換えると、僕の仕事は「**読む**」から「**見る**」に変わりました。diff を読むのではなく、動いている画面を見る。

だからでしょうか、最近は「GUI は操作するものではなく、**見るもの**になっていく」と本気で考えるようになりました。エージェントが「やること」を引き受けるほど、画面に残る役割は結果の提示に寄っていく。この話は長くなるので、また別の記事で。

---

## やっていないことのほうが、本質だった

冒頭に戻ります。

僕がレビューをやめられたのは、**頑張ったからではありません**。むしろ逆で、頑張らなくても壊れないようにしたからです。

そして念のため書いておくと、超人的なエンジニアになったわけでもありません。手を動かす量が増えたわけでもない。増えたのは、**壊れたときに勝手に赤くなるものの数**だけです。

変わったのは、ボトルネックの場所です。

- 昔: **書く速度**がボトルネックだった
- 少し前: **判断する速度**がボトルネックになった
- 今: **統合する速度**（CI、レビュー、マージ、コンフリクト）がボトルネック

そしてこの移動は、道具を整えた分だけ進みました。並列で回せる数の上限は、注意力ではなく、**道具がどれだけ状態を持ってくれるか**で決まる。同じことが品質にも言えて、**読まなくて済む量は、壊れたときに赤くなる仕組みの量で決まります**。

レビューをやめられたのではありません。**レビューを、機械に移した**だけです。

---

この記事で紹介したものは、全部この3つのリポジトリで動いています。そのまま持っていってください。

- [isamu/claude](https://github.com/isamu/claude) — 全プロジェクト共通の `CLAUDE.md` / `docs/` / レビュー用の `skills/`
- [receptron/mulmoterminal](https://github.com/receptron/mulmoterminal) — `eslint.config.js` / `.github/workflows/` / リポジトリ側の `CLAUDE.md`
- [receptron/mulmoclaude](https://github.com/receptron/mulmoclaude) — `eslint.config.mjs` / `.github/workflows/`

:::message
本文中の 📎 リンクは、**2026年7月26日時点のコミットに固定**してあります。設定は変わっていくので、最新を見たいときはリポジトリのデフォルトブランチをどうぞ。

- mulmoterminal: [`5e82524`](https://github.com/receptron/mulmoterminal/tree/5e8252440ddb7cb5e4df94e9793935fada0d5d9a)
- mulmoclaude: [`7773cd7`](https://github.com/receptron/mulmoclaude/tree/7773cd70324c3c9e4be3e143fbe4c2bd83b7c46a)
- isamu/claude: [`273144a`](https://github.com/isamu/claude/tree/273144a8bf83ff5ea2b92d78ae908cdb2ab4b541)
:::

最後に聞いてみたいのですが——**みなさんは、自分の書いた（書かせた）コードを、まだ全部読んでいますか？**

読んでいないなら、代わりに何が守ってくれていますか。読んでいるなら、いつまで読めそうですか。そこがいま、一番知りたいところです。
