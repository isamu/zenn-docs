---
title: "Claude Code Skillの配布方法 - Plugin・Marketplace・Skills CLI"
emoji: "🔌"
type: "tech"
topics: ["claudecode", "plugin", "ai", "typescript"]
published: true
publication_name: "singularity"
---

## はじめに

Claude Codeでは、繰り返し使う作業手順を**Skill**として定義し、`/skill-name`で呼び出せます。自分用に作ったSkillを他のユーザーにも共有したくなったとき、いくつかの配布方法があります。

この記事では、Skillの配布方法と、配布時にハマりやすいポイントを解説します。

## 配布方法の全体像

| 方法 | 仕組み | 対象エージェント |
|------|--------|-----------------|
| Plugin Marketplace（自前） | GitHubリポジトリ | Claude Code |
| Plugin Marketplace（公式） | Anthropic公式ストア | Claude Code |
| Skills CLI（vercel-labs/skills） | npm + SKILL.md | Claude Code, Cursor, Copilot等 40+ |

## 1. Plugin Marketplaceで配布（自前ストア）

最も一般的な方法です。GitHubにマーケットプレイス用のリポジトリを作成し、そこからPluginとして配布します。

### リポジトリ構成

マーケットプレイスには2つのリポジトリが関係します。

**マーケットプレイスリポジトリ**（カタログ）：

```
my-marketplace/
  .claude-plugin/
    marketplace.json    ← プラグイン一覧を定義
```

**プラグインリポジトリ**（Skill本体）：

```
my-plugin/
  .claude-plugin/
    plugin.json         ← プラグインのメタデータ
  skills/
    my-skill/
      SKILL.md          ← Skillの定義
```

### marketplace.json の例

```json
{
  "name": "mulmocast-plugins",
  "owner": { "name": "receptron" },
  "plugins": [
    {
      "name": "mulmocast",
      "source": {
        "source": "github",
        "repo": "receptron/mulmocast-claude-plugin"
      },
      "description": "Create video presentations from URLs and topics",
      "version": "1.0.0",
      "category": "productivity"
    }
  ]
}
```

### plugin.json の例

```json
{
  "name": "mulmocast",
  "description": "MulmoCast skills for Claude Code",
  "version": "1.0.0",
  "author": { "name": "receptron" }
}
```

### 名前の関係を整理する

Plugin Marketplaceでは、3つの名前が登場します。それぞれの関係を整理します。

```
receptron/mulmocast-claude-plugin   ← GitHubリポジトリ名
         ↓
  marketplace.json の "name": "mulmocast-plugins"   ← マーケットプレイス名
  marketplace.json の plugins[0].name: "mulmocast"   ← プラグイン名
         ↓
  インストール時: mulmocast@mulmocast-plugins
                  ^^^^^^^^^ ^^^^^^^^^^^^^^^^
                  プラグイン名  マーケットプレイス名
```

| 名前 | 定義場所 | 例 | 用途 |
|------|---------|-----|------|
| GitHubリポジトリ名 | GitHub | `receptron/mulmocast-claude-plugin` | `marketplace add` で指定 |
| マーケットプレイス名 | marketplace.json の `name` | `mulmocast-plugins` | `install` の `@` 以降 |
| プラグイン名 | marketplace.json の `plugins[].name` | `mulmocast` | `install` の `@` 以前 |

つまり：

- **`marketplace add`** には **GitHubリポジトリ名** を指定する
- **`plugin install`** には **プラグイン名@マーケットプレイス名** を指定する（GitHubリポジトリ名は使わない）

GitHubリポジトリ名はマーケットプレイスの登録時だけ使い、以降はmarketplace.json内で定義した名前で操作します。

### ユーザー側のインストール手順

```bash
# 1. マーケットプレイスを登録（GitHubリポジトリ名を指定）
claude plugin marketplace add receptron/mulmocast-claude-plugin

# 2. プラグインをインストール（プラグイン名@マーケットプレイス名）
claude plugin install mulmocast@mulmocast-plugins
```

Claude Code内からも操作可能です：

```
/plugin marketplace add receptron/mulmocast-claude-plugin
/plugin install mulmocast@mulmocast-plugins
```

## 2. Plugin Marketplace（公式ストア）

