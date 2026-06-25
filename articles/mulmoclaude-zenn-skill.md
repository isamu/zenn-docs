---
title: "MulmoClaude の作業を Zenn 記事化するスキルを作り、本体のシステムスキルにするまで"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["MulmoClaude", "Zenn", "ClaudeCode", "生成AI"]
published: true
---

MulmoClaude でやった作業を、そのまま Zenn の技術記事にして共有したい——という動機から、
「作業 → Zenn 記事」を自動化するスキルを作った。さらにそれを MulmoClaude **本体に同梱される
システムスキル**へ昇格させ、PR まで出した。この記事はその一連の流れと、途中で分かった
スキルの仕組みをまとめたもの（実際この記事自体、作ったスキルで書いている）。

## スキルとは（MulmoClaude / Claude Code）

スキルは `SKILL.md` 一枚で定義する。frontmatter に `name` と `description`、本文は
「いつ・何をするか」を二人称で書いた手順書だ。`description` が呼び出しのトリガになるので、
動詞＋名詞で始め、ユーザーが言いそうなフレーズを入れておくのがコツ。

```md
---
name: zenn-article
description: 作業内容を Zenn 技術記事(md)に変換して保存する。「Zenn にまとめて」等で発火。
---

# 本文（二人称の手順書）
```

MulmoClaude のワークスペースでは、`data/skills/<slug>/SKILL.md` に書くと
`.claude/skills/<slug>/` へ**自動ミラー**されて即ディスカバリ対象になる。ボタン一つの
登録作業もいらない。

## まず workspace スキルを 2 つ作る

最初に、自分のワークスペース専用スキルとして 2 つ用意した。

- **`zenn-article`** — 今の会話・成果物を題材に Zenn 記事 md を生成して `articles/` に保存
- **`zenn-setup`** — Zenn 連携リポジトリの検出・最新化・clone・新規作成

frontmatter は Zenn の house style に合わせる。ポイントは `topics` に必ず `MulmoClaude`
タグを入れること（Zenn のトピックは自由語なので、新しいタグもそのまま生やせる）。

```yaml
---
title: "..."
emoji: "📝"
type: "tech"
topics: ["MulmoClaude", "Zenn"]
published: true
---
```

Zenn のスラッグ規約も押さえておく。**12〜50 文字、`^[a-z0-9_-]{12,50}$`**。
そして **公開後はスラッグ＝記事 URL なので変更不可**。タイトルの英単語から読みやすい
kebab-case を作り、短ければ日付や hex を足して伸ばす。

## 本体のシステム（プリセット）スキルに昇格させる

ここからが本題。「個人スキルではなく、MulmoClaude に最初から入っている `mc-` 系の
システムスキルにしたい」という話になり、配布の仕組みを調べた。分かったのはこう。

- プリセットスキルは本体リポの
  `packages/services/workspace-setup/assets/skills-preset/mc-<name>/SKILL.md` に置く
- 同期処理（`syncPresetSkills`）が **このディレクトリを走査**し、`mc-` プレフィックス＋
  `SKILL.md` を持つ各ディレクトリをワークスペースへコピーする
- **マニフェストへの登録は不要**。`preset-list.ts` はあくまで**プラグイン**（MCP ツールや
  Vue View を持つ npm パッケージ）用で、スキルには関係しない
- 旧 `server/workspace/skills-preset.ts` は `@mulmoclaude/workspace-setup` への
  薄い shim になっていて、正本はパッケージ側の `assets/` に移っていた

つまり、**`mc-zenn/SKILL.md` を 1 枚足すだけ**で配布される。コードもテストもいじらない。
初期化・記事作成・公開の 3 ワークフローを 1 つに統合した `mc-zenn` を書いた。

## 冪等な初期化を設計する

要件として「初期化済みなら何もしない／記事を書こうとして未初期化なら初期化する」という
**冪等性**が欲しかった。判定はシンプルに、ワークスペース内の `github/zenn/articles/` が
あるかどうかを marker にする。

- ある → 準備済み。再初期化しない
- ない → **git clone**（既存リポ）か **`npx zenn init`**（新規）で初期化

新規初期化は zenn-cli の標準フローに従う。

```bash
mkdir -p github/zenn
cd github/zenn && npm init --yes && npm install zenn-cli && npx zenn init
```

記事作成ワークフローの先頭（Step 0）でこの初期化判定を呼ぶので、ユーザーは
「Zenn にまとめて」から始めても、裏で勝手にセットアップされてそのまま執筆に入れる。
なお zenn.dev 側の「GitHub からのデプロイ」連携だけはブラウザ操作なので、そこは
自動化せず手順を案内する設計にした。

## PR を出すまで

本体リポへの変更なので、ちゃんと feature ブランチを切って進めた。

1. `main` から `feat/mc-zenn-skill` を作成
2. `mc-zenn/SKILL.md` と計画書 `plans/feat-mc-zenn-skill.md` を追加（個別 `git add`）
3. プリセット同期テストを実行 → **38/38 パス**、`mc-*` 検証も green
4. `feat:` プレフィックスでコミット → push → PR 作成

`format` / `lint` / `build` / `typecheck` は対象が `ts/json/yaml/vue` のみ。今回は
**markdown だけの追加**なのでコンパイル対象はなく、ビルド系ゲートは素通り——という確認も
込みで進めた。

## まとめ・学び

- スキルは `SKILL.md` 一枚。`description` がトリガなので「いつ呼ぶか」を具体的に書く
- workspace スキルは `data/skills/` → `.claude/skills/` の自動ミラーで即使える
- 本体プリセットは `assets/skills-preset/mc-*/` に置くだけ（ディレクトリ走査で配布、登録不要）
- 冪等化は「存在チェックできる marker」を 1 つ決めるのが肝（ここでは `articles/` の有無）
- 公開物の slug は URL で不可逆。命名は慎重に

「やった作業をそのまま記事化するスキル」は、こうして自分の作業ログを資産に変えてくれる。
次はこの `mc-zenn` がマージされたら、各ユーザーの MulmoClaude に最初から入る予定だ。
