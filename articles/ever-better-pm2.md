---
title: "13年もののJSを .ts に改名しただけで型エラーが2,641件出た ── その中身は「無いデータを読んでいる」だった"
emoji: "🪜"
type: "tech"
topics: ["TypeScript", "ESLint", "JavaScript", "AI", "ClaudeCode"]
published: false
publication_name: "singularity"
---

[前の記事](https://zenn.dev/singularity/articles/small-teams-ai)で、**並列で回すには足回りが要る**と書きました。読まなくても壊れない仕組みを先に作らないと、エージェントを増やしても自分が詰まる、という話です。

そこで当然、こう思うわけです。

> **既存のリポジトリには、その足回りが無い。**

自分が最初から作ったものなら、最初から入れておけます。でも**世の中のコードの大半は、そうではありません。**

この記事は、**13年動いている他人のリポジトリに足回りを入れてみた**記録です。

結論から書きます。

> **`.js` を `.ts` に改名しただけで、型エラーが 2,641 件出ました。**
> **中身を見ていくと、どれも「そこに無いデータを読んでいる」という同じ形でした。**

---

## 古いリポジトリに strict な linter を入れると、失敗する

やったことがある人は分かると思います。

```
strict な設定を入れる
  ↓
エラーが4,000件出る
  ↓
「これは無理だ」
  ↓
全部 warn に落とす、または revert
```

**warn に落とすと、何も強制されません。** そして件数は静かに増えていきます。**入れる前より悪い**とも言えます。「入れてある」という安心だけが残るので。

かといって4,000件を先に直すのは、**始める前に終わっています。**

### だから「天井」を凍結する

そこで作ったのが [ever-better](https://github.com/isamu/ever-better) です（MIT・**ランタイム依存ゼロ**）。

考え方は1行です。

> **今日ある違反を全部「天井」として記録する。**
> **その日から、古いコードは免除、新しいコードは全ルールが適用される。**
> **天井は下がることはあっても、上がることはない。**

```bash
npx ever-better diagnose     # 何が足りないか、それぞれ何を意味するか
npx ever-better bootstrap    # 入れる。設定を生成する
npx ever-better freeze       # 今日の違反数を天井として固定
npx ever-better check        # CI ゲート。増えていたら落とす
npx ever-better prune        # 直したぶん、天井を下げる
```

**4,000件を今日直す必要はありません。** ただし**4,001件目は通りません。** これなら初日から入れられます。

Claude Code のプラグインにもしてあるので、`run ever-better on this repo` と言えば、診断・導入・凍結・そこからルール1本ずつの PR まで、勝手に進みます。

---

## 試す相手を選ぶ

自分のリポジトリで試しても意味がありません。**足回りが最初から入っているので。**

そこで **[pm2](https://github.com/Unitech/pm2)** にしました。

```
★43,254
2013年から13年
JavaScript
open issue 1,098件
```

**Node.js のプロセスマネージャとして、たぶん誰でも一度は使っています。** そして**十分に古い。**

[フォークして](https://github.com/isamu/pm2)、動かしました。

---

## 1時間で起きたこと

```
09:09  導入計画と、エージェント向けの指示を追加
09:27  Prettier でツリー全体を整形
09:34  ESLint / TypeScript / 品質ゲートを導入
09:43  違反の天井を 4,942 件で凍結
09:56  ★ ReferenceError を2件修正
09:58  macOS の CI ジョブを直す
10:14  PR 6本マージ完了
```

**約1時間**です。人が張り付いていたのは、判断のところだけです。

### 凍結した中身

72ルールで **3,645件**（このあと書く理由で、4,942 から descope した後の数字）。

```
1,356   id-length                      変数名が短すぎる
  920   @typescript-eslint/no-unused-vars
  197   max-nested-callbacks           コールバックの入れ子
  129   max-lines-per-function
  122   @typescript-eslint/no-this-alias   var self = this
   89   @typescript-eslint/no-unused-expressions
   63   sonarjs/no-unused-vars
   62   sonarjs/no-redundant-boolean
   61   sonarjs/no-nested-functions
   54   sonarjs/no-dead-store           書いたのに誰も読まない代入
   49   sonarjs/cognitive-complexity
   40   complexity
```

**`var self = this` が122件**あるあたりに、13年が見えます。

そして**この数字は「悪い」という意味ではありません。** 2013年に書き始めたコードに、2026年のルールを当てているだけです。**当時は正しかった書き方も混ざっています。**

天井を凍結する道具にしたのは、まさにそこが理由です。**「全部直せ」ではなく「これ以上増やすな」という形にする**。それなら今日から成立します。

---

## そして、linter が出したのはバグでした

ここからが本題です。

`no-undef` ── **未定義の変数を使っている**という、一番基本的なルールです。**まずこれが2件、当てました。**

### ① `pm2 monit` が、エラー時に落ちる

```js
// lib/API/Extra.js:703
that.exitCli(conf.ERROR_EXIT);
```

**`conf` という変数は、このファイルに存在しません。** 同じファイルの他の箇所は全部 `cst.ERROR_EXIT` で、`cst` は8行目で require されています。**ここだけ違う。**

`getMonitorData` が失敗したときに通ります。エラーメッセージを出したあと、**`ReferenceError: conf is not defined` を投げて落ちます。**

### ② タブ補完のインストールが、エラー時に落ちる

```js
function readRc(completer, cb) {          // ← completer を受け取る
  ...
  if (err) return cb(new Error("No " + file + " ... " + completer + " ..."));
}

function writeRc(content, cb) {           // ← 受け取っていない
  ...
  if (err) return cb(new Error("No " + file + " ... " + completer + " ..."));
}                                                                  ↑ 未定義
```

**`readRc` からコピーして引数を1つ減らしたのに、本文の参照を消し忘れています。**

`pm2 completion install` で、rc ファイルが読めないときに通ります。**本来は「No .zshrc file. You'll have to run instead: …」という親切なメッセージが出るはずの場所です。**

実際に確かめました。

```
修正前  ReferenceError: completer is not defined
修正後  Error: No .zshrc file. You'll have to run instead: pm2 completion >> ~/.zshrc
```

### 共通しているのは、実行されにくさ

**2件とも、何かが既に失敗したあとにしか通りません。**

だから13年見つかりませんでした。**普通に動いているときは、絶対に通らない場所です。** そして通ったときには、ユーザーは**作者が書いたメッセージではなく、スタックトレースを見ています。**

本家に PR を出しました → **[Unitech/pm2#6143](https://github.com/Unitech/pm2/pull/6143)**（差分4行）。

---

## 次に、TypeScript に移した

linter は「無いものを使っている」までしか見ません。**その先は型です。**

`lib/` の `.js` 49本を、`git mv` で `.ts` に**改名しただけ**にしました。**ロジックは1行も書き換えていません。**

結果です。

```
                 改名前    改名後
eslint（未抑制）   3,130  →  1,875
型エラー           2,641  →    259
```

**改名しただけで、型エラーが 2,641 件。**

（改名後の259件は、代入だけされていたフィールド18個の宣言を `API.ts` に足し、strict を掛ける対象を17ファイルに絞ったあとの数字です。**残りはまだ `strict:false` で逃がしてあります。**）

そして中身を見ていくと、**「無いデータを読んでいる」ばかりでした。**

---

## 出てきたもの

フォークに issue として11件立てました。代表的なものを挙げます。

### ① プロトタイプが「見つかった」と答えてしまう

**この1件が、この記事で一番言いたいことです。**

pm2 は、アプリ設定の `user` をユーザー表に引きにいきます。

```js
var users = passwd.getUsers();
var user_info = users[app.uid || app.user];
if (!user_info) { /* User ... cannot be found */ }
app.env.HOME = user_info.homedir;
app.uid = parseInt(user_info.userId);
```

**`getUsers()` が返すのは、プロトタイプを持つ普通のオブジェクトです。**

なので `user` が `constructor` / `toString` / `valueOf` / `hasOwnProperty` / `__proto__` のいずれかだと、**継承されたプロパティが返ってきます。**

```js
typeof users['constructor'];  // 'function'
typeof users['toString'];     // 'function'
```

**`!user_info` を通過します。** 「User cannot be found」は出ません。そして `user_info.homedir` が `undefined`、`parseInt(undefined)` で **`app.uid = NaN`**。

**そこに無いデータを読んだのに、JavaScript が答えてしまった。** それだけです。

### ② 同じ形が、別の場所にもあった

`pm2 ls --sort` も同じでした。

```
$ pm2 ls --sort constructor
TypeError: propertyName.split is not a function
```

ソート項目が正しいかを**プレーンオブジェクトへの引き**で検証していたので、`constructor` や `toString` が通ってしまいます。

**同じ間違いが、別の人が書いた別のファイルで、独立に起きています。** 個人のミスではなく、**言語の性質**です。

### ③ タイムアウトの中で落ちる

`disconnectBus` の200msフォールバックが、**インスタンスではなくエクスポートされたコンストラクタ関数**を見ていました。

```js
if (Client.sub_sock.destroy) that.sub_sock.destroy();
//  ↑ コンストラクタ。インスタンスは that
```

`undefined.destroy` を読んで TypeError。**しかも `setTimeout` の中なので、捕まえられずにプロセスが落ちます。**

### ④ その他

- **`pm2 conf`** — module 設定に `null` が1つあると、`typeof null === 'object'` で `Object.keys(null)` に入り、一覧全体がクラッシュ
- **`pm2 start --ext`** — ディレクトリの走査を **other の読み取り権限ビット（`mode & 4`）** で判定していた。pm2 が読めるかどうかとは無関係
- **docker のプロセス操作** — 失敗時にエラーを返さず例外を投げる
- **`sexec`** — 呼び出し側から渡されたオブジェクトを書き換える
- **テストスイート** — 開発者の**実際の `~/.pm2` に対して動く**。失敗するとリトライ前に `pm2 uninstall all` を実行する

---

## 全部、同じ形をしています

```
conf.ERROR_EXIT              conf という変数が存在しない
completer                    引数として受け取っていない
users['constructor']         プロトタイプが答えた。データは無い
fields['toString']           同上
Client.sub_sock              コンストラクタを見ている。インスタンスではない
Object.keys(null)            オブジェクトのつもりが null
```

**全部、そこに無いデータを読んでいる**のです。

そして**型を書いていれば、全部その場で止まります。**

> **型が無いというのは、「このデータが本当にそこにあるか」を誰も確かめていない、ということです。**

`any` にした瞬間、そのチェックは消えます。**消えたことは、どこにも記録されません。** だから13年残ります。

### 別のプロジェクトでも同じでした

同じことを別の既存プロジェクトでもやっています。**935ファイルに `any` が 1,413 箇所**ありました。

そして linter の設定ファイルには、こう書いて `no-explicit-any` が off にしてありました。

> **多すぎて今は直せない**

**責める気にはなりません。** 実際に多すぎるからです。**そして off にした瞬間から、そこは誰も確かめていない場所になります。**

`any` は「型が分からない」ではなく、**型を決める判断を、書かずに済ませた場所**です。判断を書かなければ、間違いも書かれません。**間違いが無いのではなく、間違いを確かめていないだけです。**

---

## 正直に書いておくこと

数字を出す以上、**都合の悪いほうも書きます。**

**天井が 4,933 → 3,637 に下がりましたが、そのうち 1,292 件は「直した」のではありません。** `@typescript-eslint/no-require-imports` を `.js` ファイル全体で対象外にしたぶんです。CommonJS のパッケージで `.js` ファイルが `import` を使えない以上、このルールは**新しいファイルを全部止めるだけで、線を守っていませんでした。**

**範囲を狭めたのであって、良くしたわけではありません。** ツールの出力にもそう記録してあります。**実際に直したのは4件です。**

こういうものを混ぜて「1,296件改善」と書くことはできます。**でもそう書いた瞬間、他の数字も信用されなくなります。**

---

## まとめ

- **古いリポジトリに strict な linter を入れると、4,000件出て revert される。** だから**天井を凍結する**道具を作った（[ever-better](https://github.com/isamu/ever-better)・MIT・依存ゼロ）
- ★43,254・13年ものの [pm2](https://github.com/Unitech/pm2) に当てたら、**1時間で 4,942 件を凍結**できた
- **`no-undef` だけで本物のバグが2件**。どちらも**エラー処理の中**で、13年見つかっていなかった → [本家に PR](https://github.com/Unitech/pm2/pull/6143)
- **`.js` を `.ts` に改名しただけで、型エラーが 2,641 件**（eslint も 3,130 件）
- そこから**バグを11件**切り出した。うち2件は**プロトタイプが「見つかった」と答えてしまう**もの
- **全部「そこに無いデータを読んでいる」という同じ形**だった
- **型が無いというのは、そこにデータが本当にあるかを誰も確かめていない、ということ**

古いコードを責める話ではありません。**13年動き続けているコードのほうが、たいていのコードより優秀です。**

ただ、**確かめていない場所は、確かめるまで分からない。** それだけです。

そして**確かめるのは、いまは1時間でできます。**

---

## 関連記事

- [AIに大きな仕様書を渡して一気に作らせるのは、宝くじを買うようなものです](https://zenn.dev/singularity/articles/small-teams-ai) — この記事の前提。**足回りが要る**と書いた回
- [1日500コミットは、もう読めない ── だからコードレビューをやめた](https://zenn.dev/singularity/articles/stopped-reviewing-my-code) — 読まなくても壊れない仕組みの全体像
- [strict を入れても as は止まらない](https://zenn.dev/singularity/articles/what-else-was-off) — その検査が本当に効いているかを数えた話