Anthropicが運営する公式マーケットプレイス [claude-plugins-official](https://github.com/anthropics/claude-plugins-official) に登録する方法です。

### 公式ストアの特徴

- Claude Code起動時に自動で利用可能（ユーザーがマーケットプレイスを追加する手順が不要）
- Anthropicによるレビューを経て掲載

### インストール手順（ユーザー側）

公式ストアのプラグインは、マーケットプレイスの追加なしで直接インストールできます：

```bash
claude plugin install playwright@claude-plugins-official
```

### 登録方法

自前のマーケットプレイスで動作確認した後、公式ストアへの掲載をリクエストします。詳細は [claude-plugins-official](https://github.com/anthropics/claude-plugins-official) のREADMEを参照してください。

## 3. Skills CLIで配布（vercel-labs/skills）

Pluginとは別の仕組みですが、[vercel-labs/skills](https://github.com/vercel-labs/skills) を使うと、Claude Code以外のエージェント（Cursor、Copilot、Cline等）にも対応した形でSkillを配布できます。

### 仕組み

Skills CLIは、SKILL.mdファイルを各エージェントの所定ディレクトリにシンボリックリンクで配置するツールです。

```bash
# インストール（npxで直接実行）
npx skills add owner/repo

# 一覧表示
npx skills list

# 検索
npx skills find
```

### Plugin との違い

| 項目 | Plugin | Skills CLI |
|------|--------|-----------|
| 対象エージェント | Claude Codeのみ | 40+（Claude Code, Cursor, Copilot等） |
| 配布形式 | GitHubリポジトリ + marketplace.json | GitHubリポジトリ + SKILL.md |
| インストール方法 | `/plugin install` | `npx skills add` |
| MCP Server対応 | あり | なし |
| Hooks対応 | あり | なし |

Claude Code固有の機能（MCPサーバー、Hooks等）を使う場合はPlugin、エージェント横断で使いたい場合はSkills CLIが適しています。

## Plugin Marketplace配布時のハマりポイント

ローカルで開発・テストしたSkillをPlugin Marketplaceで配布すると、ローカル開発時とは異なる挙動になる部分があります。以下は、実際に配布してみて気づいた注意点です。

### Skill名が変わる（ローカル vs マーケットプレイス）

同じSKILL.mdでも、配置場所によってスキルの呼び出し名が変わります。

**ローカルスキル**（`~/.claude/skills/story/SKILL.md`）：

```
/story
```

**プラグイン内のスキル**（`mulmocast`プラグインの`skills/story/SKILL.md`）：

```
/mulmocast:story
```

プラグインとして配布すると、**プラグイン名がプレフィックスとして付く**点に注意してください。SKILL.md内のドキュメントやサンプルで `/story` と書いている場合、ユーザーの環境では `/mulmocast:story` になります。

#### Skillから別のSkillを呼ぶ場合は特に注意

Skillの中で別のSkillを呼び出す（dispatchする）設計にしている場合、この名前の変化が深刻な問題になります。

例えば、`/create` Skillの中で `/story` Skillを呼ぶように書いていた場合：

- **ローカル開発時**: `/create` → `/story` を呼ぶ → 成功
- **マーケットプレイス配布後**: `/mulmocast:create` → `/story` を呼ぶ → **見つからない**（正しくは `/mulmocast:story`）

この問題を防ぐには、SKILL.md内でSkill名をフォールバック付きで記述しておくことが必要です：

```markdown
次のステップでは `/mulmocast:story` スキルを実行してください。
もし見つからない場合は `/story` を試してください。
```

こうしておけば、ローカル環境でもマーケットプレイス経由でも動作します。

### CLIコマンド名が変わる（開発時 vs npm配布後）

Skillの中でCLIツールを呼び出している場合、開発時と配布後でコマンド名が異なることがあります。

**開発時**（リポジトリ内で実行）：

```bash
yarn run cli images script.json
```

**npm配布後**（グローバルインストール済み）：

```bash
mulmocast images script.json
```

SKILL.md内でCLIコマンドを記載する場合は、**npm配布後のコマンド名**で書く必要があります。開発者自身はリポジトリで `yarn run cli` を使っていても、ユーザーは `mulmocast` コマンドを使うためです。

package.jsonの`bin`フィールドが、npm配布後のコマンド名を決めます：

```json
{
  "name": "mulmocast",
  "bin": {
    "mulmocast": "./dist/cli/bin.js",
    "mulmo": "./dist/cli/bin.js"
  }
}
```

#### フォールバックの記述

Skill名と同様に、CLIコマンドもフォールバックを書いておくと、開発者・ユーザーどちらの環境でも動作します：

```markdown
以下のコマンドでMulmoScriptから画像を生成してください。

mulmocast images script.json

もし `mulmocast` コマンドが見つからない場合は、
リポジトリのルートディレクトリで以下を実行してください。

yarn run cli images script.json
```

開発者がSkillをローカルでテストする際は `yarn run cli` を使い、マーケットプレイス経由でインストールしたユーザーは `mulmocast` を使うため、両方を記載しておくことで混乱を防げます。

### マーケットプレイス名とプラグイン名の衝突

marketplace.jsonの`name`とplugin.jsonの`name`を同じにすると、Linux環境でインストールに失敗する場合があります（EXDEVエラー）。名前は必ず異なるものにしてください。

```json
// marketplace.json
{ "name": "mulmocast-plugins" }    // ← "mulmocast-plugins"

// plugin.json
{ "name": "mulmocast" }             // ← "mulmocast"（異なる名前にする）
```

## まとめ

| 配布方法 | 手軽さ | 対象範囲 | 向いているケース |
|----------|--------|---------|-----------------|
| 自前マーケットプレイス | ★★☆ | Claude Code | チーム内や特定コミュニティ向け |
| 公式ストア | ★☆☆ | Claude Code | 広く一般に公開 |
| Skills CLI | ★★★ | 40+ エージェント | エージェント横断で使いたい |

## 参考リンク

- [Claude Code Plugins ドキュメント](https://code.claude.com/docs/en/plugins)
- [Plugin Marketplace 作成ガイド](https://code.claude.com/docs/en/plugin-marketplaces)
- [公式マーケットプレイス](https://github.com/anthropics/claude-plugins-official)
- [Anthropic Skills リポジトリ](https://github.com/anthropics/skills)
- [vercel-labs/skills](https://github.com/vercel-labs/skills)
- [Agent Skills 仕様](https://agentskills.io/specification)
