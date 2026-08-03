---
title: "ESLint と TypeScript の設定、ネット記事のコピペで作っていませんか？"
emoji: "📏"
type: "tech"
topics: ["TypeScript", "ESLint", "ClaudeCode", "AI", "vibecoding"]
published: true
publication_name: "singularity"
---

正直に聞きます。

**あなたのプロジェクトの `tsconfig.json` と `eslint.config.js`、どこかからコピペして、そのままになっていませんか。**

僕はなっていました。Zenn や Qiita の記事から持ってきて、動いたのでそのまま。中身を1行ずつ説明できるかというと、できません。

そして、こういうことを聞かれたら答えられますか。

- **`tsconfig` の設定**、なぜその値なのか説明できますか
- **ESLint の flat config**、旧形式との違いを把握して書いていますか
- **TypeScript 5 と 7**、何がどう変わったか言えますか（7.0 は**2026年7月8日に GA** しました。Go 製の別実装です。いま多くの人が使っている 6.0 は「最後の JavaScript 版」という位置づけの橋渡しリリースです）
- **Node 20 → 22 → 24**、何が変わって何に影響しましたか
- **`mjs` と `cjs`**、`"type": "module"` のとき、Web 向けにバンドルするとき、それぞれの解決の違いを説明できますか

**僕は無理です。** 全部は追えていません。

これ、人類には早すぎると思っています。**スレッドプログラミングと同じ種類の難しさ**です。一つ一つは理解できるのに、組み合わさった瞬間に、自分の頭の中のモデルと実際の挙動がずれる。しかも**ずれても、何も言われません。**

で、実際にずれていました。

---

`CLAUDE.md` に、はっきりこう書いてあります。エージェントが毎回読む、うちのコーディング規約です。

```markdown
NEVER use `as` type casts; MUST use type guards instead
```

**型が合わないときに `as` でごまかすな、型ガードを書け**、という意味です。

`as` は「この値はこの型だ」と**コンパイラに言い切る**構文です。検査ではありません。宣言です。

```ts
const user = JSON.parse(body) as User;   // ← 中身は何も確かめていない
user.name.toUpperCase();                 // 💥 name が無ければここで落ちる
```

`JSON.parse` が返すのは `any` で、中身が `User` である保証はどこにもありません。それでも `as User` と書けば、そこから先は `User` として扱われます。**型エラーは消えます。バグは消えません。**

型ガードなら、確かめてから通します。

```ts
const isUser = (x: unknown): x is User =>
  typeof x === "object" && x !== null && "name" in x && typeof x.name === "string";

const parsed: unknown = JSON.parse(body);
if (!isUser(parsed)) return res.status(400).json({ error: "bad payload" });
parsed.name.toUpperCase();   // ここでは本当に User
```

`x is User` は「この関数が `true` を返したら、その値は `User` だ」という宣言です。**`as` との違いは、根拠があること。** 中で実際に確かめた結果に、名前を付けているだけです。

> 💡 `"name" in x` を挟むのが要点です。これを書くと `x.name` に触れるようになるので、**ここも `as` なしで書けます**。上のコードは `--strict` でエラー0、`as` は1つも使っていません（実際にコンパイルして動かしました）。

書く量は増えます。でも増えた分が、**実行時に効く検査**です。`as` はそれを1単語で消せてしまう。だから禁止しています。

なのにコードを読んでいたら、`as` がありました。1つではなく、あちこちに。

最初は「エージェントが規約を守っていないな」と思いました。**違いました。** 設定を見に行ったら、そもそも **ESLint で禁止していませんでした。**

守らせていなかったのは、こちらです。

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

**「プリセットを入れたから大丈夫」が成り立ちません。** 抽象的な話ではないので、実際に測った表を出します。

### どのプリセットに何が入っているか（実測）

`typescript-eslint` の各プリセットを読み込んで、ルールの有効状態を出しました。

| ルール | `recommended` | `strict` | `stylistic` | `recommendedTypeChecked` | `strictTypeChecked` |
|---|---|---|---|---|---|
| `no-explicit-any` | ✅ | ✅ | — | ✅ | ✅ |
| `no-non-null-assertion` | — | ✅ | — | — | ✅ |
| **`consistent-type-assertions`** | — | **—** | **✅** | — | **—** |
| `ban-ts-comment` | ✅ | ✅ | — | ✅ | ✅ |
| `no-floating-promises` | — | — | — | ✅ | ✅ |
| `no-unsafe-assignment` | — | — | — | ✅ | ✅ |
| `await-thenable` | — | — | — | ✅ | ✅ |
| `no-base-to-string` | — | — | — | ✅ | ✅ |

