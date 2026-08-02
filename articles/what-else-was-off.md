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

ESLint と TypeScript の設定は、正直むずかしいと思います。組み合わせが多く、バージョンで変わり、抜けても何も言われない。だから AI に任せるにしても、**「入れて」と頼んだあとに、入ったかを自分で確かめる**工程は要ります。

チェックリスト、作ります。

作業中の issue はこちらです。

- [mulmoterminal#1231](https://github.com/receptron/mulmoterminal/issues/1231) — `as` の除去
- [mulmoterminal#1300](https://github.com/receptron/mulmoterminal/issues/1300) — 型情報ルールの導入（165件）
- [mulmoterminal#1301](https://github.com/receptron/mulmoterminal/issues/1301) — tsconfig の5フラグ
- [mulmoclaude#2736](https://github.com/receptron/mulmoclaude/issues/2736) — 同（4フラグ）

**自分のリポジトリでも、上の3コマンドを打ってみるといいと思います。** 入っていると思っているものが、入っているとは限りません。
