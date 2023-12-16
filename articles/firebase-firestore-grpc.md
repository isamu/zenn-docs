---
title: "Cloud Functions for Firebaseは遅い？その２"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Firebase, Functions, Firestore, gRPC]
published: true
publication_name: "singularity"
---

[前回の記事](https://zenn.dev/singularity/articles/firebase-functions-performance)で調査したように、Firebase Functionsからfirebase-adminを使ってFirebaseの他サービス(FirestoreやAuthentication)を呼び出す場合、コールドスタート時にgRPCの初期化が走りその処理にかなり時間がかかかる問題があります。特に対策をしていないFunctionsの関数を使うと、この条件のときにFunctionsの応答に4秒くらいかかるケースが有りました。

この問題は2020年ころには課題として認識されていましたが、長い間対策方法はない状況でした。[この問題について議論しているチケット](https://issuetracker.google.com/issues/158014637?pli=1) ではgRPCではなくREST APIを使って他サービスを呼び出す方法が回避策として提案されていました。2022年にこの方法が正式な対策として取り込まれれたようです。


firebase-adminの[document](https://firebase.google.com/docs/reference/admin/node/firebase-admin.firestore.firestoresettings.md?hl=ja#firestoresettings_interface)を見ると、この対策を有効にするには、Firestoreの初期化時にpreferRestをoptionとして渡す必要があります。


Firestoreの初期化の説明部分に、特に説明はなく[シレっとoptionとして](https://firebase.google.com/docs/reference/admin/node/firebase-admin.firestore?hl=ja#initializefirestore)追加されていました。

firebase-adminの[Release notes](https://firebase.google.com/support/release-notes/admin/node#version_1140_-_15_december_2022)を見ると、version 11.4.0でこの対策が入ったようです。これより古いバージョンを使っている場合は、最新にupdateしてこのoptionを有効にすると幸せになれそうです。


改めてpreferRestの説明を見ると

>FirestoreSettings.preferRest
>
>可能な場合は HTTP/1.1 REST トランスポートを使用します。
>preferRest 、gRPC を必要とするメソッドが呼び出されるまで、HTTP/1.1 REST トランスポートの使用を強制します。メソッドで gRPC が必要な場合、この Firestore クライアントは依存する gRPC ライブラリをロードし、それ以降のすべての通信に gRPC トランスポートを使用します。現在、gRPC を必要とする唯一の操作は、 onSnapshot()を使用してスナップショット リスナーを作成することです。

とあります。FunctionsでonSnapshotを使うことは少ないと思うので、このoptionを使うことで問題は解決と言えそうです。