**僕がハマったのは3行目です。**

`strict` を入れていたので、型まわりは一番厳しくしたつもりでいました。ところが **`consistent-type-assertions`（`as` を止めるルール）は `strict` に入っていません**。`stylistic`（見た目・書き方の統一）のほうに入っています。

名前から想像すると逆に思えます。`as` は「書き方」ではなく「安全性」の話に見えるので。**でも分類上は `stylistic` でした。**

### 抜けやすい組み合わせ、3つ

実測から分かった、**具体的に抜けるパターン**です。

**① `strict` を入れた → `as` は止まらない**

`stylistic` を併用しないと入りません。`strict` と `stylistic` は**排他ではなく、両方入れるもの**です。

```js
export default tseslint.config(
  ...tseslint.configs.strict,
  ...tseslint.configs.stylistic,   // ← これが無いと as が止まらない
);
```

**② `strict` を入れた → 型情報を使うルールは1つも入らない**

`no-floating-promises`（`await` の付け忘れ）も `no-unsafe-assignment`（`JSON.parse` の結果）も、**`TypeChecked` が付いたプリセットにしか入っていません**。

しかも `TypeChecked` 版は `parserOptions.projectService` を設定しないと**動きさえしません**。プリセットを足すだけでは足りない、というのがもう一段の罠です。

**③ `tsconfig` に `strict` を入れた → 6つ入らない**

こちらも測りました。`strict: true` だけを書いた設定の実効値です。

| `strict` が**含む**もの | `strict` が**含まない**もの |
|---|---|
| `noImplicitAny` | **`noUncheckedIndexedAccess`** |
| `strictNullChecks` | **`exactOptionalPropertyTypes`** |
| `strictFunctionTypes` | **`noImplicitReturns`** |
| `strictBindCallApply` | **`noPropertyAccessFromIndexSignature`** |
| `strictPropertyInitialization` | **`noImplicitOverride`** |
| `noImplicitThis` | **`noFallthroughCasesInSwitch`** |
| `useUnknownInCatchVariables` | `noUnusedLocals` / `noUnusedParameters` |
| `alwaysStrict` | |

**「`strict` を入れたから一番厳しい」ではありませんでした。** 右側は全部、個別に書かないと有効になりません。

### さらに、フレームワークのプリセットが上書きする

Vue のプロジェクトでは `@vue/tsconfig` を継承していました。実効値を出すと、こうなっています。

```
strict = true
exactOptionalPropertyTypes = true     ← @vue/tsconfig が入れてくれている
noUncheckedIndexedAccess = 未設定       ← 入っていない
verbatimModuleSyntax = true
```

**自分では書いていない設定が入っていて、書いたつもりの設定が入っていない。** どちらも実効値を出すまで分かりませんでした。

そしてこれは、**フレームワークとそのバージョン**によって変わります。Nuxt、Next、SvelteKit、それぞれ自前の tsconfig を配っていて、中身は同じではありません。

知らなかったわけではなく、**確かめたことがなかった**、というのが正直なところです。

## AI に任せられる。でも、確認は別

いまは設定もだいたい AI に書かせられます。実際そうしています。

ただ、**「入れて」と頼んだものが入ったかは、頼んだ側が確かめないと分かりません。** エージェントは「入れました」と言いますし、たいてい本当に入れています。問題は**入れていないことに誰も気づかない**ケースのほうです。

エラーが出ないので。lint も通るし、CI も緑です。**設定が抜けていることは、何も報告しません。**

（この確認を毎回やるのは面倒なので、**貼ると必ずエラーになるサンプル集**を別記事にまとめています。設定が入っていれば落ちる、入っていなければ通る。それだけのコードです。近日公開します。）

以下は、実際に何を確かめて、何が出てきたかの記録です。**一連の作業は終わりました**（`as` 149 → 0、型情報ルールの指摘 407 → 0、tsconfig 3フラグ追加）。数字は最後にまとめてあります。

