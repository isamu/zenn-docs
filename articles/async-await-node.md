---
title: "Nodeの非同期処理をasync/awaitで統一する"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Node, Async]
published: true
publication_name: "singularity"
---

Node.jsのコードを読んでいると、setTimeout, Promise, callbackが入り乱れたコードに出会うことがあります。

Node.jsの特徴として非同期処理が書きやすいのですが、書く方法も様々な方法があり、意図した書きか、それともなにかのコピペなのか判断がつかないケースがあります。
意図して非同期にしていない場合は、async/awaitを使って順番に実行されるように書いた方が、思わぬバグに遭遇したり、他の人が読むときに誤解がない事が多いです。

setTimeout, Promise, callbackをasync/awaitに書き換える方法をまとめます。

## setTimeout

```typescript
setTimeout(() => {
 foo();
}, 2000)
```

setTimeoutをラップしてsleep関数を作ります。

```typescript
const sleep = async (milliseconds: number) => {
  return await new Promise((resolve) => setTimeout(resolve, milliseconds));
};
```

async/awaitで使います。

```typescript
await sleep(2000);
await foo();
```

非同期で実行したい場合は

```typescript
(async () => {
  await sleep(2000);
  await foo();
})()
```
と、これ自体をasyncで動作させます。

## Promise

```typescript
foo().then(aa => { bar(aa) });
```

fooを非同期で実行したいというとき以外は、

```typescript
const aa = await foo();
await bar(aa);
```

と、fooを待つと良いです。

## callback

callbackはpromiseでラップしてasync/awaitにします。

callbackの例
```typescript
foo((res) => {
  bar(res);
});
```


Promiseでcallbackを包みます。

```typescript
const foo_promise = () => {
  return new Promise(
    (resolve, reject) => {
      foo(resolve);
    }
  );
};
```

await/asyncで呼び出します。

```typescript
(async () => {
  const res = await foo_promise();
  await bar(res);
})();
```

## まとめ

以上で、非同期処理をasync/awaitで統一できます。
async/awaitで統一すればNode.jsに慣れて無い人でも動作を把握しやすくなります。
