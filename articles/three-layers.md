---
title: "宣言1枚が、手元のアプリにも、共有サービスにもなる — GUI Chat Protocol / コレクション / 共有アプリ"
emoji: "🧩"
type: "tech"
topics: ["AI", "typescript", "firebase", "ClaudeCode", "設計"]
published: true
publication_name: "singularity"
---

MulmoTerminal / MulmoClaude には、チャットの中に画面を出す GUI Chat Protocol、手元でデータを扱うコレクション、他人に使ってもらう共有アプリという3つの仕組みがある。一見すると無関係な3つだが、中を読むと1本の線でつながっていた。同じ `schema.json` を、置き場所と宣言の足し方を変えて3通りに使っている。

以下は 2026-09-21 時点の実装を、各リポジトリの README・spec・ソースで確かめた内容である。確かめられなかったことは、最後にまとめた。

```
schema.json（fields / views）
    │
    ├─ storage 既定（ローカルの JSON ファイル）
    │     └─ コレクション — 自分の画面、自分のエージェント
    │
    ├─ presentCollection で返す
    │     └─ GUI Chat Protocol — チャットの中に描画
    │
    └─ storage: { type: "firestore" } + app.json
          └─ 共有アプリ — 他人のブラウザ、認可つき
```

この図の下の層から順に見ていく。

## GUI Chat Protocol：返り値に、画面向けの口をもう1つ足す

