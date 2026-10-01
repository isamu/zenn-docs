---
title: "Firebase + Vue3でOGP"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Firebase, Vue.js, JavaScript]
published: true
publication_name: "singularity"
---

# Firebase + Vue3でOGP

OGPとは、Twitter, Facebook, SlackなどでURLを共有したときに、previewとして表示される文章や画像をページごとに指定する手法です。HTMLのHeaderのMetaタグに情報を記述して読み込ませます。
Firebase + Vueの場合は、主にSPAとしてクライアント側で処理するのでOGPとはあまり相性は良くありません。また、そのためだけにSSRを使うのはToo muchです.

そこで簡単にOGPを書き出す方法は２つ紹介します。

１つは、VueをビルドするときにOGPを含むhtmlを作成し、hostingで直接参照させる方法。
もう１つはVueでビルドしたhtmlをexpress(Webサーバ）で書き換えて表示させる方法です。この場合、メタタグ部分だけexpressで書き換えるので、OGP以外はSPAとしてそのまま実装できます。

## Vue + Firebase のページの表示方法

まずFirebase + Vue3のhtmlページの作成、表示方法を説明します。

`public/index.html`に元となるhtmlファイルが有ります

vue.config.jsでBuildされるページを指定しています

```vue.config.js
  pages: {
    index: {
      entry: 'src/main.ts',
      template: 'public/index.html',
      filename: 'index.html',
    },
```

Build時にこの設定を元に`public/index.html`から`dist/index.html`が作成されます

firebase.jsonで
```
  "hosting": [{
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
      ,,,,
    ]
  }]
```
DispatchのRuleを指定。
上記設定だけであれば、ブラウザからどのページを見てもdist/index.htmlが返されます。


## Vueのビルド時にOGPのページを生成する

先程の情報を元に、OGPを差し替えたページを表示させます。
`/ogp/`以下のページでindex.htmlとは異なるOGPを表示させることをゴールとします。

`public/index.html`をコピーして`public/ogp.html`を作成。

```
<meta property="og:site_name" content="OGPテストページ"/>
<meta property="og:title" content="OGPテストページ"/>
```
など、ogp.htmlに独自のOGPを追加します。

vue.config.jsにBuild設定を追加。

```
  pages: {
    index: {
      entry: 'src/main.ts',
      template: 'public/index.html',
      filename: 'index.html',
    },
    ogp: {
      entry: 'src/main.ts',
      template: 'public/ogp.html',
      filename: 'ogp.html',
    },
```

これでVueのBuild時に`public/opg.html`から`dist/ogp.html`も作成されるようになります。

firebase.jsonに追加
```
  "hosting": [{
    "rewrites": [
      {
        "source": "/ogp/**",
        "destination": "/ogp.html"
      },
      {
        "source": "**",
        "destination": "/index.html"
      }
      ,,,,
    ]
  }]
```

この設定でbuild + deployすると `/ogp/`ではじまる全てのページで`ogp.html`が返ってくるようになります。

ogp debuggerなどそれらのページをよみこんでOGPが正しく表示されるか確認できます。


## FunctionでExpressを使って動的にOGPページを作成する

FirebaseのHosting + Functionsを使うと動的にhtmlページを作成することができます。これを使ってSSRをすることも可能ですがOGP目的であれば、もっとシンプルにすることができます。
Vueでbuildしたindex.htmlを元に、OGP部分のみをFunctionsで書き換えてブラウザに返します。


流れとしては

- functionsデプロイ時にdist/index.htmlをfunctionsにコピーし、それをテンプレートとして使うようにする
- functionsでexpressを使ってwebサーバを実装。上記templateを読み込み、ogpなどを書き換える
- hostingの設定でこの機能を使うページをこのexpressサーバにdispatchする

です。

### functionsデプロイ時にテンプレートのコピー

Vueではbuildごとにjsなどのファイル名が変わるので、必ず毎回デプロイごとにVueでbuildしたhtmlをfunctionsにコピーする必要があります。この実装をした後にhostingだけ更新すると、webサーバが返すhtmlのjs/cssファイルの場所と、実際にVueがbuildしたファイル名が異なってエラーになるので注意です。

firebase.jsonに
```
  "functions": {
    "predeploy": [
      "cp dist/index.html functions/templates/index.html",
```

とdeploy前にコピーをしています。functions以下の場所は、実装に合わせて調整をしてください。
デプロイ前はこのファイルはないので、テスト時などは手動でコピーを忘れずに。

### expressの実装

functionbs + expressの設定や実装方法は省略します。他のドキュメント、もしくは、https://github.com/Nakajima-Foundation/ownplate/ の実装を参考にしてください。

ogpのページを表示する部分でurlを元にdbからデータをとってきて、データを元にOGPを作る、htmlのタグを差し替えます。
一般的なwebサーバと同様に、injectionや、セキュリティで必要なヘッダー、xssなどの対策は特に注意をしてください（例は一部割愛したコードなので穴があります）

```
const ogpPage = async (req: any, res: any) => {
  const { restaurantName, menuId } = req.params;
  const template_data = fs.readFileSync("./templates/index.html", {
    encoding: "utf8",
  });
  try {
    res.setHeader("X-Frame-Options", "deny");
    res.setHeader("X-Content-Type-Options", "nosniff");

    const restaurant = await db.doc(`restaurants/${restaurantName}`).get();
    const menuData = await getMenuData(restaurantName, menuId);

    const siteName = ownPlateConfig.siteName;
    const title = [menuData.name, restaurant_data.restaurantName].join(" / ")
    const description = menuData.description;
    const regexTitle = /<title.*title>/;

    const url = `https://${ownPlateConfig.hostName}/r/${escapeHtml(restaurantName)}/menus/${escapeHtml(menuId)}`

    const metas = [
      `<title>${escapeHtml(title)}</title>`,
      `<meta name="description" content="${escapeHtml(description)}"/>`,
      `<meta property="og:title" content="${escapeHtml(title)}" />`,
      `<meta property="og:site_name" content="${escapeHtml(siteName)}" />`,
      `<meta property="og:type" content="website" />`,
      `<meta property="og:url" content="${url}" />`,
      `<meta property="og:description" content="${escapeHtml(description)}" />`,
    ];
    const regexBody = /<div id="app">/;

    const bodyString = [
      '<div id="app">',
      '<h1 style="font-size: 50px;">',
      escapeHtml(title),
      "</h1>",
      '<span style="font-size: 30px;">',
      escapeHtml(restaurant_data.introduction),
      "</span>",
    ].join("\n");

    res.send(
      template_data
        .replace(/<meta[^>]*>/g, "")
        .replace(regexTitle, metas.join("\n"))
        .replace(regexBody, bodyString)
    );
  } catch (e) {
    res.send(template_data);
  }
};
```

## FirebaseでDispatch

静的なページのときと同様にfirebase.jsonに設定を追加して特定のページでこのサーバからOGP付きのhtmlを返します。

```
  "hosting": [{
    "rewrites": [
      {
        "source": "/r/*/menus/*",
        "function": "apiJP",
        "region": "asia-northeast1"
      },
```

この指定では `/r/foo/menus/bar`へリクエストがあった場合に先程のwebサーバからOGPを返します。


## まとめ

expressを使ったもののほうが、ページごとに個別に柔軟なOGPを返せる一方、staticなhtmlの場合はページごとの個別のカスタマイズは難しいがfunctionsの費用がかからずレスポンスも早いといった利点があります。

それぞれの特徴を元に柔軟に使い分けることができます。


