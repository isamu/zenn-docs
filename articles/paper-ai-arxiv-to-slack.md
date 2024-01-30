---
title: "Paper AI - arXivから興味のある論文を検索し、論文のabstractを要約しSlackへ投稿するAIボット"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Firebase, ChatGPT, JavaScript, Typescript, cloudfunctions]
published: true
publication_name: "singularity"
---

Paper AIは、arXivから興味のある論文を検索し、見つかった論文のabstractを要約しSlackへ投稿するAIボットです。FirebaseのCloud Functionsで動作します。
[Singularity Society](https://www.singularitysociety.org/)で開催している半年間にわたるオンラインのハッカソン、[BootCamp](https://note.com/singsoc/n/n81886b75b852)のサブプロジェクトとして作成されました。


# 動作の仕組み

FirebaseのCloud Funstions上で動作します

- Schedule functions (cron)でFunctionsが定期実行され、arXivを検索します。検索結果をFirestoreに保存します
  - [cron_arxiv_search](https://github.com/SingularitySociety/paper_ai/blob/main/functions/src/functions/arxiv/search.ts)
    - 定期的に起動され、arXivのAPIで新しい論文を取得する
    - 取得した論文のサマリーをFirestoreに保存する

```TypeScript
export const search_arxiv = async (db: admin.firestore.Firestore) => {
  try {
    const papers = await search_arxiv_papers();

    for await (const data of papers) {
      const id = data.id.replace("http://", "").replace(/\//g, "-");
      data.authors = data.authors.map((author) => author.join(","));

      const path = `/papers/${id}`;
      const currentDoc = (await db.doc(path).get()).data();
      if (!currentDoc) {
        await db.doc(path).set(data);
      }
    }
  } catch (error) {
    console.log(error);
  }
};
```
- Firestoreに論文情報が保存されると、Triggerで関数が呼ばれ、保存された論文情報を元に、GPTに日本語で要約させます。要約した結果をSlaskに投稿します。

  - [arxiv_create_doc](https://github.com/SingularitySociety/paper_ai/blob/main/functions/src/functions/arxiv/create_doc.ts)
    - Firestoreのcreateをトリガーに起動
    - 論文サマリーをLLMに投げて、必要な情報に変換
    - Slackへpushする

```TypeScript
export const createPaperEvent = async (
  db: firebase.firestore.Firestore,
  snap: firebase.firestore.QueryDocumentSnapshot,
  context: EventContext,
) => {
  const data = snap.data();
  const { paperId } = context.params;
  if (!data) {
    console.log("no data");
    return;
  }
  const summary_data = await call_llm(data);
  if (!summary_data) {
    console.log("no summary");
    return;
  }
  await db.doc(`summaries/${paperId}`).set({
    summary: summary_data,
    id: data.id,
    title: data.title,
  });
  const message = formatPushMessage(summary_data, data);
  await pushSlask(message);

  return;
};

export const call_llm = async (data: firebase.firestore.DocumentData) => {
  const text = `title: ${data.title}\nbody: ${data.summary}"`;

  const res = await call_slashgpt(text);
  if (res.result) {
    return res.function_result;
  }
  return null;
};

```

# 設定

### Firebaseのプロジェクト設定
  - [Firebaseのページ](https://firebase.google.com/?hl=ja)でprojectを作ります。
  - Functionsを使うには課金設定が必要なのでBlazeプランに変更し、課金を有効にします。従量制なので、ほとんどコストはかかりません
  - Functionsを有効にします。

### functions:secrets にOpenAIのsecret keyとSlack bot用のtoken/channnelをセットする

[Slack apiのページ](https://api.slack.com/apps)のCreate New APPでアプリを作成します。
![](/images/paper-ai-arxiv-to-slack/11.png)
作成したアプリのOAuth tokenをコピーし、SLACKTOKENとして登録します。
![](/images/paper-ai-arxiv-to-slack/12.png)

SLACKCHANNELはbotが要約を出力するチャンネル。このチャンネルにbotを招待します。

OPENAI_API_KEYはopenaiのapikeyを指定します。

Secret Manager APIの設定を要求された場合は、それに従って有効にする。

```
firebase functions:secrets:set SLACKTOKEN --project=default
firebase functions:secrets:set SLACKCHANNEL --project=default
firebase functions:secrets:set OPENAI_API_KEY --project=default
```


- 通常のfunctionとしてデプロイする

```
firebase deploy --only functions --project=default
```

正しく設定できていると、以下のようにSlackのチャンネルにpaper aiのbotが要約を投稿するようになります。

![](/images/paper-ai-arxiv-to-slack/13.png)

初回実行時はOpenAIのAPIへのアクセスが大量になり、上限に引っかかってエラーになることがあります。２回目以降は差分となるので、エラーはほぼ出なくなります。

# カスタマイズ

### 検索とSlack投稿formatのカスタマイズ

初期の状態では、arXivにはLLMのキーワードで検索をしています。search_arxiv_papers関数を変更することで、検索内容を変更することができます。

Slackの投稿時に、formatPushMessageで整形をしています。

この２つの関数は[lib/utils.ts](https://github.com/SingularitySociety/paper_ai/blob/main/functions/src/lib/utils.ts)にあります。

```TypeScript
export const search_arxiv_papers = async () => {
  const papers = await search({
    searchQueryParams: [
      {
        include: [{ name: "LLM" }],
      },
    ],
    sortBy: "lastUpdatedDate",
    sortOrder: "descending",
    start: 0,
    maxResults: 100,
  });
  return papers;
};

export const formatPushMessage = (
  summary_data: LLMSummary,
  data: firebase.firestore.DocumentData,
) => {
  const { title, keywords, issues, methods, results } = summary_data;
  const base_title = data.title.replaceAll("\n", "");
  return [
    "------------------",
    `${base_title} (${data.id})`,
    "------------------",
    "",
    [
      `📚内容: ${title}`,
      `🔑キーワード: ${keywords}`,
      `❓問題点: ${issues}`,
      `⚒️手法:️ ${methods}`,
      `⭐結果: ${results}`,
    ].join("\n\n"),
  ].join("\n");
};
```

### プロンプトのカスタマイズ

Paper AIでは、LLMの処理に[SlashGPT](https://github.com/snakajima/SlashGPT)の[Typescript版](https://github.com/isamu/slashgpt-web)を利用しています。プロンプトは[マニフェスト](https://github.com/SingularitySociety/paper_ai/blob/main/functions/slashgpt/manifests/main/paper.yml)に書かれています。また、要約の結果を整形した状態で受け取るために、Function callingを使っています。Function callingの定義は[こちら](https://github.com/SingularitySociety/paper_ai/blob/main/functions/slashgpt/resources/functions/paper.json)。
これらをカスタマイズすることにより、要約の精度の向上や他の項目の取得が可能になります。


# 動作テスト

Paper AIを実際に動かすのは、FirebaseのFirestore, Functionsが必要となり、Trigger処理をしているので開発時に環境設定が複雑になります。

そのためlocalでテストするために、各それぞれの機能を関数にして、Firebaseに依存しない形でテスト可能にしています。

以下のスクリプトで、単体の動作が可能となる。

### 論文を検索するスクリプト

```
 npx ts-node tests/arxiv.ts
```

### GPTに要約させるスクリプト

```
 OPENAI_API_KEY=xxxx npx ts-node tests/gpt.ts
```

- OpenAIのapi keyが必要
  - このkeyはレポジトリには絶対に登録しないこと
- tests/sample.tsのarXivのapiレスポンスのデータを読み込みLLMに問い合わせる
  - プロンプトなどを変更する場合は、slashgpt/manifests/main/paper.yml を編集
  - SlashGPTを使っているので、SlashGPTのみでpaper.ymlの動作検証可能
  - sample.tsを差し替えて別の論文でも検証可能

### Slackに投稿するデータを変換するスクリプト

- データベースのデータ(arXivの返却値)とLLMのレスポンスから、Slackへの投稿メッセージに変換する関数
- 投稿時の変換をテストして、投稿メッセージを変えることができる
- LLMのレスポンスが変わった場合（function calling)、こちらのテストデータも変更する必要あり

```
 npx ts-node tests/format_data.ts
```

# TODO

以下TODOです。パッチ大歓迎です。

- llmの処理を失敗したときのretry
  - 更新日の管理やllmの結果をdbに入れる
- Slackの対話による論文LLMとの対話
  - PDFをダウンロードしてアブストだけでなくフルペーパーの要約
  - 画像サポート
- web ui
  - Firebaseに入れている既存のデータをそのまま表示させる
  - LLMとの対話を可能にする＋そのログの表示
  - 検索キーワードの追加、削除、管理
  - ユーザごとに特定のキーワードをいれておくと、検索結果にそのキーワードが含まれている場合に通知やメンションさせる


