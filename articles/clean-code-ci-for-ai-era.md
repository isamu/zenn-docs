---
title: "AIでがんがん書く時代の「きれいなコード」の守り方 — ESLint+SonarJS / jscpd / knip をCIに置く"
emoji: "🧹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ClaudeCode", "ESLint", "CI", "TypeScript", "SonarQube"]
published: true
publication_name: "singularity"
---

Claude Code や Codex でコードを書くようになって、開発のボトルネックが変わったな、と感じています。以前は「書く速度」が律速でしたが、今は違う。AI がいくらでも書いてくれるので、詰まるのは**書いたコードをきれいなまま保つ**ところです。

AI は、その場の文脈だけで「それっぽく動くコード」を出します。悪気なく、こういうことをします。

- 1つの関数に処理を全部詰め込んで、気づけば200行
- 型が面倒になると `any` で流す、変数名が `md` とか `e` みたいに短くて意味不明
- すでにある `truncate()` を知らずにもう一回書く（うちでは同じ関数が6個まで増えました）
- 「この機能もう要らない」とリファクタさせると、呼び出し元は消えるのにヘルパーだけ残る

一つひとつは些細ですが、これが積み重なるとコードベースがじわじわ腐ります。しかも厄介なのは、人間のレビューではこれらが見つけにくいこと。目の前の差分がどんなに綺麗でも、「26万行のどこかに同じ関数がもうあるか？」なんて記憶では追えません。

