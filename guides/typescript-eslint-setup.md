# TypeScript / ESLint 設定ガイド（エージェント向け）

**このファイルは、AI エージェントが他プロジェクトの設定を自動で整えるための指示書です。**
人間が読んでも構いませんが、書き方は「エージェントが手順として実行できること」を優先しています。

対象: **React / Vue / Node / モノレポ / サーバとフロントの同梱**（`receptron/mulmoterminal` はこの最後の形です）

---

## 0. 最初にやること — 推測しない

**設定ファイルを読んで判断してはいけません。** プリセットが継承されるので、ファイルの中身と実効値は一致しません。

必ずこの3つを実行して、**現状を出力させてから**始めます。

```bash
# 実際に効いている ESLint ルール（ファイルを1つ指定する）
npx eslint --print-config src/index.ts > /tmp/current-eslint.json

# 継承後の tsconfig 実効値
npx tsc -p tsconfig.json --showConfig

# あるフラグを入れたら何件出るか（入れる前に分かる）
npx tsc -p tsconfig.json --noEmit --pretty false --noUncheckedIndexedAccess 2>&1 | grep -c "error TS"
```

⚠️ **`tsc` は必ずプロジェクトのものを使う**こと。`npx tsc` が別のツールを拾う環境があります。疑わしければ `./node_modules/.bin/tsc --version` で確認します。

⚠️ **`--pretty false` を付ける**こと。付けないと ANSI カラーが混ざって `grep -c` が正しく数えられません。

---

## 1. tsconfig — `strict` だけでは足りない

### `strict: true` が含むもの（8つ・自動）

`noImplicitAny` / `strictNullChecks` / `strictFunctionTypes` / `strictBindCallApply` /
`strictPropertyInitialization` / `noImplicitThis` / `useUnknownInCatchVariables` / `alwaysStrict`

### `strict` が含まないもの — 個別に書く

```jsonc
{
  "compilerOptions": {
    "strict": true,

    // 以下は strict に含まれない。全部入れる
    "noUncheckedIndexedAccess": true,        // arr[i] を T | undefined にする
    "exactOptionalPropertyTypes": true,      // a?: string と a: string | undefined を区別
    "useUnknownInCatchVariables": true,      // strict に含まれるが明示してよい
    "noImplicitOverride": true,              // 親のメソッド上書きに override を必須化
    "noImplicitReturns": true,               // 戻る道と戻らない道の混在を禁止
    "noFallthroughCasesInSwitch": true,      // break 忘れを検出
    "noPropertyAccessFromIndexSignature": true  // 添字型へのドットアクセスを禁止
  }
}
```

### 導入の順序 — 件数を測ってから決める

```bash
for f in noUncheckedIndexedAccess exactOptionalPropertyTypes noImplicitOverride \
         noImplicitReturns noFallthroughCasesInSwitch noPropertyAccessFromIndexSignature; do
  n=$(./node_modules/.bin/tsc -p tsconfig.json --noEmit --pretty false --$f 2>&1 | grep -c "error TS")
  echo "$f: $n"
done
```

- **0件のものは即入れる**（移行作業ゼロ）
- **件数が多いものは別 PR にする**。特に `noUncheckedIndexedAccess` は数百件になり得ます

### `noUncheckedIndexedAccess` を入れるときの型付け

数が多くても**設計判断は要りません**。パターンは4つだけです。

```ts
// ① 一度 const に受けて undefined を弾く
const line = lines[i];
if (line === undefined) return;

// ② ?? で既定値
const ext = (mimeType.split(";")[0] ?? "").trim();

// ③ ?. にする
const body = wrapped?.[1] ?? content;

// ④ 境界チェック済みなら早期 return
if (!def) return null;
```

### テストは対象外にする

テストは自分で作った fixture を `rows[0]` で引きます。そこの `T | undefined` は**テスト自身が保証している値についての雑音**なので、テスト用 tsconfig で明示的に off にします。

```jsonc
// tsconfig.test.json
{ "compilerOptions": { "noUncheckedIndexedAccess": false } }  // 理由をコメントで残す
```

⚠️ **`exactOptionalPropertyTypes` はテストでも on のまま**にします。こちらは fixture の作り方の問題ではなく、型の意味の問題なので。

---

