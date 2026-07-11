---
title: "MulmoTerminal 日本語ガイド — 並行AIエージェントのコックピット"
emoji: "🖥️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["MulmoTerminal", "ClaudeCode", "Codex", "AI", "個人開発"]
published: true
publication_name: "singularity"
---

[MulmoTerminal](https://github.com/receptron/mulmoterminal) は、[Claude Code](https://claude.com/claude-code) や Codex といった AI エージェントを、ブラウザ上で**並行して監督・可視化・拡張**するためのツールです。複数のエージェントを 1 画面のダッシュボードに並べ、「どれが動いていて・どれが手を止めて待っているか」を一目で把握しながら、**1 人で何体もまとめて回せます**。待ち時間や「今どこで何を頼んだんだっけ？」に振り回されず、あなたの注意を要所だけに集中できます。`npx mulmoterminal@latest` だけで、すぐに始められます。

この記事は、公式の [日本語ガイド](https://github.com/receptron/mulmoterminal/tree/main/docs/guide/ja) を1ページにまとめたものです。

## vibe coding、こんなこと、ありませんか？

AI エージェント（**Claude Code** / **Codex**）と一緒にコーディングしていると——

- 📊 どれが何をしているのか、**状態が分からなくなる**
- 📁 **どのディレクトリ**で動いているのか見失う
- 💭 dir は分かっても、**そもそも何を頼んだんだっけ？**（元の指示を忘れる）
- 🔔 エージェントが**終わったのに気づかず**、待たせる／待ちぼうける
- 💥 ターミナルを閉じた・落ちた瞬間に**セッションが消える**
- 🌿 git の状態を見たい・フォルダを開きたいのに、**いちいちコマンドを打つ**
- ⚡ 結局、**ターミナルを軸にサクサク作業したい**だけなのに——

AI エージェントは 1 タスクに数分かかります。1 体を見張ると手が空きっぱなし。数を増やすほど、上のような
「**追いきれない**」場面が増えていきます。ボトルネックは CPU でも端末でもなく、**あなたの注意**です。

## その一つひとつに、こたえます

| こんなとき | MulmoTerminal では |
|---|---|
| 複数ターミナルの**状態**が分からない | グリッドに並べ、**状態の色**（作業中＝青／要対応＝琥珀）＋通知音で一目（→ 「基本編」） |
| **どのディレクトリ**か分からない | 各セルに dir・**プロジェクト名バッジ・色**を表示。色分けで即区別（→ 「設定方法」） |
| **元の指示**を忘れる | セルヘッダーに**直近の指示／今やっていること**を常時表示、🕘 で**ツール履歴**（→ 「機能一覧」） |
| **完了**に気づきたい | 終わった／入力待ちは**琥珀色＋通知音**で「呼ばれた」と分かる |
| **セッションを継続**したい | **tmux 永続化**で、リロード・再接続・サーバ再起動を跨いで生き続ける |
| **git / dir をサッと**開きたい | git ステータスチップ、ワンクリックで **OSのファイルマネージャ(Finder/Explorer等) / アプリ内ファイル / PR** を開く |
| **ターミナル基軸**で効率化 | 上記すべてを端末の上に載せ、**DSL で自分のワークフローに拡張**（→ 「設定方法」） |

## そのための 4 本柱

1. **監督** — グリッドは**並行エージェントのコックピット**。status の色と通知で triage し、呼ばれたセルにだけ行く。
2. **可視化** — 各エージェントの**状態・モデル・コンテキスト量・git・ツール履歴・コスト**を一目で。「どこで何を」が常に見える。
3. **自動化 & エラー調査** — スクリプトをセル内で走らせ、失敗したら**大量のログを AI が短く診断**。
4. **拡張（DSL）** — ヘッダーのボタン／チップ、ランチャ、プロジェクト設定を**小さな DSL で拡張**。どんな開発者にも合う。

![並行するエージェント端末のボード](https://raw.githubusercontent.com/receptron/mulmoterminal/main/docs/guide/images/grid-2x2.png)

## 🚀 まずは起動

[`claude`](https://claude.com/claude-code)（Claude Code）が動く環境 + **Node ≥ 22.9** があれば、コマンド 1 つで始められます
（`tmux` があるとセッションが永続化されて理想的）。

```bash
npx mulmoterminal@latest    # → http://localhost:34567 が開く
```

## この記事の構成

この先は次の順で説明します。

1. **基本編** — グリッドで今できること
2. **応用編** — シナリオ別の使い方
3. **機能一覧**（4 本柱で整理）
4. **設定方法**（設定モーダル・`config.json`・`.mulmoterminal.json`・**DSL 拡張**）

---

## 基本編 — グリッドで今できること

### グリッドは「エージェントのボード」

グリッドビューは、**複数の AI エージェントを並行して監督する**ための画面です。各セルは独立した 1 体の
エージェント（または端末）。1 つが考えている間に別のセルを進め、**要対応（琥珀色）になったものだけ**拾う——
全部を見張らずに、1 人で多数を回すのが狙いです。

MulmoTerminal には 2 つの表示モードがあり、上部ツールバーの **チャット / グリッド** アイコンで切り替えます。

- **単一ビュー** … 1 体のエージェントに**集中**する画面（左にやり取り、右に GUI パネル＝図・フォーム・画像・書類）。
- **グリッドビュー** … 複数のエージェントを並べて**同時に監督**する画面。このガイドの主役です。

![単一ビュー — 1 体に集中](https://raw.githubusercontent.com/receptron/mulmoterminal/main/docs/guide/images/single-view.png)

### エージェントを起動する（ランチャフォーム）

グリッドの空きセルには **ランチャフォーム**が出ます。ここで「何を」「どこで」動かすか選びます。

![空きセルのランチャフォーム](https://raw.githubusercontent.com/receptron/mulmoterminal/main/docs/guide/images/grid-launch-form.png)

| 部分 | 役割 |
|---|---|
| **Claude / Codex** トグル | このセルで動かす**エージェント**を選ぶ |
| **WORKING DIRECTORY** | 作業ディレクトリを入力（`▶` で起動）。よく使うディレクトリは *cwd presets* から補完 |
| **OR ISOLATE IN A WORKTREE** | git リポなら、タスク名を入れて **＋ New worktree**。作業を隔離した worktree を作って起動 |
| **OR LAUNCH** | エージェント以外の**起動コマンド**（`Shell` / `codex` / 任意）を永続端末として起動 |

### セルの読み方 — 「どこで何をしているか」

起動したセルのヘッダーは 2 段。ここに、そのエージェントの**状態・位置・作業内容**が集約されます。

![起動したセル（2段ヘッダー）](https://raw.githubusercontent.com/receptron/mulmoterminal/main/docs/guide/images/grid-one-cell.png)

*図はレイアウトを示す例です。各セルには通常 **Claude / Codex のセッション**が入り、ヘッダーにそのエージェントの状態・モデル・作業内容が出ます（図の中身は端末の例）。*

- **1 段目（情報）**：状態ドット・ディレクトリ・git チップ（`⎇ ブランチ ●変更数`）・**モデル/コンテキスト量**・
  そのエージェントが**今やっていること**・拡大/閉じる。
- **2 段目（操作）**：接続状態・添付・ファイルブラウザ・GitHub・**タイムライン 🕘**（ツール呼び出し履歴）。

> **状態は色で分かります。** 青みがかった枠＝**作業中**（考え中）、琥珀色＝**要対応**（入力待ち／未読の出力あり）、
> 通常＝アイドル。通知音も鳴るので、**画面を見張らなくても**「呼ばれた」と分かります。これがグリッドの肝です。

### たくさん並べる・ページ・並べ替え

- ツールバーの **New terminal（＋）** でセルを追加。1 ページ **9 セル**まで、あふれると次のページ（タブ）へ。
- **Toggle grid cell ordering** で並べ替えモードに入り、各セルの `◀ ▶` で位置を入れ替え。

![並行するエージェント](https://raw.githubusercontent.com/receptron/mulmoterminal/main/docs/guide/images/grid-2x2.png)

### 1 体に寄る（フィルムストリップ）

セルの **⤢**（拡大）で、そのエージェントを大きく表示し、他は下部の**フィルムストリップ**にサムネイル化。
サムネイルのヘッダー余白クリックで**切り替え**、**⤡** でグリッドへ戻ります。「全体を見渡す ↔ 1 体に寄る」を素早く行き来できます。

> **軽い強調**：いまフォーカスが当たっているセルは、**その場でわずかに持ち上がって拡大**し、どれがアクティブか一目で分かります（グリッドの配置は崩れません）。しっかり 1 体に寄りたいときだけ ⤢ を使う、と自然に使い分けられます。

![拡大（フィルムストリップ）](https://raw.githubusercontent.com/receptron/mulmoterminal/main/docs/guide/images/grid-zoom.png)

### Claude と Codex を混在

同じグリッドで、セルごとに **Claude** でも **Codex** でも起動できます。どちらも同じ端末体験・永続化・GUI パネル・
可視化の仕組みを共有。得意分野で使い分けたり、同じタスクを両者に投げて見比べたりできます。

---

## 応用編 — ユーザーシナリオ別の使い方

MulmoTerminal の 4 本柱——**監督 / 可視化 / 自動化・調査 / 拡張**——を、実際の開発の流れに落として紹介します。

### 1. 複数タスクを並行して回す（監督）

グリッドの基本にして本命の使い方です。ここが**司令塔**の中心。

1. `New terminal ＋` でセルを増やし、それぞれで別タスクの Claude / Codex を起動。
2. 1 つが考えている間に、別のセルのレビューや修正を進める。
3. **要対応（琥珀色）**になったセルだけ拾って対応する——全部を見張らなくてよい。

![並行するターミナル](https://raw.githubusercontent.com/receptron/mulmoterminal/main/docs/guide/images/grid-2x2.png)

> 各セルのヘッダーには「そのセッションが今やっていること」が出るので、どのセルが何の作業かひと目で分かります。

> 🔔 **完了は「音」でも教えてくれます。** セッションが要対応になると通知音が鳴るので、**画面を見ていなくても**気づけます。
> 音は「設定方法」で**好きな音声ファイル**に変えられます——ある開発者は、完了のたびに**銅鑼（ドラ）を「ゴーーン」**と鳴らしているとか。

### 2. worktree で隔離して安全に試す

「main を汚さずに試したい」ときは、**git worktree** で作業を隔離します。

1. git リポを作業ディレクトリにしたセルで、**OR ISOLATE IN A WORKTREE** にタスク名（例：`fix-login`）を入力。
2. **＋ New worktree** で、そのタスク専用の worktree を作ってセッションを起動。
3. 変更が溜まると、セルヘッダーに差分バッジが出ます。ワンクリックで **push / PR 作成**まで。

![worktree はランチャフォームから](https://raw.githubusercontent.com/receptron/mulmoterminal/main/docs/guide/images/grid-launch-form.png)

### 3. 複数リポジトリを横断して作業する

セルごとに**別々の作業ディレクトリ**を指定できます。フロントエンドのリポと API のリポを隣同士のセルで開き、
片方の変更に合わせてもう片方を直す、といった横断作業が 1 画面で完結します。よく使うディレクトリは
「設定方法」の *cwd presets* に登録しておくと補完が効きます。

### 4. スクリプトを走らせて、失敗を AI に要約させる

グリッドのセルは Claude だけでなく、プロジェクトの**スクリプト**——例えば `yarn dev`（開発サーバ）・`yarn test`・
`yarn build`・`yarn lint`——も動かせます。

1. そのディレクトリの `script.json` にスクリプトを定義しておき、空きセルのランチャ（または **▶ Run** メニュー）から選ぶ。

   ```json
   { "scripts": [
     { "label": "dev",  "command": "yarn dev" },
     { "label": "test", "command": "yarn test" },
     { "label": "lint", "command": "yarn lint" }
   ] }
   ```

   > **0.8.0 からはもう一つの方法**：`.mulmoterminal.json` の **`run:"shell"` ボタン**（例 `{ "run": "shell", "cmd": "yarn build" }`）でも、
   > ヘッダーのワンクリックで同じコマンドをコマンドセルとして実行できます。→ 「設定方法 › ヘッダーのカスタマイズ」
2. スクリプトは**そのセルの中で**走るので、Claude セッションの隣で結果を見られる。
3. ビルドやテストが失敗して大量のログに埋もれたら、コマンドセルの **✦ Summarize** を押す。
   出力を `claude -p` に渡し、**エラー / 警告 / 原因 / 直し方**を短くまとめて返します。
   **⧉ Copy as prompt** で「コマンド + ディレクトリ + 要約 + フォローアップ」をコピーし、任意のセッションに貼って続行できます。

### 5. Claude と Codex を併用する

同じタスクを Claude と Codex に投げて見比べる、得意分野で使い分ける、といったことがセル単位でできます。
起動時のトグルで選ぶだけで、コレクションのアクションや mulmoclaude スキルも両方で使えます。

### 6. プロジェクトを色で見分ける

セルが増えるとどれがどのプロジェクトか分かりにくくなります。各リポの `.mulmoterminal.json` に
**名前バッジ**とヘッダー・本体・枠・ドット・ボタンの**色**を設定しておくと、グリッド上で一目で区別できます（「設定方法」参照）。

![4 プロジェクトをド派手に色分け（モンドリアン／ゴッホ／ピカソ／マティス風）](https://raw.githubusercontent.com/receptron/mulmoterminal/main/docs/guide/images/grid-colors.png)

*例：モンドリアン風（黄＋赤）、ゴッホ風（夜の青＋黄）、ピカソ風（青＋赤＋黄）、マティス風（緑＋ピンク）。ヘッダー・バッジだけでなく、`colors` で**端末の中身（背景・文字）**まで染まるので、間違えようがありません。*

```json
{
  "name": "🌌 van-gogh",
  "badgeColor": "#f5b301",
  "headerColor": "#0b1a4a",
  "headerTextColor": "#f2e29b",
  "colors": { "background": "#0a1330", "foreground": "#f2e29b", "cursor": "#f5b301" }
}
```

### 7. ヘッダーによく使う操作を足す

`.mulmoterminal.json` の `buttons` / `chips` で、稼働中ターミナルのヘッダーに**自分専用のボタン**（例：`/compact` 送信、
GitHub を開く、ビルドを走らせる）や**表示チップ**を足せます。詳しくは「設定方法 › ヘッダーのカスタマイズ」へ。

---

## 機能一覧

MulmoTerminal の機能を **4 本柱**（監督 / 可視化 / 自動化・調査 / 拡張）で整理します。使い方は「基本編」・「応用編」へ。

### 1. 監督 — 並行エージェントのコックピット

| 機能 | 説明 |
|---|---|
| 並行ターミナル | 1 ページ最大 **9 セル**、あふれると**ページ（タブ）**が増える |
| 状態の色 + 通知音 | 作業中（青）/ 要対応（琥珀）/ アイドル。見張らずに「呼ばれた」と分かる |
| セル追加 / 閉じる / 並べ替え | `New terminal ＋`、各セルの `✕`、並べ替えモードの `◀ ▶` |
| 拡大 / フィルムストリップ | `⤢` で 1 体に寄り、他はサムネイル。全体 ↔ 個を素早く往復 |
| worktree 隔離 | 同じリポに複数エージェントを走らせても衝突しない git worktree |
| セッション永続化（tmux） | tmux があれば各セッションは tmux 内で走り、リロード／サーバ再起動を跨いで**再接続** |

### 2. 可視化 — どこで何をしているか

| 機能 | 説明 |
|---|---|
| そのエージェントの作業内容 | セルヘッダーに「今やっていること」を表示 |
| git ステータスチップ | `⎇ ブランチ ●変更数 ↑ahead ↓behind` を常時表示 |
| モデル / コンテキスト量 | 例 `Opus · ctx 35%` — 動作モデルと文脈の埋まり具合 |
| アクティビティタイムライン 🕘 | ツール呼び出し履歴（Bash / Read / Edit …）を新しい順にモーダル表示 |
| コスト（推定） | 設定に **セッション / 今日 / 今月** の概算コスト |
| worktree 差分バッジ | worktree セルで変更量を表示、クリックで差分パネル |
| GUI パネル | エージェントのツール呼び出しで図・フォーム・画像・書類を描画（Claude / Codex 両対応） |

### 3. 自動化 & エラー調査

| 機能 | 説明 |
|---|---|
| スクリプト実行 | そのディレクトリの `script.json` のコマンドを**セル内で**実行 |
| ✦ Summarize / Explain | 端末出力を `claude -p` に渡し、**エラー / 警告 / 原因 / 直し方**を短く要約 |
| ⧉ Copy as prompt | コマンド + ディレクトリ + 要約 + フォローアップをコピーして任意セッションに貼付 |
| 起動コマンド | Claude 以外（`Shell` / `codex` / 任意）を**永続端末**として起動 |

### 4. 拡張 — DSL で自分に合わせる

| 機能 | 説明 |
|---|---|
| ヘッダーの操作ボタン | `buttons` で `input`（テキスト送信）/ `open`（URL・ファイルマネージャ・アプリ内ビュー）/ `shell`（コマンド実行）を追加。`${変数}` と `when` 条件付き |
| ヘッダーの表示チップ | `chips` で組み込みチップの並べ替え・非表示 + カスタムチップ |
| 名前バッジ / 色 | `.mulmoterminal.json` でディレクトリごとに名前・各所の色 |
| ランチャ / cwd presets / PR repos | 設定で起動コマンド・作業ディレクトリ候補・横断 PR 対象を拡張 |
| テーマ | Midnight / Nord / Daylight / Solarized Light |

> **設定なしなら従来どおり**——ボタン/チップ/色は足したぶんだけ効き、既定の見た目は変わりません。
> 詳細は「設定方法」へ。

---

## 設定方法

設定は 3 か所にあります。**設定モーダル（⚙）**・**グローバル設定 `~/.mulmoterminal/config.json`**・
**プロジェクトごとの `<project>/.mulmoterminal.json`**。ボタン/チップは両ファイルがマージされます。

### 設定モーダル（⚙）

ツールバーの ⚙ から開きます。

![設定モーダル](https://raw.githubusercontent.com/receptron/mulmoterminal/main/docs/guide/images/settings.png)

| 項目 | 内容 |
|---|---|
| **THEME** | Midnight / Nord / Daylight / Solarized Light |
| **NOTIFICATION SOUND** | 要対応時に鳴らす音（空なら内蔵チャイム、または任意の音声ファイル） |
| **PULL REQUEST REPOS** | 横断 PR/Issue ビューが集約するリポ（`owner/repo`） |
| **LAUNCH COMMANDS** | グリッドセルで Claude 以外に起動できるコマンド（`{ label, command }`） |
| **MCP SERVERS** | 単一ビューのセッションに追加する自分の MCP サーバ |

### グローバル設定 `~/.mulmoterminal/config.json`

```json
{
  "cwdPresets": ["/Users/you/projects/acme-web", "/Users/you/projects/acme-api"],
  "launchers": [
    { "label": "Shell", "command": "$SHELL" },
    { "label": "Node REPL", "command": "node" }
  ],
  "prRepos": ["acme/web", "acme/api"],
  "userMcpServers": [],
  "buttons": [],
  "chips": null
}
```

| キー | 役割 |
|---|---|
| `cwdPresets` | ランチャの作業ディレクトリ補完 |
| `launchers` | グリッドセルの「OR LAUNCH」に並ぶ起動コマンド |
| `prRepos` | 横断 PR/Issue ビューの対象リポ |
| `buttons` / `chips` | ヘッダーのボタン/チップ（プロジェクト設定とマージ。→ 「ヘッダーのカスタマイズ（ボタン / チップ）」） |

### プロジェクトごとの `.mulmoterminal.json`

プロジェクト直下に置くと、**そのディレクトリで開いた端末（グリッドセル）**の見た目・音・ヘッダーを変えられます。

> **手で書かなくても OK**：ツールバーの設定ボタン、または任意の端末で `/mulmoterminal-config` を実行すると、**会話形式で `.mulmoterminal.json` を作れます**。色や DSL の書式を覚えていなくても、いまのディレクトリ（や最近使ったディレクトリをまとめて）設定できます。保存した変更は**再起動なしでその場に反映**されます。

#### 名前バッジと色

```json
{
  "name": "acme-web",
  "badgeColor": "#2563eb",
  "headerColor": "#0b2545",
  "headerTextColor": "#e6f0ff",
  "cellColor": "#0e1117",
  "cellBorderColor": "#1f6f4f",
  "dotColor": "#22c55e",
  "buttonColor": "#a7f3d0"
}
```

すべて `#rrggbb`。作業中/要対応の状態色は、これらの背景色より優先されます（アイドル時に反映）。

#### ターミナル自体の色（xterm パレット）

`headerColor` などが「**枠**（ヘッダー・セル）」の色なのに対し、**`colors`（と `theme`）は端末の中身（xterm）**を染めます。
`colors` は xterm の ITheme——`background` / `foreground` / `cursor` や `red` `green` … の ANSI 16 色——を上書きできます。

```json
{
  "name": "🌌 van-gogh",
  "headerColor": "#0b1a4a",
  "headerTextColor": "#f2e29b",
  "colors": { "background": "#0a1330", "foreground": "#f2e29b", "cursor": "#f5b301" }
}
```

`theme` に `midnight` / `nord` / `daylight` / `solarized` を指定するとプリセットのパレットになり、`colors` はその上へ部分上書き。
「応用編」の色分けスクショは、ヘッダー色と `colors` を組み合わせて**ヘッダーから端末の中身まで**プロジェクトごとに染めた例です。

#### ヘッダーのカスタマイズ（ボタン / チップ）

MulmoTerminal の「**拡張**」の柱がここ。稼働中ターミナルのヘッダーを、**小さな DSL** で自分のワークフローに合わせて成形できます。
どんな開発者でも、よく使う操作をワンクリックにし、見たい情報だけを出せる——それがこの仕組みの狙いです。

**ボタン**（`buttons`）— 稼働中セッションに効く、絵文字/ラベル付きの操作ボタン。設定なしなら従来どおり何も増えません。

```json
{
  "buttons": [
    { "id": "compact", "emoji": "🗜️", "label": "Compact", "run": "input", "text": "/compact", "when": "agent == claude" },
    { "id": "gh",      "emoji": "🌐", "label": "Open on GitHub", "run": "open", "open": { "url": "https://github.com/${repo}" }, "when": "isGitRepo" },
    { "id": "reveal",  "emoji": "📁", "label": "Reveal folder", "run": "open", "open": { "reveal": "${dir}" } },
    { "id": "build",   "emoji": "🔨", "label": "Build", "run": "shell", "cmd": "yarn build" }
  ]
}
```

- `run: "input"` … 稼働中の Claude/Codex に `text` を送信（例 `/compact`）。
- `run: "open"` … `url`（ブラウザ, http/https のみ）/ `reveal`（OSのファイルマネージャ: Finder/Explorer/xdg-open）/ `files`（アプリ内エクスプローラ）/ `view`（`prs`/`wiki`/`collections`/`accounting`）。
- `run: "shell"` … `cmd` をコマンドセルで実行（サーバ側で id 解決 + `${変数}` はシェルエスケープ、コマンドはブラウザに渡らない）。
- `${変数}` … `dir` `branch` `repo` `ahead` `behind` `dirty` `agent` `model` `task`。
- `when` … `isGitRepo` / `agent == …` / `repo == …`（`&&` / `||`、`&&` が優先）。

**チップ**（`chips`）— グリッドセルヘッダーの情報チップを並べ替え/非表示 + カスタム。`null`（既定）は従来どおり。

```json
{ "chips": ["ctx", "git", { "label": "env", "text": "⎇ ${branch}", "when": "isGitRepo" }] }
```

- 組み込み `git` / `diff` / `ctx` / `usage` … 並べた順に表示、書かなければ非表示。
- カスタム `{ label, text, when }` … 読み取り専用テキスト（`text` は `${変数}` 展開）。

### スクリプト `<project>/script.json`

グリッドセルで実行できるプロジェクトのスクリプト（dev サーバ・テスト・ビルドなど）。

```json
{ "scripts": [ { "label": "dev", "command": "yarn dev" }, { "label": "test", "command": "yarn test", "cwd": "." } ] }
```

### 環境変数

| 変数 | 既定 | 役割 |
|---|---|---|
| `CLAUDE_CWD` / `--cwd` | 実行したディレクトリ（`npx mulmoterminal`。サーバを直接起動した場合のみ `~/mulmoclaude`） | 既定の作業ディレクトリ（PTY の cwd）。`--cwd` でも指定可 |
| `PORT` | `34567` | サーバのポート |
| `MULMOTERMINAL_HOME` | `~/.mulmoterminal` | 管理下 git worktree のルート |
