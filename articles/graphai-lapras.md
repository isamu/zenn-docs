---
title: "GraphAI - MCPを使って年収1億円の仕事を探す"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::

GraphAIは、MCP AgentでMCPクライアントをサポートしています。
LAPRASから公開されたMCPサーバを使って、GraphAIで仕事を探したいと思います。

npmでもGraphAIのMCP Agentは公開されていますが今回はagentのレポジトリをcloneしてサンプルコードを使います。

```bash
git clone https://github.com/receptron/graphai-agents.git
cd  protocol/mcp-agent
```

gitからcloneして、mcp-agentのディレクトリーに移動します。

protocol/mcp-agentの下に、tests/run_chat.tsというGraphAIを使ってMCPサーバとチャットをするサンプルがあります。

mcpConfigを以下のように書き換えます。必要な作業はこれだけです。

```TypeScript
export const mcpConfig = {
  lapras: {
    command: "npx",
    args: [
      "-y",
      "@lapras-inc/lapras-mcp-server"
    ]
  }
};
```

起動して
```sh
npx ts-node tests/run_chat.ts
```

なにができるか聞いてみます。

```
✔ You: なにができるんだい？
私は、特定の仕事検索や求人情報の詳細取得など、仕事に関連する情報を提供することができます。また、複数のツールを同時に使用して効率的に情報を取得することも可能です。何か手伝えることはありますか？
```

仕事が探せるようです。

```
✔ You:  フルタックエンジニアのしごとある？
以下はフルスタックエンジニアに関連する求人情報です。興味のあるものがあれば、詳細を確認してみてください。

1. **Webアプリ開発エンジニア[Sansan Data Hub/Salesforce・AppExchange]**
  - 会社: Sansan株式会社
  - 勤務地: 東京都、大阪府
  - 給与: 7,140,000円〜11,270,000円
  - [詳細はこちら](https://lapras.com/jobs/6484)

2. **テクニカルリード アーキテクト［Bill One］**
  - 会社: Sansan株式会社
  - 勤務地: 東京都、大阪府、愛知県、福岡県
  - 給与: 9,940,000円〜17,850,000円
  - [詳細はこちら](https://lapras.com/jobs/5673)

3. **【フルリモート｜スーパーフレックス】日本の働き方を変えるHRテック企業でシェアトップクラスクラウド型タレントマネジメントシステム開発を推進するデータ基盤エンジニア**
  - 会社: 株式会社カオナビ
  - 給与: 5,500,000円〜9,000,000円
  - [詳細はこちら](https://lapras.com/jobs/7315)

4. **【バックエンド Java/リモートOK/お金の悩みを解決し、一生涯利用するサービスへ】業界No.1のtoC資産運用アプリ「WealthNavi」のリードエンジニアを募集！**
  - 会社: ウェルスナビ株式会社
  - 勤務地: 東京都
  - 給与: 6,000,000円〜10,000,000円
  - [詳細はこちら](https://lapras.com/jobs/8789)

興味のある求人があれば、リンクをクリックしてさらに詳細をご確認ください。
```

いろいろと紹介してくれます。条件を追加してみましょう

```
✔ You: おおお。年俸1億円くらいはある？
年俸1億円に近い求人は見当たりませんでしたが、いくつか高給の求人があります。以下にその一部を紹介します。

1. **テクニカルリード [iOS]**
  - 会社: Sansan株式会社
  - 勤務地: 東京都、大阪府、福岡県
  - 給与: 10,100,000円〜18,060,000円
  - [詳細はこちら](https://lapras.com/jobs/6225)

2. **テクニカルリード アーキテクト[Eight]**
  - 会社: Sansan株式会社
  - 勤務地: 東京都
  - 給与: 10,010,000円〜18,060,000円
  - [詳細はこちら](https://lapras.com/jobs/6270)

3. **テクニカルリード アーキテクト［データ化］**
  - 会社: Sansan株式会社
  - 勤務地: 東京都、大阪府
  - 給与: 10,010,000円〜18,060,000円
  - [詳細はこちら](https://lapras.com/jobs/2738)

4. **【プロダクト責任者】法務・契約業務の変革を進め、ビジネスにスピードをもたらす「MNTSQ CLM」**
  - 会社: MNTSQ株式会社
  - 勤務地: 東京都
  - 給与: 10,000,000円〜15,000,000円
  - [詳細はこちら](https://lapras.com/jobs/7657)

これらの求人は高給ですが、年俸1億円には達していません。興味がある場合は、リンクをクリックして詳細を確認してください。
? You:
```

ちょっと希望の条件には当てはまらなかったので、今回は応募は見送りですが、Sansanだと好条件だということがわかりました。

GraphAIとMCPを使うと、簡単に仕事を探すことが出来ます！！