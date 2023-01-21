---
title: "Firebase Hosting + Functions でRegion指定がサポートされています"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Firebase, VueJs, JavaScript]
published: true
---


# 背景

長い間、Firebaseで動的コンテンツを扱う場合、FunctionsのRegionが指定できずus-central1しか使えないとい成約がありました。この為、動的コンテンツは米国サーバを使うことになり日本からアクセスする場合に、遅延が避けられない問題がありました。

# 設定方法

久しぶりにFirebaseの[document](https://firebase.google.com/docs/hosting/functions)を見ていたところ


> 指定されていない場合は、デフォルトでus-central1ます。ソース コードが利用できない場合、CLI はデプロイされた関数からリージョンを検出しようとします。関数が複数のリージョンにある場合、CLI では、 hosting構成でregionを指定する必要があります。


とあり、regionの指定方法が書かれてあったので、[おもちかえり.com](https://omochikaeri.com/)試してみました。

[firebase.json](https://github.com/Nakajima-Foundation/ownplate/blob/master/firebase.json)

```
  "hosting": [{
    "rewrites": [
      {
        "source": "/r/*",
        "function": "apiJP",
        "region": "asia-northeast1"
      },
    ]
  }]
```

[functions](https://github.com/Nakajima-Foundation/ownplate/blob/master/functions/src/wrappers/apiJP.ts)
```
export default functions
  .region("asia-northeast1")
  .runWith({
    memory: "1GB" as "1GB",
  })
  .https.onRequest(express.app);

```

こちらでデプロイしたところ、問題なく動作しました。応答速度も改善されました。

多くの記事が古く、region指定ができないと書いてあるので、今後要注意です。

