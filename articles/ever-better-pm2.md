---
title: "1日500コミットできる環境を整えるツールを作った ── TS/JS なら Claude Code に渡すだけ"
emoji: "🪜"
type: "tech"
topics: ["TypeScript", "ESLint", "JavaScript", "AI", "ClaudeCode"]
published: false
publication_name: "singularity"
---

[1日500コミット](https://zenn.dev/singularity/articles/diy-terminal-500-commits)も、[コードレビューをやめた](https://zenn.dev/singularity/articles/stopped-reviewing-my-code)も、**足回りがあって初めて成立します。** 読まなくても壊れない状態を先に作らないと、エージェントを何体並べても自分が詰まります。

ここで正直に書いておくと、**その足回りが最初から入っていたことは、一度もありません。**

プロトタイプのつもりで始めるので、設定は適当です。動くものができてから、途中で入れて、少しずつ厳しくしていく。**毎回そうなります。**

そして毎回、**「次は最初から入れよう」と誓います。** そして次も入っていません。

しかも自分のものだけではありません。**人のプロジェクトに入ることもあります。** そこには当然、自分の足回りはありません。

つまり、**この作業は毎回発生します。**

```
自分の新しいリポジトリ    プロトで始めるので、途中から入れる
自分の古いリポジトリ      入っていない期間のぶんが溜まっている
人のプロジェクト          入っていない。しかも自分で最初から作れない
```

**毎回、同じことを手でやっていました。**

> **もう苦労しないために、道具にしました。**

**[ever-better](https://github.com/isamu/ever-better)**（MIT・**ランタイム依存ゼロ**）。TS/JS のリポジトリを渡せば、**足りない品質ツールを診断して、入れて、いまの違反数を「天井」として固定します。**

Claude Code のプラグインにしてあるので、実際にはこう使います。

```
run ever-better on this repo
```

**これだけです。** そこから先は勝手に進みます。

後半は、**★43,254・13年もののリポジトリに実際に当てた記録**です。**型エラーが2,641件出ました。**

---

## なぜ必要なのか

古いリポジトリに strict な linter を入れたことがある人は、これを知っていると思います。

```
strict な設定を入れる
  ↓
エラーが4,000件出る
  ↓
「これは無理だ」
  ↓
全部 warn に落とす、または revert
```

**warn に落とすと、何も強制されません。** そして件数は静かに増えます。**入れる前より悪い**とも言えます。「入れてある」という安心だけが残るので。

かといって4,000件を先に直すのは、**始める前に終わっています。**

### 天井を凍結する

考え方は1行です。

> **今日ある違反を全部「天井」として記録する。**
> **その日から、古いコードは免除、新しいコードは全ルールが適用される。**
> **天井は下がることはあっても、上がることはない。**

**4,000件を今日直す必要はありません。** ただし**4,001件目は通りません。** これなら初日から入れられます。

---

## 何をするツールか

CLI は5つです。

```bash
npx ever-better diagnose     # 読むだけ。何が足りないか、それぞれ何を意味するか
npx ever-better bootstrap    # 入れる。設定を生成する
npx ever-better freeze       # 今日の違反数を天井として固定
npx ever-better check        # CI ゲート。1件でも増えていたら落とす
npx ever-better prune        # 直したぶん、天井を下げる
```

`freeze` が要です。**ルールごとに現在の違反数を記録**して `.ever-better/state.json` に持ちます。`check` はそれと比べるだけです。

### Claude Code に渡すとどうなるか

```
/plugin marketplace add isamu/ever-better
/plugin install ever-better
```

そのうえで `run ever-better on this repo` と言うと、**この順で PR が飛んできます。**

```
1  整形の PR（Prettier）
2  ツール導入の PR（ESLint / TypeScript / ゲート）
3  凍結の PR（天井を固定）
4  以降、ルール1本ずつの PR
```

4番目が本体です。**1ルールずつ、違反を直して、直せるようにするために pure 関数を切り出して、テストを書いて、天井を下げます。**

**手つかずのリポジトリだと、これは数本では済みません。** 走らせる前に知っておいたほうがいいです。

### CLI だけでも、止血まではできる

エージェントを使わない場合でも、**「これ以上増やさない」状態は CLI だけで作れます。**

```
diagnose / bootstrap / freeze / check   →  CLI だけで完結する
違反を減らす                            →  ここから先はエージェントの仕事
```

`bootstrap` は依存の追加・設定生成・3プラットフォームの CI・`dependabot.yml` まで入れます。**そこまで済ませて `freeze` すれば、その日から件数は増えません。**

**減らすほうは、コードを読んで直してテストを書く作業**なので、CLI にはありません（`prune` は「直したあとに天井を下げる」だけです）。

**止血は CLI、治療はエージェント。** そう分かれています。

### 決めないことは、決めない

**コードから決められることは全部決めます。** そうでないものは **issue を立てて先へ進みます。**

- 挙動が曖昧なもの（ここは throw すべきか、retry か、log か）
- 公開 API の変更
- それ自体が別プロジェクトになる規模のリファクタ
- **そのリポジトリではそのルールが単に間違っている**場合

issue には**選択肢と、自分ならどれを採るか**を書きます。**判断を人に返して、手は止めない。**

---

## で、実際に試した

自分のリポジトリで試しても意味がありません。**足回りが最初から入っているので。**

**[pm2](https://github.com/Unitech/pm2)** を選びました。

```
★43,254
2013年から13年
JavaScript
open issue 1,098件
```

Node.js のプロセスマネージャとして、**たぶん誰でも一度は使っています。** そして十分に古い。

[フォークして](https://github.com/isamu/pm2)、`run ever-better on this repo` と言いました。

### 1時間で、ここまで来ました

```
09:09  導入計画と、エージェント向けの指示を追加
09:27  Prettier でツリー全体を整形
09:34  ESLint / TypeScript / 品質ゲートを導入
09:43  違反の天井を 4,942 件で凍結
09:56  ★ ReferenceError を2件修正
09:58  macOS の CI ジョブを直す
10:14  PR 6本マージ完了
```

**人が張り付いていたのは、判断のところだけです。**

### 凍結した中身

72ルールで3,645件（後述の理由で 4,942 から descope した後の数字）。

```
1,356   id-length                          変数名が短すぎる
  920   @typescript-eslint/no-unused-vars
  197   max-nested-callbacks               コールバックの入れ子
  129   max-lines-per-function
  122   @typescript-eslint/no-this-alias   var self = this
   89   @typescript-eslint/no-unused-expressions
   63   sonarjs/no-unused-vars
   62   sonarjs/no-redundant-boolean
   61   sonarjs/no-nested-functions
   54   sonarjs/no-dead-store              書いたのに誰も読まない代入
   49   sonarjs/cognitive-complexity
   40   complexity
```

**`var self = this` が122件**あるあたりに、13年が見えます。

**この数字は「悪い」という意味ではありません。** 2013年に書き始めたコードに、2026年のルールを当てているだけです。**当時は正しかった書き方も混ざっています。**

天井を凍結する形にしたのは、まさにそれが理由です。**「全部直せ」ではなく「これ以上増やすな」という形にする**。それなら今日から成立します。

---

## そして、linter が出したのはバグでした

`no-undef` ── **未定義の変数を使っている**という、一番基本的なルールです。**まずこれが2件、当てました。**

### `pm2 monit` が、エラー時に落ちる

```js
// lib/API/Extra.js:703
that.exitCli(conf.ERROR_EXIT);
```

**`conf` という変数は、このファイルに存在しません。** 他の箇所は全部 `cst.ERROR_EXIT` で、`cst` は8行目で require されています。**ここだけ違う。**

`getMonitorData` が失敗したときに通ります。エラーを出したあと、**`ReferenceError: conf is not defined` を投げて落ちます。**

### タブ補完のインストールが、エラー時に落ちる

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

実際に確かめました。

```
修正前  ReferenceError: completer is not defined
修正後  Error: No .zshrc file. You'll have to run instead: pm2 completion >> ~/.zshrc
```

**2件とも、何かが既に失敗したあとにしか通りません。** だから13年見つかりませんでした。**普通に動いているときは、絶対に通らない場所です。**

本家に PR を出しました → **[Unitech/pm2#6143](https://github.com/Unitech/pm2/pull/6143)**（差分4行）。

---

## 次に、TypeScript に移した

linter は「無いものを使っている」までしか見ません。**その先は型です。**

`lib/` の `.js` 49本を、`git mv` で `.ts` に**改名しただけ**にしました。**ロジックは1行も書き換えていません。**

```
                 改名前    改名後
eslint（未抑制）   3,130  →  1,875
型エラー           2,641  →    259
```

**改名しただけで、型エラーが 2,641 件。**

（259 は、代入だけされていたフィールド18個の宣言を足し、strict を掛ける対象を17ファイルに絞ったあとの数字です。**残りはまだ `strict:false` で逃がしてあります。**）

そして中身を見ていくと、**「無いデータを読んでいる」ばかりでした。**

---

## 出てきたもの

フォークに issue を11件立てました。代表的なものです。

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

なので `user` が `constructor` / `toString` / `valueOf` / `hasOwnProperty` / `__proto__` のいずれかだと、**継承されたプロパティが返ります。**

```js
typeof users['constructor'];  // 'function'
typeof users['toString'];     // 'function'
```

**`!user_info` を通過します。** 「User cannot be found」は出ません。そして `parseInt(undefined)` で **`app.uid = NaN`**。

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

### ④ そのほか

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

そして linter の設定には、**多すぎて今は直せない**という趣旨のコメントとともに、`no-explicit-any` が off にしてありました。

**責める気にはなりません。** 実際に多すぎるからです。**そして off にした瞬間から、そこは誰も確かめていない場所になります。**

`any` は「型が分からない」ではなく、**型を決める判断を、書かずに済ませた場所**です。判断を書かなければ、間違いも書かれません。**間違いが無いのではなく、間違いを確かめていないだけです。**

---

## 正直に書いておくこと

数字を出す以上、**都合の悪いほうも書きます。**

**天井が 4,933 → 3,637 に下がりましたが、そのうち 1,292 件は「直した」のではありません。** `@typescript-eslint/no-require-imports` を `.js` ファイル全体で対象外にしたぶんです。CommonJS のパッケージで `.js` が `import` を使えない以上、このルールは**新しいファイルを全部止めるだけで、線を守っていませんでした。**

**範囲を狭めたのであって、良くしたわけではありません。** ツールの出力にもそう記録してあります。**実際に直したのは4件です。**

こういうものを混ぜて「1,296件改善」と書くことはできます。**でもそう書いた瞬間、他の数字も信用されなくなります。**

---

## まだ出来たばかりです

**[ever-better](https://github.com/isamu/ever-better) は数日前に書き始めたものです。** この記事に出てくる pm2 が、実質的な最初の実戦投入でした。

なので、**足りないところだらけだと思っています。**

- 診断が拾えていないツールがある
- ルールの初期セットが、あなたのリポジトリには合わないかもしれない
- モノレポや、`.js` と `.ts` が混ざった構成での挙動
- Claude Code 以外から使ったときの手触り

**issue をどんどん立ててください。** 「ここで詰まった」「この判定はおかしい」「こういうリポジトリだと動かない」── **どれも歓迎します。直します。**

> **[github.com/isamu/ever-better/issues](https://github.com/isamu/ever-better/issues)**

**うまくいった報告も、同じくらいありがたいです。** どんなリポジトリで何件凍結できたか、それだけでも次の改善の材料になります。

---

## まとめ

- **1日500コミットには足回りが要る。** でも**最初から入っていたことは一度もない**。毎回、途中から入れて厳しくしていく
- そこを埋める **[ever-better](https://github.com/isamu/ever-better)** を作った（MIT・依存ゼロ）。**今日の違反数を天井として固定し、そこから増やさない**
- **TS/JS なら、Claude Code に `run ever-better on this repo` と言うだけ**。整形 → 導入 → 凍結 → ルール1本ずつの PR、と勝手に進む
- ★43,254・13年ものの [pm2](https://github.com/Unitech/pm2) に当てたら、**1時間で 4,942 件を凍結**できた
- **`no-undef` だけで本物のバグが2件**。どちらもエラー処理の中で、13年見つかっていなかった → [本家に PR](https://github.com/Unitech/pm2/pull/6143)
- **`.js` を `.ts` に改名しただけで、型エラーが 2,641 件**（eslint も 3,130 件）
- そこから**バグを11件**切り出した。うち2件は**プロトタイプが「見つかった」と答えてしまう**もの
- **全部「そこに無いデータを読んでいる」という同じ形**だった

古いコードを責める話ではありません。**13年動き続けているコードのほうが、たいていのコードより優秀です。**

ただ、**確かめていない場所は、確かめるまで分からない。** それだけです。

そして**確かめるのは、いまは1時間でできます。**

---

## 関連記事

- [AIに大きな仕様書を渡して一気に作らせるのは、宝くじを買うようなものです](https://zenn.dev/singularity/articles/small-teams-ai) — この記事の前提。**足回りが要る**と書いた回
- [1日500コミットは、もう読めない ── だからコードレビューをやめた](https://zenn.dev/singularity/articles/stopped-reviewing-my-code) — 読まなくても壊れない仕組みの全体像
- [strict を入れても as は止まらない](https://zenn.dev/singularity/articles/what-else-was-off) — その検査が本当に効いているかを数えた話