だったら、こういう衛生管理は機械にやらせよう、という話です。この記事では、Claude Code をコアに開発している実プロダクト [MulmoClaude](https://github.com/receptron/mulmoclaude)（Vue + TypeScript、`packages/` に25超のワークスペースがあるモノレポ）で実際に使っている構成を、設定ファイルの実物つきで紹介します。

### そもそも「DRY」とは

この先「DRY」という言葉を何度か使うので、先に一言だけ。DRY は "Don't Repeat Yourself"（繰り返すな）の略で、**同じ処理や同じ知識を、コードのあちこちに重複させず一箇所にまとめる**、という設計の基本原則です。

なぜ重複が悪いかというと、仕様が変わったときに全部直す羽目になるからです。冒頭の `truncate()` が6箇所にコピペされていたら、挙動を1つ変えるのに6箇所直さないといけない。そして必ず直し忘れが出て、片方だけ古い挙動のまま残る——これがバグになります。だから「同じことは一箇所に書く」。地味ですが、コードが増えるほど効いてきます。

分かりやすいのは、**ユーザーの権限を判定するコード**です。「このユーザーはこの操作をしていいか」という判定が、コピペで10箇所に散っているとします。ここで権限の仕組みを変えることになったら、間違いなく10箇所すべてを直さないといけない。そして9箇所直して1箇所見落とせば、そこだけ古いルールで通ってしまう——権限判定でこれが起きると、ただのバグでは済まず、セキュリティホールになります。

「AI に探させればいいのでは」と思うかもしれませんが、AI もコードベース全体を把握しているわけではありません。grep で拾えない書き方（同じ判定を少しだけ違う書き方でコピペしてある、など）は普通に見落とします。そもそも「10箇所ある」ことを人間も AI も知らない、というのが本当の問題です。

逆に、判定が `canEditDocument(user, doc)` という関数1つにまとまっていれば、変更するのはその関数の中だけ。呼び出し元が10箇所あろうが100箇所あろうが、直す場所は1つです。**サーバー側とフロント（Web）側で同じ判定をする**ような場面ではさらに効きます。判定ロジックを共通のパッケージに置いて両方から呼ぶようにしておけば、「サーバーだけ直してフロントが古いまま」という食い違いが構造的に起きなくなります。

なので、AI がいくらでも書き直せる前提に立ったとしても、結論は人間が手で書いていた頃と変わりません。**同じ処理をコピペせず、きれいなコードと明確な関数名で関数に切り出しておく**こと。むしろ、コードが増える速度が上がったぶん、この規律の重要性は上がっています。

### 余談：きれいなコードは、AIにとっても「安い」

DRY で小さいコードは、保守性だけでなく **AI のトークン消費**という意味でも得だ、というのは案外見落とされます。

理屈はシンプルで、エージェントがコードを読んだり直したりするとき、コード量がそのままコンテキスト消費になるからです。`truncate()` が6実装あれば挙動を把握するのに6箇所読む羽目になるし、大きい関数は丸ごと読み込まれる。逆に小さくまとまっていれば、限られたコンテキスト窓で一度に見渡せる範囲が広がるし、grep で該当箇所に一直線で辿れる。編集の差分も小さくなって、書き戻すトークンも減ります。

つまり「人間が丁寧に書くときのように、DRY でコンパクトに」という規律は、AI に任せる時代はむしろ効きが強くなる。速く大量に書けるぶん、密度が効いてくるわけです。

ただ一点だけ補足すると、DRY はバイト数を減らすことではありません。helper を切り出せば行数はむしろ増えることもある。効くのは「直す場所が1箇所になる」ことであって、やりすぎて無理に共通化すると、今度は「helper と全呼び出し元」を跨いで読む羽目になり、かえってコンテキストが増えます。狙いはあくまでモジュラーで単一責任なコードで、「たまたま似てるだけ」のものは共通化せずに残す。この線引きこそ、これから紹介するツール群が支えてくれるものです。

## 4本柱を機械に守らせる

役割の違う4つを重ねます。1つでは必ず穴が空くからです。

| 観点 | ツール | 何を捕まえるか |
|---|---|---|
| 大きさ・複雑度 | ESLint コアルール | 長い関数・深いネスト・多すぎる引数 |
| 後で困る書き方・規約 | SonarJS + strict TS | 追いにくいロジック・`any`・暗号みたいな名前・依存の逆流 |
| 重複 | jscpd | コピペコード（ファイル横断のトークン一致） |
| デッドコード | knip | 未使用 export・未使用ファイル・孤児 |

なぜ4つも要るかというと、それぞれ「見える範囲」が違うからです。ESLint の `no-unused-vars` は1ファイルの中しか見ないし、jscpd はコピペしか見ない、knip は export が使われているかしか見ない。重ねて初めて面が埋まります。

紹介の順番は、まず毎回・毎ファイル効く土台の ESLint / SonarJS から入って、そのあとファイル横断の専用スキャン（jscpd / knip）を足す、という流れにします。すでに ESLint を使っている人が多いはずなので、そこを強くするのが一番コスパがいいからです。

## 1. 関数とファイルの大きさ、複雑度をESLintで縛る

まずは土台から。AI は「とりあえず1つの関数に全部書く」のが得意なので、放っておくと巨大関数が増えていきます。ここは ESLint のコアルールで上限を決めてしまうのが早い（[eslint.config.mjs](https://github.com/receptron/mulmoclaude/blob/main/eslint.config.mjs)）。

```js
rules: {
  "max-lines-per-function": ["error", { max: 50, skipBlankLines: true, skipComments: true, IIFEs: true }],
  complexity: ["error", { max: 15 }],   // 循環的複雑度
  "max-depth": ["error", { max: 4 }],   // ネストの深さ
  "max-params": ["error", { max: 6 }],  // 引数の数
}
```

地味なポイントですが、`skipBlankLines` と `skipComments` を付けています。こうしておくと、コメントを丁寧に書いた小さい関数が「長い」と誤判定されません。コメントは書いてほしい、でも測りたいのは実装の量だけ、という意図です。

### 既存コードにどう入れるか — 「新規だけ先に厳しく」する

正直、ここが一番実戦的な話だと思います。50行ルールを、すでに大きく育ったコードベースにいきなり「違反＝エラー（CIを失敗させる）」で入れたら、何百件もの既存違反が一斉に引っかかって、直しきれない超巨大な修正になってしまいます。かといって諦めると、いつまでも汚いままです。

ここで使うのが「**新しく書くコードにだけ、先にルールを効かせる**」というやり方です。ESLint には、違反したときに「エラー（`error`：CIを止める）」にするか「警告（`warn`：知らせるだけで止めない）」にするかを選べる仕組みがあるので、これを次のように使い分けます。

1. **基本はエラー（`error`）にする。** これで、これ以降に新しく書かれる50行超の関数は、CIで弾かれて入れられなくなります。「悪化を止める（＝もう増やせない）」わけです。
2. **今すでにある違反だけ、警告（`warn`）に緩める例外リストを作る。** ESLint はファイル単位でルールを上書きできるので、既存の長い関数があるファイルだけを名指しで警告に落とします。

```js
// eslint.config.mjs の末尾（既存違反の“見逃しリスト”）
{
  files: ["server/api/routes/collections.ts"],   // 既に長い関数があるファイル
  rules: { "max-lines-per-function": "warn" },   // ここだけ「警告」に緩める
}
```

3. **あとは時間を見つけて少しずつ関数を分割し、直したものからこの例外リストの記載を消していく。** 新しいファイルをこのリストに足すのは禁止——ここに追加が発生したら、それは「新しく汚いコードが増えた」合図なので。

この「新しいぶんは厳しく、既存の借金は少しずつ返す」という型は、関数の長さに限らず、どんな厳しめルールを後から入れるときにも使えます。AIとの相性も良くて、AIがこれから書くコードには最初から厳しいルールを効かせ、過去の負債はそれとは別の作業として計画的に返す、と分けて考えられます。

:::message alert
とはいえ、一番のおすすめは**最初からルールを入れておくこと**です。ここまで「新規だけ先に厳しく」「見逃しリスト」といった工夫を書いてきましたが、これらは全部「後から厳しくする」ための後始末のテクニックです。プロジェクトが大きくなってからルールを入れると、この返済作業が必ず発生します——数百件の違反を1つずつ直す地味な修正は、正直かなりしんどい。

だから、新しくプロジェクトを始めるなら、**コードを1行書く前に厳しめのルールを入れてしまう**のを強く勧めます。最初から `error` にしておけば、そもそも汚いコードが1行も入らないので、見逃しリストすら要りません。「後で厳しくすればいい」は、たいてい「後で大変になる」とセットです。
:::

ちなみにテストは別扱いにしています。`describe` / `it` ブロックや fixture ビルダーは長くて自然なので、テストだけ `max-lines-per-function` を off に。10個の `it()` を抱えた `describe()` に本番コードの可読性ルールを当てても意味がないですからね。

## 2. 「動くけど後で困るコード」をSonarJS+strictで潰す

同じ ESLint の上に、もう一枚重ねます。AI が生みがちな、**動くけれど後で困る書き方**を締める層です。

バグではありません——ちゃんと動く。でも、読みにくかったり、直しにくかったり、少し変更するとすぐ壊れそうだったり。今は平気でも後で面倒を呼ぶ、そういう書き方のことです。長すぎる関数、深いネスト、`any` での握りつぶし、意味の取れない変数名あたりが代表格です。

MulmoClaude は `sonarjs.configs.recommended` と `tseslint.configs.strict` をベースに、こうした書き方を見つけるルールをさらにいくつか強めています。

一番効いているのが認知的複雑度です。

```js
"sonarjs/cognitive-complexity": "error",
```

循環的複雑度が「分岐がいくつあるか」を数えるのに対して、認知的複雑度は「人間が読んで追うのがどれだけ辛いか」を測ります。ネストが深いほど重く加算されるのが特徴で、分岐の数自体は少なくても3重ネストで頭が痛くなるような関数を捕まえてくれる。AI が書きがちな入れ子だらけのロジックにちょうど効きます。

`any` も本番コードでは禁止です。

```js
"@typescript-eslint/no-explicit-any": "error",   // 本番コード（テストだけ warn）
```

AI は型が面倒になると `any` で流しにかかるので、ここは `error` で止めます（`any` は「型チェックをやめる」宣言なので、これが1つ紛れると周りの型安全もなし崩しになりがちです）。プロジェクトの規約ファイル（Claude Code に読ませる `CLAUDE.md`）でも「`as` による強制的な型変換は使わず、型ガードで安全に絞り込む」を徹底しています。

ただし正直に書いておくと、このルールが止めてくれるのは**ソースに明示的に書かれた `any` だけ**です。型定義のないライブラリから流れ込んでくる推論上の `any`、`as unknown as Foo` の二段キャスト、`JSON.parse()` の戻り値あたりは、`no-explicit-any` を素通りします。つまり「`any` を禁止した」と「型がついている」は同じではありません。

実際にどれだけ型がついているかを測るには、[type-coverage](https://github.com/plantain-00/type-coverage) という別のCLIが使えます。ESLint のルールではなく、jscpd や knip と同じ「ファイル横断の専用スキャン」の仲間です。

```bash
npx type-coverage --at-least 95 --detail --strict
```

コードベース中の識別子（変数・引数・プロパティなど）のうち型がついているものの割合を `(11 / 14) 78.57%` のような形で出して、下限を切ります。`--detail` を付けると、型のついていない識別子が位置つきで並びます。

`--strict` は付けておくのを勧めます。手元の小さいファイルで試したところ、これを付けないと `as unknown as Foo` のような二段キャストが「型がついている」側に数えられてしまいました。上に挙げた抜け道のうち、推論 `any` と `JSON.parse()` の戻り値は既定でも拾えますが、キャストで black box にした箇所は `--strict` を付けて初めて数に入ります。

もう一段強くやるなら、`@typescript-eslint` の `no-unsafe-*` 系（`any` 由来の値の代入・呼び出し・メンバーアクセスを禁止する）を有効にする手もあります。これらは**型情報を使うルール**なので、パーサに型情報を渡す設定（`projectService`）が必要です。

MulmoClaude で実際に入れた構成がこれです（[PR #2183](https://github.com/receptron/mulmoclaude/pull/2183)）。

```js
// projectService は SonarJS 自身の型情報ルールも起こす。手で列挙すると将来の
// ルール追加で CI が落ちるので、plugin の metadata から導出する。flat config では
// 「名指し＝有効化」なので、recommended で既に有効なものだけに絞る（そうしないと
// recommended が意図的に off にしているルールまで有効化してしまう）。
const rec = sonarjs.configs.recommended?.rules ?? {};
const isEnabled = (level) => {
  const severity = Array.isArray(level) ? level[0] : level;
  return severity !== undefined && severity !== "off" && severity !== 0;
};
const sonarTypeAwareRulesAsWarn = Object.fromEntries(
  Object.entries(sonarjs.rules ?? {})
    .filter(([name, rule]) => rule?.meta?.docs?.requiresTypeChecking && isEnabled(rec[`sonarjs/${name}`]))
    .map(([name]) => [`sonarjs/${name}`, "warn"]),
);

export default [
  // ...既存の設定
  {
    // テスト/e2e は入れない。型プログラムを小さく保つため
    files: ["server/**/*.ts", "src/**/*.ts", "packages/**/src/**/*.ts"],
    languageOptions: {
      parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname },
    },
    rules: {
      // (1) no-explicit-any をすり抜ける any
      "@typescript-eslint/no-unsafe-assignment": "warn",
      "@typescript-eslint/no-unsafe-member-access": "warn",
      "@typescript-eslint/no-unsafe-argument": "warn",
      "@typescript-eslint/no-unsafe-call": "warn",
      "@typescript-eslint/no-unsafe-return": "warn",
      // (2) 型チェッカにしか見えないバグ
      "@typescript-eslint/no-base-to-string": "warn",
      "@typescript-eslint/no-floating-promises": "warn",
      "@typescript-eslint/no-misused-promises": "warn",
      "@typescript-eslint/await-thenable": "warn",
      ...sonarTypeAwareRulesAsWarn,
    },
  },
];
```

設定上のポイントは4つです。

**`strictTypeChecked` を丸ごとではなく、ルールを名指しする。** コストは「ルールの数」ではなく「型プログラムを構築したこと」に付くので、絞っても速くはなりません（フルスコープで +25% 程度）。絞る目的は速度ではなく、出力を読める量に保つことです。丸ごと入れると `restrict-template-expressions` のような style 系が過半を占めて、`any` と正しさの指摘が沈みます。

**どのルールを選ぶかは、一度丸ごと動かして出力を見てから決める。** 先に絞ると、`no-floating-promises` 系が拾う実バグ——付け忘れた `await`、同期専用 API に渡した async コールバック——の存在に気づけません。構文だけを見るルールには原理的に見えない種類のバグです。

**全部 `warn` にする。** これ単体では CI を落とさず、出たぶんは「ゲート」ではなく「これから返すバックログ」になります。次章の「止めるか、知らせるか」でいう後者の扱いです。

**SonarJS の巻き添えを metadata で処理する。** `projectService` を有効にすると、型プログラムがなくて休眠していた SonarJS の型情報ルールが、プリセットの `error` severity で一斉に目を覚まします。上のコードで `warn` に落としていますが、ここで効いているのが「手書きリストは腐る」と「flat config では名指し＝有効化」の2点で、どちらも ESLint 設定で一般に効く注意点です。

名前の短さも見ています。ここで少し、そもそも「良い名前とは何か」を整理させてください。読みやすいコードの物差しは、突き詰めると一つです——**他人（そして未来の自分、そして AI）が、そのコードを理解するのにかかる時間を最小にすること**。名前も、この一点に効くかどうかで判断できます。

具体的には、こういう考え方です。

- **名前は「小さなコメント」**。だから名前に情報を詰める。`get` より `fetch` / `download` のように、何をするかまで踏み込んだ具体的な動詞を選ぶ。
- **単位や状態を名前に入れる**。`delay` より `delayMs`、`size` より `sizeMb`、`url` より `untrustedUrl`。読んだだけで前提がわかる。
- **汎用的すぎる名前を避ける**。`tmp` / `data` / `retval` / `foo` は、よほど寿命の短い変数以外では情報量ゼロ。
- **誤解されない名前にする**。`filter` は「除外する」のか「残す」のか曖昧。`selectUnread` のように読み手が一意に取れる語を選ぶ。
- **スコープが狭ければ短くてもよい**。3行で閉じるループの `i` は問題ないが、関数をまたいで生きる変数ほど説明的にする。

これらは変数だけの話ではありません。**関数名やクラス名もまったく同じ**で、`process()` や `handle()` のような“何でも屋”の名前ではなく、その関数が何をするかが名前で語られている状態を目指します。

ESLint はこの「情報を詰めた名前」を、下限のところで機械的に後押しできます。

```js
"id-length": ["error", { min: 3, properties: "never", exceptions: ["_", "i", "j", "ok"] }],
```

`fs` / `os` のような名前空間 import や、`id` / `md` / `ms` みたいな略語を弾いて、3文字以上の名前に誘導するルールです(ループ変数の `i`/`j`、成功/失敗を表す `ok`、捨て変数の `_` だけ例外)。結果として `import * as fs` ではなく `{ readFileSync }` のような named import、`md` ではなく `markdown`、`ms` ではなく `delayMs` のように、読んで一瞬でわかる名前へ自然に寄っていきます。AI は油断すると `e` や `md` みたいな省略名を平気で置くので、これがいい歯止めになります。

もちろん `id-length` が保証するのは「3文字以上」という最低ラインだけで、`data2` のような“長いだけの悪い名前”までは防げません。そこは人間とレビューの仕事ですが、機械が最低ラインを守ってくれるだけでも、コードベース全体の底上げになります。

このあたりは他にも地味に効くルールを積んでいます。`eqeqeq`(== の型強制を禁止)、`no-else-return`、`prefer-destructuring`、`no-param-reassign`、`import/no-cycle`(循環 import 禁止)、`no-non-null-assertion`(`!` 禁止)あたり。一つひとつは小さいですが、AI が量産する「動くけど雑」なコードをまとめて締めてくれます。

### おまけ：設計の約束事もESLintで守れる

応用ですが強力なので、少し背景から紹介させてください。

コードが大きくなると、自然と「層」に分かれます。たとえば「画面（UI）」「業務ロジック」「データ保存」といった具合に。このとき健全な設計では、**依存の向きが一方通行**であることが大事です。画面が業務ロジックを呼ぶのは自然ですが、その逆——データ保存の層が画面のコードを呼ぶ——が起きると、何がどこに依存しているのか分からなくなり、一部だけ差し替えたり切り出したりが効かなくなります。「依存の向きが壊れる」というのはこういう状態です。

AI は目の前を動かすことに集中するので、「近くにある関数を import すれば動く」と、この向きを平気で逆流させます。人間のレビューでも、import 文一行の向きの間違いは見落としがちです。

そこで MulmoClaude では、「プラグイン（あとから足す機能のコード）は、本体側の内部設定やサーバー処理を直接 import してはいけない」という向きのルールを、ESLint の `no-restricted-imports` で機械的に禁止しています。

```js
{
  files: ["src/plugins/*/**/*.ts", "src/plugins/*/**/*.vue"],
  rules: {
    "no-restricted-imports": ["error", {
      patterns: [
        // 「この場所からの import は禁止」を指定し、代わりの手段を message で案内する
        { group: ["**/config/**"], message: "プラグインは本体の設定を直接 import しないこと。代わりに ../api を使う。" },
        { group: ["**/server/**"], message: "プラグインはサーバー側の処理を import しないこと（型だけは許可）。", allowTypeImports: true },
      ],
    }],
  },
}
```

こうすると、設計上の約束事そのものを CI で強制できます。しかも違反したとき `message` で「代わりにこうして」と案内できるので、人間にも AI にも親切です。あわせて、AがBを、BがAを import し合ってしまう「循環参照」を禁止する `import/no-cycle` を入れておくと、時間とともに設計が崩れていくのをかなり防げます。

## 3. コピペコードを見つける — jscpd

ここからはファイル横断のスキャン。ESLint は1ファイルの中しか見てくれないので、その外側を埋めていきます。

冒頭にも書いた `truncate()` が6実装、という話。AI は近くに似たコードがあると、既存を探さずにもう一回書くんですよね。これは誰かがサボったわけではなく、人間の記憶が数十万行をカバーできないだけです([#1304](https://github.com/receptron/mulmoclaude/issues/1304))。

[jscpd](https://github.com/kucherenko/jscpd) はトークン列レベルのコピペ検出器で、v5 系は Rust 製。26万行を0.1秒台で舐めるので、CI に置いてもコストはほぼゼロです。まずは手元で試せます。

```bash
npx jscpd@5 . --format typescript --ignore "**/node_modules/**,**/dist/**"
```

CI に置くとき、一つ知っておいてほしい落とし穴があります。`--threshold 5`(重複率5%超で失敗)を置きたくなるんですが、これでは新規重複が止まりません。16万行に対して20行コピペしても、全体率は 3.27% から 3.28% くらいしか動かない。閾値には永遠に届かず、`truncate()` が6個に増える事故は1件も止まらないわけです。

そこで MulmoClaude では、jscpd の結果を **SARIF**（静的解析の結果を表す、業界標準のファイル形式）で書き出し、**GitHub の Code Scanning**（解析結果を受け取って PR 上に警告を表示してくれる GitHub の機能）に渡しています。Code Scanning は「この PR で新しく増えた指摘」を、マージ先のブランチと比べて差分で見せてくれます。だから全体の母数に薄められることなく、この変更が新しく持ち込んだ重複だけを捕まえられるわけです([duplication-scan.yaml](https://github.com/receptron/mulmoclaude/blob/main/.github/workflows/duplication-scan.yaml)、[PR #2129](https://github.com/receptron/mulmoclaude/pull/2129))。

```yaml
# duplication-scan.yaml（要点）
- name: Run jscpd
  run: |
    jscpd . \
      --format typescript \
      --ignore "**/node_modules/**,**/dist/**,**/*.d.ts,**/test/**,**/e2e/**" \
      --reporters console,sarif --output report
- name: Upload SARIF to Code Scanning
  uses: github/codeql-action/upload-sarif@<sha>  # v3
  with:
    sarif_file: report/jscpd-report.sarif
    category: jscpd
```

:::message
jscpd 単体の詳しい話(`--format` で絞る理由、行%とトークン%の違い、`ai` レポーターなど)は別記事「[コピペコードを機械的に潰す](https://zenn.dev/singularity/articles/jscpd-dry-detection-mono)」にまとめました。
:::

重複が見つかったら、そのまま AI に「この2箇所を共通関数に抽出して」と渡せるのも今どきです。検出は決定的(トークン一致)なので機械が得意、「共通化すべきか、たまたま似てるだけか」の判断は人間や LLM が得意。役割分担がきれいに決まります。

## 4. デッドコードを消す — knip

もう一つ、ファイル横断で見たいのが**デッドコード**、つまり「もう誰からも使われていないのに残っているコード」です。AI にリファクタさせると、呼び出し元は消えたのに、その関数（ヘルパー）や export だけが取り残される、というのがしょっちゅう起きます。ESLint の `no-unused-vars` は1ファイル内しか見ないので、どこからも import されない export や、丸ごと誰も読み込まないファイルはすり抜けてしまいます。

ここで [knip](https://github.com/webpro-nl/knip) を使います。

:::message
ts-prune を思い浮かべた方へ。ts-prune は「未使用 export」検出の定番でしたが、今は knip が上位互換になっています。knip は未使用 export に加えて、未使用ファイル・未使用依存・未解決 import まで一つで見てくれる(ts-prune は export だけ、かつメンテも停滞気味)。新規に入れるなら knip でよいと思います。MulmoClaude も knip を採用しています。
:::

knip は「エントリーポイントから到達できないもの」を洗い出す仕組みなので、entry の指定が肝になります([knip.json](https://github.com/receptron/mulmoclaude/blob/main/knip.json))。

```json
{
  "$schema": "https://unpkg.com/knip@6/schema.json",
  "entry": [
    "server/index.ts",
    "src/main.ts",
    "src/plugins/index.ts",
    "scripts/**/*.{mjs,ts}",
    "batch/**/*.ts",
    "test/**/*.ts", "e2e/**/*.ts"
  ],
  "project": ["server/**/*.ts", "src/**/*.{ts,vue}"],
  "ignore": ["**/dist/**", "**/*.d.ts", "src/plugins/_generated/**"],
  "ignoreExportsUsedInFile": true,
  "rules": {
    "exports": "warn",
    "types": "warn",
    "files": "off",
    "dependencies": "off"
  }
}
```

`entry` には `server/index.ts` や `src/main.ts` だけでなく、`scripts/**` や `test/**` も入れておきます。ここを漏らすと、テストからしか呼ばれない helper を「使われていない」と誤判定してしまうので。MulmoClaude では今のところ `exports` と `types` だけを見て、未使用ファイルや依存は off にしています。理由は次の CI の話とつながります。

MulmoClaude の [dead-code-scan.yaml](https://github.com/receptron/mulmoclaude/blob/main/.github/workflows/dead-code-scan.yaml)([PR #2153](https://github.com/receptron/mulmoclaude/pull/2153))は、CIを絶対に失敗させない作りにしています。理由は二つ。一つは、既存の未使用コードがそれなりに溜まっているので、いきなり「止める」設定にすると、全員が反射的に「この行は無視して」という注釈を貼り始めて無意味になること。もう一つは、knip は「変更前と比べた差分」を出す機能を持たないので、報告が「この変更で増えたぶん」ではなく「今この瞬間に存在する全部」になること。だから止める設定にはせず、人間が「これは本物か、この変更で悪化したか」を判断するための材料として置いています。

```yaml
# 結果は PR の要約欄とコード上の警告に出すだけ（末尾の || true でCIは止めない）
- name: Run knip
  run: |
    yarn --silent knip --reporter compact >> "$GITHUB_STEP_SUMMARY" || true
    yarn --silent knip --reporter github-actions || true
```

## CIで「止める」か「知らせるだけ」か

4本をCIに並べるとき、実は大事なのが、全部を同じ「失敗させる」扱いにしないことです。

CIでの扱いには、大きく2つあります。一つは**止める**——違反が見つかったらCIをわざと失敗させ、直すまでマージできなくする厳しいやり方。もう一つは**知らせるだけ**——結果は出すけれどCIは止めず、人間がその報告を見て「これは直すべきか」を判断するやり方です。

「全部止める設定にすれば安心では？」と思うかもしれませんが、逆効果になることがあります。**信頼できない検査でCIを止めると、みんな「とりあえず通したいから」と無視コメント（`// eslint-disable` のような、ルールを黙らせる注釈）を貼り始め**、せっかくの検査が形骸化してしまうからです。だから原則は「**その検査が信頼できて初めて、止める設定にする**」。ここは実際に運用してみて一番腹落ちした部分でした。

MulmoClaude の切り分けはこうなっています。

| 検査 | CIでの扱い | なぜそうしたか |
|---|---|---|
| ESLint + SonarJS | **止める**（違反でCI失敗） | 前項の「新規だけ厳しく」で既存の負債と切り離してあり、新しく増えたぶんだけ確実に止められる。信頼できるのでゲートにできる |
| jscpd（重複） | **知らせる**（PRに新規ぶんを差分表示） | 前述のとおり全体%では新規を止められないため、GitHubのCode Scanning（PR上に警告を出す仕組み）に渡し、「この PR で増えた重複」だけを見せる |
| knip（デッドコード） | **知らせるだけ**（CIは止めない） | knip は「この PR の差分」ではなく「今ある全部」を出すため、まずは人間が読むための材料に留める |

ポイントは、「新しく増えたぶんだけを正確に指せる検査」は止める側に回せて、「今ある全部しか出せない検査」はまず知らせる側に置く、という切り分けです。

もう一つ付け加えると、CI はあくまで「書いた後」に捕まえる仕組みなので、「書く前」に重複を防ぐ人間側の仕組みも要ります。MulmoClaude には [docs/shared-utils.md](https://github.com/receptron/mulmoclaude/blob/main/docs/shared-utils.md) という共有ヘルパーのカタログがあって、「新しい helper を書く前にここを見る、追加したら1行足す」を運用ルールにしています。`truncate()` の6実装も、`formatBytes()` の2実装も、この予防がなかったから起きた、とそのファイルに明記されています。AI に書かせるときは、このカタログを AI にも読ませておくと効きます。「既存の helper を使え」を人間だけでなく AI にも効かせられるので。

## まずはここから

新規プロジェクトなら、この順で入れるのがおすすめです。

```bash
# まず土台
yarn add -D eslint typescript typescript-eslint eslint-plugin-sonarjs eslint-plugin-import @eslint/js eslint-config-prettier
# 次に専用スキャン
yarn add -D jscpd knip
```

最小の `eslint.config.mjs` はこのくらいから始めれば十分です。

```js
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";
import sonarjs from "eslint-plugin-sonarjs";
import importPlugin from "eslint-plugin-import";
import prettier from "eslint-config-prettier";

export default [
  eslint.configs.recommended,
  sonarjs.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  {
    languageOptions: {
      parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname },
    },
    plugins: { import: importPlugin },   // import/no-cycle を使うのに必要
    rules: {
      "max-lines-per-function": ["error", { max: 50, skipBlankLines: true, skipComments: true }],
      complexity: ["error", { max: 15 }],
      "max-depth": ["error", { max: 4 }],
      "sonarjs/cognitive-complexity": "error",
      "@typescript-eslint/no-explicit-any": "error",
      "id-length": ["error", { min: 3, exceptions: ["_", "i", "j"] }],
      "import/no-cycle": "error",
    },
  },
  prettier,
];
```

`strict` ではなく `strictTypeChecked` を指定しているのが地味なポイントです。`any` 由来の値が黙って流れていくのを `no-unsafe-*` 系が止めてくれます。

そして**これを丸ごと入れられるのは新規プロジェクトの特権**です。前の章で「style 系の指摘で埋まる」と書いたのは、既に育ったコードベースに後から入れた場合の話でした。ゼロから始めるなら、`no-confusing-void-expression` も `restrict-template-expressions` も最初から満たしながら書くだけなので、返済すべき違反自体が発生しません。逆に言えば、後から入れるプロジェクトは丸ごとではなく、前章のようにルールを名指しするほうが定着します。

CI 側は、ESLint は「止める」設定で、jscpd と knip は「知らせる／差分で見せる」設定で置きます。既存コードに厳しいルールを入れるときは、いきなり全部 `error` にせず、まず新しく書くぶんだけ厳しくして既存は少しずつ直す、という段取りを思い出してもらえれば。

## おわりに

AI 時代のボトルネックは「書く速度」ではなく「きれいなまま保つ速度」に移りました。そしてきれいでコンパクトなコードは、保守性だけでなく AI のトークン効率という面でも得をします。だからこそ、衛生管理は機械に任せてしまうのが合理的です。

やることを一言でまとめると、見える範囲の違う4本(ESLint / SonarJS / jscpd / knip)を重ね、「新しく増えたぶんを正確に指せる検査」はCIを止める側に、「今ある全部しか出せない検査」は知らせる側に分け、既存の負債は「新規だけ先に厳しく」の要領で少しずつ返す。そして機械の検出(CI)と人間の予防(共有ヘルパーのカタログ)を両輪で回す。これだけです。

こうしておくと、「AI に思い切り書かせる」ことと「コードベースを綺麗に保つ」ことが、ちゃんと両立します。速く書けるようになったぶん、守りを機械化しておく価値も上がっている、というのが今の実感です。
