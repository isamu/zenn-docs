---
title: "ESLint と TypeScript の設定、抜けてないと言い切れますか"
emoji: "📏"
type: "tech"
topics: ["TypeScript", "ESLint", "ClaudeCode", "AI", "vibecoding"]
published: false
publication_name: "singularity"
---

自分の規約には、はっきりこう書いてあります。

```markdown
NEVER use `as` type casts; MUST use type guards instead
```

なのにコードを読んでいたら、`as` がありました。1つではなく、あちこちに。

「規約を守っていないな」と思って設定を見に行ったら、そもそも **ESLint で禁止していませんでした。**

ちなみに、これは自分だけの話ではないようです。r/ExperiencedDevs に、こういう相談が上がっていました（239 upvote / 203 コメント）。

**[Senior dev keeps type asserting everything in TypeScript – how do I approach this?](https://www.reddit.com/r/ExperiencedDevs/comments/1mi5iuk/senior_dev_keeps_type_asserting_everything_in/)**

> **シニア開発者がTypeScriptで何でも型アサーションしている – どう対処すればいい？**
>
> コードはAIツールのCursorで書かれたように見えることが多く、**コンパイラを喜ばせるためだけに `as` が多用されている**ことがあります。

上位の回答はほぼ全部同じでした。

> **あなたのリンターはこれを許可せず、CIビルドを失敗させるべきです。**

その通りです。**うちは規約に書いただけで、止めていませんでした。**

## なぜ抜けたか

言い訳ではなく、構造の話として書きます。

ESLint と TypeScript の設定は、**組み合わせが多すぎます。**

- ESLint 本体の設定（flat config になってから書き方が変わった）
- `typescript-eslint` のプリセット（`recommended` / `strict` / `stylistic`、それぞれに `TypeChecked` 版がある）
- プラグイン（`sonarjs`、`import`、`vue`…）とその推奨設定
- **`tsconfig` が複数**（app / server / test / node …）、しかも**継承する**
- Vue なら `vue-eslint-parser`、Node なら別の設定、モジュール解決の違い
- そして **Node のバージョン、ESLint のバージョン、TypeScript のバージョン**で、有効なものが変わる

**「プリセットを入れたから大丈夫」が成り立ちません。** どのプリセットが何を含むか、それが継承後にどうなるかは、**組み合わせごとに違います**。

僕の場合はこうでした。`typescript-eslint` の `strict` を入れていたので、型まわりは締まっていると思っていた。**`strict` に `consistent-type-assertions` は入っていません。** 知らなかったわけではなく、**確かめたことがなかった**。

## AI に任せられる。でも、確認は別

いまは設定もだいたい AI に書かせられます。実際そうしています。

ただ、**「入れて」と頼んだものが入ったかは、頼んだ側が確かめないと分かりません。** エージェントは「入れました」と言いますし、たいてい本当に入れています。問題は**入れていないことに誰も気づかない**ケースのほうです。

エラーが出ないので。lint も通るし、CI も緑です。**設定が抜けていることは、何も報告しません。**

（このあたり、チェックリストを作ろうと思っています。「新しいリポジトリで最初に確認する項目」みたいなやつを。）

以下は、実際に何を確かめて、何が出てきたかの記録です。

---

## まず `as` を止めた

`assertionStyle: "never"` を入れて、ESLint に数えさせました。**149箇所**。

grep では90箇所しか見えていませんでした。複数行にまたがるものや、思いつかなかった書き方を取りこぼしていたわけです。**「だいたい数える」では足りませんでした。**

そして直し始めたら、**バグが出ました。**

### キャストが `undefined` を黙らせていた

最初のファイルにこれがありました。

```ts
viewComponent: markdownPlugin.viewComponent as unknown as Component
```

Vue の型不一致だろうと思っていました。パッケージごとに別の Vue 型を持つ、よくあるやつです。

**違いました。** キャストを外すと、本当のエラーが出ます。

```
Type 'Component | undefined' is not assignable to type 'Component'
```

プラグインのプロトコル側が `viewComponent?: Component` と **optional で宣言**していました。View を持たないプラグインがあり得るので、仕様としてはそれで正しい。**キャストは、その `undefined` の可能性を黙らせていました。**

通れば `undefined` がレジストリに入り、「登録されているのに描けないツール」ができます。壊れ方は白い枠か描画時エラーで、**原因から遠い場所に出ます**。

顕在化はしていませんでした。対象パッケージが全部ちゃんと View を出荷していたので。**動いていないバグではなく、開いていた穴**です。

### 検査より先に型を主張していた

```ts
const mode: ViewWriteMode = body.mode === undefined ? "upsert" : (body.mode as ViewWriteMode);
if (!VIEW_WRITE_MODES.includes(mode)) { /* 400 */ }
```

**「これは有効なモードだ」と宣言した次の行で、「それは有効なモードか？」と聞いています。** 実行時の結果は同じですが、順序が逆でした。

### 同じ応答なのに、1フィールドだけ検査していた

外部から来る JSON の5フィールドのうち、**1つだけ `unknown` として扱い、残り4つは `string | null` と宣言して無検査で受けて**いました。コメントには「untrusted JSON だから unknown にしている」と書いてあるのに、隣の4つは信じている。

### そして、外すだけで消えるキャストが6件

```ts
CollectionCardView as Component
AccountingView as Component
```

**外しても型エラーが出ません。** `.vue` の SFC 型は最初から `Component` に代入できます。誰かが1つ書いて、次の人が真似して、以後誰も必要性を再検証しない。

**やっぱり型は重要でした。** キャストで黙らせていたのは、ちゃんと言いたいことがある側でした。

---

## 本家の型が壊れている場合だけ、cast を残した

全部消せたわけではありません。**allowlist に2件**残っています。どちらも**型が間違っているのが自分のコードではない**ケースです。

たとえばこれ。

```ts
// @modelcontextprotocol/sdk@1.30.0 writes `class StreamableHTTPServerTransport implements Transport`,
// yet types that class's onclose/onerror/onmessage accessors `T | undefined` while Transport spells
// them `?: T`. Under exactOptionalPropertyTypes the class therefore fails the interface it claims;
// the sibling WebStandardStreamableHTTPServerTransport declares them correctly, which is what makes
// this a declaration bug rather than a real mismatch. Upstream issue (open, names this exact
// workaround): https://github.com/modelcontextprotocol/typescript-sdk/issues/2083
await server.connect(transport as unknown as Transport);
```

`implements Transport` と書いてあるクラスが、その `Transport` を満たしていません。**兄弟クラスは正しく宣言されている**ので、実装の問題ではなく**宣言のバグ**です。

こういうものは、こちらでどう書いても直りません。なので:

1. **本家に issue を立てる**（[typescript-sdk#2083](https://github.com/modelcontextprotocol/typescript-sdk/issues/2083)。この回避策そのものが記載されています）
2. **allowlist に入れて、理由と「いつ消せるか」を書く**
3. **インラインの `eslint-disable` は使わない**

3つ目が大事だと思っています。インラインの disable は**現場に埋もれて見えなくなります**。設定ファイルの allowlist なら、**増えたら気持ち悪い**。

もう1件も同じ理由（プラグインのプロトコル側のジェネリクスが、ホスト側で証明不可能な形になっている）です。そして allowlist の先頭にはこう書いてあります。

> Nothing here is "we could not be bothered": a host-side fix was written and merged for the one case that had one.

**直せるものは、向こうを直しました。** 残っているのは、こちらでは直しようがない2件だけです。

---

## で、他の設定も見直した

`as` の一件で「設定は抜けるものだ」と分かったので、**他も測ることにしました。**

### やり方：ファイルを読まない。実効値を出す

設定ファイルを読んでも分かりません。プリセットが継承されるからです。**最終的に何が有効か**を出力させます。

```bash
npx eslint --print-config src/main.ts
```

型まわりだけ抜き出した結果です。

| ルール | 状態 |
|---|---|
| `no-explicit-any` | 🔴 error |
| `no-non-null-assertion` | 🔴 error |
| `consistent-type-assertions` | 🔴 error（今回入れた） |
| **`no-unsafe-assignment`** | ❌ **未設定** |
| **`no-unsafe-member-access`** | ❌ **未設定** |
| **`no-floating-promises`** | ❌ **未設定** |
| **`no-misused-promises`** | ❌ **未設定** |
| **`await-thenable`** | ❌ **未設定** |

未設定のものに共通点があります。**全部、型情報を要するルール**です。

つまり**型情報を使うルールが1つも入っていませんでした。**

### これは `as` を消しても残る穴

`no-explicit-any` は入っています。だから `any` は止まっていると思っていました。

**止まっていません。** このルールは **`any` という字が書かれている場所しか見られない**からです。

- **型定義のないライブラリ**から流れ込む値
- **`JSON.parse()` の戻り値**
- `as unknown as T` を消しても、**消した先が `any` なら同じ**

そして構文ルールでは**原理的に**捕まらないものがあります。**`await` の付け忘れ**は、構文としては完全に正しい。

入れて数えたら **165件**、所要 **11秒**でした。

| ルール | 件数 |
|---|---|
| `no-unsafe-member-access` | 65 |
| `no-unsafe-assignment` | 49 |
| **`no-floating-promises`** | **24** |
| `no-unsafe-argument` | 14 |
| その他 | 13 |

`no-unsafe-*` の139件は「型が付いていない」で、必ずしもバグではありません。**`no-floating-promises` の24件**が問題です。Promise を作って `await` も `.catch()` も付けていないので、**握り潰された失敗が24箇所ある可能性**があります。

### ちなみに「重いから入れていない」ではありませんでした

そう思っていたのですが、姉妹プロジェクトの設定に当時の判断が書き残してありました。

> that program is the whole cost (**measured: five rules cost the same as all 44**)

**コストは型プログラムの構築に付きます。** 5個に絞っても44個入れても同じ時間です。**絞っても速くなりません。**

絞った理由は**出力の量**でした。`strictTypeChecked` を丸ごと入れると1,213件出て、**そのうち439件が1ルールのスタイル指摘**。本当に見たい2つが埋もれます。

**「全部入れる」と「入れない」の間に「選んで入れる」がある。** ただしその選択は、一度全部動かして出力を見ないとできません。

## そして、もっと厄介なものが見つかった

作業のついでに、非 null アサーション（`!`）も調べました。こちらは `tseslint.configs.strict` 経由で **すでに error として有効**です。

なのに `src/components/GuiPanel.vue` に**3箇所生き残っていました**。

確かめると、こうなります。

| 場所 | 結果 |
|---|---|
| `.ts` に `m.get("x")!.a` を書く | `error  Forbidden non-null assertion` |
| **Vue の `<template>` 内**に同じものを書く | **何も報告されない** |

**Vue のテンプレート内の式は、typescript-eslint のルールが走りません。** `vue-eslint-parser` は `<script>` を typescript-eslint に渡しますが、テンプレートの式は同じようには扱われないためです。

つまりこうなっていました。

- リポジトリ全体で `!` が**この3つしか無い**のは、ルールがちゃんと効いている証拠
- **そしてその3つは、ちょうどルールが見られない場所にある**

偶然ではありません。**止められる場所では止まっていて、止められない場所に溜まっていた。** 水が低いところに流れるのと同じです。

そして同じことが、今回入れた `as` のルールにも当てはまります。**テンプレートに書けば、すり抜けます。**

いま、テンプレート内に `as` はありません。でも「`error` にしたから安全」ではない。**ルールが見ていない場所がある**、という認識のほうが大事でした。

> **lint が通ったことは、lint が見たことを意味しません。**

## 他のフレームワークは？ 全部試しました

「うちは React だけど」と思った方のために、実際に測りました。**同じコードを各フレームワークで書いて、ESLint にかけたもの**です。

```ts
// どれも同じ内容。script 側とテンプレート側の両方に置いた
const bad = m.get("x")!.a;              // ← script / 関数本体
{ m.get("y")!.a }                        // ← テンプレート / JSX
{ ({} as { q: number }).q }              // ← テンプレート / JSX
```

結果です。

| | script 側 | **テンプレート / JSX 側** |
|---|---|---|
| **React**（`.tsx`） | ✅ 検出 | ✅ **検出** |
| **Solid / Preact**（`.tsx`） | ✅ 検出 | ✅ **検出** |
| **Svelte**（`.svelte`） | ✅ 検出 | ✅ **検出** |
| **Astro**（`.astro`） | ✅ 検出 | ✅ **検出** |
| **Vue**（`.vue`） | ✅ 検出 | ❌ **素通り** |

**Vue だけでした。**

正直に言うと、これは意外でした。「テンプレート言語は全部同じ穴があるだろう」と思って測り始めたのですが、**Svelte も Astro もちゃんと見ています**。

### なぜ Vue だけなのか

パーサーの構造の違いです。

- **React / Solid / Preact** — JSX は **TypeScript の文法の一部**です。パーサーが1つの木にするので、JSX の中の式もそのまま TypeScript の式として扱われます
- **Svelte / Astro** — テンプレート言語ですが、パーサーがマークアップ内の式を **TypeScript の式として木に組み込みます**
- **Vue** — `vue-eslint-parser` は `<script>` を typescript-eslint のパーサーに渡しますが、`<template>` の式は別扱いになり、型情報を要するルールが走りません

つまり **Vue の実装上の選択**であって、テンプレート言語だから避けられない、というものではありませんでした。

### Vue を使っているなら

- `eslint-plugin-vue` の**テンプレート専用ルール**は効きます（`vue/no-...` 系）。効かないのは **typescript-eslint 側**のルール
- **テンプレートに式を書きすぎない**のが現実的な回避策です。ロジックを `<script>` の computed に出せば、そこは見られます
- ちなみにうちの3箇所も、`v-if` のナローイングが**兄弟の属性に持ち越せない**せいで、テンプレート内で `!` を書く形になっていました。**computed に出すのが正解**です

## 「厳しい戦い」は、もう発生しない

Reddit のスレッドで、僕が一番おもしろいと思ったのはこのコメントでした。

> TypeScriptの最大の問題は、提供されるすべての安全でない逃避ハッチです。**チーム全体でそれらを決して使わないことに合意するのは、しばしば厳しい戦いです。** リンタールールは役立ちますが、チーム全体でそのリンタールールに合意させる必要があります。

「合意させる」。ここです。

同じスレッドの他の回答も、ほとんどが**政治の話**でした。「マネージャーに報告して承認を得る」「RFC プロセスで投票にかける」「バグを追跡して証拠を集める」「1on1 で話す」。

技術的には答えが出ているのに、**入れられない**。厳しい戦いだから。

前回の記事で、僕はこう書きました。

> lint の厳しさは、ずっと「機械の正しさ」と「開発者の忍耐」のトレードオフでした。緩めるという判断は、技術的な判断ではなく社会的な判断だった。**その社会的コストがゼロになったなら、天秤は片側にしか傾きません。**

このスレッドは、その「社会的コスト」が**実在することの証拠**でした。203 件のコメントのうち、かなりの部分が「どうやって同僚を説得するか」に費やされています。

**エージェントは説得が要りません。** ルールを書けば、次から守ります。文句も言わないし、来週になって緩和 PR を出してくることもない。

僕が90箇所も溜めてしまったのは、**説得の相手がいないのに、ルールを機械に入れる作業を後回しにしていた**からでした。一番安かった作業を、やっていなかった。

## 型は、もう人間が書いてメンテするものではない

ここまで書いて、たどり着いた結論です。

TypeScript の型を、人間が書いて、人間がレビューして、人間がメンテナンスする。この前提はもう成り立ちません。**量が違います。**

だからといって「型はいらない」わけではなく、むしろ逆です。**AI が書くからこそ、型は要ります。** 静的解析が効かない言語だと、エージェントが壊したことに誰も気づけない。

変わるのは、人間が書く場所です。

| | 以前 | これから |
|---|---|---|
| 型そのもの | 人間が書く | **AI が書く** |
| 型のレビュー | 人間がやる | **静的解析がやる** |
| **何を許さないか** | 暗黙のルール・レビューでの指摘 | **人間が書く。ここだけ。** |

`as` を1つずつ指摘して回るのは、もう人間の仕事ではありません。**「`as` を許さない」と1行書くのが人間の仕事**です。

そして僕は、その1行を書き忘れていました。

---

## tsconfig も同じだった

コンパイラ側も見ました。ここも**継承するので、ファイルを読んでも実効値は分かりません**。

```bash
npx tsc -p tsconfig.server.json --showConfig
```

| 設定 | app | server | test |
|---|---|---|---|
| `strict` | ✅ | ✅ | ✅ |
| **`noUncheckedIndexedAccess`** | ❌ | ❌ | ❌ |
| **`useUnknownInCatchVariables`** | ❌ | ❌ | ❌ |
| **`noImplicitReturns`** | ❌ | ❌ | ❌ |
| **`noPropertyAccessFromIndexSignature`** | ❌ | ❌ | ❌ |
| **`noImplicitOverride`** | ❌ | ❌ | ❌ |

`strict` は付いています。だから安心していました。**`strict` はこれらを含みません。**

### 全部、0件で入った

フラグを立ててコンパイルして、エラーを数えました。

```
noUncheckedIndexedAccess            server 0 / app 0
useUnknownInCatchVariables          server 0 / app 0
noImplicitReturns                   server 0 / app 0
noPropertyAccessFromIndexSignature  server 0 / app 0
noImplicitOverride                  server 0 / app 0
```

**5つとも0件。** 移行作業ゼロで、今日入れて何も壊れません。姉妹プロジェクトでも測ったら、そちらも**4つとも0件**でした。

**入れていなかった理由は「大変だから」ではなく、「測っていなかったから」**でした。

### 設定の欠落が、別の設定を off にさせていた

これが一番こたえました。

`eslint.config.js` に、あるルールを off にした理由が書いてあります。

```js
// 型の上で結果が変わらない比較を不要と指摘するルール。
// noUncheckedIndexedAccess が off だと実行時に必要なガードまで
// 「常に真」に見えるので、消すと落ちる。
"sonarjs/different-types-comparison": "off",
```

**「`noUncheckedIndexedAccess` が off だから、このルールは使えない」**と書いてあります。そしてその `noUncheckedIndexedAccess` は、**0件で入る**。

理由をコメントに残すのは正しい習慣です。ただ、**その理由が「別の設定が足りないから」だったときに、上流を直しに行く**ところまでが対でした。書いてあったのに、上流を直すという発想にならなかった。

---

## まとめ：確認は3コマンドで済む

やることは単純でした。**設定ファイルを読むのではなく、実効値を出力させて数える。**

```bash
# 実際に効いている ESLint ルール
npx eslint --print-config <file>

# 継承後の tsconfig 実効値
npx tsc -p <config> --showConfig

# そのフラグを入れたら何件出るか
npx tsc -p <config> --noEmit --<flag>
```

3つとも数秒から十数秒です。**やらなかった理由は、思いつかなかったからでした。**

そして分かったこと。

- 規約に書いてあっても、**機械が止めていなければ守られない**
- `strict` や `recommended` は、**あなたが思っているものを含んでいるとは限らない**
- **`any` は構文だけでは見えない。** 型情報を使うルールが要る
- キャストで黙らせていたものには、**ちゃんと言いたいことがあった**（`undefined` が入り得る、検査より先に主張している、そもそも不要）
- **型が間違っているのが本家の場合だけ、cast を残す。** ただし本家に issue を立て、allowlist に理由と「いつ消せるか」を書く
- **設定の欠落が、別の設定を off にさせることがある**
- **Vue のテンプレート内は typescript-eslint から見えない。** 測ったら React / Solid / Preact / Svelte / Astro は全部見えていて、**Vue だけ**だった
- **lint が通ったことは、lint が見たことを意味しない**

ESLint と TypeScript の設定は、正直むずかしいと思います。組み合わせが多く、バージョンで変わり、抜けても何も言われない。だから AI に任せるにしても、**「入れて」と頼んだあとに、入ったかを自分で確かめる**工程は要ります。

チェックリスト、作ります。

作業中の issue はこちらです。

- [mulmoterminal#1231](https://github.com/receptron/mulmoterminal/issues/1231) — `as` の除去
- [mulmoterminal#1300](https://github.com/receptron/mulmoterminal/issues/1300) — 型情報ルールの導入（165件）
- [mulmoterminal#1301](https://github.com/receptron/mulmoterminal/issues/1301) — tsconfig の5フラグ
- [mulmoclaude#2736](https://github.com/receptron/mulmoclaude/issues/2736) — 同（4フラグ）

**自分のリポジトリでも、上の3コマンドを打ってみるといいと思います。** 入っていると思っているものが、入っているとは限りません。

---

## 関連記事

AI に思い切り書かせるための6本です。どこから読んでも大丈夫ですが、この順に並んでいます。

1. [ターミナルを自作したら、1日のコミット数が500を超えて、生産性がバグった話](https://zenn.dev/singularity/articles/diy-terminal-500-commits) — 並列運用の始まり。道具そのものを作った話
2. [1日500コミットは、もう読めない ── だからコードレビューをやめた](https://zenn.dev/singularity/articles/stopped-reviewing-my-code) — 読まなくても壊れない仕組みの全体像
3. [ユーザーの困りごとは、その日のうちに直す ── 中央値1.2時間、最速9分](https://zenn.dev/singularity/articles/issue-median-one-hour) — 機械に移せなかったものは何か
4. **ESLint と TypeScript の設定、抜けてないと言い切れますか** ← **いまここ**
5. [AIでがんがん書く時代の「きれいなコード」の守り方](https://zenn.dev/singularity/articles/clean-code-ci-for-ai-era) — ESLint / SonarJS / jscpd / knip を CI に置く実装編
6. [jscpd で重複コードを機械的に潰す](https://zenn.dev/singularity/articles/jscpd-dry-detection-mono) — 重複検出の詳細。全体監査と CI 差分チェックの二段構え
