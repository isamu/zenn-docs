---
title: "Firestoreの設計ノウハウ"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Firebase, Firestore, GCP, JavaScript]
published: true
---

Firestoreでサービスを設計するときに事前に知っておくと良いポイントです。


# indexに使えそうなデータは、documentに入れておく

document pathで指定できるデータもdocumentにデータとして入れておくと、collectionGroupでデータを取得するときに役に立つ事がある。
また、当初は表示等で使わない関連するidもdocumentに入れておくと、後でspecが変更になった時にデータ取得に役に立つことがある。

特にdocumentのdocument.idをデータに入れておくとよい。collectionGroupに対しては、firestore.FieldPath.documentId()が使えないので、collectionGroupに対してid指定で多くのデータを取る場合にin Arrayのクエリが使えるようになる。
`addDoc` (firebase 8以下では`add()`)を使うと自動的にidがセット&保存されるので、
```
const id = doc(collection(db, "hoge")).id
```
firebase8以下では
```
const id = db.collection("hoge").doc().id
```
として、書き込み前にidをとってdataにidを追加してから`setDoc`でデータを保存するとよい。

# データを分割する

 - Firestoreでは基本的にdocument全体でしかread/write制御できない
    -  firestore.rulesを細かく書いていけば対策はできるが、テストや管理が大変
 - データ取得時にdocumentのデータが全て返ってくる
    - SQLのカラムを指定してデータをとる、という仕組みがない

ということで、aclの粒度の違うデータや、データ取得時に必要なデータごとにdocumentを分けたほうがよい。
ただ、細かく分けるとfirestore.rulesやデータの読み書きの処理が増えるので気をつける。

# フラグなどのデータの扱い
Firestoreでは、firestore.rulesで指定がない限り、document内のデータはwrite/create権限があるユーザは自由に追加/変更することができる。
 - プログラム側で想定していないkeyを使うことが可能
 - 型も変えられる
 - フロントでvalidationしたとしても直接書き込まれたらvalidationを回避可能
 - フロントエンドで書き込みを想定してないデータを勝手に書き込まれる可能性がある

対策としては

- ユーザが読み書きすると困るデータはdocumentを分けて write権限を与えない
- firestore.rulesでユーザが更新できないようにする
- firestore.rulesでvalidationを書く
- フロントでは書き込みできないようにしてfunctionsで更新する

など。

# collectionの命名

collectの名前はcollectionGroupを使う場合を想定して、uniqにつけると良い。secure, private, dataなどgeneralな名前はつけないほうが良い。
例えばuser以下のpublicというcollection(user/{userId}/public)と group以下にも同様のcollection(group/{groupId}/public)があった場合に、
publicをcollectionGroupで取得すると、userとgroupのデータの両方が対象となる。
それを避けるために、userPublicData, groupPublicDataなど、それぞれのcollection名をユニークにするとよい

firestoreのsubCollectionは親のcollectionがなくても作れるので、推測するとpathは単なる仮想的なデータのラベルで、collectionごとのflatなKVSのような実装になっていると思われる。そうするとcollection/subCollectionの縛られず、collection名に紐づく単純なKVSとして考えると良い（おそらくそういう実装になっているはず）

# firestoreの不得意なこと
- １つのdocumentに対して writeが1write/1s という制限があるので、カウンターなどは苦手
- in queryは一度に10件しかとれないので気を付ける