## 2. ESLint — プリセットの分担を知る

### どのプリセットに何が入っているか（実測）

| ルール | `recommended` | `strict` | `stylistic` | `recommendedTypeChecked` | `strictTypeChecked` |
|---|---|---|---|---|---|
| `no-explicit-any` | ✅ | ✅ | — | ✅ | ✅ |
| `no-non-null-assertion` | — | ✅ | — | — | ✅ |
| **`consistent-type-assertions`** | — | **—** | **✅** | — | **—** |
| `ban-ts-comment` | ✅ | ✅ | — | ✅ | ✅ |
| `no-floating-promises` | — | — | — | ✅ | ✅ |
| `no-unsafe-*` | — | — | — | ✅ | ✅ |
| `await-thenable` | — | — | — | ✅ | ✅ |
| `no-base-to-string` | — | — | — | ✅ | ✅ |

**罠が2つあります。**

1. **`as` を止める `consistent-type-assertions` は `stylistic` にしかない。** `strict` を入れても入りません
2. **型情報を使うルールは `TypeChecked` 版にしかない。** しかも `parserOptions` の設定なしでは**動きさえしません**

### ベースの構成

```js
import tseslint from "typescript-eslint";
import sonarjs from "eslint-plugin-sonarjs";

export default [
  ...tseslint.configs.strict,       // any 禁止、! 禁止
  ...tseslint.configs.stylistic,    // ← as を止めるのはこちら
  sonarjs.configs.recommended,
  {
    files: ["**/*.{ts,tsx,mts,cts}", "**/*.vue"],
    rules: {
      "@typescript-eslint/consistent-type-assertions": ["error", { assertionStyle: "never" }],
      "@typescript-eslint/no-explicit-any": "error",
      "@typescript-eslint/no-non-null-assertion": "error",
      "@typescript-eslint/ban-ts-comment": "error",
    },
  },
];
```

⚠️ **`strictTypeChecked` を丸ごと入れてはいけません。** 実測で 1,213件出て、**そのうち439件が `restrict-template-expressions` 1つ**でした。スタイル指摘が本命を埋めます。**ルールを名指しで入れます**（次項）。

なお**絞っても速くなりません**。コストは型プログラムの構築に付くので、5個でも44個でも同じです（実測）。絞る目的は**出力の量**です。

---

## 3. 型情報を使うルール — 別ブロックで足す

### 最低限これだけ

```js
{
  files: ["server/**/*.ts", "src/**/*.ts", "common/**/*.ts"],
  ignores: ["**/*.spec.ts", "**/*.test.ts"],   // 型プログラムを小さく保つ
  languageOptions: {
    parser: tseslint.parser,
    parserOptions: {
      project: ["./tsconfig.app.json", "./tsconfig.server.json"],
      tsconfigRootDir: import.meta.dirname,
    },
  },
  rules: {
    "@typescript-eslint/no-floating-promises": "warn",
    "@typescript-eslint/no-misused-promises": "warn",
    "@typescript-eslint/await-thenable": "error",
    "@typescript-eslint/no-base-to-string": "error",
  },
}
```

### `projectService: true` か、`project` の明示か

**モノレポや tsconfig が複数ある構成では `project` を明示します。**

`projectService: true` は root の `tsconfig.json` を見ますが、それが `references` だけを持つ構成だと、**サービスが `server/**` のファイルを配置できず、大量のパースエラーになります**（実測: 321件）。

| 構成 | 指定 |
|---|---|
| tsconfig が1つ | `projectService: true` |
| **app / server / test に分かれている** | **`project: [...]` で全部名指し** |
| モノレポ（workspaces） | 各パッケージの tsconfig を名指し、または `projectService` + `allowDefaultProject` |

---

## 4. フレームワーク別の注意

### Vue — 2ブロックに分ける（一番間違えやすい）

**① `.vue` のブロックで、型情報を `vue-eslint-parser` 越しに渡す**

```js
{
  files: ["**/*.vue"],
  languageOptions: {
    parser: vueParser,
    parserOptions: {
      parser: tseslint.parser,                 // vue-eslint-parser がこれに渡す
      project: ["./tsconfig.app.json"],
      tsconfigRootDir: import.meta.dirname,
      extraFileExtensions: [".vue"],           // 必須
    },
  },
}
```