[receptron/gui-chat-protocol](https://github.com/receptron/gui-chat-protocol) にあり、npm に publish 済みである。

通常の tool call は、実行結果をテキストで LLM に返す。このプロトコルでは返り値を2つに分け、LLM が読む `message` の横に、UI が読む `data` を置く。

```ts
return {
  message: "9月の支出をグラフにしました",   // LLM が読む。会話はこれで続く
  data: {
    type: "chart",                          // UI が読む。type で描画先が決まる
    option: { /* ECharts の設定 */ },
  },
};
```

LLM から見ると、標準の function calling と区別がつかない。システムプロンプトでツール定義を受け取り、呼び、テキストを受け取るだけなので、新しく覚える概念もない。既存の tool call に口を1つ足しただけの仕組みだから、MCP とも function calling とも競合しない。

パッケージは次のように分かれている。

| | |
|---|---|
| `src/types.ts` / `schema.ts` / `inputHandlers.ts` | フレームワーク非依存のコア |
| `src/vue.ts` / `src/react.ts` | Vue 3 / React 18-19 のアダプタ型 |
| `spec/GUI_CHAT_PROTOCOL.md` | 設計と動機 |
| `spec/CREATING_A_PLUGIN.md` | プラグインの作り方 |

UI を持たないロジックはコアだけで書け、フレームワークごとに分かれるのは描画の型付けだけである。画面からエージェントへ向かう入力も扱える。`inputHandlers.ts` が `FileInputHandler` と `ClipboardImageInputHandler` を定義していて、チャットに落としたファイルや貼り付けた画像を、プラグインが受け取れる。

いま動いているプラグインを並べると、次の表になる。

| ツール | 出るもの |
|---|---|
| `presentChart` | ECharts |
| `presentForm` | 構造化された入力フォーム |
| `presentDocument` | Markdown / Marp スライド |
| `presentHtml` | 自己完結した HTML ページ |
| `presentCollection` | スキーマ駆動のデータ画面 |
| `presentShapeScript` | 対話的な3D |
| `presentMulmoScript` | 動画・スライドの絵コンテ |
| `@mulmoclaude/accounting-plugin` | 複式簿記の帳簿 |
| `@mulmochat-plugin/generate-image` | 画像生成 |

この中の `presentCollection` が、次の層への入口になっている。コレクションの画面も、このプロトコルのプラグインの1つとしてチャットに描かれる。

## コレクション：schema.json がデータの形と画面を両方決める

コレクションの本体は `@mulmoclaude/collection-plugin` で、自己申告は Schema-driven Collections plugin である。1つのコレクションは、ディレクトリ1つにまとまっている。

```
<slug>/
  schema.json      データモデルと UI の宣言
  SKILL.md         エージェントへの説明書
  meta.json        作者・slug・説明・ライセンス
  views/*.html     任意。カスタムビュー
  seed/items/*.json 任意。サンプルレコード
  manifest.json    取り込み時に使うファイル一覧
```

中心になるのは `schema.json` で、たとえば映画リストなら次のように書く。

```json
{
  "title": "映画リスト",
  "icon": "movie",
  "dataPath": "data/movies/items",
  "primaryKey": "id",
  "fields": {
    "title":  { "type": "string", "label": "タイトル", "required": true },
    "genre":  { "type": "enum", "label": "ジャンル", "values": ["SF", "ドラマ"] },
    "poster": { "type": "image", "label": "ポスター" }
  },
  "views": [
    { "id": "cinema", "label": "シネマ", "file": "views/cinema.html", "capabilities": ["read"] }
  ]
}
```

これを置くだけで、表・入力フォーム・カレンダーが描画される。ホスト側にコレクション固有のコードはなく、自前の画面が欲しければ `views[]` に足せばよい。データは `dataPath` の下に、1件1ファイルの JSON として並ぶ。DB を使っていないので、git に入れられるし、エディタで開けるし、`rm` で消せる。

エージェントとの接続は `SKILL.md` が受け持つ。

```markdown
---
name: movies
description: 個人の映画リストコレクション。ユーザーが「この映画を追加して」
  「この映画を観た、★4」などと言ったら使う。記録は dataPath に1件1ファイルの
  JSON で保存する。CRUD は manageCollection で行う。
---

## レコードの形
- `id` — kebab-case slug、主キー
- `watched` — boolean。`rating` と `watchDate` は `watched` のときだけ表示
```

書いてあるのは「こう言われたらこれを使え」と「データはこういう形だ」の2つだけで、どちらも自然言語である。エージェントはこれを読み、`manageCollection` で JSON を読み書きする。ただし、スキーマを変えたら `SKILL.md` も自分で直す必要があり、そこは自動では追従しない。

作ったコレクションは、レジストリの [receptron/mulmoclaude-collections](https://github.com/receptron/mulmoclaude-collections) で配る。`scripts/build-index.mjs` が `collections/` を走査して `index.json` を生成し、バックエンドはこれを1回の GET で取って Discover のカタログにする。取り込み時に取得するファイルは、コレクションごとの `manifest.json` に並んでいる。投稿するときは `collections/<自分の GitHub ログイン名>/<slug>/` に足して PR を出す。他人の名前空間に出そうとすると、CI が弾く。

## 共有アプリ：同じスキーマを、他人のブラウザへ

共有アプリを作るのは `@receptron/sharedapp` で、自己申告は「The shared-app compiler: `app.json` in, Firestore documents out — and what publish refuses before it writes any of them.」である。

共有アプリのスキーマを見ると、コレクションとの差は思ったより小さい。

```json
{
  "title": "設問",
  "icon": "quiz",
  "primaryKey": "id",
  "storage": { "type": "firestore" },     ← ここだけ
  "fields": { ... }
}
```

変わるのは `storage` だけで、`fields` も `views` もコレクションと同じ形をしている。置き場所も `.claude/skills/<name>/schema.json` のままである。`sharedapp` は `@mulmoclaude/core` から `CollectionSchema` 型と `isValidCollectionName`、`isSafeCustomViewPath` を peer で借りていて、型を共有している。機能の設計名も、もともと「shareable collections」だった。

コレクションに足すのは、リポジトリのルートに置く `app.json` で、1リポジトリが1アプリになる。

```json
{
  "aid": "(init が書く)",
  "name": "講演アンケート",
  "slug": "talk-survey",
  "protocol": "1.0.0",
  "members": {
    "owner@example.jp": { "*": "owner" }
  },
  "collections": {
    "responses": { "submitOnly": true, "statusField": "status" }
  },
  "public": {
    "enabled": true,
    "read": ["questions"],
    "submit": {
      "responses": {
        "auth": "verifiedEmail",
        "emailField": "email",
        "idFrom": "auth.uid",
        "stampField": "answeredAt",
        "initialStatus": "submitted",
        "createFields": ["email", "name", "answers", "comment", "answeredAt", "status"]
      }
    }
  },
  "views": [
    { "id": "public", "audience": "public", "path": "views/survey.html", "collections": ["questions"] },
    { "id": "desk",   "audience": "member", "path": "views/desk.html",   "collections": ["questions", "responses"] }
  ]
}
```

定義はコミットするが、回答はコミットしない。スキーマとビューはリポジトリのファイルとして持ち、レコードはクラウドの store に入る。権限は、メールアドレスを並べて決める。人を招くときは行を足して publish するだけで、招かれた側はこのリポジトリもこのマシンも要らない。

### publish は変換し、同時に止める

publish は `app.json` を Firestore の文書に変換する。そのとき、安全にできない宣言は何も書く前に拒否する。

```
app.json ──► projectApp        ──► apps/{aid}, apps/{aid}/config/public, schemas
         ├─► projectPublish    ──► 同じ文書データを、呼び出し側の書き込み順で
         ├─► projectAppViews   ──► apps/{aid}/{member,roster}/config
         └─► publishProblems   ──► 拒否。何も書く前に
```

拒否の判断は `publishChecks.ts` の検査が受け持つ。その一部を挙げる。

```
bindsSubmitterIdentity / submitOnlyProblems / identityBindings / authProblems
mailProblems / submitShapeProblems / capCeilingProblems / windowProblems
keyFieldCountProblems / coherenceProblems / statusCoherenceProblems
initialTransitionProblems
```

Firestore のルールは人が書かない。宣言から生成し、安全にできない形なら publish そのものが止まる。ソース中のコメントには「a linter is not a substitute for a rule」とある。MulmoServer の側には、publish の出力を Firestore rules エミュレータに通すテストがあり、「publish が書くものが `firestore.rules` の許すものと一致する」ことを確かめている。両リポジトリを通じて、これを証明しているテストはこの1本しかない。

### protocol は契約の版

射影にはすべて `protocol` が刻まれる（`src/appProtocol.ts`）。レンダラ（MulmoServer）は別にリリースされ、1ヶ月古いブラウザで動くこともある。文書の中でそれを知らせる手段は、この番号しかない。

| | |
|---|---|
| MAJOR | 読み手が理解していなければならない変更。古い読み手は部分描画せず拒否する。だから読み手が先に出る |
| MINOR | 古い読み手が安全に無視できる追加（`views[].live`、`views[].limit`） |
| PATCH | どちらでもない |

`APP_PROTOCOL` は、いまも 1.0.0 のままである。manifest スキーマが `.strict()` なので、知らないキーを渡された古いビルドは、落ちずに止まる。だからキーを足しても番号は動かない。番号が動くのは「既存キーの意味が変わる」ときだけで、その変化はスキーマからは見えない。

どのキーが何のためにあるかは、テンプレート9種（`server/skills/mulmoterminal-shared-app/templates/`）が実例で示している。

| キー | 何のため | 実例 |
|---|---|---|
| `assignee` | 名指しの人だけが承認、自分のぶんだけ | salon |
| `stampField` / `window.fromField` | 先着順と、クラスごとの開放時刻 | gym |
| `idFrom: "field"` / `mirror` | 事前に一覧できる予約単位 | meeting-room |
| `views[].live` | 見ている最中に動くページ | live-poll |
| `views[].limit` | コレクションがアプリの年齢とともに増える形 | append-feed |
| `writerDelete` | 放棄された行をオーナーが解放する | project-board |
| `names` / `idFrom: "auth.uid"` | 名前を一度登録してから作業を取る | project-board |
| `agents[]` | 参加者が人でなく AI | ai-council |
| `public.enabled: false` + `public.submit` | 名前が矛盾して見える組み合わせ | append-feed |

## どこで分かれ、どこでつながっているか

3つを並べると、同じ `schema.json` の使い道が3通りあるだけだと分かる。

| | 置き場所 | 誰が読む | 足すもの |
|---|---|---|---|
| コレクション | ローカルの JSON ファイル | 自分 + 自分のエージェント | `SKILL.md` |
| GUI Chat Protocol 経由 | 同上 | チャットの中 | `presentCollection` が `data.type` で返す |
| 共有アプリ | Firestore | 他人のブラウザ | `storage: firestore` + `app.json` |

違うのはデータの置き場と、誰に見せるかで、フィールドの定義もビューも共通している。パッケージの継ぎ目も、この分かれ方に沿っている。

| 層 | パッケージ | 依存の向き |
|---|---|---|
| プロトコル | `gui-chat-protocol` | 何にも依存しない。型だけ |
| コレクション runtime | `@mulmoclaude/core/collection` | discovery / store / Firestore backend / host seam |
| コレクション UI | `@mulmoclaude/collection-plugin` | Vue の面（chat View/Preview、embeds、calendar、record modal、i18n） |
| publish compiler | `@receptron/sharedapp` | `@mulmoclaude/core` を peer で（型3つだけ） |
| ホスト | MulmoTerminal | 上を全部束ね、deploy / publish / unpublish を持つ |
| レンダラ | MulmoServer | 公開ビューを描く。別リリース |

`sharedapp` を `core` から切り出した理由も、記録に残っている。切り出す前の90日で、24コミットが `@mulmoclaude/core` のリリース（8パッケージ + フル CI）を通っていたのに、MulmoClaude 自身はそのコードを一行も使っていなかった。

## 確かめたこと / 確かめていないこと

一次情報で確認したのは、次のことである。

- `gui-chat-protocol` の返り値2口・パッケージ構成・入力ハンドラ（README と spec）
- コレクションの構成と `schema.json` / `SKILL.md` の実物（`mulmoclaude-collections` の3件）
- 共有アプリのスキーマが `storage: { type: "firestore" }` 以外コレクションと同形であること（`templates/survey.md`）
- `publishChecks.ts` の検査名一覧、`protocol` の MAJOR/MINOR/PATCH 規則（sharedapp の README）
- パッケージ間の peer 依存（sharedapp README）

各テンプレートの宣言が実際に publish を通るかは、動かしていないので確かめられていない。`presentCollection` が返す `data` の具体的な形は、dist が minify されていて追えなかった。MulmoServer 側のレンダリング経路も、まだ見ていない。
