---
title: "その設定、本当に効いてますか ── 貼ると必ずエラーになるサンプル17本"
emoji: "🧪"
type: "tech"
topics: ["TypeScript", "ESLint", "設定", "AI", "vibecoding"]
published: false
publication_name: "singularity"
---

自分のリポジトリで、規約に「`as` を使うな」と書いてあるのに `as` が90箇所ありました。調べたら **ESLint で禁止していませんでした**（→ [前回](https://zenn.dev/singularity/articles/what-else-was-off)）。

そのとき思ったのが、**「設定が入っているかどうかを、目で確かめる方法がない」**ということでした。

設定ファイルを読んでも分かりません。プリセットが継承されるからです。`strict` を入れたからといって、何が有効になったのかは書いてありません。

なので、**貼ると必ずエラーになるコード**を用意しました。

## 使い方

1. 新しいファイルを作って、下のコードを貼る
2. `npx tsc --noEmit` と `npx eslint` を走らせる
3. **エラーが出なかった行が、あなたの設定に抜けているもの**

それだけです。エラーが出れば設定は効いています。出なければ入っていません。

:::message
この記事のサンプルは**全部実際に走らせて確認**しています。TypeScript 6.0.3 / typescript-eslint で、「設定 OFF のときは通り、ON のときだけ落ちる」ことを1本ずつ検証しました。
:::

---

## A. tsconfig 編（9項目）

まとめて1ファイルに貼れます。

```ts
// ① strict — 引数に型がない
export function a1(x) { return x + 1; }

// ② strictNullChecks — null を string に入れている
export const a2: string = null;

// ③ noUncheckedIndexedAccess — 配列の要素が undefined かもしれない
const arr: string[] = [];
export const a3 = arr[0].length;

// ④ exactOptionalPropertyTypes — 省略可のプロパティに undefined を明示的に入れている
type T4 = { a?: string };
export const a4: T4 = { a: undefined };

// ⑤ useUnknownInCatchVariables — catch した値の型を決めつけている
export function a5() { try { throw 1 } catch (e) { return e.message } }

// ⑥ noImplicitReturns — 戻り値がある道とない道が混ざっている
export function a6(b: boolean) { if (b) return 1; }

// ⑦ noPropertyAccessFromIndexSignature — 添字型にドットでアクセスしている
type T7 = { [k: string]: string };
export const a7 = (({} as T7)).foo;

// ⑧ noImplicitOverride — 親のメソッドを override と書かずに上書きしている
class P8 { m() {} }
export class C8 extends P8 { m() {} }

// ⑨ noFallthroughCasesInSwitch — case が break なしで次に落ちている
export function a9(n: number) {
  switch (n) { case 1: console.log(1); case 2: return 2; }
  return 0;
}
```

**それぞれ何が問題なのか**を、1つずつ書きます。

### ① `strict` — 引数の型が書かれていない

```ts
export function a1(x) { return x + 1; }
```

`x` に型がありません。この関数に**何を渡してもコンパイルが通ります**。

```ts
a1("abc");   // "abc1" が返る（文字列連結になる）
a1(null);    // 1 が返る
a1([]);      // "1" が返る
```

呼び出し側が間違えても、**誰も教えてくれません**。`strict` を入れると「`x` の型が指定されていない」と怒られます。

> 📌 `strict` は**8つの設定の束**です。以下の②⑤もこの中に含まれます。ただし③④⑥⑦⑧⑨は**含まれません**。ここが一番の落とし穴でした。

### ② `strictNullChecks` — `null` が素通りする

```ts
export const a2: string = null;
```

「文字列」と宣言した変数に `null` を入れています。これが通ると、こういうことが起きます。

```ts
const name: string = getUser().name;   // 実は null かもしれない
name.toUpperCase();                    // 💥 実行時エラー
```

**JavaScript で一番多いエラー**（`Cannot read property of null`）を、コンパイル時に潰すための設定です。

### ③ `noUncheckedIndexedAccess` — 配列の外を読んでも気づけない

```ts
const arr: string[] = [];
export const a3 = arr[0].length;
```

`arr` は**空の配列**です。`arr[0]` は存在しないので `undefined` になります。そこから `.length` を読むと落ちます。

TypeScript は既定で **`arr[0]` の型を `string` だと言います**。本当は `string | undefined` なのに。

```ts
const items = ["a", "b"];
const third = items[2];       // 型は string。実際は undefined
third.toUpperCase();          // 💥
```

この設定を入れると `string | undefined` になり、**チェックしないと使えなくなります**。

### ④ `exactOptionalPropertyTypes` — 「無い」と「undefined が入っている」の区別

```ts
type T4 = { a?: string };
export const a4: T4 = { a: undefined };
```

`a?: string` は「**`a` は無くてもいい**」という意味です。「**`a` に `undefined` を入れてもいい**」ではありません。

普通は同じに見えますが、違いが出る場面があります。

```ts
const patch = { name: undefined };
Object.assign(user, patch);       // user.name が undefined で上書きされる
// 「変更しない」つもりが「消す」になった
```

`{}` を渡していれば何も起きませんでした。**この設定は、実際にバグを見つけたことがあります**（詳細は[前回](https://zenn.dev/singularity/articles/what-else-was-off)）。

### ⑤ `useUnknownInCatchVariables` — catch した値は何が来るか分からない

```ts
export function a5() { try { throw 1 } catch (e) { return e.message } }
```

`e.message` を読んでいますが、**そこに `message` がある保証はありません**。JavaScript は何でも throw できます。

```ts
throw new Error("x");   // e.message → "x"
throw "文字列";          // e.message → undefined
throw 1;                // e.message → undefined
throw null;             // e.message → 💥 落ちる
```

この設定を入れると `e` の型が `unknown` になり、**確かめてからでないと使えなくなります**。

```ts
catch (e) {
  const msg = e instanceof Error ? e.message : String(e);
}
```

### ⑥ `noImplicitReturns` — 戻り値がある道とない道が混ざる

```ts
export function a6(b: boolean) { if (b) return 1; }
```

`b` が真なら `1` が返り、**偽なら何も返りません**（`undefined` になります）。

```ts
const n = a6(false);   // undefined
n + 1;                 // NaN
```

書いた人はたぶん `else` を書き忘れています。**書き忘れを検出する**設定です。

### ⑦ `noPropertyAccessFromIndexSignature` — 存在しないキーを打ち間違えても気づけない

```ts
type T7 = { [k: string]: string };
export const a7 = (({} as T7)).foo;
```

`{ [k: string]: string }` は「**どんな文字列のキーでもいい**」という型です。なので `.foo` も `.fooo` も `.あいうえお` も、全部通ります。

```ts
const config: Record<string, string> = loadConfig();
config.databaseUrl;    // 通る
config.databaseUrI;    // 打ち間違い（l → I）も通る。undefined になる
```

この設定を入れると `config["databaseUrl"]` と**角括弧で書かないといけなくなります**。書き方が変わることで「これは動的なキーだ」と目に見えるようになる、という趣旨です。

### ⑧ `noImplicitOverride` — 親のメソッドを気づかずに壊す

```ts
class P8 { m() {} }
export class C8 extends P8 { m() {} }
```

子クラスが親と同じ名前のメソッドを定義しています。**意図的な上書きなのか、名前がぶつかっただけなのか**、コードからは分かりません。

危ないのは逆のケースです。**親側のメソッド名が変わったとき**、子の上書きは静かに「ただのメソッド」に変わり、**呼ばれなくなります**。

この設定を入れると `override m() {}` と明示することになり、**親に対応するものが無ければエラー**になります。

### ⑨ `noFallthroughCasesInSwitch` — `break` の書き忘れ

```ts
switch (n) { case 1: console.log(1); case 2: return 2; }
```

`case 1:` に `break` がないので、**`n` が 1 のとき `case 2:` も実行されます**。

```ts
a9(1);   // 1 を表示して、さらに 2 を返す
```

意図的な fallthrough もありますが、**ほとんどは書き忘れ**です。意図的なら `// falls through` とコメントを書けば通ります。

---

## B. ESLint 編（8項目）

こちらもまとめて貼れます。

```ts
// ⑩ no-explicit-any
export const b10: any = 1;

// ⑪ no-non-null-assertion
const m = new Map<string, { x: number }>();
export const b11 = m.get("k")!.x;

// ⑫ consistent-type-assertions（assertionStyle: "never"）
export const b12 = {} as { y: number };

// ⑬ ban-ts-comment
// @ts-ignore
export const b13 = 1;

// ⑭ no-floating-promises ★型情報が要る
async function work(): Promise<void> {}
export function b14() { work(); }

// ⑮ no-unsafe-assignment ★型情報が要る
export const b15 = JSON.parse("{}");

// ⑯ await-thenable ★型情報が要る
export async function b16() { return await 1; }

// ⑰ no-base-to-string ★型情報が要る
export const b17 = `${{}}`;
```

### ⑩ `no-explicit-any` — 型チェックを自分で切っている

```ts
export const b10: any = 1;
```

`any` は「**この変数については型チェックをしないでください**」という宣言です。そこから先、何をしても怒られません。

```ts
const data: any = fetchSomething();
data.user.name.toUpperCase();    // 全部通る。実行時に落ちる
```

TypeScript を使う意味が、その変数の周りだけ消えます。

### ⑪ `no-non-null-assertion` — 「絶対にある」と言い切っている

```ts
const m = new Map<string, { x: number }>();
export const b11 = m.get("k")!.x;
```

`Map.get()` は**キーが無ければ `undefined`** を返します。`!` は「**いや、絶対にある**」とコンパイラに言い切る記号です。

```ts
const user = users.get(id)!;    // いない場合を考えていない
user.name;                      // 💥 Cannot read property 'name' of undefined
```

書いた時点では本当に「絶対にある」かもしれません。問題は**半年後にそうでなくなったとき、誰も教えてくれない**ことです。

### ⑫ `consistent-type-assertions` — 「この型だと思え」と命令している

```ts
export const b12 = {} as { y: number };
```

**空のオブジェクトに「`y` という数値がある」と宣言**しています。当然ありません。

```ts
const config = {} as Config;
config.apiUrl;    // 型の上では string。実際は undefined
```

`as` は**確認ではなく命令**です。コンパイラは「そう言うなら」と黙ります。

> 📌 **代わりに型注釈を使うと、確認してくれます。**
> ```ts
> const config: Config = {};   // ← エラーになる（足りないと言われる）
> ```
> 見た目はほぼ同じで、**検査されるかどうかが正反対**です。

### ⑬ `ban-ts-comment` — エラーを黙らせている

```ts
// @ts-ignore
export const b13 = 1;
```

`@ts-ignore` は「**次の行のエラーを無視しろ**」です。何のエラーだったかは記録されません。

`@ts-expect-error` なら「エラーが出るはず」という意味になり、**エラーが出なくなったら逆に怒ってくれます**。使うならこちらです。

### ⑭ `no-floating-promises` — `await` を忘れている ★

```ts
async function work(): Promise<void> {}
export function b14() { work(); }
```

`work()` を呼んでいますが、`await` も `.catch()` も付いていません。**構文としては完全に正しい**ので、TypeScript は何も言いません。

何が起きるか。

```ts
function save() {
  db.write(data);        // await を忘れた
  console.log("保存完了");  // 書き込み前に表示される
}
```

さらに悪いことに、**`db.write` が失敗しても誰も気づきません**。エラーが握り潰されて、成功したように見えます。

> ⚠️ **これは実バグを生みます。** 自分のリポジトリで測ったら **24箇所**ありました。

### ⑮ `no-unsafe-assignment` — 型のない値が流れ込んでいる ★

```ts
export const b15 = JSON.parse("{}");
```

`JSON.parse` の戻り値は **`any`** です。`any` という字はどこにも書いていないので、**⑩の `no-explicit-any` では止まりません**。

```ts
const config = JSON.parse(text);       // any
config.databse.url;                    // 打ち間違いも通る 💥
```

外部から来るデータ（API のレスポンス、設定ファイル、型定義のないライブラリ）は、**だいたいここから入ってきます**。

### ⑯ `await-thenable` — Promise でないものを待っている ★

```ts
export async function b16() { return await 1; }
```

`1` は Promise ではありません。`await 1` は何も待たずにそのまま `1` になります。

これ自体は無害ですが、**書いた人は何かを待っているつもり**です。

```ts
const user = await getUser(id);   // getUser が async でなくなった
                                   // → await は意味を失うが、誰も教えない
```

### ⑰ `no-base-to-string` — `"[object Object]"` になる ★

```ts
export const b17 = `${{}}`;
```

オブジェクトを文字列に埋め込むと、**`"[object Object]"`** になります。

```ts
console.log(`ユーザー: ${user}`);
// → ユーザー: [object Object]

throw new Error(`失敗: ${detail}`);
// → 失敗: [object Object]   ← 原因が分からないエラーメッセージ
```

**ログとエラーメッセージで一番よく見る事故**です。

---

## ★印の4つには、追加の設定が要ります

⑭⑮⑯⑰ は **型情報を要するルール**で、`projectService` を有効にしないと**動きさえしません**。

```js
// eslint.config.js
{
  files: ["src/**/*.ts", "server/**/*.ts"],
  languageOptions: {
    parser: tseslint.parser,
    parserOptions: {
      projectService: true,                    // ← これ
      tsconfigRootDir: import.meta.dirname,
    },
  },
  rules: {
    "@typescript-eslint/no-floating-promises": "warn",
    "@typescript-eslint/no-unsafe-assignment": "warn",
    "@typescript-eslint/await-thenable": "warn",
    "@typescript-eslint/no-base-to-string": "warn",
  },
}
```

⚠️ **`tseslint.configs.strict` を入れていても、これらは入りません。** `strict` は型情報を使わないルールだけです（型情報版は `strictTypeChecked`）。

⚠️ そして **`strictTypeChecked` を丸ごと入れるのは、たぶんやめたほうがいいです。** 自分のリポジトリで測ったら 1,213件出て、**そのうち439件が1つのスタイルルール**でした。本当に見たいものが埋もれます。**上の4つのように名指しで入れる**ほうが実用的です。

なお速度については、**絞っても速くなりません**。コストは型プログラムの構築に付くので、5個でも44個でも同じです（実測）。絞る理由は出力の量です。

---

## ⚠️ Vue を使っている人へ：テンプレート内は見られていません

これは測って驚いたところです。

`<template>` の中に書いた `!` や `as` は、**typescript-eslint のルールが1つも走りません**。

```vue
<script setup lang="ts">
const m = new Map<string, { a: string }>();
const bad = m.get("x")!.a;        // ← error になる
</script>

<template>
  <div>{{ m.get("y")!.a }}</div>  <!-- ← 何も言われない -->
</template>
```

`vue-eslint-parser` は `<script>` を typescript-eslint に渡しますが、**テンプレートの式は同じようには扱われない**ためです。

他のフレームワークも測りました。**Vue だけでした。**

| | script 側 | テンプレート / JSX 側 |
|---|---|---|
| React（`.tsx`） | ✅ | ✅ |
| Solid / Preact | ✅ | ✅ |
| Svelte | ✅ | ✅ |
| Astro | ✅ | ✅ |
| **Vue** | ✅ | ❌ |

回避策は **テンプレートに式を書きすぎないこと**です。ロジックを `<script>` の computed に出せば、そこは見られます。

---

## チェックリスト（印刷用）

新しいリポジトリで、最初に確認する項目です。

### tsconfig

- [ ] `strict`
- [ ] `noUncheckedIndexedAccess` ← **`strict` に含まれない**
- [ ] `exactOptionalPropertyTypes` ← **含まれない**
- [ ] `noImplicitReturns` ← **含まれない**
- [ ] `noPropertyAccessFromIndexSignature` ← **含まれない**
- [ ] `noImplicitOverride` ← **含まれない**
- [ ] `noFallthroughCasesInSwitch` ← **含まれない**

### ESLint

- [ ] `no-explicit-any`
- [ ] `no-non-null-assertion`
- [ ] `consistent-type-assertions`（`assertionStyle: "never"`）
- [ ] `ban-ts-comment`
- [ ] **`projectService: true`**（以下4つの前提）
- [ ] `no-floating-promises` ← **一番バグを生む**
- [ ] `no-unsafe-assignment`
- [ ] `await-thenable`
- [ ] `no-base-to-string`

### 確認コマンド

```bash
# 実際に効いている ESLint ルール（継承後）
npx eslint --print-config src/index.ts

# 継承後の tsconfig 実効値
npx tsc -p tsconfig.json --showConfig

# そのフラグを入れたら何件出るか（入れる前に分かる）
npx tsc -p tsconfig.json --noEmit --noUncheckedIndexedAccess
```

3つとも数秒です。

---

## 既存のプロジェクトに入れるとき

いきなり `error` にすると、既存の違反で全部真っ赤になります。そうすると人は `// eslint-disable` を貼り始め、**ルールが形骸化します**。

順番はこうです。

1. **`warn` で入れる。** CI は緑のまま、件数だけ見える
2. **`--noEmit --<flag>` で何件出るか測る。** 意外と0件のことがあります（うちは tsconfig の5つ全部が0件でした）
3. 1ファイルずつ直す
4. 直しきれないものは**設定ファイルに理由付きで書く**（インラインの `eslint-disable` は使わない）
5. **最後に `error` に上げる**

そして一番の推奨は、**新しいプロジェクトなら1行目を書く前に全部入れること**です。そのときだけ、返済すべき違反が存在しません。

---

## まとめ

- **設定ファイルを読んでも、何が効いているかは分からない。** プリセットが継承されるので
- **`strict` は6つの重要な設定を含んでいない**
- **`any` は構文だけでは見えない。** `JSON.parse` も型なしライブラリも素通りする
- **`no-floating-promises` が一番バグを生む。** `await` 忘れはエラーを握り潰す
- **Vue のテンプレート内は見られていない。** 他のフレームワークは見られている
- **貼って走らせれば、抜けは数秒で分かる**

このサンプルをリポジトリに `check-config.ts` として置いておいて、たまに走らせるのもいいと思います。エラーが減っていたら、**どこかの設定が外れています。**

---

## 関連記事

AI に思い切り書かせるための7本です。どこから読んでも大丈夫ですが、この順に並んでいます。

1. [ターミナルを自作したら、1日のコミット数が500を超えて、生産性がバグった話](https://zenn.dev/singularity/articles/diy-terminal-500-commits) — 並列運用の始まり。道具そのものを作った話
2. [1日500コミットは、もう読めない ── だからコードレビューをやめた](https://zenn.dev/singularity/articles/stopped-reviewing-my-code) — 読まなくても壊れない仕組みの全体像
3. [ユーザーの困りごとは、その日のうちに直す ── 中央値1.2時間、最速9分](https://zenn.dev/singularity/articles/issue-median-one-hour) — 機械に移せなかったものは何か
4. [ESLint と TypeScript の設定、抜けてないと言い切れますか](https://zenn.dev/singularity/articles/what-else-was-off) — 規約に書いても、機械が止めていなければ守られない
5. **その設定、本当に効いてますか ── 貼ると必ずエラーになるサンプル17本** ← **いまここ**
6. [AIでがんがん書く時代の「きれいなコード」の守り方](https://zenn.dev/singularity/articles/clean-code-ci-for-ai-era) — ESLint / SonarJS / jscpd / knip を CI に置く実装編
7. [jscpd で重複コードを機械的に潰す](https://zenn.dev/singularity/articles/jscpd-dry-detection-mono) — 重複検出の詳細。全体監査と CI 差分チェックの二段構え
