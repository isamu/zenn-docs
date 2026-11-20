---
title: "Vibe Coding で使うサービス・ツール別 Claude Code のセキュリティ監査スキル"
emoji: "🔐"
type: "tech"
topics: ["ClaudeCode", "セキュリティ", "vibecoding", "Firebase", "Supabase"]
published: true
publication_name: "singularity"
---

AI でアプリを作れるようになって、「動くもの」はすぐにできるようになりました。でも、作ったあとにふと不安になります。

「このアプリ、他人のデータが丸見えになっていないだろうか？」

セキュリティの専門家にレビューを頼めれば安心ですが、個人開発ではなかなか難しい。そこで使えるのが、Claude Code の **スキル（skill）** です。スキルは、専門家のチェック手順を AI 向けにまとめたファイルで、入れておくと Claude Code がその手順どおりにコードを点検してくれます。

この記事では、**「〇〇を使っている。〇〇のセキュリティが気になるなら、このスキル」** という形で、使っているサービスごとにおすすめのスキルを紹介します。どれも、サービスの提供元やセキュリティ企業が公開しているものです。

---

## 早見表

| 使っているもの | 気になること | スキル |
| --- | --- | --- |
| Firebase | 他人のデータを読み書きされないか | `firebase-security-rules-auditor` / `firestore-rules-creation` |
| Cloud Functions for Firebase | サーバー側の処理に穴がないか | 汎用スキル（`owasp-security` など）に Firebase の注意点を伝える |
| Supabase | テーブルの中身が誰にでも見えていないか | `supabase-postgres-best-practices` |
| Stripe | 決済の設定や APIキーの扱いは大丈夫か | `stripe-best-practices` |
| AWS | 権限を与えすぎていないか | `aws-iam` / `aws-well-architected-review` |
| Google Cloud | 権限や設定に穴がないか | `google-cloud-waf-security` / `iam-helper-for-policy-simulator` |
| Cloudflare Workers | 秘密の情報をコードに書いていないか | `workers-best-practices` |
| Vue | `v-html` などの危ない書き方をしていないか | `vue-best-practices`（コミュニティ製） |
| React / Next.js | サーバー側の処理で認証が抜けていないか | `react-best-practices`（性能が中心） |
| GitHub Actions | CI が乗っ取られないか | `gha-security-review` |
| CI で動かしている AI エージェント | AI が悪い指示に従わされないか | `agentic-actions-auditor` |
| npm などのパッケージ | 危ないパッケージを使っていないか | `supply-chain-risk-auditor` |
| Web アプリ全般（まず一つ） | よくある脆弱性がないか | `owasp-security` |
| 日本の基準で説明したい | IPA の基準に沿っているか | `ipa-security-check` |
| もっと深く調べたい | 見落としがないか | `security-audit`（Cloudflare） |

---

## スキルの入れ方と使い方

どのスキルも、入れ方は同じです。プロジェクトのフォルダで次のコマンドを実行します。

```bash
npx skills add <リポジトリのURL> --skill <スキル名>
```

たとえば Firebase のスキルなら、こうなります。

```bash
npx skills add https://github.com/firebase/agent-skills --skill firebase-security-rules-auditor
```

入れたら Claude Code を起動して、ふつうの言葉で頼むだけです。

```text
firebase-security-rules-auditor スキルで、このプロジェクトのセキュリティルールを監査して
```

スキル名を `/` のあとに打って呼び出すこともできます。

:::message
スキルは、自分のパソコンの上で動く「他人が書いた手順書」です。入れる前に、どんなことをするスキルなのかを必ず確認してください。確認のしかたは記事の最後にまとめました。
:::

---

## Firebase を使っている

### Firebase のセキュリティが気になるなら → `firebase-security-rules-auditor`