先に言っておくと、**バグはそれなりに出ました。** 型が「ある」と言っていたのに API が返していないフィールド、`"[object Object]"` を黙って画面に出していた文字列化、実行する人によって答えが変わり得た並び順。そして**検査していたつもりの検査が動いていなかった**のが2つ。

---

## まず `as` を止めた

`assertionStyle: "never"` を入れて、ESLint に数えさせました。**149箇所**。

（結論から書くと、**この149箇所は全部片付きました**。いまルールは `error` で入っていて、後述の「本家の型が壊れている」2件だけが例外として残っています。以下はその過程で出てきたものです。）

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

（この記事を書いている間に `no-floating-promises` と `no-misused-promises` は入りました。以下は**入れる前に測った数字**です。）

### これは `as` を消しても残る穴

`no-explicit-any` は入っています。だから `any` は止まっていると思っていました。

**止まっていません。** このルールは **`any` という字が書かれている場所しか見られない**からです。

- **型定義のないライブラリ**から流れ込む値
- **`JSON.parse()` の戻り値**
- `as unknown as T` を消しても、**消した先が `any` なら同じ**

そして構文ルールでは**原理的に**捕まらないものがあります。**`await` の付け忘れ**は、構文としては完全に正しい。

試しに入れて数えたら **165件**、所要 **11秒**でした。

| ルール | 件数 |
|---|---|
| `no-unsafe-member-access` | 65 |
| `no-unsafe-assignment` | 49 |
| **`no-floating-promises`** | **24** |
| `no-unsafe-argument` | 14 |
| その他 | 13 |

`no-unsafe-*` の139件は「型が付いていない」で、必ずしもバグではありません。**`no-floating-promises` の24件**が問題です。Promise を作って `await` も `.catch()` も付けていないので、**握り潰された失敗がある可能性**があります。

### 全部読んだら、このルールでは実バグ0件でした

先に結論を書きます。**この予想は外れました。**

`.vue` の配線を直したら件数は 24 → **66件**に増えたのですが、**66件を全部読んで、実バグは1つもありませんでした**。内訳です。

| パターン | 件数 |
|---|---|
| `router.push(…)` | 16 |
| 意図的な再取得（`loadDiff` / `refreshUsage` など） | 約30 |
| `nextTick(…)` | 7 |
| `.then(…)` 連鎖 | 5 |
| `socket.join` / `close()` など | 4 |

**全部「意図的に待たない」ものでした。**

### それでも直す価値はありました

じゃあ無駄だったかというと、逆です。**66件を `void` で明示しました。**

```ts
void router.push("/settings");   // ← 意図的に待たない、とコードに書く
```

理由は1つです。

> **66件が出続ける状態では、本物の `await` 忘れが紛れても気づけません。**

これ以降 `no-floating-promises` の警告が出たら、それは**本当に誰かが `await` を忘れた**という意味になります。**警告を0に戻すことが、警告に意味を持たせる**ことでした。

（`void` は unhandled rejection を防ぐわけではありません。lint を黙らせるだけです。ただしサーバ側は `unhandledRejection` を拾ってログに出す仕組みがあり、ブラウザはコンソールに報告するので、**「意図的に待たない」という表明としては正直**だと判断しました。）

ちなみに `void` を禁じる `sonarjs/void-use` というルールがあり、これと**正面から矛盾**します。`await` 忘れを捕まえる側を選びました。**ルール同士がぶつかることもある**、という例です。

### ただし、バグが0件だったのは「このルール」だけでした

ここは誤解を招くので、はっきり書き直します。

**`no-floating-promises` の66件に実バグは無かった。でも、他のルールは実バグを出しました。** 全部片付けたので、出てきたものを並べます。

#### 型が「ある」と言っていたフィールドを、API は返していなかった

`/api/session/:id` の応答に `SessionDetail` という型を付けていました。ガードを足したら、**テストが4件落ちました。**

**このエンドポイントは `id` を返していませんでした。** 型だけが「ある」と言っていて、実際には来ない。`as` で型を主張していたので、誰も気づきませんでした。型を消しました。

#### 呼び出し側が名乗った型を、検証せずに返していた

```ts
const data = await fetchJson<Config>("/api/config");   // ← 中身は確かめていない
```

`fetchJson<T>` は**受け取った JSON をそのまま `T` として返していました。** 呼び出し側が `Config` と書けば `Config` になる。書いただけです。

