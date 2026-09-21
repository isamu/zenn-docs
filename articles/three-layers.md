---
title: "宣言1枚が、手元のアプリにも、共有サービスにもなる — GUI Chat Protocol / コレクション / 共有アプリ"
emoji: "🧩"
type: "tech"
topics: ["AI", "typescript", "firebase", "ClaudeCode", "設計"]
published: true
publication_name: "singularity"
---

MulmoTerminal / MulmoClaude には、一見すると無関係な3つの仕組みがあります。
これは**1本の線**で、同じ `schema.json` が置き場所と宣言の足し方で3通りに使われます。

**2026-09-21 時点の実装に基づきます。** 確かめていないことは最後に明記しました。

## 3つは別機能ではない

MulmoTerminal / MulmoClaude には、一見すると無関係な3つの仕組みがある。

- **GUI Chat Protocol** — エージェントの返り値で画面を描く
- **コレクション** — スキーマ駆動のデータアプリ
- **共有アプリ** — 他人が使えるように公開する

**これは1本の線だ。** 同じ `schema.json` が、置き場所と宣言の足し方で3通りに使われる。

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

以下、下の層から順に。

---

## 1. GUI Chat Protocol — 返り値にもう1つ口を足す

[receptron/gui-chat-protocol](https://github.com/receptron/gui-chat-protocol)。npm に publish 済み。

### 何をしているか

通常の tool call は、実行結果をテキストで LLM に返す。このプロトコルは**返り値を2つに分ける**。

```ts
return {
  message: "9月の支出をグラフにしました",   // LLM が読む。会話はこれで続く
  data: {
    type: "chart",                          // UI が読む。type で描画先が決まる
    option: { /* ECharts の設定 */ },
  },
};
```

**LLM 側からは、標準の function calling と区別がつかない。** システムプロンプトでツール定義を
受け取り、呼び、テキストを受け取る。新しい概念を覚える必要がない。

つまり**新しいアーキテクチャではなく、既存の tool call への加算**であり、MCP とも function
calling とも競合しない。

### パッケージ構成

| | |
|---|---|
| `src/types.ts` / `schema.ts` / `inputHandlers.ts` | フレームワーク非依存のコア |
| `src/vue.ts` / `src/react.ts` | Vue 3 / React 18-19 のアダプタ型 |
| `spec/GUI_CHAT_PROTOCOL.md` | 設計と動機 |
| `spec/CREATING_A_PLUGIN.md` | プラグインの作り方 |

UI を持たないロジックはコアだけで書ける。描画の型付けだけがフレームワークごとに分かれる。

### 入力方向もある

`inputHandlers.ts` が `FileInputHandler` と `ClipboardImageInputHandler` を定義している。
**チャットに落としたファイルや貼り付けた画像**を、プラグインが受け取れる。

### いま動いているプラグイン

| ツール | 出るもの |
|---|---|
| `presentChart` | ECharts |
| `presentForm` | 構造化された入力フォーム |
| `presentDocument` | Markdown / Marp スライド |
| `presentHtml` | 自己完結した HTML ページ |
| `presentCollection` | **スキーマ駆動のデータ画面** ← 次の層 |
| `presentShapeScript` | 対話的な3D |
| `presentMulmoScript` | 動画・スライドの絵コンテ |
| `@mulmoclaude/accounting-plugin` | 複式簿記の帳簿 |
| `@mulmochat-plugin/generate-image` | 画像生成 |

**`presentCollection` がこのプロトコルの一プラグインである**という事実が、次の層への接続点になる。

---

## 2. コレクション — スキーマがデータモデルと UI の両方を宣言する

`@mulmoclaude/collection-plugin`。自己申告は **Schema-driven Collections plugin**。

### 1つのコレクションはディレクトリ1つ

```
<slug>/
  schema.json      データモデルと UI の宣言
  SKILL.md         エージェントへの説明書
  meta.json        作者・slug・説明・ライセンス
  views/*.html     任意。カスタムビュー
  seed/items/*.json 任意。サンプルレコード
  manifest.json    取り込み時に使うファイル一覧
```

### schema.json

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

これを置くと**表・入力フォーム・カレンダー**が描画される。**ホスト側にコレクション固有の
コードは無い。** `views[]` を足すと自前の画面も足せる。

データは **1件1ファイルの JSON** が `dataPath` の下に並ぶ。DB は無い。git に入るし、
エディタで開けるし、`rm` で消える。

### SKILL.md — エージェント側の接続

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

**「こう言われたらこれを使え」と「データはこういう形だ」を自然言語で書いてあるだけ。**
エージェントはこれを読み、`manageCollection` で JSON を読み書きする。

⚠️ **スキーマを変えたら SKILL.md も直す必要がある。** そこは自動ではない。

### レジストリ

[receptron/mulmoclaude-collections](https://github.com/receptron/mulmoclaude-collections)。

- `scripts/build-index.mjs` が `collections/` を走査して **`index.json`** を生成
  （バックエンドが1回の GET で取る Discover のカタログ）
- コレクションごとに **`manifest.json`**（取り込み時に取得するファイル一覧）
- 投稿は `collections/<自分の GitHub ログイン名>/<slug>/` に足して PR。
  **他人の名前空間には出せない**（CI が弾く）

---

## 3. 共有アプリ — 同じスキーマを、他人のブラウザへ

`@receptron/sharedapp`。自己申告は **「The shared-app compiler: `app.json` in, Firestore
documents out — and what publish refuses before it writes any of them.」**

### コレクションとの差は、実は小さい

共有アプリのスキーマは、こうなる。

```json
{
  "title": "設問",
  "icon": "quiz",
  "primaryKey": "id",
  "storage": { "type": "firestore" },     ← ここだけ
  "fields": { ... }
}
```

**`storage` が変わるだけで、`fields` も `views` もコレクションと同じ形。**
置き場所も `.claude/skills/<name>/schema.json` で変わらない。

`sharedapp` は `@mulmoclaude/core` から **`CollectionSchema` 型と `isValidCollectionName`、
`isSafeCustomViewPath` を peer で借りている**。**型を共有している**のがその証拠で、
機能の設計名も「shareable collections」だった。

### 足すのは app.json

**1リポジトリが1アプリ。** ルートの `app.json` が宣言。

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

**定義はコミットされ、回答はコミットされない。** スキーマとビューはリポジトリのファイル、
レコードはクラウドの store。

**権限はメールアドレスの列挙。** 招待は行を足して publish するだけで、相手はこのリポジトリも
このマシンも要らない。

### publish は compiler であり、gate でもある

```
app.json ──► projectApp        ──► apps/{aid}, apps/{aid}/config/public, schemas
         ├─► projectPublish    ──► 同じ文書データを、呼び出し側の書き込み順で
         ├─► projectAppViews   ──► apps/{aid}/{member,roster}/config
         └─► publishProblems   ──► 拒否。何も書く前に
```

`publishChecks.ts` が持っている検査の一部:

```
bindsSubmitterIdentity / submitOnlyProblems / identityBindings / authProblems
mailProblems / submitShapeProblems / capCeilingProblems / windowProblems
keyFieldCountProblems / coherenceProblems / statusCoherenceProblems
initialTransitionProblems
```

**Firestore のルールを人が書かない。** 宣言から生成され、安全にできない形は publish 自体が
止まる。ソース中のコメントに **「a linter is not a substitute for a rule」** とある。

そして **MulmoServer が、publish の出力を Firestore rules エミュレータに通すテスト**を
持っている。両リポジトリを通じて「publish が書くものが `firestore.rules` の許すものと一致する」
ことを証明している唯一のテスト。

### protocol — 契約の版

すべての射影に `protocol` が刻まれる（`src/appProtocol.ts`）。**レンダラ（mulmoserver）は
別リリースで、1ヶ月古いブラウザで動きうる**ので、文書の中でそれを知らせる唯一の手段。

| | |
|---|---|
| **MAJOR** | 読み手が理解していなければならない変更。古い読み手は**部分描画せず拒否する**。だから読み手が先に出る |
| **MINOR** | 古い読み手が安全に無視できる追加（`views[].live`、`views[].limit`） |
| **PATCH** | どちらでもない |

`APP_PROTOCOL` は **いまも 1.0.0**。キーを足しても番号は動かない。manifest スキーマが
`.strict()` なので、**知らないキーを渡された古いビルドは、落とさずに止まる**からだ。
動くのは「既存キーの意味が変わる」ときだけで、それはスキーマに見えない。

### 宣言に出てくる主なキー

テンプレート9種（`server/skills/mulmoterminal-shared-app/templates/`）が、どのキーが何のために
あるかを実例で書いている。

| キー | 何のため | 実例 |
|---|---|---|
| `assignee` | 名指しの人だけが承認、自分のぶんだけ | salon |
| `stampField` / `window.fromField` | 先着順と、クラスごとの開放時刻 | gym |
| `idFrom: "field"` / `mirror` | 事前に一覧できる予約単位 | meeting-room |
| `views[].live` | 見ている最中に動くページ | live-poll |
| `views[].limit` | コレクションがアプリの**年齢**とともに増える形 | append-feed |
| `writerDelete` | 放棄された行をオーナーが解放する | project-board |
| `names` / `idFrom: "auth.uid"` | 名前を一度登録してから作業を取る | project-board |
| `agents[]` | 参加者が人でなく AI | ai-council |
| `public.enabled: false` + `public.submit` | 名前が矛盾して見える組み合わせ | append-feed |

---

## 分岐点はどこか

**同じ `schema.json` が、3通りに使われる。**

| | 置き場所 | 誰が読む | 足すもの |
|---|---|---|---|
| **コレクション** | ローカルの JSON ファイル | 自分 + 自分のエージェント | `SKILL.md` |
| **GUI Chat Protocol 経由** | 同上 | チャットの中 | `presentCollection` が `data.type` で返す |
| **共有アプリ** | Firestore | 他人のブラウザ | `storage: firestore` + `app.json` |

**分岐しているのは、データの置き場と、誰に見せるかだけ。** フィールド定義もビューも共通。

---

## 継ぎ目の整理

| 層 | パッケージ | 依存の向き |
|---|---|---|
| プロトコル | `gui-chat-protocol` | 何にも依存しない。型だけ |
| コレクション runtime | `@mulmoclaude/core/collection` | discovery / store / Firestore backend / host seam |
| コレクション UI | `@mulmoclaude/collection-plugin` | Vue の面（chat View/Preview、embeds、calendar、record modal、i18n） |
| publish compiler | `@receptron/sharedapp` | `@mulmoclaude/core` を **peer** で（型3つだけ） |
| ホスト | MulmoTerminal | 上を全部束ね、deploy / publish / unpublish を持つ |
| レンダラ | MulmoServer | 公開ビューを描く。**別リリース** |

`sharedapp` が `core` から切り出された理由も記録されている。**切り出す前の90日で24コミットが
`@mulmoclaude/core` のリリース（8パッケージ + フル CI）を通っていて、MulmoClaude 自身はその
コードを一行も使っていなかった。**

---

## 確かめたこと / 確かめていないこと

**一次情報で確認した**

- `gui-chat-protocol` の返り値2口・パッケージ構成・入力ハンドラ（README と spec）
- コレクションの構成と `schema.json` / `SKILL.md` の実物（`mulmoclaude-collections` の3件）
- 共有アプリのスキーマが `storage: { type: "firestore" }` 以外コレクションと同形であること
  （`templates/survey.md`）
- `publishChecks.ts` の検査名一覧、`protocol` の MAJOR/MINOR/PATCH 規則（sharedapp の README）
- パッケージ間の peer 依存（sharedapp README）

**確かめていない**

- 各テンプレートの宣言が実際に publish を通るか（動かしていない）
- `presentCollection` が返す `data` の具体的な形（dist が minify されていて追えていない）
- MulmoServer 側のレンダリング経路
