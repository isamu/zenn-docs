---
title: "Strix × Ollama で自分のサイトの穴を AI に探させる"
emoji: "🦉"
type: "tech"
topics: ["security", "pentest", "ollama", "llm", "strix"]
published: false
---

## この記事でやること

[Strix](https://github.com/usestrix/strix) は Apache 2.0 で公開されている AI ペネトレーションテストツールです。「自律型の AI ハッカー」が実際にコードを動かし、脆弱性を見つけ、動く実証コード（PoC）まで作って検証してくれます。静的解析ツールのように「怪しい行」を並べるだけで終わらず、本当に悪用できるかまで踏み込むのが特徴です。

通常はクラウドの大規模モデルに API キーをつないで使いますが、この記事では **Ollama + ローカルモデル** で動かす構成をまとめます。コードと LLM 推論を外部クラウドに送らずに検査できます（Web 検索などのツールが動くときは、その通信だけは別途発生します）。

そのうえで、

1. Strix で **自分のサイトのどんな穴を探せるのか**
2. 検査を回すと **どんな結果が出てくるのか**
3. その結果から **どう対策するのか**

までを一通り説明します。

:::message alert
Strix は実際に攻撃コードを実行します。**自分が所有・管理しているサイトやコード、または明示的に許可を得た対象にだけ**使ってください。他人のサイトへの無断スキャンは違法になり得ます。
:::

---

## 1. 必要なもの

- macOS / Linux / Windows
- Docker（Docker Desktop 推奨。Strix は検査対象をコンテナ内で動かします）
- Ollama（ローカル LLM ランタイム）
- `uv`（Strix の実行に使用）

---

## 2. Ollama のインストールとモデル取得

### macOS

```bash
brew install ollama
ollama --version
ollama pull qwen3-coder-next:latest
```

### Linux

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull qwen3-coder-next:latest
```

### Windows

[Ollama for Windows](https://ollama.com/download/OllamaSetup.exe) をインストールし、PowerShell で:

```powershell
ollama pull qwen3-coder-next:latest
```

ダウンロード後、サーバーが動いているか確認します。

```bash
ollama list
curl http://localhost:11434/api/tags
```

### Strix 向けの Ollama 設定（ここを外すと無言で失敗する）

Strix は巨大なシステムプロンプトと多数のツール定義（tool schema）をモデルに渡し、モデルが返す **ツール呼び出し（tool call）** を使って検査を進めます。ここが噛み合わないと、スキャンは「エラーも出さずに何も見つけない」で終わります。実行前に次を必ず設定してください。

- **ツール呼び出しに対応したモデル／テンプレートを使う。** 新しめの Ollama は、テンプレートがツール非対応だと `tools param requires --jinja flag` で弾きます。
- **推論系モデル（qwen3 系など）は thinking を切る。** thinking を残すと、ツール呼び出しが構造化された `tool_calls` ではなく本文（content）側に漏れ、Strix が認識できません。Modelfile やパラメータで `think: false` にするか、非 thinking の派生モデルを使います。
- **`num_ctx` を 16k〜32k 以上に上げる。** Ollama の既定コンテキストは小さく、ツール定義がプロンプトから溢れて切り捨てられると、モデルは正しい呼び出しを出さなくなります。

:::message alert
怖いのは、**短いテスト用プロンプトだと正常に見えるのに、本番のスキャンだけ失敗する**ことです（`num_ctx` 不足やツール定義の切り捨てが原因）。「動作確認は通ったのに検出結果が 0 件」という場合、まずこの設定を疑ってください。
:::

例（Modelfile でコンテキストと thinking を固定する）:

```bash
cat > Modelfile <<'EOF'
FROM qwen3-coder-next:latest
PARAMETER num_ctx 32768
EOF
ollama create qwen3-coder-next-strix -f Modelfile
# 以降、STRIX_LLM を ollama/qwen3-coder-next-strix:latest に向ける
```

---

## 3. Docker の確認

Strix 本体は Ollama のモデルを使いますが、検査対象のアプリやツールは Docker コンテナで動かします。

```bash
docker info
# エラーが出る場合は Docker Desktop を起動
open -a Docker   # macOS
```

---

## 4. 環境変数で Ollama を指定

`~/.zshrc` または `~/.bashrc` に追記します。

```bash
# Strix に Ollama のローカルモデルを使わせる
export STRIX_LLM="ollama/qwen3-coder-next:latest"
export LLM_API_BASE="http://localhost:11434"
```

反映と確認:

```bash
source ~/.zshrc   # または source ~/.bashrc
echo $STRIX_LLM
echo $LLM_API_BASE
```

---

## 5. Strix のインストール

```bash
cd ~/ss
git clone https://github.com/usestrix/strix.git
cd strix

# 依存関係
uv sync --no-dev

# 結果ビュアー（TUI）のビルド
make tui-build

# 動作確認
uv run strix --help
```

---

## 6. Strix は自分のサイトの「どんな穴」を探せるのか

Strix は、人間のペンテスターと同じ流れ（偵察 → 攻撃 → 検証）を自律的に回します。手元の構成で使える主な能力は次のとおりです。

### 検査できる対象（target）

`--target` には複数の形を渡せます。自分のサイトを色々な角度から当てられます。

| 対象の種類 | 指定例 |
|------|------|
| ローカルのソースコード | `--target ./ownplate` |
| 動いている Web アプリ | `--target https://staging.example.com` |
| GitHub リポジトリ | `--target https://github.com/you/repo` |
| API 仕様 | `--target ./openapi.yaml` / `--target ./collection.postman_collection.json` |
| ドメイン / IP | `--target example.com` / `--target 192.168.1.42` |

コードと動作中アプリを **同時に** 渡すこともできます（ソースを読んで当たりをつけ、実アプリで悪用を試す、という人間に近い動き）。

```bash
uv run strix \
  --target ./ownplate \
  --target https://staging.ownplate.example \
  --instruction "Firebase を使った注文サイト。認証と注文データのアクセス制御を重点的に"
```

### 探せる脆弱性の種類

Strix は代表的な Web 脆弱性を一通りカバーします（PoC で悪用を実証するものを含む）。

- **アクセス制御の不備 / IDOR** — 他人の注文・ユーザー情報を ID 書き換えで覗けるか
- **認証・認可の欠陥** — ログインを飛ばせるか、権限昇格できるか
- **XSS**（格納型・反射型）
- **SQL インジェクション / NoSQL インジェクション**
- **SSRF**（サーバーを踏み台に内部へアクセス）
- **CSRF**
- **XXE**
- **RCE**（リモートコード実行）
- **SSTI**（テンプレートインジェクション）
- **パストラバーサル**
- 設定・シークレットの漏れ（Firebase ルールの緩さ、公開された鍵など）

### 使い方のバリエーション

```bash
# 偵察重視の軽いスキャン（CI 向け）
uv run strix --target ./ownplate --scan-mode quick -n

# じっくり深く（デフォルト）
uv run strix --target ./ownplate --scan-mode deep

# 変更差分だけを検査（PR レビュー用）
uv run strix --target ./ownplate --scope-mode diff --diff-base origin/main

# 予算と手数の上限を切る
uv run strix --target ./ownplate --max-budget-usd 5 --max-turns 60
```

`-n`（`--non-interactive`）を付けると TUI を出さずに実行し、完了で終了します。CI やバッチ実行向けです。

---

## 7. 実行してみる

```bash
cd ~/ss/strix
uv run strix \
  -t ~/ss/ownplate \
  -n \
  --instruction "Firebaseプロジェクトのセキュリティチェックを実施してください"
```

実行が終わると、結果は `./strix_runs/<run 名>/` に保存されます（例: `strix_runs/ownplate_3b40/`）。

---

## 8. どんな結果が出てくるか

1 回の run で、次のファイルが `strix_runs/<run 名>/` に生成されます。

| ファイル | 中身 |
|------|------|
| `penetration_test_report.md` | 人が読むレポート（要約・手法・分析・推奨） |
| `findings.sarif` | 機械可読の検出結果（GitHub Code Scanning に取り込める SARIF 形式） |
| `vulnerabilities.json` | 検出した脆弱性の一覧（ビューアーが読む。検出結果があるときに生成） |
| `coverage.json` | 何を検査し、どこに穴（gap）が残ったかの記録 |
| `run.json` | run の全メタデータ |
| `strix.log` | 実行ログ |

### 1 件 1 件の検出結果（finding）の形

脆弱性が見つかると、Strix は単なる警告ではなく、次を揃えた 1 件のレポートとして記録します。

- **severity**（深刻度）: CVSS v3 の内訳から算出され、`critical / high / medium / low / info` に分類
- **confidence**（確信度）: `high / medium / low`
- **PoC**: 動的に見つかった脆弱性では、実際に悪用する `poc_script_code`（スクリプト／ペイロード）と証拠（evidence）が必須。これらが無いと登録されない（依存ライブラリの CVE など別種の検出結果は別経路で扱われます）
- **fix_effort**（修正コスト）: `trivial / low / medium / high`
- 深刻度を上下させうる条件や、確信度の根拠

この「動的な脆弱性は PoC が無ければ登録しない」という設計のおかげで、静的解析にありがちな大量の誤検知に埋もれずに済みます。

### 「何も出なかった」ときの読み方

実際に ownplate の **コードだけ** を検査した実行例（軽量なローカルモデル `llama3.1:8b` を使用）では、レポートは「重大な問題なし」で終わり、`findings.sarif` の `results` は空、`coverage.json` は次のようになっていました。

```json
"summary": { "surfaces_reviewed": 0, "findings_filed": 0, "gaps": 1 },
"gaps": [
  { "kind": "agent_recorded_no_coverage",
    "detail": "Root Agent ran ... without recording any coverage." }
]
```

ここで大事なのは、**「検出結果 0 件」＝「安全」ではない**ということです。`surfaces_reviewed: 0` と `gaps: 1` は「検査面をほとんど踏めていない（穴が残っている）」というサインです。ローカルモデルで手数や探索が足りない、対象の起動まで届いていない、あるいは **ツール呼び出しが成立していない（§2 の `num_ctx`・thinking 設定）** といった場合にこうなります。

結果を受け取ったら、

- `coverage.json` の `gaps` と `surfaces_reviewed` を必ず確認する
- 0 件なら `--scan-mode deep` に上げ、`--instruction` で検査してほしい箇所を具体的に指示し、動作中アプリの URL も `--target` に足して **再実行** する

という読み方・回し方をします。

---

## 9. 結果からどう対策するか

### 検出結果が出た場合

深刻度と修正コストで優先順位を付け、種類ごとに対処します。

| 見つかった穴 | 典型的な対策 |
|------|------|
| IDOR / アクセス制御不備 | サーバー側で所有者チェック。Firebase なら Security Rules で `request.auth.uid` と所有者を照合 |
| 認証・認可の欠陥 | 重要操作はサーバー側で再検証。クライアント判定に依存しない |
| XSS | 出力エスケープ、`v-html` を避ける、CSP の導入 |
| SQLi / NoSQLi | パラメータ化クエリ、入力バリデーション |
| SSRF | 外向きリクエストの宛先を許可リスト化、内部アドレスを拒否 |
| CSRF | トークン検証、`SameSite` クッキー |
| シークレット漏れ | 鍵をコードから除去しローテーション、環境変数へ |

Strix は各 finding に修正の方向性（remediation）と PoC を付けてくれるので、**まず PoC を自分で再現して本物だと確認 → 修正 → 同じ PoC が通らなくなるか再検証**、という流れが取れます。

### 修正したら差分だけ再検査する

修正後は全体を回し直さなくても、変更部分だけを当てられます。

```bash
uv run strix --target ./ownplate --scope-mode diff --diff-base origin/main
```

### CI に組み込んで「入れない」ようにする

`--fail-on` を付けると、指定した深刻度以上の検出結果が出たとき Strix が終了コード 2 を返します。CI はこれを失敗として扱うので、PR ごとに差分を検査し、危険なコードが本番に入る前に止められます。

```bash
uv run strix \
  --target ./ownplate \
  --scope-mode diff --diff-base origin/main \
  --scan-mode quick \
  -n \
  --fail-on high
```

`findings.sarif` を GitHub Code Scanning にアップロードすれば、検出結果を PR 上の注釈として表示できます。

---

## 10. 結果を共有する

### ビュアーで確認

```bash
uv run strix view            # 最新の run を開く
uv run strix view ownplate_3b40
```

### PDF レポートを生成

```bash
cd ~/ss/llm/strix
uv run scripts/generate_pdf_report.py \
  ~/ss/llm/strix/strix_runs/ownplate_3b40 \
  ~/ss/llm/strix/strix_runs/ownplate_3b40/report.pdf
```

---

## 11. トラブルシューティング

### "Ollama connection refused"

```bash
curl http://localhost:11434/api/tags   # 応答がなければ未起動
ollama serve
```

### "Docker not available"

```bash
open -a Docker   # macOS
docker info
```

### "Model not found"

```bash
ollama list
ollama pull qwen3-coder-next:latest
```

### 検出結果が 0 件で `gaps` だけ出る

- `coverage.json` の `surfaces_reviewed` が 0 なら検査が届いていない
- **まず §2 の Ollama 設定（`num_ctx` 16k〜32k、thinking オフ、ツール対応テンプレート）を確認する** — ツール呼び出しが成立しないと、この症状が最も起きやすい
- `--scan-mode deep` に上げる、`--max-turns` を増やす、`--instruction` を具体化する
- 動作中アプリの URL を `--target` に追加する
- ローカルモデルで探索が浅い場合は、より大きなモデルや（許可があれば）クラウドの API を併用する

---

## 推奨設定まとめ

| 項目 | 設定 |
|------|------|
| モデル | `ollama/qwen3-coder-next:latest` |
| API Base | `http://localhost:11434` |
| スキャンモード | 日常は `standard`、本格検査は `deep`、CI は `quick` |
| スコープ | PR レビューは `--scope-mode diff --diff-base origin/main` |
| CI ゲート | `--fail-on high`（必要に応じて `critical`） |

---

## 関連リンク

- [Strix GitHub](https://github.com/usestrix/strix)
- [Strix Docs](https://docs.strix.ai)
- [Ollama Docs](https://ollama.com/docs)