`read` 引数（検証関数）を**必須**にしました。オプションにすると既定が「全部通す」になって元に戻るので。

#### 保存時は検証していて、読み込み時は検証していなかった

設定の保存パスにはバリデーションがありました。**読み込みパスには無かった。** 壊れた設定エントリはそのまま起動時に入ってきます。

#### 文字列化が、壊れた値を黙って通していた

これが一番効きました。17箇所ありました。

```ts
const slug = String(x ?? "");
```

`x` がオブジェクトだと **`"[object Object]"`** になります。例外は出ません。そしてその文字列が**コレクションの slug としてルックアップに使われ、セッションのタイトルとして画面に出ます。**

問題は「表示が変」ではありません。

> **「値が無い」と「値が壊れている」が区別できなくなる。**

どちらも「何か変な文字列」として同じ経路を流れていきます。`readString`（文字列ならそれ、でなければ `""`）に寄せました。人間向けのエラーメッセージだけは、**何を拒否したのか分からなくなる**ので `JSON.stringify` する側を選んでいます。

#### `await` が、読み手に嘘をついていた

```ts
res.json(await readWikiIndex(workspace));
```

`readWikiIndex` は**同期関数**です。`await` は何も待っていません。**「ここは I/O だから遅い」という嘘の合図**を出していただけでした。

#### 並び順が、実行する人によって変わり得た

ファイル名のソートに `localeCompare` を使えという指摘が出ました。**が、対象はこれです。**

```
2026/08/02/rollout-2026-08-02T09-00-00-<uuid>.jsonl
```

ゼロ埋めの日付と ISO タイムスタンプを**ロケール順に並べると、答えがマシンによって変わります。** 呼び出し側が意味しているのはコード単位順なので、それを明示する関数に切り出して、**`localeCompare` と結果が分かれるケース（`"B"` vs `"a"`）をテストに入れて**固定しました。

**指摘に従っていたら、バグを入れていました。**

#### 常に偽のガードが1つ

```ts
for (const m of s.matchAll(re)) {
  const start = m.index;
  if (start === undefined) continue;   // ← ここに来ない
```

`matchAll` は global 正規表現を要求し、返すマッチには**仕様上つねに `index` が入ります**。死にコードでした。

#### `any` の入口は、たった3つでした

145件の `no-unsafe-*` が server に散らばって見えましたが、出どころは3種類だけです。**これは他のプロジェクトでもそのまま使えると思います。**

| 入口 | なぜ `any` になるか |
|---|---|
| **`await import(name)`** | 動的 import の戻りは `any`。そこから辿る `mod.default` も `mod.execute()` も**全部型検査の外** |
| **`JSON.parse(...)`** | 戻りが `any` |
| **`req.body`** | Express の型が `any` |

3つとも「外から来た値」です。**境界に1つ関数を置いて全部そこを通す**だけで消えました。

ここで1つ発見がありました。**`typeof x === "function"` では絞りきれません。** `Function` にしか絞れず、`Function` の呼び出しは `any` を返すので、結果がまた型検査の外に出ます。

そして、この `any` のために**わざわざ書かれた回避策**が見つかりました。

```ts
// Re-checked rather than reusing the guard above: `req.body` is `any`, and narrowing it there
// does not survive to here — the call would take `any` and typecheck would not notice.
```

**「上でガードしたのに、`any` だから絞り込みがここまで残らない」** と書いてあります。型が通るようになって、この再チェックごと消えました。**設定の欠落は、コードの形にも跡を残します。**

### そして、検査していたつもりの検査が動いていなかった

最後にこれです。作業中に**別の穴**が見つかりました。

```
yarn typecheck   →   server と test を一切見ていない
```

root の `tsconfig.json` が app と node しか参照しておらず、`vue-tsc -b` はその参照を辿るだけだったので、**server のコードは1行も型検査されていませんでした。**

「緑だった」は「検査した」の証拠になりません。なので、**4領域それぞれに `const x: number = "nope"` を仕込んで**確かめました。

| 仕込んだ場所 | 修正前 | 修正後 |
|---|---|---|
| `server/config/workspace.ts` | 素通り | ✓ 検出 |
| `src/utils/focusTrap.ts` | ✓ | ✓ |
| `test/common/readString.spec.ts` | 素通り | ✓ 検出 |
| `test/server/git/prs.spec.ts` | 素通り | ✓ 検出 |

