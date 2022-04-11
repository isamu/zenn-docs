---
title: "functions:secrets:setが追加され、Functionsのconfigを使った環境変数が非推奨になった話"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Firebase, Security, Functions]
published: true
---

[Firebaseのセキュリティチェックリスト](https://firebase.google.com/support/guides/security-checklist)を見ていると、


> CloudFunctionの環境変数に機密情報を入れないでください
>多くの場合、自己ホスト型のNode.jsアプリでは、環境変数を使用して秘密鍵などの機密情報を含めます。 CloudFunctionsではこれを行わないでください。 Cloud Functionsは関数の呼び出し間で環境を再利用するため、機密情報を環境に保存しないでください。

とありました。(2022/04/11)
以前は、`functions:config:set` で秘匿情報を登録し、 `functions.config().some.key`で取り出すという方法が一般的だったのですが、いつに間にか推奨させないようになっていました。「呼び出し間で環境を再利用」とあるので、コンテナを共有あるいは使い回しをし、他のプロジェクトで使われていた古い設定が残るようです。以前からこの問題があり、セキュリティ上の理由で推奨をためたのか、それともコンテナの効率化で変更になったのか理由はわかりませんが、新しい方法が用意されたようです。

Firebaseのドキュメントの[環境を構成する](https://firebase.google.com/docs/functions/config-env)を見てみると、npmの `firebase-functions v3.18.0`と、CLIの`firebase-tools v 10.2.0`の組み合わせで、秘匿情報は functions:secrets:setを使って管理するようになったようです。その他にも、このバージョンから.envを使った環境変数の設定をサポートをするようになったようです。秘匿ではない情報はこちらで管理すると楽ですね。まずは、これらのパッケージをupdateしましょう。


`functions:secrets:set`での登録の方法は`config`とほぼ同じで

```
firebase functions:secrets:set SECRET_NAME
```
で、秘匿情報をセットし、

```
exports.processPayment = functions
  // Make the secret available to this function
  .runWith({ secrets: ["SECRET_NAME"] })
  .onCall((data, context) => {
    const myBillingService = initializeBillingService(
      // reference the secret value
      process.env.SECRET_NAME
    );
    // Process the payment
  });
```
と、runWithに、必要な秘匿情報の名前を渡すと process.env.SECRET_NAME のように環境変数としてアクセスできるようです。

基本的には、firebase functions:config:set と似たような方法で使うことができます。

[firebase-functionsのChangeLog](https://github.com/firebase/firebase-functions/releases/tag/v3.18.0)をみると、

>Add new runtime option for setting secrets.

とありますね。


Functions secretsは、実際にはGoogle Cloud Secret Managerで情報を管理しているので、若干のコストがかかります。 6 個のシークレット、１シークレットあたり毎月 10,000 回のアクセス オペレーションまでは無料で、それ以外が、アクセス オペレーション 10,000 回あたり $0.03かかります。
詳しくは[Secret Manager の料金](https://cloud.google.com/secret-manager/pricing)を参照してください。

試してみると、
```
$ firebase functions:secrets:set TEST

Error: HTTP Error: 403, Secret Manager API has not been used in project xxxxxxxxxx before or it is disabled. Enable it by visiting https://console.developers.google.com/apis/api/secretmanager.googleapis.com/overview?project=xxxxxxxxxx then retry. If you enabled this API recently, wait a few minutes for the action to propagate to our systems and retry.
```
と、まずはSecret Manager APIを有効にする必要があります。

```
$ firebase functions:secrets:set TEST
? Enter a value for TEST [hidden]
✔  Created a new secret version projects/998434940151/secrets/TEST/versions/1
```
登録できました。


`functions:config`で管理していた情報は、やはめに`functions:secrets:set`に移行したほうが良さそうです
使わなくなった config内の変数は`firebase functions:config:unset paramname`で削除する必要があるのでお忘れなく。





