---
title: "GraphAI 0.5.9"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

GraphAI 0.5.9がリリースされました。
主な変更点は以下。

## 深いnestedのnamed inputsサポート
```typescript
inputs: {a: {b: {c: [":data"], d: ":data2"}, e: ":data3" } }
```

## inputsでfunctionを追加
```typescript
 inputs: { length: ":arrayData.length()", flatArray: ":arrayData.flat()" }
```

サポートするのは
- array:
  - length()
  - flat()
- object:
  - keys()
  - values()
- string:
  - jsonParse()
  
## inputsでtemplateを書くことができる

```typescript
 inputs: { template: "this is a ${:item}. hello ${:userName} "}
```

## 補足
functionは

```typescript
:objectDataString.jsonParse().values().flat().length()
```

のようにmethod chainできます。

templateもprops, function使えます

```typescript
inputs: { template: "this is ${:item.color}. hello ${:user.name}. array length is ${:arrayData.length()}"}
```