そして気づいたのは、**規約のほうにその跡があった**ことです。

> `yarn typecheck` alone passes while CI fails. 3つ全部走らせろ

これは**この欠落を運用でカバーするための注意書き**でした。参照を 2 → 5 にしたら、1本で済むようになりました。**規約に「気をつけろ」と書いてあるものは、たいてい設定で直せます。**

姉妹プロジェクトでは、同じ形の穴がもう1つありました。

```
eslint src server test e2e e2e-live packages     ← scripts/ batch/ config/ が無い
```

`yarn lint` が **`scripts/` に一度も届いていませんでした。** そこにあるのは、ビルド駆動・リリース監査・npm smoke ── **CI が「この PR をマージしてよいか」を判断するために動かしているコード**です。**92 errors** が放置されていて、うち**9件はまさに禁止したはずの `as`** でした。

> **ゲートを作るコードが、ゲートの外にいた。**

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

### さらに、`.vue` の `<script>` にも型情報は自動では届いていなかった

これは記事を書いている途中で判明したことです。

型情報ルールを入れたつもりでいたのに、**`.vue` ファイルには効いていませんでした**。配線を直したら `no-floating-promises` が **34件 → 66件**に増えて気づきました。**32件が `.vue` の `<script>` に隠れていた**わけです。

必要だったのは2つです。

**① `.vue` のブロックに `project` と `extraFileExtensions` を渡す**

```js
files: ["**/*.vue"],
languageOptions: {
  parserOptions: {
    parser: tseslint.parser,
    project: ["./tsconfig.app.json"],       // ← これ
    tsconfigRootDir: import.meta.dirname,
    extraFileExtensions: [".vue"],          // ← これ
  },
},
```

**② ルールは、`languageOptions` を持たない別のブロックで足す**

ここが罠でした。型情報ルールのブロックの `files` に `.vue` を足すと、そのブロックに書いた `parser: tseslint.parser` が **`vue-eslint-parser` を置き換えてしまいます**。全 SFC がパースできなくなります。

```
1:8  error  Parsing error: '>' expected
```

flat config は**後のブロックが前を上書きする**ので、**ルールだけのブロックに分ける**のが正解でした。一度これを踏みました。

コストは `yarn lint` が 31秒 → 41秒。

そして **`<template>` の中は、これをやっても素通りのまま**です。届いたのは `<script>` までで、テンプレートは別扱いでした。

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

### ここで諦めかけたのですが、埋められました

しばらく「Vue の構造上の問題だから、テンプレートに式を書かないよう気をつけるしかない」と思っていました。**それは間違いでした。**

見落としていたのは、**typescript-eslint のルールが届かないだけで、AST 自体は歩ける**ということです。`eslint-plugin-vue` の **`vue/no-restricted-syntax`** が、テンプレートの AST を歩く唯一のルールでした。同じ禁止をセレクタとして書けば済みます。

```js
// .vue の override ブロック
"vue/no-restricted-syntax": [
  "error",
  {
    selector: "TSAsExpression",   // ← as
    message: "Do not use any type assertions — narrow in <script> and pass the result to the template.",
  },
],
```

`!` を止めたいなら `TSNonNullExpression` を足します。

姉妹プロジェクトに入れたら、**host とプラグインで16箇所**出てきました。**ルールを `error` にした後も、ずっとゲートの外にいた16箇所**です。

そして直してみたら、**6箇所はそもそも不要**でした。

```vue
<div v-if="entry.spec.type === 'stdio'">
  {{ (entry.spec as StdioSpec).command }}   <!-- v-if が既に絞っている -->
</div>
```

Vue のテンプレートは `v-if` でナローイングが効きます。**キャストを書いた時点では必要だったのかもしれませんが、その後の変更で不要になっていた。** 誰も気づかなかったのは、**そこを見ているルールが無かったから**です。

残りは、DOM のチェックを `<script>` の名前付きハンドラに移しました。

```vue
<!-- before -->
@input="onChange(($event.target as HTMLTextAreaElement).value)"

<!-- after -->
@input="onInput"
```

