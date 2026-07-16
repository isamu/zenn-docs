---
title: "MulmoTerminal 設定リファレンス — 何を足すと何が変わるか、1つずつ"
emoji: "⚙️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["MulmoTerminal", "ClaudeCode", "設定", "TypeScript", "個人開発"]
published: false
publication_name: "singularity"
---

[MulmoTerminal](https://github.com/receptron/mulmoterminal) は `npx mulmoterminal` の一発で動きますが、**設定を足していくほど手に馴染みます**。この記事は、その設定を **1つずつ「こう書くと、こうなる」** で並べたリファレンスです。全体像やグリッドの使い方は [日本語ガイド](https://zenn.dev/singularity/articles/mulmoterminal-guide-ja) と [グリッドビューの記事](https://zenn.dev/singularity/articles/mulmoterminal-grid-view) をどうぞ。

設定は **3つの層** に分かれています。上から順に紹介します。

| 層 | 置き場所 | 何を決めるか | 反映 |
| --- | --- | --- | --- |
| 環境変数 / `.env` | 起動時の環境 | ポート・使う CLI・作業ディレクトリなど**サーバの起動条件** | 起動時 |
| グローバル設定 | `~/.mulmoterminal/config.json` | プリセット・通知・ヘッダーボタンなど**アプリ全体の挙動** | 設定モーダル（⚙）は即時、一部は再起動 |
| ディレクトリ設定 | `<project>/.mulmoterminal.json` | そのフォルダで開くターミナルの**見た目と音** | 次にそのセルを開いたとき |

その前に、初回だけ走らせておくと楽な `init` から。

## 0. 初期設定 — `npx mulmoterminal init`

```bash
npx mulmoterminal@latest init
```

これは **冪等な初期設定コマンド**です。何度実行してもよく、既存の設定を上書き・補完します。やること:

- **環境チェック**：Node のバージョン、`claude` / `tmux` / `gh` / `codex` が入っているかを確認し、足りなければ入れ方（コマンド）を表示します。
- **作業ディレクトリの自動登録**：`~/.claude/projects/` にある Claude Code の履歴を読み、**最近使ったプロジェクトのパスを `cwdPresets` に登録**します。次項のランチャーに、いつものフォルダが最初から並びます。
- **設定スキルの起動（任意）**：Claude Code が入っていれば、`/mulmoterminal-config` スキルを起動して対話的に設定を詰められます。

```bash
npx mulmoterminal init --dry-run   # 何を書き込むかだけ表示（変更しない）
```

> 履歴からパスを拾うとき、プロジェクト名そのものは復元できません（`/`・`.`・`-` がすべて `-` にエンコードされるため）。なので **ディレクトリ名を推測せず、履歴（transcript）に記録された実際の `cwd` を読んで**登録しています。存在しないパスは登録されません。

---

## 1. 環境変数 / `.env`（サーバの起動条件）

サーバは環境変数で設定します。`.env` を置けば `node --env-file-if-exists=.env` 経由で読まれます（**すべて既定値があるので `.env` は無くても動きます**）。

例（`.env`、gitignore 推奨）:

```
CLAUDE_CWD=/Users/you/my-project
```

主なものを「足すとどうなるか」で:

- **`PORT`（既定 `34567`）** — 開く URL のポート。`PORT=8080` にすると `http://localhost:8080`。
- **`CLAUDE_CWD`（既定：`npx` を叩いたディレクトリ）** — 各 `claude` を動かす作業ディレクトリ。**サイドバーにどのプロジェクトのセッションが並ぶかを決めます**。`npx mulmoterminal --cwd ./foo` でも指定でき、相対パスも可。ただし `.env` に書く場合は**絶対パス**で（`~` は展開されません）。
- **`CLAUDE_BIN`（既定 `claude`）** — 起動する Claude Code のバイナリ。別ビルドを使うときにパスを差し替えます。
- **`CLAUDE_PERMISSION_MODE`（既定 `auto`）** — 各 `claude` 起動時に渡す権限モード。
- **`MT_TITLE_MODEL`（既定 `haiku`）** — セルのヘッダーに出る **AI 生成タイトル** に使うモデル。直近のやり取りを安いモデルで要約します。`--model` エイリアスでもフルの model id でも可。
- **`CODEX_BIN` / `CODEX_MODEL` / `CODEX_HOME`** — Codex 側の起動バイナリ・モデル・ホーム（既定 `~/.codex`）。
- **`MULMOTERMINAL_HOME`（既定 `~/.mulmoterminal`）** — 管理下の **git worktree** を置くルート。
- **`WAIT_REAP_GRACE_MS`（既定 `1800000` = 30分）** — **待機中**のバックグラウンドセッションを、この時間放置すると自動回収します。`0` 以下にすると回収しません。

（Docker サンドボックス系・アップデート通知の opt-out もありますが、詳細は README に譲ります。）

---

## 2. グローバル設定 — `~/.mulmoterminal/config.json`

ユーザーごとの UI 設定です。**設定モーダル（⚙）** で編集すると、このファイルへ保存されます（`GET`/`POST /api/config`）。JSON を直接編集してもかまいません。フィールドを1つずつ:

### `cwdPresets` — ランチャーのフォルダ候補

```jsonc
{
  "cwdPresets": [
    { "label": "payments", "path": "/Users/you/work/payments" },
    { "label": "blog",     "path": "/Users/you/src/blog" }
  ]
}
```

新しいターミナルを起動するときの **クイック選択候補**。ここに足したフォルダが、ランチャーフォームにチップで並びます（`init` が履歴から自動で埋めてくれるのもこれ）。チップを押すとそのパスが入力欄に入り、そこから resume するセッションを選べます。

### `soundFile` — 注意音（アプリ全体）

```jsonc
{ "soundFile": "/Users/you/sounds/ping.mp3" }
```

セッションが**要対応**になったときに鳴らす音を、自分の音声ファイルに差し替えます。**絶対パス**で指定。空/未設定なら、Web Audio API で合成した内蔵チャイムが鳴ります（**音声ファイルは同梱していない**ので npm パッケージは軽いまま）。ファイルは `GET /api/sound` で配信され、読めない/音声でないときはチャイムにフォールバックします。

### `prRepos` — 横断 PR/Issue ビューの対象

```jsonc
{ "prRepos": ["receptron/mulmoterminal", "you/your-app"] }
```

**PRs & Issues** ビューが、ここに挙げた `owner/repo` の open な PR・Issue を（あなたの `gh` ログインで）横断集約します。追った複数リポジトリの「レビュー待ち」を一望できます。

### `launchers` — Claude 以外の起動メニュー

```jsonc
{
  "launchers": [
    { "label": "codex", "command": "codex" },
    { "label": "shell", "command": "$SHELL" }
  ]
}
```

グリッドセルのランチャーに、Claude 以外の起動肢を足します。素のシェル、`codex`、任意の対話コマンド。`{ label, command }` の配列です。

### `userMcpServers` — MCP サーバの追加

```jsonc
{ "userMcpServers": [ { "id": "mytools", "url": "http://localhost:8931/mcp" } ] }
```

**単一ビュー**の Claude セッションの `--mcp-config` に、HTTP MCP サーバをマージします。`{ id, url }` の配列。次に開くセッションから有効になります（Docker サンドボックス内では `localhost` は `host.docker.internal` 経由で届きます）。

### `buttons` — ヘッダーのアクションボタン

各ターミナルのヘッダーに並ぶ**操作ボタン**を定義します。`buttons` を**省略**すると既定（📎 ファイルパス選択・📂 OS のファイルマネージャで開く）のまま。**設定すると既定を置き換える**ので、減らす・並べ替える・差し替えるが自由にできます。

1つのボタンは `id` / `label` と、`run` の種類で決まります:

- **`"run": "shell"`** — コマンドを実行。
- **`"run": "input"`** — エージェントにテキストを送る。
- **`"run": "open"`** — 何かを開く。`open` の対象は次のいずれか:
  - `url` … URL を開く
  - `reveal` … OS のファイルマネージャで表示
  - `files` … アプリ内ファイルエクスプローラ（例：`"open": { "files": "${dir}" }`）
  - `view` … 内蔵オーバーレイ
  - `terminal` … ディレクトリ → 隣に `$SHELL` の新セルを開く
  - `pr: true` … 現ブランチの PR を開く（open な PR が無いときはボタン自体が隠れる）
  - `pickFile: true` … OS のファイルダイアログ → 選んだパスを挿入

`${dir}` / `${branch}` / `${repo}` … はライブのコンテキストに置換され、`when`（例 `"isGitRepo"`）で表示条件を絞れます。手で書くのが面倒なら **`/mulmoterminal-config` スキル**が妥当な JSON を対話的に生成します。per-dir の `buttons` は、グローバルのものに **`id` 単位でマージ**されます。

### `chips` — ヘッダーの情報チップ

ヘッダーに出す**情報チップ**（ブランチ名などの表示）。省略すると既定セットのまま、`[]` にすると内蔵チップを**全部隠せます**。

### `pushEnabled` — 完了時のスマホ通知（Web Push）

```jsonc
{ "pushEnabled": true }
```

`true` にすると、**バックグラウンドのタスクが完了したとき**に、登録済みデバイスへ Web Push を送ります（タイトル＝プロジェクトのディレクトリ、本文＝直近のプロンプト）。既定は OFF。送信は別サービス `mulmoserver` の `sendPush` が担い、**RemoteHost チャンネルが接続中のときだけ**動きます（その Google サインインが通知認証を供給）。RemoteHost 未接続 or デバイス未登録なら no-op。

### `worklogEnabled` / `worklogIntervalHours` — 開発ワークログ

```jsonc
{ "worklogEnabled": true, "worklogIntervalHours": 6 }
```

`worklogEnabled: true`（＋**再起動**。スケジューラは起動時にタスクを読むため）で、内蔵の定期バッチを登録します。`worklogIntervalHours`（既定 `6`、`1`〜`168` にクランプ）ごとに Claude セッションが立ち上がり、**保存した全作業ディレクトリ（`cwdPresets`）で前回以降にやったこと**をレビューして、マネージャー視点の短いレポートを書きます。同じリポジトリの複数クローンは1セクションに統合。出力は wiki に ISO 週ごとの週次ページ（`data/wiki/pages/dev-log-YYYY-www.md`）として残り、`#worklog` タグから辿れます。**毎回 LLM を回すのでトークンを消費**します（既定 OFF）。1つの「ハブ」インスタンスだけで回すのが吉。

---

## 3. ディレクトリ設定 — `<project>/.mulmoterminal.json`

プロジェクトのフォルダに `.mulmoterminal.json` を置くと、**そのフォルダで開いたターミナル（＝グリッドのセル）だけ**、見た目と音を持てます。アプリ全体のテーマはそのまま、**そのセルだけ**ディレクトリのテーマが手動のテーマ選択を上書きします。全フィールド任意で、無い/壊れたファイルは無視されます。

```jsonc
{
  "name": "PROD · payments",            // このディレクトリのターミナルに出すバッジ
  "badgeColor": "#cf222e",              // バッジ色（#rrggbb）
  "headerColor": "#190a23",             // セルヘッダー背景
  "headerTextColor": "#ffffff",         // セルヘッダー文字色
  "cellColor": "#101014",               // セル本体の背景
  "cellBorderColor": "#2a2a4e",         // セルの枠線色
  "dotColor": "#00e676",                // 待機時のステータスドット
  "buttonColor": "#c7cdf0",             // ヘッダーのアイコンボタン色
  "theme": "nord",                      // xterm パレット: midnight | nord | daylight | solarized
  "colors": { "background": "#190a23", "cursor": "#ff2e63" }, // パレットの個別上書き
  "sound": "./.mulmoterminal/alert.mp3" // 注意音（このディレクトリからの相対パス）
}
```

1つずつ、足すとどうなるか:

- **`name`** — ターミナル/セルのヘッダーに出す**バッジ**のラベル。「PROD」「staging」など、環境の取り違えを目で防げます。
- **`badgeColor`** — バッジの背景色（`#rrggbb`）。文字色は自動でコントラストを取ります。
- **`headerColor`** — ヘッダーの**背景**色（グリッドのヘッダー行＋単一ビュー）。作業中/ブロック中はステータスの色が優先され、**待機時**にこの色が出ます。
- **`headerTextColor`** — ヘッダーの**文字**色（パス・タイトル・プロンプト）。
- **`cellColor`** — セル**本体の背景**色（ターミナルを囲む枠）。
- **`cellBorderColor`** — セルの**枠線**色。作業中/ブロック中のステータス枠はこの上に優先表示されます。
- **`dotColor`** — **待機時**のステータスドット色。作業中/要対応の色は変えないので、活動シグナルは保たれます。
- **`buttonColor`** — ヘッダーの**アイコンボタン**色（展開・閉じる・添付・フォルダなど）。
- **`theme`** — このディレクトリのターミナルの xterm パレット（`midnight` / `nord` / `daylight` / `solarized`）。
- **`colors`** — `theme`（未設定ならアプリのテーマ）の上に重ねる**個別上書き**。キーは xterm の `ITheme` 名（`background`・`foreground`・`cursor`・`selectionBackground`・16 の ANSI 色 …）、値は hex（`#rgb`/`#rrggbb`/`#rrggbbaa`）。未知のキー・不正な値は捨てられます。
- **`sound`** — このディレクトリのセッションの注意音。**ディレクトリからの相対パス**（`GET /api/dir-sound` で配信）。

> **セキュリティ**：`sound` は相対パス専用で、絶対パスや `../` でディレクトリを抜ける指定は拒否されます。パスは HTTP リクエストからは取らないので、開いたプロジェクトがプレイヤーに任意ファイルを差せません。変更は**次にターミナルを開いたとき**に反映されます（ライブ監視はしません）。

### 使いどころ

本番のリポジトリだけ `name: "PROD"` ＋ 赤い `headerColor` にしておく、といった**「どのセルがどの環境か」を一目で分かる**運用が定番です。色分けは事故防止に効きます。

---

## まとめ — どこをいじると何が変わるか

- **起動条件**（ポート・使う CLI・作業ディレクトリ）→ **環境変数 / `.env`**
- **アプリ全体の挙動**（プリセット・通知・ヘッダーボタン・ワークログ）→ **`~/.mulmoterminal/config.json`**（⚙ でも編集可）
- **セルごとの見た目と音**（バッジ・色・テーマ・音）→ **`<project>/.mulmoterminal.json`**

まず `npx mulmoterminal@latest init` で土台を作り、あとは使いながら上の3層に足していけば、自分の運用に馴染んでいきます。

- リポジトリ: https://github.com/receptron/mulmoterminal
- npm: https://www.npmjs.com/package/mulmoterminal
- あわせて読みたい: [日本語ガイド](https://zenn.dev/singularity/articles/mulmoterminal-guide-ja) / [グリッドビュー](https://zenn.dev/singularity/articles/mulmoterminal-grid-view)
