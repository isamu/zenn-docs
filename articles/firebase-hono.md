---
title: "Firebase Functions でHonoをwebサーバとして使う"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Firebase, VueJs, JavaScript, hono]
published: true
publication_name: "singularity"
---


Firebase functionsでwebサーバ、apiサーバを立てる場合、expressを使うことが多いと思います。[Hono](https://hono.dev/)は軽量でパワフル、導入が用意なwebフレームワークです。expressのかわりのhonoを使ってみたいと思います。

honoは Cloudflare Workers,  AWS Lambda, Lambda@Edgeなどで使うためのadapterが用意されています。その[adapter](https://github.com/honojs/hono/tree/main/src/adapter)を参考にすれば他の環境で動作させることが可能となります。


今回は、lambda-edgeのhandler.tsを参考に、firebase functionsの対応をしました。基本はrequestを変換しRequestをhono.fetchに渡し、返ってきたresponseを環境に合わせて変換すればよさそうです。

出来上がったものは[こちら](https://github.com/isamu/firebase-vue3-startup-kit/pull/15/files)

```typescript
import { Hono } from "hono";
import { Request as FunctionRequest, Response } from "firebase-functions";

// eslint-disable-next-line @typescript-eslint/no-explicit-any
const handle = (app: Hono<any>) => {
  return async (req: FunctionRequest, resp: Response) => {
    const url = new URL(`${req.protocol}://${req.hostname}${req.url}`);

    const headers = new Headers()

    Object.keys(req.headers).forEach((k) => {
      headers.set(k, req.headers[k] as string);
    })
    const body = req.body;

    const newRequest = ["GET", "HEAD"].includes(req.method) ? new Request(url, {
      headers,
      method: req.method,
    }) : new Request(url, {
      headers,
      method: req.method,
      body,
    }) 
    const res = await app.fetch(newRequest);
    resp.json(await res.json());
  };
};

const app = new Hono();

app.get("/api/test", (c) => c.json({message: "Hono!"}));

export const server = handle(app);
```

デプロイ後webページにリクエストすると

```
{"message":"Hono!"}
```

と返ってきました。


[この記事によれば](https://zenn.dev/yusukebe/articles/0c7fed0949e6f7)

> Hono[炎] と名付けました。Cloud「flare」

とありますが、「Fire」baseとも相性が良いようです。