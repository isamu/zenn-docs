---
title: "「as を数えたら90箇所」の続き ── ついでに他も測ったら、もっと大きい穴があった"
emoji: "📏"
type: "tech"
topics: ["TypeScript", "ESLint", "ClaudeCode", "AI", "vibecoding"]
published: false
publication_name: "singularity"
---

前回、規約に「`as` を使うな」と書いてあるのに90箇所あった、という話を書きました。

その修正を進めながら、ふと思いました。

**他にも同じことが起きているのでは。**

規約に書いただけで機械が止めていないものが、`as` 以外にもあるはずです。そう思って測ったら、**`as` より大きい穴が2つ**出てきました。

:::message
この記事は[「as を使うな」と規約に書いた。数えたら90箇所あった](https://zenn.dev/singularity/articles/as-in-the-age-of-agents)の続きです。前回を読んでいなくても分かるように書いていますが、動機はそちらにあります。
:::

## まず、何が有効になっているのかを出す

設定ファイルを読んでも分かりません。プリセットが継承されるので、**最終的に何が有効なのか**は別途出す必要があります。

ESLint にはそのためのコマンドがあります。

```bash
npx eslint --print-config src/main.ts
```

これで、そのファイルに実際に適用されるルールが全部出ます。型まわりだけ抜き出したのがこれです。

| ルール | 状態 |
|---|---|
| `no-explicit-any` | 🔴 error |
| `no-non-null-assertion` | 🔴 error |
| `consistent-type-assertions` | 🔴 error（前回入れた） |
| `ban-ts-comment` | 🔴 error |
| **`no-unsafe-assignment`** | ❌ **未設定** |
| **`no-unsafe-member-access`** | ❌ **未設定** |
| **`no-unsafe-call` / `-return` / `-argument`** | ❌ **未設定** |
| **`no-floating-promises`** | ❌ **未設定** |
| **`no-misused-promises`** | ❌ **未設定** |
| **`await-thenable`** | ❌ **未設定** |
| **`no-base-to-string`** | ❌ **未設定** |

未設定のものには共通点があります。**全部、型情報を要するルール**です。

つまり **型情報を使うルールが、1つも入っていませんでした。**

## なぜこれが `as` より大きい穴なのか

`no-explicit-any` は入っています。だから `any` は止まっている、と思っていました。

**止まっていません。** このルールは **`any` という字が書かれている場所しか見られない**からです。

構文だけでは見えない `any` がこれだけあります。

- **型定義のないライブラリ**から流れ込んでくる値
- **`JSON.parse()` の戻り値**（`any` です）
- `as unknown as T` の二段キャスト（前回消しましたが、**消した先が `any` なら同じこと**）

そして、構文ルールでは**原理的に**捕まらないものがあります。

- **`await` の付け忘れ** — 構文としては完全に正しい
- 同期しか受けない API に **async コールバックを渡す** — 型を見ないと分からない
- オブジェクトが **`"[object Object]"`** になる — 文字列連結として合法

**前回消した `as` は「嘘をついているが依存していない」ものが多かった**のですが、こちらは違います。`await` の付け忘れは、**エラーが握り潰されて何も起きなかったように見えます**。

## 入れて数えた：165件

9ルールを有効にして走らせました。

| ルール | 件数 |
|---|---|
| `no-unsafe-member-access` | 65 |
| `no-unsafe-assignment` | 49 |
| **`no-floating-promises`** | **24** |
| `no-unsafe-argument` | 14 |
| `no-unsafe-call` | 8 |
| `no-unsafe-return` | 3 |
| **`no-misused-promises`** | **2** |
| `await-thenable` | 0 |
| `no-base-to-string` | 0 |
| **合計** | **165** |

**所要 11 秒**です（テストを除いた本体のみ）。

`no-unsafe-*` の139件は「型が付いていない」であって、必ずしもバグではありません。**問題は `no-floating-promises` の24件**です。これは「Promise を作ったのに `await` も `.catch()` も付けていない」なので、**握り潰された失敗が24箇所ある可能性**があります。

## 「重いからやめた」ではなかった

型情報ルールを入れていない理由として、僕は「重いから」だと思っていました。**違いました。**

姉妹プロジェクトの設定ファイルに、当時の判断が書き残してありました。

> that program is the whole cost (**measured: five rules cost the same as all 44**)

**コストは型プログラムの構築に付きます。** ルールを5個に絞っても、44個入れても、同じ時間がかかる。**絞っても速くなりません。**

では何のために絞ったのか。**出力の量**でした。

> The rest of `strictTypeChecked` ... is dominated by style — **`restrict-template-expressions` alone accounts for 439 of its 1213 findings**

`strictTypeChecked` を丸ごと入れると1,213件出て、**そのうち439件が1ルールのスタイル指摘**です。本当に見たい2つ（構文で見えない `any` と、`await` の付け忘れ）が、その中に埋もれます。

**「全部入れる」と「入れない」の間に、「選んで入れる」がある。** そしてその選択は、一度全部動かして出力を見ないとできません。

## tsconfig も測った

同じ発想で、コンパイラ側も見ました。ここにも罠があります。

**`tsconfig` は継承するので、ファイルを読んでも実効値は分かりません。** これで出します。

```bash
npx tsc -p tsconfig.server.json --showConfig
```

結果です。

| 設定 | app | server | test |
|---|---|---|---|
| `strict` | ✅ | ✅ | ✅ |
| **`noUncheckedIndexedAccess`** | ❌ | ❌ | ❌ |
| **`useUnknownInCatchVariables`** | ❌ | ❌ | ❌ |
| **`noImplicitReturns`** | ❌ | ❌ | ❌ |
| **`noPropertyAccessFromIndexSignature`** | ❌ | ❌ | ❌ |
| **`noImplicitOverride`** | ❌ | ❌ | ❌ |

`strict` は付いています。だから大丈夫だと思っていました。

**`strict` はこれらを含みません。** `strict` は8つのフラグの束で、上の5つはその外側にあります。よく知られたことのはずですが、**自分のリポジトリで確認したことはありませんでした**。

## 全部、0件で入った

フラグを立ててコンパイルして、エラーを数えました。

```
noUncheckedIndexedAccess            server 0 / app 0
useUnknownInCatchVariables          server 0 / app 0
noImplicitReturns                   server 0 / app 0
noPropertyAccessFromIndexSignature  server 0 / app 0
noImplicitOverride                  server 0 / app 0
```

**5つとも0件。** 移行作業ゼロで、今日入れて何も壊れません。

姉妹プロジェクトでも測ったら、そちらも**4つとも0件**でした。

つまり、**入れていなかった理由は「大変だから」ではなく、「測っていなかったから」**でした。大変かどうかを確かめないまま、入っていない状態が続いていた。

## 設定の欠落が、別の設定の欠落を呼んでいた

これが一番こたえた発見です。

`eslint.config.js` に、あるルールを off にした理由が書いてありました。

```js
// 型の上で結果が変わらない比較を不要と指摘するルール。
// noUncheckedIndexedAccess が off だと実行時に必要なガードまで
// 「常に真」に見えるので、消すと落ちる。
"sonarjs/different-types-comparison": "off",
```

読み直すと、こう言っています。

**「`noUncheckedIndexedAccess` が off だから、このルールは使えない」**

そして `noUncheckedIndexedAccess` は、**0件で入ります**。

つまりこの off は、**先に tsconfig を直していれば必要なかった**可能性が高い。穴が穴を呼んで、しかも**その理由がちゃんとコメントに書いてあった**。書いてあるのに、上流を直すという発想にならなかった。

理由をコメントに残すのは正しい習慣です。ただ、**その理由が「別の設定が足りないから」だったときに、上流を直しに行く**ところまでが対でした。

## で、何が言えるか

前回の記事の結論はこうでした。

> `CLAUDE.md` は**読ませるもの**、ESLint は**止めるもの**。この2つは別。

今回はその先です。**止める側の設定も、書いただけでは効いていません。**

- `no-explicit-any` は入っていた。でも**構文で見える `any` しか止めていなかった**
- `strict` は入っていた。でも**`strict` が含まないものは入っていなかった**
- ルールを off にした理由は書いてあった。でも**その理由の上流は直していなかった**

どれも「入れ忘れ」ではありません。**入っていると思っていた**のが実態です。

そして共通する対処は1つでした。

> **設定ファイルを読むのではなく、実効値を出力させて、数える。**

```bash
npx eslint --print-config <file>        # 実際に効いているルール
npx tsc -p <config> --showConfig        # 継承後の実効値
npx tsc -p <config> --noEmit --<flag>   # そのフラグを入れたら何件出るか
```

3つとも、実行に数秒から十数秒しかかかりません。**やらなかった理由は、思いつかなかったからです。**

## まとめ

- 型情報を要するルールが**1つも入っていなかった**。入れたら **165件**
- そのうち **`no-floating-promises` の24件**は、握り潰された失敗の可能性がある
- **`strict` は `noUncheckedIndexedAccess` などを含まない。** 5つ足りなかった
- そしてその5つは、**全部0件で入った**。大変だったのではなく、測っていなかった
- **設定の欠落が別の設定を off にさせていた**。理由はコメントに書いてあったのに、上流を直しに行かなかった
- 対処は共通で、**実効値を出力させて数える**。数秒で終わる

作業は issue にしてあります。

- [mulmoterminal#1300](https://github.com/receptron/mulmoterminal/issues/1300) — 型情報ルールの導入（165件）
- [mulmoterminal#1301](https://github.com/receptron/mulmoterminal/issues/1301) — tsconfig の5フラグ
- [mulmoclaude#2736](https://github.com/receptron/mulmoclaude/issues/2736) — 同（4フラグ）

**自分のリポジトリでも、3つのコマンドを打ってみるといいと思います。** 入っていると思っているものが、入っているとは限りません。

---

## 関連記事

AI に思い切り書かせるための7本です。どこから読んでも大丈夫ですが、この順に並んでいます。

1. [ターミナルを自作したら、1日のコミット数が500を超えて、生産性がバグった話](https://zenn.dev/singularity/articles/diy-terminal-500-commits) — 並列運用の始まり。道具そのものを作った話
2. [1日500コミットは、もう読めない ── だからコードレビューをやめた](https://zenn.dev/singularity/articles/stopped-reviewing-my-code) — 読まなくても壊れない仕組みの全体像
3. [ユーザーの困りごとは、その日のうちに直す ── 中央値1.2時間、最速9分](https://zenn.dev/singularity/articles/issue-median-one-hour) — 機械に移せなかったものは何か
4. [「as を使うな」と規約に書いた。数えたら90箇所あった](https://zenn.dev/singularity/articles/as-in-the-age-of-agents) — 規約に書くことと、規約で止めることは別だった
5. **「as を数えたら90箇所」の続き ── ついでに他も測ったら、もっと大きい穴があった** ← **いまここ**
6. [AIでがんがん書く時代の「きれいなコード」の守り方](https://zenn.dev/singularity/articles/clean-code-ci-for-ai-era) — ESLint / SonarJS / jscpd / knip を CI に置く実装編
7. [jscpd で重複コードを機械的に潰す](https://zenn.dev/singularity/articles/jscpd-dry-detection-mono) — 重複検出の詳細。全体監査と CI 差分チェックの二段構え
