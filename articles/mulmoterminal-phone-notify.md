---
title: "Claude Code のタスクが終わったらスマホに通知が来る開発環境を作った（MulmoTerminal）"
emoji: "📱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["MulmoTerminal", "ClaudeCode", "WebPush", "PWA", "個人開発"]
published: false
publication_name: "singularity"
---

Claude Code に長めのタスクを投げると、待っている時間がもったいない。かといって席を立つと、**いつ終わったか気づけない**。とくに [MulmoTerminal](https://github.com/receptron/mulmoterminal) でエージェントを並列に回していると、「どれかが手を止めて待っている」のを見逃しがちです。

そこで、**バックグラウンドのタスクが完了したらスマホに Web Push が飛ぶ**ようにしました。さらに、通知を受けたら **スマホからそのまま操作** もできます。

> 長いタスクを投げて席を立つ → 手が空いたらスマホがブルッと鳴る → 外出先でも続きを触れる

この記事では、その仕組みと使い方をまとめます。

## MulmoTerminal とは

[MulmoTerminal](https://github.com/receptron/mulmoterminal) は、**Claude Code（と Codex）をブラウザから並列に監督する**ためのツールです。`npx mulmoterminal` の一発でローカルサーバが立ち上がり、複数のエージェントセッションをグリッドに並べて、それぞれが **作業中 / 要対応 / 完了** のどれかを色で一望できます。並列運用の全体像は [グリッドビューの記事](https://zenn.dev/singularity/articles/mulmoterminal-grid-view) を参照してください。

本記事はその中の **スマホ通知（Web Push）＋モバイルからの操作（RemoteHost）** に絞った話です。

## なにが嬉しいか

MulmoTerminal はもともと、要対応セルを **画面上のベルと通知音** で知らせてくれます。でもそれは「PC の前にいる」前提。**席を外していると気づけない**し、スマホから様子を見ることもできませんでした。

Web Push を入れると:

- **バックグラウンドで走らせたタスクが終わった瞬間、スマホに OS 通知** が届く
- 通知を受けて、**スマホのブラウザ（PWA）からセッションを覗いたり、短い返信を送ったり** できる

「投げっぱなしにして散歩に出て、通知が来たら戻る」みたいな運用が現実的になります。

## 仕組み

構成はこうなっています。

```
[MulmoTerminal server (ローカル)]
   │ タスク完了（Stop フック）を検知
   ▼
   sendPush を呼ぶ ── HTTPS ──▶ [mulmoserver の Cloud Function]
                                     │ FCM で配信
                                     ▼
                              [あなたのスマホ / PWA]  ← OS 通知
```

ポイントは **通知の送信をサーバ側でやっている** ことです。ブラウザの Notification API に頼るのではなく、Claude のアクティビティフック（`Stop` イベント）を受けたローカルサーバが、通知配信サービス（`mulmoserver`）の Cloud Function を叩きます。配信先（どの端末に送るか）は、サインイン済みユーザーの uid からサーバ側で解決されるので、**PC の画面を閉じていてもスマホには届きます**。

通知の認証は **RemoteHost チャンネルの Google サインイン** が供給します。つまり「スマホから操作する RemoteHost」と「スマホへ通知する Web Push」は、同じ Google アカウントで束ねられています。

### どのタイミングで飛ぶか

初期は「今見ていないバックグラウンドのセッションが完了したとき」だけ飛ぶ設計でした（見ているペインは、そこにいるので鳴らさない ＝ 通知音と同じ考え方）。

ただ実際に使うと、「**自分がドライブしている本命のセッションを投げて席を立った**」ケースで飛ばず、不便でした。`active`（見ているペイン）はペイン選択ベースで、ウィンドウのフォーカスとは連動しないためです。そこで **すべての完了ターンで飛ぶ**ように変えました（見ているペインでも飛ぶ）。「毎回スマホが鳴るのはうるさい」という判断もあり得るので、そこは好みで調整できるようにしていく予定です。

### 接続が切れても静かに死なない

Web Push の送信には、サーバが RemoteHost セッション（の ID トークン）を持っている必要があります。ところが **サーバを再起動すると、そのセッションはメモリから消えます**（セッション本体はブラウザ側の localStorage に保持する「ブラウザ保持（case A'）」方式）。

初期実装はここに穴があって、「UI は接続中の表示のまま、サーバはセッションを失っている」状態だと、**通知音は鳴る（クライアント側）のに Web Push は黙って no-op**（`result: null`）になっていました。ハマりどころだったので、**クライアントがソケット再接続などを検知したら自動でセッションを再送する自己修復** を入れて解消しています。

## 使い方

### 1. セットアップ

まず（初回だけ）初期設定します。環境チェックと、Claude の履歴からの作業ディレクトリ登録をまとめてやってくれます。

```bash
npx mulmoterminal@latest init   # Node / claude / tmux などのチェック＋初期設定
npx mulmoterminal               # http://localhost:34567 が開く
```

### 2. ターミナル側（PC）

1. ツールバーの **📱 RemoteHost**（`phonelink` アイコン）を開き、**Connect（Google サインイン）**。スマホと **同じ Google アカウント** で。
2. **設定（⚙）→ Web Push notifications** の **「Notify my devices when a task finishes」** を ON（既定は OFF）。

### 3. スマホ側（PWA）

1. スマホのブラウザで **[https://mulmoserver.web.app](https://mulmoserver.web.app)** を開く（RemoteHost パネルの **QR コード** でもOK）。
2. **同じ Google アカウント** でサインイン。
3. **通知を有効化**（この端末を通知先として登録）。
4. **ホーム画面に追加**（PWA 化）しておくと配信が安定します。

これで、バックグラウンドのタスクが完了するたびにスマホへ通知が飛びます。

### 通知が来ないとき

- **見ているペインのタスク**を終わらせていませんか？ → 見ていないセッションでも試す
- **RemoteHost が未接続** → もう一度 Connect（サーバ再起動後などはリロードで再接続）
- スマホ側で **通知が有効化されていない / 端末未登録** → PWA で有効化
- **同じ通知が複数回**届く → スマホに古い端末登録が残っている可能性。mulmoserver 側で登録し直すと解消

## まとめ

- MulmoTerminal は、Claude Code のタスク完了を **サーバ送信の Web Push でスマホに通知** できます。
- さらに **RemoteHost** で、通知を受けたスマホからそのまま操作できます。
- 「投げて席を立ち、通知で呼び戻される」開発ループが現実的になります。

セットアップは `npx mulmoterminal@latest init` → 起動 → RemoteHost を Connect → Web Push を ON → スマホで PWA を開く、の順です。並列運用そのものの話は [グリッドビューの記事](https://zenn.dev/singularity/articles/mulmoterminal-grid-view) と [日本語ガイド](https://zenn.dev/singularity/articles/mulmoterminal-guide-ja) もどうぞ。

- リポジトリ: https://github.com/receptron/mulmoterminal
- npm: https://www.npmjs.com/package/mulmoterminal
