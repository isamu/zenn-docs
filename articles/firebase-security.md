---
title: "Firebaseでサービスを公開するときに気を付けるセキュリティポイント"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Firebase, Security, JavaScript]
published: true
publication_name: "singularity"
---

Firebaseでサービスを公開するときに、最低限必要なセキュリティーのポイントをまとめています。
Firebaseは、現在もすごい勢いで開発が進んでいるので、セキュリティーに関するポイントやノウハウも日々更新されます。最新のドキュメントや、変更点を調べることをおすすめします。この記事を含め、ネットの記事は古い可能性があります。2022年前後でも、App Check導入、Cross service Rulesの導入、AuthenticationのIdentity Platform導入、Workload Identity連携の導入など、日々更新されています。

公式の[Firebaseのセキュリティチェックリスト](https://firebase.google.com/support/guides/security-checklist)は必ず目を通しましょう。


# GCPインフラ
  - Firebaseで使うGoogle Accountは個人のアカウントを使わないでGoogle Workspace/Cloud Identityのものを使う
  - 管理権限の不正ログインがないか確認するため、Cloud Identityのaudit logで定期的にログインのログを確認する
  - Google Accountの2FA(MFA)は、必ず有効にする
  - パスワードなども制限できるなら、ランダムな文字列の組み合わせで推測できないようにする
    - 定期的な更新は推奨しない
    - 他のパスワードの使いまわしは厳禁。
    - 複数のユーザでアカウント共有禁止。
  - Firebaseの管理権限をもったユーザは最小限に保つ。与える権限は必要最小に。
    - 不要になったユーザは定期的に無効にする
  - IAM, Service keyの権限も定期的に見直しをする
  - Security Command Centerを有効にして定期的に問題がないか確認をする
    - 有料プランだと年間300万円かかるので、無料のプランで。
  - Firebaseへのアクセス(ログイン)や各種操作をしたときにEventarcを使って通知が来るように設定をする
    - Cloud Run + Triggerで通知をするなど [参考になる記事](https://medium.com/google-cloud-jp/eventarc-が-ga-になったので-試しに-firebase-console-へのアクセスをトリガーに-cloud-run-を実行してみる-540c7352ed70)
  - firebaseのCI用の認証トークンはCI等では使わない
    - `firebase login:ci`で生成されるトークン
    - 以下の特徴のため、漏れると危険
      - グーグルアカウントに紐付いている強力なトークン
      - トークンを取り消すにはトークンの情報が必要なので保存しておく必要がある
      - 今までに発行したトークンの一覧を確認する方法がない
    - 
# Firebase

## App Check
  - Firebaseは公開されているapi keyなどを使えばどこからでもアクセスができる
  - App Checkを導入することにより、バックエンドに対してのアクセス元を制限できる
    - [公式ドキュメント](https://firebase.google.com/docs/app-check)
    - reCAPTCHA3などで、正しいリクエスト以外を弾くしくみ

## Firestore
  - Firestoreはsecurity rulesをしっかり設定する
    - 全世界に読み書き権限を与えない。使わない場合は、全部読み書きできないようにしておく。
    - ユーザの権限ごとに読み書きをしっかり設定する
      - customClaim, email認証などを使って権限をわける
    - ユーザが更新すべきでない、参照すべきでないデータは、Functionsで読み書きすることも考える
      - Firebaseは基本的にはクライアントで読み書きするが、必要なときにはFunctionsを使う
    - 再帰ワイルドカード (`{document=**}`)
      - 再帰ワイルドカードが定義されていると、そのルールが定義よりも深い階層全て適用されるため、想定をしていない権限が許可されている事があるので、利用には注意する
  - １つのdocumentに、ユーザが書き換え可能なものと、システムが更新するものは極力混ぜない
    - 混ざった場合には security ruleでユーザがその項目を更新したり、create時にその項目を勝手に設定できないように気を付ける
    - create時にdefault値が必要な場合には、Triggerを使ってcreate時にsetするなど。
  - 定期的に攻撃者目線で、問題ないか確認する
    - 自社システムをアタックするハックデイを開催してもよい。
  - 個人情報は特に注意
    - 読み書き権限は特に。
    - 公開可能な情報と、documentレベルでしっかり分ける
    - collectionGroupでうっかりとれないように注意する
    - 基本は不要なデータは取らないようにする
    - 不要になった個人情報は定期的に削除する
  - Firestore全体のバックアップを設定する
    - [データのエクスポートをスケジュールする](https://firebase.google.com/docs/firestore/solutions/schedule-export)

## Authentication
  - AuthenticationはFirebaseのものではなく、GCPのIdentity Platform を使うのがよい
    - Identity Platformを使うと、多要素認証,ブロッキング関数などが使える
    - Firebase Authenticationと互換性がある
    - [Identity Platform と Firebase Authentication の違い](https://cloud.google.com/identity-platform/docs/product-comparison?)
  - 認証の仕組みとしては、OAuth2.0プロバイダーが最も安全
    - Google, Appleの認証を使えないか検討をする
    - email/password認証を使う場合は、ブルートフォース攻撃を防ぐために、サインインエンドポイントの割り当ての制限をする
  - Identity Toolkit APIのquotaを設定する

## Storage
  - Storage.rulesでwrite権限やファイルサイズ、ファイル種別の制限をして、ユーザが不法なファイルをアップロードできないようにする。
  - uploadされた画像は、resizeなどをして、そのときにexifを削除して他のユーザへ表示するようにして、オリジナルの画像は表示させない
  - 2022/09にリリースされた[Cross service Rules](https://firebase.blog/posts/2022/09/announcing-cross-service-security-rules) による、Storage.rulesからfirestoreを参照できるようになったので、そちらを使う
  
## Hosting
  - HTTP Headerを設定して、不正なリクエストを防ぐ(Functionsでexpressを使っている場合は、そちらも忘れずに)
```
    "headers": [{
        "source": "**",
        "headers": [
          { "key" : "Content-Security-Policy", "value": "frame-ancestors 'none'"},
          { "key" : "X-Frame-Options", "value" : "deny" },
          { "key" : "X-Content-Type-Options", "value" : "nosniff" },
          { "key" : "X-XSS-Protection", "value" : "1; mode=block" },
          { "key" : "X-Permitted-Cross-Domain-Policies", "value" : "none" },
          { "key" : "Referrer-Policy", "value": "no-referrer" }
        ]
    }]
```
  
  - 必要なら開発系にアクセス制限(IPや認証)を設定する
    - express経由等の設定が必要。簡易ならweb層で

##  Function
  - 一般的なバックエンドのセキュリティ対策をする
    - 変数チェック、権限チェック、インジェクション
  - ユーザ属性の管理
    - 例えば特権ユーザを作るなら、customClaimを使うなどして、他のユーザがFunctionsを叩いても動作しないようにする
    - email認証のユーザとSMS認証のユーザで動作を変えるならauth.token.phone_numberとauth.token.emailを確認する
    - email確認済みのユーザかauth.token.email_verifiedで確認する、など
  - APIのDoS対策にrate limitを導入する
  - Functionsのログに秘匿情報を出さないようにする
  - 環境変数に機密情報を入れない
    - Cloud Functionsは関数の呼び出し間で環境を再利用するため、機密情報を環境に保存しない
    - Cloud SecretManagerを使う
  - 必要ならFunctionsの操作ログを自前で実装する
    - functions.logger.logを使って、Cloud loggingで参照しやすいように。
  - DoS攻撃で課金が増える可能性があるので、インスタンスの最大数の上限を設定しておく
    - maxInstancesを指定する
    
# API Key
  - API Keyは公開可能なものと、そうでないものは区別して厳重に管理
    - firebaseConfigは公開可能
    - Firebase Admin SDKのprivate keyは絶対に公開しない。アプリやwebに組み込まない。レポジトリにも登録しない。
    - `firebase login:ci`で生成されるトークンは使わない
  - AWS/GitHubの外部のシステムと連携してfirebaseのリソースを操作する場合は、private keyではなくWorkload Identity 連携を使えるか検討する
    - https://cloud.google.com/iam/docs/workload-identity-federation
  - 公開可能なものは、設定を確認しておく（アクセス可能な範囲等）
  - 秘匿な鍵は厳重に管理。仕組みとして.gitignoreを使うなどしてGitに登録できないようにする。

# バックエンドサービスの監視とアラートを設定する
   - Firestore, Hosting, Storageは監視とアラートの設定を追加

# その他
  - Node.jsやnpmのpackageなどは、なるべく最新のバージョンに上げる
    - GithubのDependabot alertsなどを有効にして、脆弱性のあるパッケージは確認して、影響がある内容であれば必ずバージョンを上げる。
  - audit logで取れるものはとっておく。
    - インフラ周りはCloud Logging
    - functionsのログは、functions.loggerを活用
  - オープンソースであればCodeQLを利用するして、脆弱性の静的診断をする
  - FirestoreやStorageのTriggerでFunctionsを呼び出すときは、Loopしていないか確認をする
    - 無限課金される

# Firebaseへの要望
  - HostingやfuncitonsにWAFが欲しい
  - もっと詳細なAudit logが欲しい
  - クラウドレベルで、各種サービスへのにIP制限などの機能がほしい