**② 型情報ルールは、`languageOptions` を持たない別ブロックで足す**

```js
{
  files: ["src/**/*.vue"],
  // languageOptions を書かない。書くと vue-eslint-parser が置き換わり、全 SFC が壊れる
  rules: {
    "@typescript-eslint/no-floating-promises": "warn",
    "@typescript-eslint/no-misused-promises": "warn",
  },
}
```

⚠️ 型情報ブロックの `files` に `.vue` を足すと、そこに書いた `parser: tseslint.parser` が **`vue-eslint-parser` を置き換えます**。flat config は後のブロックが前を上書きするためです。結果:

```
1:8  error  Parsing error: '>' expected
```

**全 SFC がパースできなくなります。** ルールだけのブロックに分けるのが正解です。

⚠️ **`<template>` の中は、これをやっても typescript-eslint のルールが走りません。** 届くのは `<script>` までです。テンプレートに式を書きすぎず、`<script>` の computed に出してください。

### React / Solid / Preact（`.tsx`）

**追加の配線は不要**です。JSX は TypeScript の文法の一部なので、`.tsx` を `files` に含めれば JSX 内の式もルールが走ります。

```js
files: ["src/**/*.{ts,tsx}"]
```

### Svelte / Astro

テンプレート内も**カバーされます**（Vue と違います）。それぞれのパーサーを噛ませて、`parserOptions.parser` に `tseslint.parser` を渡します。

```js
{ files: ["**/*.svelte"], languageOptions: { parser: svelteParser, parserOptions: { parser: tseslint.parser } } }
{ files: ["**/*.astro"],  languageOptions: { parser: astroParser,  parserOptions: { parser: tseslint.parser } } }
```

### Node 専用プロジェクト

`.vue` / `.tsx` のブロックは不要です。`tsconfig` が1つなら `projectService: true` で足ります。

### モノレポ

- 型情報ブロックの `project` に**各パッケージの tsconfig を全部**列挙する
- または `projectService: true` + `allowDefaultProject` で拾えない範囲を補う
- **`ignores` にビルド成果物（`dist` / `build`）を必ず入れる**

---

## 5. 導入の順序（既存プロジェクト）

いきなり `error` にすると既存の違反で真っ赤になり、**人は `// eslint-disable` を貼り始めます**。そうなるとルールは形骸化します。

```
1. warn で入れる（CI は緑のまま、件数だけ見える）
2. 件数を測る（0件のものは即 error にできる）
3. 1ファイルずつ直す。PR は分割する
4. 直せないものは設定ファイルに理由付きで書く（インライン disable は使わない）
5. 最後に error へ上げる
```

**新規プロジェクトなら、1行目を書く前に全部入れます。** そのときだけ返済すべき違反が存在しません。

---

## 6. 例外の書き方

**インラインの `// eslint-disable` は使いません。** 現場に埋もれて見えなくなります。

設定ファイルの allowlist に、**1エントリ1理由**で書きます。

```js
{
  // 型が間違っているのが自分のコードではない場所。それぞれ、どの upstream の
  // 何が直れば消せるかを書く。直ったらエントリを削除する。
  files: [
    // @modelcontextprotocol/sdk が `implements Transport` と書いたクラスで
    // onclose/onerror/onmessage を `T | undefined` と宣言している（Transport は `?: T`）。
    // exactOptionalPropertyTypes 下でインターフェースを満たさない。宣言のバグ。
    // upstream: https://github.com/modelcontextprotocol/typescript-sdk/issues/2083
    "server/routes/mcp-routes.ts",
  ],
  rules: { "@typescript-eslint/consistent-type-assertions": "off" },
}
```

**本家の型が壊れている場合は、本家に issue を立ててから allowlist に入れます。** 「面倒だから」で入れるものではありません。

---

## 7. ルール同士が矛盾することがある

実例です。

| | |
|---|---|
| `no-floating-promises` | 意図的に待たない Promise は `void p` で明示せよ |
| `sonarjs/void-use` | `void` 演算子を使うな |

**正面から矛盾します。** `await` 忘れを捕まえる側を選び、`void-use` を off にしました。**理由を config に書いて**おきます。

同様に、型情報を入れると **これまで動いていなかったルールが一斉に目を覚まします**。その中には偽陽性が混ざります。

