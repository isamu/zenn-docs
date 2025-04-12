---
title: "Cloud Functions for FirebaseのonCallでSSE Streamingがサポートされました"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Firebase, Functions, LLM, Streaming]
published: true
publication_name: "singularity"
---


Firebase FunctionsのonCallでstreamingがサポートされました。
npmを最新にし(firebase-toolsも）、functions v2を使います。

`response.sendChunk`で途中のデータを送って、最後にreturnとしてまとめて結果を返します。

functionsのコードは以下。

```TypeScript
import { onCall } from "firebase-functions/v2/https";

export const sleep = async (milliseconds: number) => {
  return await new Promise((resolve) => setTimeout(resolve, milliseconds));
};

exports.streamingCall = onCall(
  {
    region: "asia-northeast1",
    timeoutSeconds: 10,
  },
  async (request, response) => {
    for await (const num of ["1", "2", "3", "4", "5"]) {
      response?.sendChunk({ num });
      await sleep(1000);
    }
    return {num: "12334"};
  });
```
と、`streamingCall`関数を定義します。

フロントではstreamingCallを呼びます。

```
import { getFunctions, httpsCallable } from "firebase/functions";

const functions = getFunctions(firebaseApp, "asia-northeast1");
const streamingFcuntion = httpsCallable(functions, "streamingCall");

(async () => {
  const { stream, data } = await streamingFcuntion.stream();
  for await (const chunk of stream) {
    console.log(chunk);
  }
  const allData = await data;
  console.log(allData);
})()
```

AIプログラミングで活用できます。