```ts
// script 側。外れたら早期 return（キャストは素通りさせていた）
function onInput(e: Event) {
  if (!(e.target instanceof HTMLTextAreaElement)) return;
  onChange(e.target.value);
}
```

**キャストが隠していたのは「チェックを書く場所を間違えている」ことでした。** DOM の型チェックは本来 `<script>` に置くものです。

### そして、この記事を書いている今も1件残っています

正直に書いておくと、**メインのリポジトリにはこのルールをまだ入れていません。** grep したら1件ありました。

```vue
<!-- src/components/SettingsField.vue:14 -->
@input="$emit('update:modelValue', ($event.target as HTMLInputElement).value)"
```

このファイルに対する設定を出すと、こうです。

```console
$ npx eslint --print-config src/components/SettingsField.vue | jq '.rules["@typescript-eslint/consistent-type-assertions"]'
[2, {"assertionStyle": "never"}]          ← error として効いている

$ npx eslint src/components/SettingsField.vue
                                          ← 何も出ない
```

**`error` に設定されているルールが、同じファイルの中の違反を報告しない。** `yarn lint` は 0 errors です。

[issue にしました](https://github.com/receptron/mulmoterminal/issues/1339)。**「149箇所を0にした」と書いた作業は、実は 149 → 1 でした。**

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

そのまま入れました。`useUnknownInCatchVariables` と `noImplicitOverride` は宣言どおり0件で通っています。

⚠️ **ただし訂正があります。** `noUncheckedIndexedAccess` だけは、リポジトリ全体で測り直したら **445件**（app 47 / server 71 / test 327）でした。上の「0件」は**サンプル1本での試算**で、全体ではありません。**0件だと一般化したのは早すぎました。**

これも入れました。**本体118件を全部直して**有効化しています（テストの327件は対象外にしました。テストは自分で作った fixture を `rows[0]` で引くので、そこの `T | undefined` は**テスト自身が保証している値についての雑音**だからです）。

118件の直し方は4通りで、**どれも設計判断が要りませんでした。**

| 直し方 | 例 |
|---|---|
| 一度 const に受けて undefined を弾く | `const line = lines[i]; if (line === undefined) …` |
| `??` で既定値 | `(mimeType.split(";")[0] ?? "").trim()` |
| `?.` にする | `wrapped?.[1] ?? content.replace(…)` |
| 境界チェック済みなら早期 return | `if (!def) return null;` |

**「コンパイラが正しく、コードが暗黙に知っていたことを書く」だけ**でした。

### 設定の欠落が、別のルールを嘘つきにしていた

これが一番おもしろかったところです。

姉妹プロジェクトの `eslint.config.mjs` に、あるルールを off にした理由が書いてありました。

```js
// `different-types-comparison` は、型の上で結果が変わらない比較を不要だと指摘する。
// これを消すと本物のガードが剥がれる。ルールの前提は
// noUncheckedIndexedAccess が有効になって初めて成り立つ ── そのとき見直すこと。
"sonarjs/different-types-comparison": "off",
```

**「`noUncheckedIndexedAccess` が off だから、このルールは信用できない」**と書いてあります。そしてその `noUncheckedIndexedAccess` は、**0件で入る**。

### そして、実際にそうなりました

このあと型情報の lint を入れたところ、**この予想が実証されました。**

`different-types-comparison` が初めて実際に走り、**9件の指摘が出ました。そして9件すべてが偽陽性**でした。

| 箇所 | ルールの指摘 | 実際 |
|---|---|---|
| `process.argv[2]` の判定 | 「`=== undefined` は常に偽」 | **引数なしで起動すれば undefined**。消すと `--help` が壊れる |
| `params[key]` の判定 | 同上 | **キーが無ければ undefined**。消すと省略可の引数が必須になる |
| 辞書引きの結果 | 「`!== undefined` は常に真」 | **辞書に無ければ undefined**。消すと未翻訳がキャッシュに入る |
| 分割代入の余り | 同上 | `const [code, ...rest] = params` |
| 配列の前要素 | 同上 | `order[idx - 1]` |

**指摘に従ってガードを消すと、実際に落ちます。**

### そして `noUncheckedIndexedAccess` を入れたら、9件のうち5件が消えた

見立てが正しかったことは、入れてみて確定しました。

```
different-types-comparison:  9件 → 4件
lint の警告全体:            33件 → 25件
```

**型が実態に追いついた瞬間に、ルールが正しく評価するようになった**わけです。残った4件も、調べたら**修正の過程で本当に不要になった再チェック**でした。

つまり最終的にはこうです。

> **偽陽性が9件出ていたのは、ルールのせいでも、コードのせいでもなかった。設定が1つ欠けていたせいだった。**

なぜこうなるか。`noUncheckedIndexedAccess` が無いと、TypeScript は `arr[i]` の型を `T` だと言います（本当は `T | undefined`）。**ルールはその嘘を信じて「この比較は無意味だ」と判定します。**

つまり:

> **設定が1つ欠けていたせいで、別のルールが「正しい防御を消せ」と言ってくる状態になっていた。**

ルールが悪いのではありません。**渡している型が嘘だから、正しい推論をして間違った結論に着く**。

理由をコメントに残すのは正しい習慣です。ただ、**その理由が「別の設定が足りないから」だったときに、上流を直しに行く**ところまでが対でした。書いてあったのに、上流を直すという発想にならなかった。

---

## 終わったので、数字を置いておきます

一連の作業は終わりました。最終状態です。

| | 最初 | いま |
|---|---|---|
| `as` 型アサーション | 149 | **0**（allowlist 2件・理由と消せる条件つき） |
| `no-unsafe-*`（本物） | **407** | **0** — 5ルールとも **error** |
| `await-thenable` / `no-base-to-string` | 19 | **0** — どちらも **error** |
| `no-floating-promises` | 102 | **0**（66件は `void` で意図を明示） |
| 型情報つき sonarjs 8種 | 18 | 3件修正・4種 error・4種 off・1種 warn |
| `yarn lint` | — | **0 errors / 11 warnings** |

残る11件の warning は、**外部 API の `deprecation` 5件**（代替が無いか、外部由来）、**`max-lines` 5件**、**正規表現1件**です。**「0にする」ではなく「意味のある数字にする」**のが目的だったので、ここで止めています。

### 入れなかったものもあります

`tsconfig` の5フラグのうち、**2つは入れませんでした。** 「今はやらない」ではなく、**入れない**という判断です。

**`noImplicitReturns`（58件）** — 指摘のほとんどが Express ハンドラのこの形です。

```ts
if (bad) return res.status(400).json({ error });
res.json(result);   // ← 明示的な return が無い
```

**これは Express の書き方として正しい**ものです。ハンドラの戻り値は誰も見ていません。通すには58箇所に意味の無い `return` を足すことになり、**型の穴は1つも塞がりません。**

**`noPropertyAccessFromIndexSignature`（1785件）** — `obj.key` を `obj["key"]` に書き換えるだけです。**アクセスの安全性は何も変わりません。**

`noUncheckedIndexedAccess` の118件は「コンパイラが正しく、コードが暗黙に知っていたことを書く」作業でした。この2つは違います。**件数ではなく、その作業で型の穴が塞がるかどうか**が分かれ目でした。

テストについても分けました。`noUncheckedIndexedAccess` は**テストだけ off** です。テストは自分で作った fixture を `rows[0]` で引くので、そこの `T | undefined` は**テスト自身が保証している値についての雑音**だからです。233件が0件になります。**フラグの価値は出荷されるコードにあります。**

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
- **`any` は構文だけでは見えない。** 型情報を使うルールが要る。そして `any` の入口は**動的 import / `JSON.parse` / `req.body` の3つ**にほぼ集約される
- キャストで黙らせていたものには、**ちゃんと言いたいことがあった**（`undefined` が入り得る、検査より先に主張している、そもそも不要）
- **型が間違っているのが本家の場合だけ、cast を残す。** ただし本家に issue を立て、allowlist に理由と「いつ消せるか」を書く
- **設定の欠落が、別の設定を off にさせることがある**
- **偽陽性を見たら、まず「別の設定が足りないのでは」と疑う。** ルールが間違っているのではなく、渡している型が嘘だから正しく推論して間違った結論に着く
- **Vue のテンプレート内は typescript-eslint から見えない**（React / Solid / Preact / Svelte / Astro は全部見えていて Vue だけ）。ただし **`vue/no-restricted-syntax` で埋められる**
- **ゲートを作るコードほど、ゲートの外にいやすい。** `scripts/` も、テストも、`.vue` も
- **緑だったことは、検査したことの証拠にならない。** 疑ったら**わざと壊して、落ちることを確かめる**
- **lint が通ったことは、lint が見たことを意味しない**

そして、いちばん効いたのはこれでした。

> **件数の多さは、作業量の指標であって、価値の指標ではない。**

118件の `noUncheckedIndexedAccess` は全部直す価値がありましたが、1785件の `noPropertyAccessFromIndexSignature` は1件も価値がありませんでした。**「その作業で型の穴が塞がるか」だけが分かれ目**です。

ESLint と TypeScript の設定は、正直むずかしいと思います。組み合わせが多く、バージョンで変わり、抜けても何も言われない。だから AI に任せるにしても、**「入れて」と頼んだあとに、入ったかを自分で確かめる**工程は要ります。

### issue と結果

一連の作業で立てた issue と、その顛末です。

| issue | 中身 | 結果 |
|---|---|---|
| [mulmoterminal#1231](https://github.com/receptron/mulmoterminal/issues/1231) | `as` の除去 | ✅ 149 → **0**（allowlist 2件） |
| [mulmoterminal#1300](https://github.com/receptron/mulmoterminal/issues/1300) | 型情報ルールが1つも入っていない | ✅ `no-unsafe-*` 407件 → **0**、5ルールとも error |
| [mulmoterminal#1301](https://github.com/receptron/mulmoterminal/issues/1301) | tsconfig の5フラグ | ✅ 3つ導入（118件修正）・**2つは入れない判断**で close |
| [mulmoterminal#1312](https://github.com/receptron/mulmoterminal/issues/1312) | `yarn typecheck` が server と test を見ていない | ✅ 参照を 2 → 5 に |
| [mulmoterminal#1339](https://github.com/receptron/mulmoterminal/issues/1339) | **テンプレートの `as` が error なのにすり抜ける** | ⏳ この記事を書いていて見つけた |
| [mulmoclaude#2692](https://github.com/receptron/mulmoclaude/issues/2692) | `as` の除去（187箇所） | ⏳ 進行中（テンプレート16件は完了） |
| [mulmoclaude#2736](https://github.com/receptron/mulmoclaude/issues/2736) | tsconfig の4フラグ | ✅ 全部導入 |
| [gui-chat-protocol#30](https://github.com/receptron/gui-chat-protocol/issues/30) | 本家の generic が検証していない | ✅ 2.0.0 で `parse` 必須化 |

**自分のリポジトリでも、上の3コマンドを打ってみるといいと思います。** 入っていると思っているものが、入っているとは限りません。

僕は、**この記事を書きながらもう1件見つけました。**

そしてこの記事を出したあとに、**もう1件**出ました。ここに「2つのルールが矛盾するので片方を off にした」と書いた箇所があります ── **その理由自体が間違っていました。** 実装を読んだら矛盾していなかった。有効に戻したら、66件出ると思っていたものが**3件**でした。

設定ファイルに書いた理由は、**間違っていても自信たっぷりのまま生き残ります。** その話は別記事にします。

---

## 関連記事

AI に思い切り書かせるための6本です。どこから読んでも大丈夫ですが、この順に並んでいます。

1. [ターミナルを自作したら、1日のコミット数が500を超えて、生産性がバグった話](https://zenn.dev/singularity/articles/diy-terminal-500-commits) — 並列運用の始まり。道具そのものを作った話
2. [1日500コミットは、もう読めない ── だからコードレビューをやめた](https://zenn.dev/singularity/articles/stopped-reviewing-my-code) — 読まなくても壊れない仕組みの全体像
3. [ユーザーの困りごとは、その日のうちに直す ── 中央値1.2時間、最速9分](https://zenn.dev/singularity/articles/issue-median-one-hour) — 機械に移せなかったものは何か
4. **ESLint と TypeScript の設定、ネット記事のコピペで作っていませんか？** ← **いまここ**
5. [AIでがんがん書く時代の「きれいなコード」の守り方](https://zenn.dev/singularity/articles/clean-code-ci-for-ai-era) — ESLint / SonarJS / jscpd / knip を CI に置く実装編
6. [jscpd で重複コードを機械的に潰す](https://zenn.dev/singularity/articles/jscpd-dry-detection-mono) — 重複検出の詳細。全体監査と CI 差分チェックの二段構え
