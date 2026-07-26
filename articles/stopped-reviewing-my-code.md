---
title: "1日500コミットは、もう読めない ── だからコードレビューをやめた"
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

エージェントを並列で回すようになって、コミットは1日500を超えました。この数字自体は、正直かなり気に入っています。

ただ、この記事はその自慢ではなく、**その量を自分が読めなくなった後の話**です。読む時間が足りなくなった人間が、読まずに済ませるために何をしたか。動機としては、わりと後ろ向きなところから始まっています。

## 前提：人間のレビューはスケールしない

少し前、[ターミナルを自作したら、1日のコミット数が500を超えて、生産性がバグった話](https://zenn.dev/singularity/articles/diy-terminal-500-commits)という記事を書きました。この記事は、その続きにあたります。

並列でエージェントを走らせると、書かれるコードの量が人間の読める速度を軽く超えます。

問題は量そのものではありません。**量に対して、自分の読む速度が追いつかなくなった**ことでした。

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

### sonarjs も入れて、かなり厳しめにする

サイズと複雑さだけでは足りません。両リポジトリとも、`eslint-plugin-sonarjs` と `eslint-plugin-security` を **recommended ごと**入れています。

```js
export default [
  js.configs.recommended,
  ...tseslint.configs.strict,        // ← recommended ではなく strict
  ...pluginVue.configs["flat/recommended"],
  sonarjs.configs.recommended,
  security.configs.recommended,
  // ...
];
```

📎 [`eslint.config.js#L9-L15`](https://github.com/receptron/mulmoterminal/blob/5e8252440ddb7cb5e4df94e9793935fada0d5d9a/eslint.config.js#L9-L15)

その上で、個別にさらに締めています。

| ルール | 何を捕まえるか |
|---|---|
| [`sonarjs/cognitive-complexity`](https://github.com/receptron/mulmoclaude/blob/7773cd70324c3c9e4be3e143fbe4c2bd83b7c46a/eslint.config.mjs#L303) | 循環的複雑度より「人間の読みにくさ」に近い指標 |
| [`sonarjs/no-ignored-exceptions`](https://github.com/receptron/mulmoclaude/blob/7773cd70324c3c9e4be3e143fbe4c2bd83b7c46a/eslint.config.mjs#L296) | catch して握りつぶしている箇所 |
| [`sonarjs/assertions-in-tests`](https://github.com/receptron/mulmoclaude/blob/7773cd70324c3c9e4be3e143fbe4c2bd83b7c46a/eslint.config.mjs#L321) | **アサーションのないテスト** |

最後のものが個人的に好きです。設定ファイルにこう書いてあります。

> assertion-less test can't slip in unnoticed — reviewers caught them by eye before, CI enforces it now.

**「今までは人間が目で見つけていたものを、CI がやる」。** この記事で言いたいことが、この1行に詰まっています。

外すルールにも理由を書きます。mulmoterminal では `sonarjs/no-os-command-from-path` を `bin/` でだけ切っています。**ユーザーがインストールした CLI（claude / gh / tmux / codex / git）を PATH から起動するのがこのツールの仕事**なので、このルールは前提そのものと喧嘩するからです。

### 型を、もう一つの規約として使う

そして型です。ここが一番大事だと思っています。

`CLAUDE.md` が自然言語で書いた規約なら、**型は機械が強制する規約**です。そして僕が diff を読まなくなった以上、**読む役を型にやってもらう**しかありません。

なので、逃げ道を塞いでいます。

| ルール | 設定 |
|---|---|
| [`@typescript-eslint/no-explicit-any`](https://github.com/receptron/mulmoclaude/blob/7773cd70324c3c9e4be3e143fbe4c2bd83b7c46a/eslint.config.mjs#L193) | error |
| [`@typescript-eslint/no-non-null-assertion`](https://github.com/receptron/mulmoclaude/blob/7773cd70324c3c9e4be3e143fbe4c2bd83b7c46a/eslint.config.mjs#L213) | error |
| [`@typescript-eslint/consistent-type-assertions`](https://github.com/receptron/mulmoclaude/blob/7773cd70324c3c9e4be3e143fbe4c2bd83b7c46a/eslint.config.mjs#L258) | error |

加えて、グローバルの `CLAUDE.md` 側にこう書いてあります。

- `as` によるキャストは使わない。**型ガード**（`const isXxx = (x: unknown): x is Type => ...`）を書く
- Zod スキーマからは `z.infer<typeof schema>` で導出する。**同じ型を二重に定義しない**
- `any` は使わない
- lint / 型エラーを `eslint-disable` / `@ts-ignore` / `@ts-expect-error` で黙らせない。**根本を直す**

最後の1行がかなり効きます。エージェントは、放っておくと**エラーを消す最短経路**を選びます。`@ts-ignore` は最短経路です。禁止しておかないと、**型が守ってくれる範囲が静かに減っていきます**。

### 型情報を使う lint は「コストに見合うものだけ」

型を本気で使うなら、`projectService` を有効にして **TypeScript のプログラムを丸ごと構築する** lint パスが要ります。ただし重い。設定ファイルに実測が書いてあります。

> that program is the whole cost (measured: five rules cost the same as all 44), and it runs the full scope in ~26s over the untyped pass.

**5ルールでも44ルールでもコストは同じ**。プログラムの構築が全部で、ルールの数はほぼ関係ない。

なのに全部は入れていません。理由もそこに書いてあります。

> The rest of `strictTypeChecked` ... is dominated by style — `restrict-template-expressions` alone accounts for 439 of its 1213 findings — which would bury the two things type information is actually needed for here

1213件のうち439件が、たった1つのスタイルルール。それに埋もれさせないために、**型情報でなければ絶対に捕まえられない2種類**に絞っています。

1. `no-explicit-any` では構造的に見えない `any` —— 型のないライブラリからの値、`JSON.parse()`、`as unknown as T` の二段キャスト
2. 構文ルールでは原理的に捕まえられない間違い —— **`await` の付け忘れ**、同期専用 API に渡された async コールバック、`"[object Object]"` になった文字列化

そしてこの4つは、**バックログをゼロにしてから error に上げて**あります。

```js
"@typescript-eslint/no-base-to-string": "error",
"@typescript-eslint/no-floating-promises": "error",
"@typescript-eslint/no-misused-promises": "error",
"@typescript-eslint/await-thenable": "error",
```

> Drained to zero and ratcheted to `error` — a dropped `await` or an async callback handed to a sync-only API is a real bug, and the backlog for these three is empty, so there is nothing to grandfather. **Re-introducing one now fails CI instead of joining a warning list nobody reads.**

📎 [`mulmoclaude eslint.config.mjs#L525-L575`](https://github.com/receptron/mulmoclaude/blob/7773cd70324c3c9e4be3e143fbe4c2bd83b7c46a/eslint.config.mjs#L525-L575) ／ 運用方針は [`docs/lint-policy.md`](https://github.com/receptron/mulmoclaude/blob/7773cd70324c3c9e4be3e143fbe4c2bd83b7c46a/docs/lint-policy.md)

**「誰も読まない警告リストに1件足す」を許さない。** ゼロまで削ってから error に上げる。この「drain してから ratchet する」の考え方は、後で出てくる重複スキャンの話（初日からブロックすると回避の作法が育つ）とちょうど裏表になっています。

### 人間なら耐えられない厳しさが、AI なら成立する

ここまで並べておいて何ですが、**この設定は人間のチームなら3日で緩和 PR が飛んできます**。

関数は60行まで。ネストは4段まで。`any` 禁止。非 null アサーション禁止。キャスト禁止。テストにはアサーション必須。握りつぶした catch は CI が落とす。

人間が書いていたら、「いちいちうるさい」「今それは本質じゃない」となります。実際、昔の僕もそう思っていました。

でも、**書いているのが AI なら文句を言いません**。指摘されたら直すだけです。

これは地味に大きな変化だと思っています。これまで lint の厳しさは、「機械の正しさ」と「開発者の忍耐」のトレードオフでした。**緩めるという判断は、技術的な判断ではなく社会的な判断だった。**

その社会的コストがゼロになったなら、天秤は片側にしか傾きません。

しかも僕にとって、これは贅沢ではなく必需品です。**僕は diff を読んでいないので、うるさく言ってくれる存在がいないと困る。**

### だから、型のない言語をもう選びたくない

同じ理由で、言語の選び方も変わりました。

**静的解析が効かない言語を、なるべく使いたくない。**

型のない言語だと、この記事に書いたことの半分が成立しません。「読まなくても壊れない」の"読まなくても"を支えているのは、結局のところ**書いた瞬間に機械が読んでくれること**だからです。型がないと、その最初の読み手がいなくなる。

昔は「型は書くのが面倒」というコストが確かにありました。今は、**その面倒を払うのが僕ではありません。**

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

### そもそも、チーム全員が Mac

3 OS で回している一番の理由はこれかもしれません。**僕らのチームには Linux 使いも Windows 使いもいません。**

つまり、Linux / Windows の CI が**唯一の実機**です。手元にない環境のバグは、ローカルでは一生再現できません。

そして、これが効くのはリリース前だけではありません。**ユーザーからバグレポートが来たときに、そこで再現させられる。** 「Windows で動きません」という Issue に対して、「手元に Windows がないので分かりません」と返さずに済む。CI が、テスト環境であると同時に**再現環境**になっています。

これは正直、想定していなかった副産物でした。3 OS の CI は「品質のため」に入れたつもりだったのに、いま一番効いているのは**サポートのため**だったりします。

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

### Windows 専用のテストケースを書く

さらに、Windows のためのテストケースを個別に書いています。いま16個の spec ファイルが `win32` を意識したケースを持っています。

目的は「Windows で動くこと」だけではありません。むしろ**他をいじったときに Windows をデグレさせないこと**のほうが大きい。パス処理や区切り文字は、Linux 前提の"きれいなリファクタ"で簡単に壊れます。そして壊れても、**Mac の上では誰も気づきません**。

書き方にコツがあって、**プラットフォームを引数で渡す**ようにしています。

```ts
expect(namesAWindowsDevice("docs/NUL/readme.md", "win32")).toBe(true);
expect(namesAWindowsDevice("docs\\CON\\readme.md", "win32")).toBe(true);
```

（`NUL` や `CON` は Windows の予約デバイス名で、ファイル名に使えません）

関数の中で `process.platform` を見るのではなく引数で受け取ると、**Windows の挙動を Mac 上でテストできます**。daily の Windows CI を待たずに、手元で赤くできる。

📎 [`test/server/files/pathContainment.spec.ts#L163-L182`](https://github.com/receptron/mulmoterminal/blob/5e8252440ddb7cb5e4df94e9793935fada0d5d9a/test/server/files/pathContainment.spec.ts#L163-L182)

これは前に書いた「境界では依存を引数で渡す」の実例でもあります。時計、ホームディレクトリ、プロセス、そして**プラットフォーム**。テストしやすさは、だいたい引数の形で決まります。

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

……と書いてはあるのですが、正直に言うと、**最後の「人間が確認」は、最近ほとんど形骸化しています**。ボットが全部 OK を出して CI がグリーンなら、僕はだいたい押すだけです。

そして、**そこも全自動にしたいと思っています。**

怖くないのか、と言われれば、少しは怖い。でも考えてみると、僕が最後に押している「確認」の中身は、**すでに機械が出した結論をなぞっているだけ**です。なぞるだけの工程を人間に残すのは、安心のためであって品質のためではない。

それでもまだ人間を残しているのは、品質の問題ではなく**責任の所在**の問題だと思っています。だから全自動に進むなら、たぶん次にやるべきは「もっと賢いレビュー」ではなく、**「何かあったときに確実に戻せること」**——ロールバックと、後から追える記録——のほうです。そっちが今の宿題です。

### その先：判断そのものを学習させる

もうひとつ考えているのが、**判断の基準そのものを機械に渡す**ことです。

この記事ではずっと「機械に移した」と書いてきましたが、上でやっている分類——ボットの指摘を「本物の修正／妥当な nitpick／誤検知」に振り分けるところ——は、まだ僕がやっています。そしてこの判断は、**過去の自分の判断の積み重ね**でできています。だとしたら、それを学習させた"判断 AI"に寄せられるはずです。

レビューを機械に移し、マージを機械に移し、最後に**判断の基準**を移す。並べてみると、順番としてはこれが自然な気がしています。

---

## 人間が残った場所

ここまで自動化して、僕が手を動かしているのはここだけになりました。

- **UI**（見ないと分からない）
- **複雑なもの**（仕様そのものが怪しいとき）
- **人間がテストするしかないもの**（体験、手触り、通知のタイミング）

言い換えると、僕の仕事は「**読む**」から「**見る**」に変わりました。diff を読むのではなく、動いている画面を見る。

だからでしょうか、最近は「GUI は操作するものではなく、**見るもの**になっていく」と本気で考えるようになりました。エージェントが「やること」を引き受けるほど、画面に残る役割は結果の提示に寄っていく。この話は長くなるので、また別の記事で。

---

## この状態だと、普通のターミナルではもう無理です

ここまで書いてきた仕組みは、全部「**読まなくても壊れない**」ためのものでした。

でも、壊れないことと、**回せる**ことは別です。

エージェントを6本並列で走らせると、ターミナルは6枚になります。どれが終わったのか、どれが自分待ちなのか、6枚のスクロールを目でスキャンして探す。この探す時間は、セッション数に比例して増えます。そして **lint も型もテストも CI も、この問題を1ミリも解決してくれません**。

なので、そこは道具のほうを作りました。**[MulmoTerminal](https://github.com/receptron/mulmoterminal)** —— ブラウザで動くターミナルです。

```bash
npx mulmoterminal
# → http://localhost:34567 が開きます
```

これだけです。Node 22.9+ と `claude` CLI があれば、設定は要りません。インストールしたくなければ `npx` のままで構いません（常用するなら `npm install -g mulmoterminal`）。

何をしてくれるかというと:

- **グリッドで全セッションが一望できる。** セルの枠色が状態を表します —— 作業中 / 自分待ち / 完了
- **入力待ちで音が鳴る。** 画面外で詰まったやつが自分から呼んでくる
- **スマホに Web Push。** 散歩中でも、詰まった1本だけ捌ける
- **裏は tmux。** ブラウザを閉じても、サーバを再起動しても、セッションは生きたまま
- **セルごとに git worktree。** 同じリポジトリを複数のエージェントが同時に触っても衝突しない。グリッドから commit / push / PR まで
- **Claude Code と Codex の両対応**、モデルもセッション単位で選べます

正直に書くと、**この記事に書いた運用は、この道具なしでは回りません**。厳しい lint も、3 OS の CI も、レビューの自動ループも、「並列で走らせている」ことが前提になっているからです。逆に言えば、並列に回し始めた瞬間から、これらは全部必要になります。

MIT です。使い方は[日本語ガイド](https://receptron.github.io/mulmoterminal/guide/ja/)にまとめてあります。1コマンドで試せるので、**いま複数のターミナルを行ったり来たりしている人は、一度だけ試してみてください。**

---

## やっていないことのほうが、本質だった

冒頭に戻ります。

僕がレビューをやめられたのは、**頑張ったからではありません**。むしろ逆で、頑張らなくても壊れないようにしたからです。

超人的なエンジニアになったわけでも、手を動かす量が増えたわけでもありません。増えたのは、**壊れたときに勝手に赤くなるものの数**だけです。

変わったのは、ボトルネックの場所です。

- 昔: **書く速度**がボトルネックだった
- 少し前: **判断する速度**がボトルネックになった
- 今: **統合する速度**（CI、レビュー、マージ、コンフリクト）がボトルネック

そしてこの移動は、道具を整えた分だけ進みました。並列で回せる数の上限は、注意力ではなく、**道具がどれだけ状態を持ってくれるか**で決まる。同じことが品質にも言えて、**読まなくて済む量は、壊れたときに赤くなる仕組みの量で決まります**。

レビューをやめられたのではありません。**レビューを、機械に移した**だけです。

そして今やりたいのは、**自分の判断を AI に学習させて、その AI に判断させること**です。

ボットのどの指摘を直して、どれを「意図的」と返したか。どの PR をマージして、どれを差し戻したか。その履歴は、全部 GitHub に残っています。**過去の自分の判断のログ**です。

それを学習させれば、途中の判断——いま僕がやっている分類——は僕でなくてもよくなるはずです。そして最終的には、マージまで自動で回したい。

---

## で、楽になったのかというと

ここまで自動化して、では楽になったのか。

**逆です。忙しくなりました。**

思いついたことが、その日のうちに動くようになりました。動くと触ります。触るとまた思いつきます。**アイデアの回転が上がったぶん、やることが増えました。**

減ったのは「待っている時間」と「読んでいる時間」であって、仕事そのものではなかった。

機械に仕事を奪われるはずじゃなかったのかな、と思います。

奪われたのは、たぶん仕事ではなく**言い訳**のほうでした。「時間がないからできない」が使えなくなった。空いた手にアイデアが流れ込んできて、結局また埋まっている。

---

最後に聞いてみたいのですが——**みなさんは、自分の書いた（書かせた）コードを、まだ全部読んでいますか？**

読んでいないなら、代わりに何が守ってくれていますか。読んでいるなら、いつまで読めそうですか。そこがいま、一番知りたいところです。

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
