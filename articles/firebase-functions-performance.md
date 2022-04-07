---
title: "Firebase Functions for Firebaseは遅い？"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Firebase, Functions, Firestore, gRPC]
published: true
---

FirebaseでFucntionsをクライアントから同期的に呼びだすと、応答にとても時間がかかることがあります。特にFunctionsの関数がデフォルトの設定の場合、数時間の時間を空けて呼び出すと、通常ならすぐに応答する関数も4秒以上かかることがあります。それらを改善するいくつかの方法を試したのでまとめます

# ホットスタンバイにする


[コールド スタートの数を減らす](https://firebase.google.com/docs/functions/manage-functions?hl=ja#reduce_the_number_of_cold_starts)を参考に、ホットスタンバイのインスタンスを指定する

```
exports.getAutocompleteResponse = functions
    .runWith({
      // Keep 5 instances warm for this latency-critical function
      minInstances: 5,
    })
    .https.onCall((data, context) => {
      // Autocomplete a user's search term
    });
```

これでhotstandbyになるという記事があったが、確認方法がないので実際にhotstandbyになっているか不明。あまり効果がなかった。

# 近くのリージョンを利用する
defaultではus-central1(米国中部)なのでasia-northeast1(東京)に指定する

```
exports.JpFunc = functions
  .region("asia-northeast1")
  .https.onCall(async (data, context) => {
    // Some func
    });
```
RTT分の効果はあり


# メモリを割り当てる
defaultでは256MBなので、増やす。増やすだけで起動速度が改善される。

```
exports.JpFunc = functions
  .runWith({
    memory: "1GB" as "1GB",
  })
  .https.onCall(async (data, context) => {
    // Some func
    });
```

劇的に効果があり。単純な関数ならこれだけでもよい。4s -> 200ms以下くらいになった。遅い関数でも1s程度になる。

# それでも初回時に遅い問題

上記の対策をしても特定の同期関数で1s程度はかかる関数があります。処理の内容を考えても100ms以下で処理できる内容で、連続して呼び出すと2回目以降は100ms以下で応答が返ってきます。時間がかかる関数の特徴を調べると、Firestoreを使っている関数とAuthenticationを使っている関数、つまりFirebaseのほかサービスを呼び出しているという共通点がわかりました。

調べていくと、こちらの[チケット](https://issuetracker.google.com/issues/158014637?pli=1)に、gRPCの初期化にとても時間がかかるという情報がありました。
初期化に時間がかかるので、一度インスタンスが起動してFirebaseが初期化されている状態だと、すぐに応答が返ってくる理由もわかります。

改善方法としては、

- Functions内ではFirebaseの他サービスを極力使わない
- 同期関数ではなく非同期関数で処理する
  - Triggerを利用する
  - Functionsでレスポンスは返すが、処理は非同期で続ける
- 定期的にFunctionsを呼び出して、Hot standbyの状態を維持する
- Firestoreは、別のライブラリを使って処理する
  - 上記チケット内でgRPCを使わないFirestoreのライブラリが紹介されていました
  - [@bountyrush/firestore](https://www.npmjs.com/package/@bountyrush/firestore)
- 1s程度なのでUI側で工夫して、遅延を許容する

などがあります。

特にカスタム認証を使う場合には、どうしてもFunctions側でAuthenticationを使う必要があり、代替え案が今のところないので、1s程度の遅延をUIでごまかす必要があります。


# まとめ
まとめると、

- Functionsの同期関数は以下の設定を追加する

```
exports.getAutocompleteResponse = functions
  .region("asia-northeast1")
  .runWith({
    // Keep 5 instances warm for this latency-critical function
    minInstances: 5,
    memory: "1GB" as "1GB",
  })
  .https.onCall((data, context) => {
    // Autocomplete a user's search term
  });
```

- Function内でFirestore, Authenticationを使う場合は初期化に時間がかかるので注意する

です。なにか、他にも良い改善案があれば、コメントで教えて下さい。