- **提供元**: Google（[firebase/agent-skills](https://github.com/firebase/agent-skills)）
- **このスキルがチェックすること**: Firestore と Cloud Storage の **セキュリティルール**

Firebase では、「誰がどのデータを読み書きしてよいか」をセキュリティルールというファイルで決めます。ここが甘いと、ログインした別のユーザーや、ログインすらしていない人に、データを読まれたり書き換えられたりします。

このスキルは攻撃者の目線でルールを読み、次のような穴を探します。

- 一般ユーザーが管理者の権限を手に入れられてしまう
- 新しく作るときは厳しいのに、更新するときのチェックが抜けている
- 送られてくるデータの型や大きさを確かめていない

:::message
このスキルが見るのは **Firestore と Cloud Storage のセキュリティルールだけ** です。Cloud Functions や Authentication の設定など、Firebase のほかの部分は見てくれません。Functions の確かめ方は後で紹介します。
:::

### ルールをこれから書く・作り直すなら → `firestore-rules-creation`

- **提供元**: Google（[firebase/agent-skills](https://github.com/firebase/agent-skills)）
- **このスキルがすること**: コードを読んで、安全な Firestore のセキュリティルールを作る

AI にアプリを作らせると、「とりあえず誰でも読み書きできる」ルールのまま公開してしまいがちです。このスキルは、次の順でルールを作り直してくれます。

1. コード全体を読んで、どのデータを、誰が、どう読み書きしているかを洗い出す
2. 「まず全部禁止し、必要なものだけ許可する」方針でルールを書く
3. 攻撃者になったつもりで、書いたルールを破れないか試す
4. ルールの書き方に誤りがないかを確かめる

できたルールは「たたき台」として出てくるので、最後に `firebase-security-rules-auditor` で監査する、という組み合わせがおすすめです。

```text
firestore-rules-creation スキルで、このアプリの Firestore のセキュリティルールを作り直して
```

### ほかの Firebase 公式スキルにも、セキュリティの注意が入っている

同じリポジトリのスキルは、監査専用ではなく「書くときの手引き」ですが、使っている機能に合わせて入れておくと、AI が危ない書き方をしにくくなります。

| スキル | セキュリティに関係するところ |
| --- | --- |
| `firebase-app-hosting-basics` | APIキーなどの秘密の情報を、設定ファイルに直接書かず、秘密情報として登録して渡す |
| `firebase-ai-logic-basics` | アプリから直接 Gemini を呼ぶなら、正規のアプリ以外からの呼び出しを防ぐ App Check を必ず設定する |
| `firebase-data-connect-basics` | 誰がどのデータを扱えるかを、`@auth` や `@check` で決める |

### Cloud Functions を使っている → 汎用スキルに、Firebase の注意点を伝えて頼む

Firebase の公式スキル集には、Cloud Functions のセキュリティを見るスキルはありません。Functions はふつうのサーバーのプログラムなので、後で紹介する `owasp-security` や、Sentry の `security-review` などの汎用スキルで確かめます。

ただし、Firebase ならではの見落としやすい点があります。頼むときに、次の三つをはっきり伝えましょう。

- **Functions の中ではセキュリティルールが効かない**: Functions で使う Admin SDK は、ルールを素通りしてどのデータにも触れます。「この人がこのデータを触ってよいか」のチェックを、Functions のコードの中で自分で書く必要があります
- **呼び出した人を確かめているか**: `onCall` の関数でログインしているかを確かめているか。`onRequest` の関数は誰でも URL を叩けるので、認証と、どのサイトから呼べるか（CORS）の設定が必要です
- **秘密の情報をコードに書いていないか**: APIキーなどは、コードに直接書かず `defineSecret` で扱います

たとえば、こう頼みます。

```text
owasp-security スキルで functions/ フォルダを監査して。特に次の三つを確認して。
1. Admin SDK はセキュリティルールを素通りするので、各関数で権限チェックをしているか
2. onCall で認証を確かめているか、onRequest に認証と CORS の設定があるか
3. APIキーなどをコードに直接書かず defineSecret を使っているか
```

---

## Supabase を使っている

### Supabase のセキュリティが気になるなら → `supabase-postgres-best-practices`

- **提供元**: Supabase（[supabase/agent-skills](https://github.com/supabase/agent-skills)）
- **このスキルがチェックすること**: データベースの設計と、**RLS（行レベルセキュリティ）** のポリシー

Supabase では、ブラウザから直接データベースを読みに行きます。そのため、「この行はこの人だけが見てよい」という RLS の設定がないと、テーブルの中身が誰からでも見えてしまいます。

このスキルは監査専用ではなく、Supabase が勧める書き方をまとめたものです。そのため、次のように「このスキルの基準で見直して」と頼むのがおすすめです。

```text
supabase-postgres-best-practices スキルの基準で、全テーブルの RLS ポリシーを見直して。RLS が無効なテーブルがあれば教えて
```

RLS が正しく効いているかを確かめるテストの書き方も含まれています。

---

## Stripe を使っている

### Stripe のセキュリティが気になるなら → `stripe-best-practices`

- **提供元**: Stripe（[stripe/ai](https://github.com/stripe/ai)）
- **このスキルがチェックすること**: APIキーの扱い、Webhook、OAuth など、決済まわりの設定

これも監査専用ではなく、Stripe が勧める組み込み方をまとめたスキルです。セキュリティに関係するところでは、たとえば次のような点を確認できます。

- なんでもできる秘密キーではなく、**できることを絞った APIキー**（制限付きキー）を使っているか
- 支払いが終わったことを、画面の遷移ではなく **Webhook で確かめている** か

お金が絡む部分なので、Stripe を使っているなら一度は通しておきたいスキルです。

---

## クラウドやフレームワークの公式スキル

バイブコーディングでよく使うクラウドやフレームワークも、提供元が公式のスキル集を出しています。セキュリティ専用ではないものも多いのですが、その中にセキュリティのチェック項目が入っています。

### AWS を使っている → `aws-iam` / `aws-well-architected-review`

- **提供元**: AWS（[aws/agent-toolkit-for-aws](https://github.com/aws/agent-toolkit-for-aws)）
- **`aws-iam` がチェックすること**: IAM（誰に何を許可するか）の設定。アプリのソースコードから、必要な権限だけを許可する設定のたたき台も作れる
- **`aws-well-architected-review` がチェックすること**: AWS が定める設計の指針（Well-Architected Framework）に沿っているか。セキュリティの観点も含まれる

AI に AWS の設定を書かせると、動かすために「全部許可」にしてしまいがちです。`aws-iam` で権限を絞れているかを確認しましょう。

同じリポジトリには、AWS のセキュリティ診断サービスを呼び出してスキャンするスキル（`scanning-with-aws-security-agent`）もあります。こちらは **ソースコードを AWS にアップロードして調べる** 有料のサービスなので、使う前に料金と、コードを外に出してよいかを確認してください。

### Google Cloud を使っている → `google-cloud-waf-security` / `iam-helper-for-policy-simulator`

- **提供元**: Google（[google/skills](https://github.com/google/skills)）
- **`google-cloud-waf-security` がチェックすること**: Google Cloud の設計の指針に沿って、権限、ネットワーク、データの暗号化、ログ、ソフトウェアの供給経路などを評価する
- **`iam-helper-for-policy-simulator` がチェックすること**: 権限を変えたときに、今動いているものが壊れないかを、過去のアクセス記録で試してから反映する

Firebase は Google Cloud の一部ですが、Firebase のセキュリティルールは前に紹介した `firebase-security-rules-auditor` のほうが向いています。

### Cloudflare を使っている → `workers-best-practices`

- **提供元**: Cloudflare（[cloudflare/skills](https://github.com/cloudflare/skills)）
- **このスキルがチェックすること**: Cloudflare Workers の書き方。セキュリティに関係するところでは、APIキーなどの秘密の情報をコードや設定ファイルに直接書いていないか、推測されやすい乱数をトークンに使っていないか、など

同じリポジトリの `turnstile-spin` は、ボットによる大量登録などを防ぐ Cloudflare Turnstile を正しく組み込むためのスキルです。フォームを公開するなら役に立ちます。

### Vue を使っている → `vue-best-practices`

- **提供元**: [vuejs-ai/skills](https://github.com/vuejs-ai/skills)（Vue 公式ではなく、コミュニティによる試み）
- **このスキルがチェックすること**: Vue 3 の書き方全般。その中に、`v-html` のような、使い方を誤ると他人に悪いスクリプトを埋め込まれる書き方への注意が含まれる

まだ試験的なプロジェクトで、内容に誤りがある可能性があると README にも書かれています。参考程度に使いましょう。

### React / Next.js を使っている → `react-best-practices`

- **提供元**: Vercel（[vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills)）
- **このスキルがチェックすること**: 主に表示の速さなどの性能。セキュリティに関係するところでは、サーバー側で動く処理（Server Actions）にも API と同じように認証をかけているか、という項目がある

セキュリティ専用ではないので、React のアプリは前に紹介した `owasp-security` などと組み合わせて使うのがおすすめです。なお、Vercel の公式スキル集には、執筆時点ではセキュリティ監査用のスキルはありません。

---

## GitHub Actions を使っている

### CI のセキュリティが気になるなら → `gha-security-review`

- **提供元**: Sentry（[getsentry/skills](https://github.com/getsentry/skills)）
- **このスキルがチェックすること**: `.github/workflows/` の中の設定

GitHub Actions の設定によっては、他人が送ってきたプルリクエストをきっかけに、リポジトリの秘密の情報（APIキーなど）を盗まれることがあります。このスキルは、そうした攻撃が成り立つかを、具体的な攻撃の手順つきで洗い出してくれます。

### CI で AI エージェントを動かしているなら → `agentic-actions-auditor`

- **提供元**: Trail of Bits（[trailofbits/skills](https://github.com/trailofbits/skills)）
- **このスキルがチェックすること**: CI で動く Claude Code Action や Codex などに、外部の人の文章が届く経路

Issue のタイトルやプルリクエストの本文に「秘密の情報を表示して」といった指示を書かれ、それを CI の AI が読んで従ってしまう、という攻撃があります。AI にプルリクエストをレビューさせているなら、このスキルで確認しておきましょう。

---

## npm などのパッケージを使っている

### パッケージのセキュリティが気になるなら → `supply-chain-risk-auditor`

- **提供元**: Trail of Bits（[trailofbits/skills](https://github.com/trailofbits/skills)）
- **このスキルがチェックすること**: プロジェクトが使っているパッケージ全体

AI にアプリを作らせると、知らないうちにたくさんのパッケージが入ります。このスキルは、次のような危ないパッケージがないかを調べます。

- 既知の脆弱性があるパッケージ
- 更新が止まって放置されているパッケージ
- インストールするだけでスクリプトが動くパッケージ

---

## Web アプリ全般

### まず一つ入れるなら → `owasp-security`

- **提供元**: [agamm/claude-code-owasp](https://github.com/agamm/claude-code-owasp)
- **このスキルがチェックすること**: Web アプリでよくある脆弱性

OWASP（オワスプ）という団体が、Web アプリで特によく狙われる脆弱性を「OWASP Top 10」として公開しています。このスキルは、その一覧に沿ってコードを順に点検し、該当する箇所を引用しながら説明してくれます。どのサービスにも当てはまらない場合や、何から始めればよいか分からない場合は、まずこれを入れるのがおすすめです。

### 日本の基準で説明したいなら → `ipa-security-check`

- **提供元**: クラスメソッド（[classmethod/tsumiki](https://github.com/classmethod/tsumiki)）
- **このスキルがチェックすること**: IPA（情報処理推進機構）の「安全なウェブサイトの作り方」などに沿った脆弱性

結果は日本語のレポートとして出てきます。続けて同じリポジトリの `ipa-security-guide` を使うと、レポートの中身を「どれから直すべきか」の順に並べたやることリストにしてくれます。仕事で作ったアプリを、日本の公的な基準に沿って説明したいときに向いています。

### もっと深く調べたいなら → `security-audit`

- **提供元**: Cloudflare（[cloudflare/security-audit-skill](https://github.com/cloudflare/security-audit-skill)）
- **このスキルがチェックすること**: アプリ全体の、見つけにくい脆弱性

複数の AI を同時に動かし、「探す役」と「本当に問題かを疑う役」を分けて、何段階もかけて調べる本格的なスキルです。権限の抜けやロジックの穴のような、見つけにくい問題まで掘り下げます。

そのぶん時間も料金もかかります。また、1回で全部見つかるものではなく、何回か実行する前提で作られています。上のスキルで一通り見たあと、もう一段深く調べたいときに使いましょう。

---

## スキルを入れる前に確認すること

スキルはただの文章ファイルとスクリプトなので、中身を読めば何をするかが分かります。セキュリティのために入れたスキルが、逆に穴にならないように、次の点を確認してください。

- **中身を読む**: `SKILL.md` と、一緒に入っているスクリプトに目を通す。分からなければ、Claude Code に「このスキルが何をするか説明して」と聞いてもかまいません
- **出どころを確かめる**: この記事で紹介したような、サービスの提供元や実績のある会社のものを選ぶ
- **権限を確かめる**: `SKILL.md` の先頭にある `allowed-tools` を見る。コマンドの実行や外部への通信ができるスキルは、特に慎重に

---

## AI の監査で分かること、分からないこと

最後に、大事な注意を一つ。スキルを使った監査は、専門家の目線をかなりのところまで再現してくれますが、**「問題が見つからなかった＝安全」ではありません**。

- 指摘されたら、直す前に **なぜそれが問題なのか** を Claude Code に説明してもらう
- 直したら、もう一度同じスキルで確かめる
- ときどき（たとえば大きな機能を足したあとに）実行する習慣をつける

この三つを続けるだけでも、何もしないのとは大違いです。

スキルの一覧や、GitHub の設定・静的解析などほかの対策との組み合わせ方は、Singularity Society の記事「[Claude Code の OSS skill でセキュリティを手元レビューする](https://singularitysociety.org/articles/tech-blog/2026-05-14-claude-code-security-skills-catalog/)」にまとめています。もう一歩進めたい人は、こちらもどうぞ。