⚠️ **偽陽性を見たら、まず「別の設定が足りないせいでは」と疑ってください。** 実例:

`sonarjs/different-types-comparison` が9件の偽陽性を出しました。`noUncheckedIndexedAccess` が無いと `arr[i]` の型が `T` になるので、**実行時に必要なガードが「常に真」に見えていた**のが原因です。フラグを入れたら **9件 → 4件**になりました。

> **ルールが間違っているのではなく、渡している型が嘘だから、正しく推論して間違った結論に着く。**

---

## 8. 検証コマンド — 設定が効いているか確かめる

このファイルを作って、`tsc` と `eslint` を走らせます。**エラーが出なかった行が、抜けている設定**です。

```ts
// === tsconfig ===
export function c1(x) { return x + 1; }                       // strict
export const c2: string = null;                               // strictNullChecks
const arr: string[] = []; export const c3 = arr[0].length;    // noUncheckedIndexedAccess
type T4 = { a?: string }; export const c4: T4 = { a: undefined };  // exactOptionalPropertyTypes
export function c5() { try { throw 1 } catch (e) { return e.message } }  // useUnknownInCatchVariables
export function c6(b: boolean) { if (b) return 1; }           // noImplicitReturns
type T7 = { [k: string]: string }; export const c7 = (({} as T7)).foo;  // noPropertyAccessFromIndexSignature
class P8 { m() {} } export class C8 extends P8 { m() {} }     // noImplicitOverride
export function c9(n: number) { switch (n) { case 1: console.log(1); case 2: return 2; } return 0; }  // noFallthroughCasesInSwitch

// === ESLint ===
export const e1: any = 1;                                     // no-explicit-any
const m = new Map<string, { x: number }>(); export const e2 = m.get("k")!.x;  // no-non-null-assertion
export const e3 = {} as { y: number };                        // consistent-type-assertions
// @ts-ignore
export const e4 = 1;                                          // ban-ts-comment
async function work(): Promise<void> {} export function e5() { work(); }  // no-floating-promises ★
export const e6 = JSON.parse("{}");                           // no-unsafe-assignment ★
export async function e7() { return await 1; }                // await-thenable ★
export const e8 = `${{}}`;                                    // no-base-to-string ★
```

★は型情報が必要なので、`parserOptions` の設定なしでは反応しません。

⚠️ Vue のプロジェクトでは、**同じコードを `.vue` の `<template>` に貼っても反応しません**。これは仕様なので、テンプレート内は別途注意します。

---

## 9. チェックリスト

### tsconfig
- [ ] `strict`
- [ ] `noUncheckedIndexedAccess`（テストは off にしてよい）
- [ ] `exactOptionalPropertyTypes`（**テストも on**）
- [ ] `useUnknownInCatchVariables`
- [ ] `noImplicitOverride`
- [ ] `noImplicitReturns`
- [ ] `noFallthroughCasesInSwitch`
- [ ] `noPropertyAccessFromIndexSignature`

### ESLint
- [ ] `tseslint.configs.strict`
- [ ] **`tseslint.configs.stylistic`**（`as` を止めるのはこれ）
- [ ] `consistent-type-assertions` を `assertionStyle: "never"` で error
- [ ] 型情報ブロック（`project` を明示 or `projectService`）
- [ ] `no-floating-promises` / `no-misused-promises` / `await-thenable` / `no-base-to-string`
- [ ] テストは型情報ブロックの `ignores` に入っている
- [ ] Vue なら**2ブロックに分かれている**
- [ ] 例外はインライン disable ではなく allowlist に理由付き

### 運用
- [ ] `yarn typecheck` が**全 tsconfig を1コマンドで**カバーしている
- [ ] CI で lint と typecheck の両方が走る
- [ ] warn のものが「返すべきバックログ」として認識されている

---

## 参照実装

- [receptron/mulmoterminal](https://github.com/receptron/mulmoterminal) — Vue + Node サーバ同梱。`eslint.config.js` に**各判断の理由がコメントで残っています**
- [receptron/mulmoclaude](https://github.com/receptron/mulmoclaude) — モノレポ（workspaces）

**設定を変えるときは、必ず理由をコメントに書いてください。** 後から来た人（や別のエージェント）が同じ調査をやり直さずに済みます。
