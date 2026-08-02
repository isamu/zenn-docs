# zenn-docs

[Zenn](https://zenn.dev/singularity) の記事と、プロジェクト設定のガイドを置いています。

## 📁 構成

| ディレクトリ | 中身 |
|---|---|
| [`articles/`](articles/) | Zenn の記事（`published: false` は下書き） |
| [`books/`](books/) | Zenn の本 |
| **[`guides/`](guides/)** | **エージェント向けの設定指示書**（記事ではない） |

## 🤖 guides/ — 他プロジェクトから参照する設定ガイド

**AI エージェントが他プロジェクトの設定を自動で整えるための指示書**です。記事とは目的が違うので分けてあります。

| ファイル | 用途 |
|---|---|
| [`typescript-eslint-setup.md`](guides/typescript-eslint-setup.md) | **TypeScript / ESLint の設定**。React / Vue / Node / モノレポ / サーバ+フロント同梱に対応 |

### 使い方

他のプロジェクトの `CLAUDE.md` などから、こう参照します。

```markdown
TypeScript / ESLint の設定は、以下のガイドに従うこと。

https://github.com/isamu/zenn-docs/blob/master/guides/typescript-eslint-setup.md

- 設定ファイルを読んで判断しない。ガイドの「0. 最初にやること」の3コマンドで実効値を出してから始める
- 既存プロジェクトなら warn から段階的に。いきなり error にしない
```

エージェントに直接渡す場合:

```
guides/typescript-eslint-setup.md の内容に従って、このプロジェクトの設定を点検して。
まず現状を測って、抜けている項目と、それを入れたら何件出るかを報告してから、進め方を相談して。
```

### このガイドが答えること

- `strict` を入れても入らない**6つの設定**
- **`as` を止めるルールが `strict` ではなく `stylistic` にある**という罠
- 型情報を使うルールの配線（`projectService` か `project` 明示か）
- **Vue で2ブロックに分けないと全 SFC が壊れる**件
- 既存プロジェクトへの段階的な入れ方
- **貼って走らせれば抜けが分かる検証コード**

内容は [receptron/mulmoterminal](https://github.com/receptron/mulmoterminal) と [receptron/mulmoclaude](https://github.com/receptron/mulmoclaude) での実測と実運用に基づいています。

## ✍️ 記事を書く

```bash
npx zenn preview        # プレビュー
npx zenn new:article    # 新規記事
```

* [📘 Zenn CLI の使い方](https://zenn.dev/zenn/articles/zenn-cli-guide)
