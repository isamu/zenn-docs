---
title: "Firebaseとvueで爆速でアプリを作るfirebase-vue3-startup-kit"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Firebase, Functions, Firestore, Vue3]
published: true
publication_name: "singularity"
---

FirebaseとVue3で爆速でwebサイトを作るfirebase-vue3-startup-kitの紹介です。
FirebaseはGoogleが提供するモバイルおよびウェブ開発プラットフォームです。Web上で簡単に設定するだけで、hosting, 認証、databaseなどがすぐに、簡単に使えます。特にdatabaseのfirestoreはサーバいらずでフロントからセキュアに、簡単にデータの読み書きでき、websockのコードを書くことなしにリアルタイムのデータ変更を受けることができます（GraphQLのSubscription相当）
Vueのreactiveな仕様ととても相性がよいので、この２つの組み合わせてwebを作ると爆速で、かつリアルタイムにデータが反映するサイトを構築できます。



従来、FirebaseとVue3で環境を作る場合、

```
yarn create vite my-vue-app --template vue
```
でvueを初期化した後に、

```
firebase init 
```
でfirebaseを初期化し、それぞれの設定や連携するためのコードを追加していく必要があります。

毎回、同じような作業をするのは時間のロスなので予めそれらの初期化と設定をしたレポジトリを作りました。（実際は２年前に作り、継続してメンテしています）

https://github.com/isamu/firebase-vue3-startup-kit

使い方は簡単。

## Vue設定

1. レポジトリをforkするか、Use this templateでレポジトリを作る
2. localにcloneする
```
git clone {your repository url}
```
3. yarn install
```
cd {your repository dirctory}
yarn install
```

4. functionsを使う場合はfunctionsディレクトリーでyarn install
```
cd functions
yarn install
```

# Firebaseの設定

1. Firebaseのコンソールで新しいプロジェクトを作る
2. web appを追加する
3. web appの設定をコピーしレポジトリにコピーする(src/config/project.ts)
4. Functionsなど課金が必要なサービスを使う場合はプラン変更
5. 必要なfirebaseの設定を追加する
 - Hosting
   - webのhosting. Vueをアップロードしてブラウザから使う場合に必要。
   - Funcitonsでwebサーバを使う場合も必要(Functionsの関数のみの場合は不要)
 - Authentication
   - ユーザ認証。email/password, Google認証などのOAuth連携、電話認証など。
 - Firestore Database
   - データベース
 - Storage
   - ファイルの管理
 - Functions
   - バックエンドサーバ
 - App Check
   - FirestoreやFunctionsのセキュリティの仕組み
   - 本番環境では有効にして、設定を追加したほうが良い

6. firebaseのプロジェクト名をレポジトリの.firebasercのfir-vue-startup-kit部分にコピー
7. Firebaseのcliをインストールしてない場合はインストールする
```yarn global add firebase-tools```

以上で必要な設定は完了

```
yarn run serve
```

するとlocalでvueが立ち上がります。


firebaseのデプロイは

```
yarn run build
firebase deploy --project=default 
```

で出来ます。
