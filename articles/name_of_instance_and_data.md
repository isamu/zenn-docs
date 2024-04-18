---
title: "データとインスタンスの変数名"
emoji: "🚀"
type: "idea"
topics: [Node.js, naming, Tech]
published: true
publication_name: "singularity"
---

Node.jsでコードを書いていると、あるデータを扱うクラスを作ることがあります。

```typescript
type HogeData = {
 ....
}
```

```typescript
class Hoge {
 ....
}
```

このときに、データやクラスのインスタンスの名前を適当につけると、後々わかりづらくなることがあります。大きくわけると、単数で扱う、arrayで持つ、ハッシュ(Object)の値として持つの３パターンあります。一般的にはarrayは複数形にすれば良いのですが、データを格納する変数名はは~~Dataと命名されることあります。Dataは複数形なので、は~~Dataを複数にはできません。
この例でもHogeDataと、データにはDataと命名しています。Datasだと少し違和感があります。

Dataの複数の場合はDataSetとし、objectの場合はDictonaryとすればスッキリします。

```typescript
hogeData: HogeData
hogeDataSet: HogeData[]
hogeDataDictonary: Record<string, HogeData>
```

```typescript
hoge: Hoge
hoges: Hoge[]
hogeDictonary: Record<string, Hoge>
```

