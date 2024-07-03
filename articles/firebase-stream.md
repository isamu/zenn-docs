---
title: "Cloud Functions for FirebaseでAIアプリの為にHTTPサーバストリーミングを使う"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Firebase, Functions, LLM, Streaming]
published: true
publication_name: "singularity"
---

## 最初に結論

- Functionsのv2を使う (Cloud Run)
- https.onRequestでhttp serverで実装(expressなど)
- Firebase Hosting経由ではなく、直接Functionsのendpointをたたく

以上で、HTTP Streamingが動きます

## はじめに

LLMやRAGを使ったAIサービスにおいて、HTTPでのテキストストリームは不可欠です。Firebaseでは、FunctionsでのStreamingが利用できませんでした。その変わりに、FunctionsでFirestoreにリアルタイムにデータを書き込み、クライアント側でFirestoreのonSnapshotを使いリアルタイムのデータ取得は可能ですが、FirestoreのR/Wが増えたり、またFirestore経由での遅延が問題になります。

Cloud Functions for FirebaseではHTTPのストリーミングはサポートされていませんでした。
V2でも試したところ、同様にStreamは動きませんでした。

しかし、調べたところCloud Runは2020年にHTTP サーバー ストリーミングに対応しているようで、Cloud Runを利用しているCloud Functions for Firebase V2ではストリーミングはできそうです。

https://cloud.google.com/blog/ja/products/serverless/cloud-run-now-supports-http-grpc-server-streaming?hl=ja

stackoverflowなどを調べてみると、Firebaseではbufferingをしているから結果がまとめて送られてくる、という情報がありました。試しにFirebaseを経由しない、すなわちFirebase hostingのendpointを使うのではなくCloud Functionsを直接叩いてみたところ、見事ストリーミングでデータを取得することができました。
くCloud Functions for Firebaseのエンドポイントは、Firebaseのwebコンソールなどで確認することができます。

## 実装

以下、実装のサンプルです。

```TypeScript

import * as express from "express";
import * as functions from "firebase-functions/v2";

const sleep = async (milliseconds: number) => {
  return await new Promise((resolve) => setTimeout(resolve, milliseconds));
};

const stream_response = async (req: express.Request, res: express.Response) => {
  res.setHeader("Content-Type", "text/event-stream;charset=utf-8");
  res.setHeader("Cache-Control", "no-cache, no-transform");
  res.setHeader("X-Accel-Buffering", "no");

  for await (const num of ["1", "2", "3", "4", "5"]) {
    res.write(num + "\n");
    await sleep(1000);
  }

  res.write("___END___");
  res.write("12345");
  return res.end();
};

const app = express();
app.use(express.json());
app.get("/v2_api/stream", stream_response);

export default functions.https.onRequest(
  {
    region: "asia-northeast1",
    timeoutSeconds: 600,
  },
  app,
);
```

`firebase-functions/v2` をimportしたときに

```
Unable to resolve path to module 'firebase-functions/v2'
```

のエラーがでる場合は、

```
yarn add -D eslint-import-resolver-typescript
```
を追加して、`.eslintrc`に

```
  settings: {
    "import/resolver": {
      typescript: {},
    },
  },
```

を追加してください。


## まとめ

Cloud Functions for FirebaseでHTTPのストリーミングに対応する方法を紹介しました。

この方法を使うことで、FirebaseのみでAIサービスを実装することが可能となります。

Firebase Vue3のスタートアップキットでもサンプルとして実装しているので、こちらも参考にどうぞ。

https://github.com/isamu/firebase-vue3-startup-kit/tree/main/functions


