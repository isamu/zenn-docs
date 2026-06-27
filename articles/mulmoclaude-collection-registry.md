---
title: "MulmoClaude のコレクションを共有する：キュレーション済み GitHub レジストリの設計と実装"
emoji: "📦"
type: "tech"
topics: ["MulmoClaude", "TypeScript", "Vue", "GitHub", "個人開発"]
published: true
---

## はじめに

[MulmoClaude](https://github.com/receptron/mulmoclaude) には「コレクション」という機能がある。`schema.json` でフィールドを定義すると、ホスト側がそのスキーマからテーブル・カレンダー・カンバン・カスタムビューを自動でレンダリングする——コレクションごとのホストコードはゼロの、**スキーマ駆動データアプリ**だ。読書リスト、映画リスト、投資ウォッチリストといったものを、エージェントとの会話だけで作れる。

```
data/skills/<slug>/
├── SKILL.md        # エージェント向けの使い方
├── schema.json     # フィールド定義（これが UI を決める）
├── views/*.html    # 任意のカスタムビュー
└── templates/*.md
data/<name>/items/<id>.json   # レコード本体
```

ここで自然に出てくる欲求が「**自分が作った良いコレクションを、他のユーザにも使ってほしい**」だ。本記事は、その配布機構——**キュレーション済みレジストリ**——をどう設計・実装したかのまとめ。設計判断、セキュリティ、UX、そして開発中にレビューボットや CodeQL から学んだことまで含めて書く。

## 全体像：producer → curate → consumer ループ

配布は 3 者のループになる。

1. **寄稿者 (producer)** — 自作コレクション（`SKILL.md` + `schema.json` + views + 任意の seed データ）を、公式レジストリ `receptron/mulmoclaude-collections` に **Pull Request** で送る。
2. **キュレーター (curate)** — CI（自動検証）＋人手で審査し、問題なければ merge する。
3. **利用者 (consumer)** — MulmoClaude の `/collections` 「Discover（発見）」タブから、レジストリを参照して良いコレクションを見つけ、ワンクリックで **取り込む (import)**。

```
寄稿者 ──PR──▶ receptron/mulmoclaude-collections ──CI生成──▶ index.json (GitHub Pages)
                          ▲                                          │
                          │ merge                                    │ 1 GET + cache
                     キュレーター                                     ▼
                                                   利用者の MulmoClaude「Discover」タブ
                                                              │ import（再検証して着地）
                                                              ▼
                                                   data/collections/<slug>/
```

ポイントは「GitHub をそのまま叩くのではなく、**バックエンドが index ファイルを見てデータを出す**」ところ。利用者のクライアントは GitHub API を直接叩かず、サーバが配信用インデックスを 1 回取得してキャッシュする。

## 設計判断

### 1. 配信は index.json + per-collection manifest.json

レジストリ側の CI（GitHub Actions）が、各コレクションのメタデータを集約した `index.json` と、コレクションごとの `manifest.json` を生成し、**GitHub Pages** で公開する。

- Discover タブ：`index.json` を 1 回 GET してカタログ表示（TTL キャッシュ）。
- 取り込み：選んだコレクションの `manifest.json` ＋ raw ファイルだけを取得。

GitHub Trees API を毎回叩く方式も検討したが、「全カタログは 1 ファイル取得＋キャッシュ、個別取り込みは必要分だけ」の方がレイテンシ・レート制限の両面で素直だった。`index.json` は `schemaVersion` 付きの**公開契約**として設計し、将来の任意 URL 追加にも備えている。

### 2. 作者名前空間とリネーム衝突回避

レジストリ内のレイアウトは作者で名前空間を切る：

```
collections/<author>/<slug>/
├── SKILL.md
├── schema.json
├── meta.json          # author, slug, version, title, license, ...
└── seed/items/*.json  # 任意（寄稿者の実データ）
```

利用者が取り込むとき、`dataPath` をローカル規約 `data/collections/<localSlug>/items` に正規化する。ローカルで slug が衝突したら、**別 slug にリネームして取り込む**（既存を壊さない）。既に同じものを取り込んでいれば「更新」として扱う。

### 3. 信頼境界：index は表示用、実体は再検証

`index.json` は**表示のためのデータ**に過ぎない。取り込み時には、fetch した実体（schema・レコード）を `packages/core` の検証器で**もう一度検証してから**ローカルに着地させる。レジストリを信用しすぎない、というのが基本姿勢だ。

取り込み自体も原子的にした。一時ディレクトリに展開 → 既存をバックアップ → アトミックに入れ替え → 失敗したらロールバック、という staged-swap で「途中で失敗して半端な状態が残る」を防ぐ。

### 4. なりすまし防止：author = GitHub login を CI で検証

`meta.json` の `author` は GitHub アカウント名とし、レジストリ側の CI が **PR 作成者の login と一致するか**を検証する。さらにパスの `collections/<author>/...` セグメントとも一致を必須にして、他人の名前空間への寄稿を弾く。

```js
// レジストリの validate スクリプト（概念）
const prAuthor = process.env.PR_AUTHOR;        // CI が渡す
if (meta.author !== prAuthor) fail("meta.author must equal the PR author");
if (pathAuthorSegment !== meta.author) fail("path namespace must equal meta.author");
```

## Export（producer 側）の実装

寄稿バンドルを作る `writeCollectionExport` は、ユニットテストしやすいように「明示的なディレクトリ＋ワークスペースルートを受け取る純粋な核」と、ライブのワークスペースから解決する薄い glue に分けた。ここで効いてきた実装上の判断が 3 つある。

### credential は hard-fail、PII は警告

seed（寄稿者の実データ）を同梱できるが、そのまま公開すると事故になる。秘密情報のパターン（AWS キー、`ghp_…` トークン、秘密鍵、`sk-…` 等）を検出したらそのレコードは**スキップして警告**、メールアドレスのような PII は**警告しつつ本人責任で残す**。credential の検出だけは妥協せず弾く。

### path-injection 対策：resolveWithin

`author` / `slug` はリクエスト由来なので、パス式に流れる箇所は path traversal の温床になる。正規表現 `^[a-z0-9-]+$` でアンカー検証していても、**CodeQL は正規表現 `.test()` をサニタイザと認識しない**（後述）。CodeQL が認識し、かつ実際に堅牢な対策は「resolve してから封じ込めを確認する」バリアだ。

```ts
// segments を root 配下に解決し、root の外に出たら null
function resolveWithin(root: string, ...segments: string[]): string | null {
  const resolvedRoot = path.resolve(root);
  const target = path.resolve(resolvedRoot, ...segments);
  return target === resolvedRoot || target.startsWith(resolvedRoot + path.sep)
    ? target
    : null;
}
```

`outDir`（export 先）と `skillDir` / `dataDir`（ワークスペース配下）に適用し、外に出たら 400 で弾く。seed のファイル名には `path.basename()` を噛ませる。これで CodeQL の 7 件の high severity アラートが消えた。

### 必須ファイルの欠落は hard-fail

`SKILL.md` と `schema.json` は必須、`screenshot.png` は任意。当初これらを同じ「無ければスキップ」で扱っていたため、`SKILL.md` 欠落でも export が「成功」してしまい、下流のキュレーションで落ちる不完全バンドルが作れてしまった。必須ファイルの存在を **出力ディレクトリを触る前に** 検査し、欠けていれば即エラーにした。

```ts
async function findMissingRequired(skillDir: string): Promise<string | null> {
  for (const file of ["SKILL.md", "schema.json"]) {
    if ((await pathKind(path.join(skillDir, file))) !== "file") return file;
  }
  return null;
}
```

## Contribute ボタン：なぜ「ボタン → エージェント」なのか

利用者が「発見・取り込み」できるようになったら、次は寄稿側の入口だ。コレクションのカードに「Contribute（共有）」ボタンを置いた。ここで重要な設計判断がある。

**ボタンを押したら何が起きるべきか？** バックエンドのエンドポイントを叩いてローカルにバンドルを作るだけでは、**寄稿は完了しない**。レジストリへの Pull Request を開くには git と `gh` が要るが、アプリのサーバはユーザの代わりに GitHub PR を作れない。

そこで「ボタン → エージェント」にした。ボタンは、寄稿手順を埋め込んだプロンプトで**新しいチャットを起動する**（`startChat`）。あとはエージェントが、ワークスペースのファイルからバンドルを組み、レジストリを clone し、index を再生成・検証し、`gh` で PR を開く——という full loop を実行する。発見性（ボタン）とエージェントの git/gh 能力を組み合わせる形だ。

```ts
// ボタンのハンドラ：プロンプトで新規チャットを起動するだけ
cui.startChat(
  t("collectionsView.contributePrompt", { title, slug }),
  cui.generalRoleId,
);
```

### いきなり走らせない：確認ダイアログ

ただ、このボタンは**押した瞬間にエージェントが走り、export して GitHub PR を開く**。一発で取り返しのつかない操作なので、間に確認ダイアログを挟んだ。

```ts
async function startContributeChat(collection: CollectionSummary): Promise<void> {
  const ok = await cui.confirm({
    message: t("collectionsView.contributeConfirm", { title: collection.title }),
    confirmText: t("collectionsView.contribute"),
    variant: "primary",
  });
  if (!ok) return;
  cui.startChat(/* 寄稿プロンプト */);
}
```

「『◯◯』コレクションをシェアしますか？スキルが起動してコレクションを export し、レジストリに GitHub PR を作成します。」——確認してから初めて `startChat` が走る。Playwright の e2e で、確認パス（ちょうど 1 回起動）・キャンセルパス（0 回）・キーボード操作の 3 ケースを回している。

## 開発プロセスでの学び

このシリーズはレビューボット（Codex / CodeRabbit）と CI を回しながら小さく積み上げた。汎用的に役立った気づきをいくつか。

- **CodeQL は正規表現ガードをサニタイザと見なさない。** `^[a-z0-9-]+$` で弾いていても path-injection は消えない。`path.resolve` ＋ 接頭辞チェックの「封じ込めバリア」が、スキャナにも実際の防御にも効く正解。
- **「成功したけど不完全」を作らない。** 必須・任意を同じ「無ければスキップ」で処理すると、必須欠落でも成功扱いになる。必須は副作用の前に検査して hard-fail。
- **i18n は 8 ロケールを型でロックステップ。** 1 つのキーを足すと型エラーで 8 ファイル全部に追加を強制される。エージェント向けの長文プロンプトも全ロケール用意した。
- **vue-i18n はメッセージ中の `<...>` を HTML と誤検知して警告する。** プロンプトに `collections/<author>/<slug>/` のような角括弧パスを書くと XSS ヒューリスティックに引っかかる。角括弧を外して散文にすれば消える（エージェントは正確なレイアウトをヘルプから学ぶので情報は欠けない）。
- **ボタンは「何をするか」だけでなく「誰が実行できるか」で設計する。** GitHub PR はサーバには作れない。だから「ボタン → エンドポイント」ではなく「ボタン → エージェント」を選んだ。

## まとめ

スキーマ駆動コレクションの配布を、**index 配信 + 作者名前空間 + 取り込み時再検証 + author 認証**という骨格で組んだ。producer 側は「ボタンがエージェントを起動して export→PR を完遂する」形にし、誤発火を防ぐ確認ダイアログを挟んだ。

ユーザ生成コンテンツの配布を作るときは、(1) 信頼境界（外部データは表示用と割り切り、着地前に再検証）、(2) なりすまし防止（作者 = 認証済み ID）、(3) 取り返しのつかない操作の確認、の 3 点がそのまま効く。スキーマ駆動 UI と「ボタン → エージェント」の組み合わせは、ホストコードを増やさずに機能を足せて気持ちがいい。
