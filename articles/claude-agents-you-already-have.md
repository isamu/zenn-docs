---
title: "Claude Code に並列セッションの一覧が入っているのを知っていますか（claude agents）"
emoji: "🛰️"
type: "tech"
topics: ["claudecode", "AI", "vibecoding", "個人開発", "ターミナル"]
published: false
publication_name: "singularity"
---

Claude Code を何個も並べて動かしている人に聞きたいのですが、**これ、打ったことありますか。**

```bash
claude agents
```

全セッションが状態別に並びます。**Working / Needs input / Completed。** 各行に、そのセッションが何をしているかの1行要約つき。

**Claude Code 本体の機能です。** 何もインストールしません。

私はこれを、**並列で動かすためのツールを自分で作ったあとに**知りました。

## いつから入っていたのか

changelog を見ました。

```
## 2.1.139
- Added agent view (Research Preview): a single list of every Claude Code session
  — running, blocked on you, or done. Run `claude agents` to get started.
```

**2.1.139。** そして worktree のほうはもっと前です。

```
## 2.1.49
- Added `--worktree` (`-w`) flag to start Claude in an isolated git worktree
```

`claude --worktree feature-auth` で、worktree を切ってそこでセッションを起動します。**cleanup も `.worktreeinclude`（worktree に持ち込むファイルの指定）も、大きなモノレポ向けの `worktree.sparsePaths` も**あります。`--tmux` を足せば tmux セッションも作ります。

**両方とも、私が並列管理のツールを作り始めるより前からありました。**

## 何ができるのか

[公式ドキュメント](https://code.claude.com/docs/en/agent-view)と手元で確認した範囲です。

| | |
|---|---|
| 一覧 | 状態でグループ化（Pinned / Ready for review / **Needs input** / Working / Completed） |
| **要約** | **Haiku が各セッションの作業内容を1行で**書く |
| 状態 | アイコン + 色（Working=アニメーション / Needs input=黄 / Completed=緑 / Failed=赤） |
| **PR** | 行の端に `#1234` のバッジ |
| 通知 | ターミナル通知、タブタイトルに待ち件数（`2 awaiting input · claude agents`） |
| Peek | `Space` で最新の出力だけ覗く（開かずに） |
| 起動 | 下の入力欄にプロンプトを打つと、新しいバックグラウンドセッションが立つ |

**⚠️ 1つ注意。** 一覧されるのは**バックグラウンドのセッション**です。いま対話中のものは、バックグラウンドに送るまで出てきません。ここを知らないと「動いていない」と思います。

## なぜ誰も知らないのか

**Research Preview のまま、静かに入っているから**だと思います。大きな告知がなく、CLI のサブコマンドとして増えているだけなので。

実際、私が話を聞いた範囲では**一度も出てきませんでした**。

| 出てきた言葉 | 回数 |
|---|---|
| VS Code | 11 |
| worktree | 10 |
| Cursor | 8 |
| tmux | 2 |
| **`claude agents`** | **0** |

ユーザーインタビュー6人と、自発的に書かれた記事2本を数えたものです。

そのうち1本は「**VS Code 拡張もターミナル6分割もいらなくなるかもしれない**」というタイトルで、**まさにこの領域を能動的に調べて書かれた記事**でした。それでも本体の機能に触れていません。

もちろん n=6 で、しかも全員が私のツールのユーザーなので**標本は偏っています**。「誰も知らない」とは言えません。言えるのは「**話を聞いた範囲では、誰も知らずに手で解いていた**」までです。

海外の Reddit スレッドも見ました。「並列でエージェントを動かすツールを教えて」という質問に **20 以上のツール名**が挙がっているのに、**`claude agents` は1件も出てきません**。

## それでも足りなかったところ

ここからは、なぜ私が別のものを作ったかの話です。**足りているなら読む必要はありません。**

`claude agents` は **1行1セッション**の一覧です。`Space` を押すと最新の出力を覗けます。

**これは分類には最高です。** どれが待っているか、どれが終わったかが一目で分かる。

**でも、読むのには向いていません。**

私がユーザーに話を聞いて、一番強く出てきた困りごとがこれでした。

> ターミナルを6分割すると、1つ1つのペインが小さすぎる。**4000字の出力を読むのにスクロールとリサイズが必要**になる

エージェントは長い返答を返します。**それを読みながら、他の5体が走り続けているのを見ていたい。** 1行の要約は、この用途を解きません。かといって1つずつ開くと、他が見えなくなる。

そこで私は「**複数のライブ端末を、同時に、そのまま見せる**」ほうに賭けました。要約しない方向です。

これが正しいかは、正直まだ分かりません。**要約で足りる人のほうが多い可能性は普通にあります。** ただ、少なくとも私と、話を聞いた6人にとっては、そこが痛みでした。

## 何が言いたいか

**まず本体を試してください。** 無料で、インストールもなく、作っているのはエージェント本体のチームです。

```bash
claude agents
claude --worktree my-feature
```

そのうえで足りなければ、この分野には **20 以上のツール**があります。Vibe Kanban（★27,500）、Nimbalyst（旧 Crystal・iOS/Android アプリつき）、Parallel Code（Gemini 対応）、Conductor（Mac）、Claude Squad（TUI）── ほぼ全部が OSS で、worktree ベースで、ローカルで動きます。

**「MIT だから」「ローカルだから」は差別化ではなく、この分野の最低ライン**でした。選ぶなら「**自分はどこで詰まっているのか**」から逆に選ぶほうが早いです。

- **読めない** → 複数のライブ画面
- **レビューできない** → diff 中心
- **見失う** → ボード / タスク
- **隔離できない** → **どれも解決しません**（devcontainer か VM を下に敷く）

私が作っているのは1つ目です（[MulmoTerminal](https://github.com/receptron/mulmoterminal)、MIT）。**ただ、この記事で言いたかったのはそこではなく、`claude agents` のほうです。**

知らずに tmux を6分割していた人が、私を含めて何人もいたので。

---

## 関連記事

- [git worktree、便利なのは知っていた。使えなかったのは、覚えておくことが多すぎたから](https://zenn.dev/singularity/articles/worktree-cost-zero)
- [ESLint と TypeScript の設定、ネット記事のコピペで作っていませんか？](https://zenn.dev/singularity/articles/what-else-was-off)
- [1日500コミットは、もう読めない ── だからコードレビューをやめた](https://zenn.dev/singularity/articles/stopped-reviewing-my-code)
