---
title: "OpenAIをTypeScriptのStreamingで使う３つの方法"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ChatGPT, LLM, Tech, OpenAI, SlashGPT]
published: true
publication_name: "singularity"
---

OpenAIなどのStreamingをサポートしているAPIを使う場合の３つの例です。

## ジェネレータ関数で使うその１
for of でloopを回す。
Streamの結果はloop内でとれるが、ジェネレータ関数の結果はうけとれない。

```typescript
import "dotenv/config";
import OpenAI from "openai";

export async function *streamFunciton() {
  const openai = new OpenAI();

  const chatStream = await openai.beta.chat.completions.stream({
    messages: [{ role: "user", content: "日本の歴史について200文字でまとめてください" }],
    model: "gpt-3.5-turbo",
    stream: true,
  });

  for await (const message of chatStream) {
    const token = message.choices[0].delta.content;
    if (token) {
      yield token
    }
  }

  const chatCompletion = await chatStream.finalChatCompletion();
  return chatCompletion;
}


const main = async () => {
  for await (const token of streamFunciton()) {
    console.log(token);
  }
};

```

## ジェネレータ関数で使うその2

その１とジェネレータ関数と同じ。ただし、イテレーターは使わないでwhileで自分で回す
{ done: true, value: result}で最後の結果がとれる
個人的にはletが出てくるのが気になる。

```typescript
import "dotenv/config";
import OpenAI from "openai";

export async function *streamFunciton() {
  const openai = new OpenAI();

  const chatStream = await openai.beta.chat.completions.stream({
    messages: [{ role: "user", content: "日本の歴史について200文字でまとめてください" }],
    model: "gpt-3.5-turbo",
    stream: true,
  });

  for await (const message of chatStream) {
    const token = message.choices[0].delta.content;
    if (token) {
      yield token
    }
  }

  const chatCompletion = await chatStream.finalChatCompletion();
  return chatCompletion;
}


const main = async () => {
  const generator = streamFunciton()
  let result = await generator.next();
  while (!result.done) {
    const value = await result.value;
    console.log(value)
    result = await generator.next();
  }
  console.log(JSON.stringify(result.value));

};
```


## その３ callbackを使う

Streaimngデータはcallbackで受け取って、結果は関数の結果として受け取る。
Streaming不要な場合はcallbackを渡さなければ良いので、使い分けるときも楽。

```typescript
export async function streamCallbackFunciton(callback: (token: string) => void) {
  const openai = new OpenAI();

  const chatStream = await openai.beta.chat.completions.stream({
    messages: [{ role: "user", content: "日本の歴史について200文字でまとめてください" }],
    model: "gpt-3.5-turbo",
    stream: true,
  });

  for await (const message of chatStream) {
    const token = message.choices[0].delta.content;
    if (token) {
      callback(token);
    }
  }

  const chatCompletion = await chatStream.finalChatCompletion();
  return chatCompletion;
}

const main = async () => {
  const callbackResult = await streamCallbackFunciton((token) => {
    console.log(token);
  });
  console.log(callbackResult);
  
};

main();

```

そのまま動くコードはこちら。
https://github.com/isamu/samples/blob/master/typescript/streaming/generator.ts