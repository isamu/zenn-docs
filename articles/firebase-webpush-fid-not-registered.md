---
title: "Firebase Web PushのFID送信が失敗する理由"
emoji: "🔔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase", "fcm", "webpush", "typescript", "pwa"]
published: true
publication_name: "singularity"
---

## はじめに

PWA に Web Push を実装していて、こんな状況にハマりました。

- クライアントの **FID 登録は成功**している（HTTP 200、有効な push endpoint も発行済み）
- なのに Admin SDK で **その FID に送ると `messaging/installation-id-not-registered`** が返る
- しかし同じ端末の **registration token に送ると成功**する

「登録は通ってるのに送れない」という一見矛盾した状況で、原因は意外なところにありました。**Cloud Functions の `httpsCallable()` が、裏で `messaging.getToken()` を呼んで FID を無効化していた**のです。

この記事では、Firebase の新しい **FID (Firebase Installation ID) ベースの Web Push** で起きるこの罠と、回避策を共有します。環境は `firebase@12.16` / `firebase-admin@14.1` です。

## 前提：Firebase の Web Push が FID ベースに移行した

まず背景から。最近の Firebase では Web Push の仕組みが変わっています。

- **クライアント (`firebase/messaging`)**: `messaging@0.13.0`（`firebase@12.14.0`）以降、`getToken()` が **deprecated** になり、`register()` + `onRegistered()` で **FID** を取得する方式が推奨に。
- **サーバ (`firebase-admin`)**: `14.1` で `token` 指定が deprecated、`fid`/`fids` 指定が推奨に。`sendEachForMulticast({ fids })` は `14.0` で追加された正式な機能です。

なので「最新のまま・非推奨 API を使わない」で組むと、自然とこうなります。

```ts
// クライアント: onRegistered を先に登録してから register() を呼ぶ
//（register() は onRegistered ハンドラ未設定だと throw する）
import { getMessaging, onRegistered, register } from "firebase/messaging";

const messaging = getMessaging();
onRegistered(messaging, (fid) => {
  // この FID をサーバへ送って保存する
  saveFidToServer(fid);
});
await register(messaging, { vapidKey, serviceWorkerRegistration });
```

```ts
// サーバ (Cloud Functions): その uid の全 FID へ送信
await getMessaging().sendEachForMulticast({
  fids, // ← token ではなく fids
  notification: { title, body },
});
```

これで動く……はずでした。

## 症状：登録は 200 成功、でも FID 送信だけ失敗

`onRegistered` で FID は取れる。サーバに保存もできる。ところが送信すると:

```
code: "messaging/installation-id-not-registered"
```

最初は「Admin SDK の FID 送信がまだバックエンド未対応なのでは？」と疑いましたが、これは**間違い**でした。`sendEachForMulticast({ fids })` は Admin 14.x の正式機能です。

決定的な切り分けとして、**同じ端末の registration token に FCM HTTP v1 API で直接送ってみる**と、あっさり成功します。

```bash
curl -s -X POST "https://fcm.googleapis.com/v1/projects/<project>/messages:send" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"message":{"token":"<fid>:APA91b...","notification":{"title":"t","body":"b"}}}'
# => {"name":"projects/<project>/messages/xxxx"}  ← 成功
```

つまり **登録自体は正常で、「FID を指定した送信」だけが壊れている**。トークンなら届く。ということは、送信経路ではなく**登録状態の問題**です。

## 切り分け：`fcmregistrations` への POST が「2回」走っている

`fetch` をラップして `fcmregistrations.googleapis.com`（FID 登録 API）への通信を覗くと、Enable の一連の流れで **登録 POST が 2 回** 走っていました。

1. **1回目**: `register()` による FID 登録。レスポンスは `registrations/{fid}`。
2. **2回目**: なぜか **`token` フィールドを含むレスポンス**（`registrations/{fid}:APA91b...`）。これは**旧形式（getToken 相当）の登録**です。

この 2 回目が犯人でした。**FID 登録の後にもう一度、旧形式の登録が走って FID を上書き・無効化**していたのです。

## 原因：`httpsCallable()` が内部で `messaging.getToken()` を呼ぶ

では 2 回目の登録は誰が呼んでいるのか。FID をサーバへ送るのに **`httpsCallable()`（`firebase/functions`）** を使っていたのですが、その内部にありました。

`@firebase/functions` のソースを見ると、callable は**呼び出しのたびに** messaging トークンを取得し、`Firebase-Instance-ID-Token` ヘッダとして付与します。

```js
// @firebase/functions/dist/index.cjs.js（抜粋）
async getMessagingToken() {
  // ...
  return await this.messaging.getToken();   // ← 旧形式の getToken を呼ぶ
}
// ...
const messagingToken = await this.getMessagingToken();
// ...
if (context.messagingToken) {
  headers['Firebase-Instance-ID-Token'] = context.messagingToken; // 毎回付与
}
```

流れを整理するとこうです。

```
register()             → FID 登録（正常）
  ↓
httpsCallable(登録関数)  → 内部で messaging.getToken()
  ↓
                         旧形式の FCM 登録が作られ、FID が送信ターゲットとして無効化される
  ↓
sendEachForMulticast({ fids }) → installation-id-not-registered
```

同時に発行される registration token は有効なので、「トークン送信だけは成功する」という症状とも一致します。Firebase JS SDK の 12.15〜12.16 あたりの FID 移行に、この不整合が残っているようです。

## 対策：callable を `httpsCallable` ではなく「生 fetch」で叩く

犯人が `httpsCallable()` の内部 `getToken()` だと分かれば、対策はシンプルです。**`firebase/functions` を経由せず、生の `fetch` で onCall 関数を叩けばよい**。認証は Firebase Auth の ID トークンを `Authorization: Bearer` で自分で付けます。onCall 関数はこのヘッダから `request.auth` を解決するので、**サーバ側は onCall のまま変更不要**です。

```ts
import { getIdToken } from "firebase/auth";
import { auth } from "./firebase";

const REGION = "asia-northeast1";
const PROJECT_ID = "your-project-id";

// httpsCallable は内部で messaging.getToken() を呼び、FID を壊す。
// Push 経路の関数呼び出しはこれで代替する。
export const callFunctionRaw = async <T>(name: string, data: unknown): Promise<T> => {
  const user = auth.currentUser;
  if (!user) throw new Error("not signed in");
  const idToken = await getIdToken(user);

  const res = await fetch(`https://${REGION}-${PROJECT_ID}.cloudfunctions.net/${name}`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${idToken}`, // ← callable の認証はこれで足りる
    },
    body: JSON.stringify({ data }), // callable プロトコルは { data } でラップ
  });
  if (!res.ok) throw new Error(`${name} failed: ${res.status}`);

  const payload: { result: T } = await res.json(); // 返り値は { result } に入る
  return payload.result;
};
```

これで FID 登録の後に `getToken()` が走らなくなり、`sendEachForMulticast({ fids })` が正しく届くようになりました。実機（iOS 16.4+ ホーム追加 PWA / Android Chrome）で `sent 1/1` ＋通知到達を確認できました。

:::message
onCall を生 fetch で叩くと CORS はどうなる？という点ですが、onCall（gen2）はブラウザからの callable 呼び出し用に CORS を自前で処理するため、別オリジン（localhost や hosting ドメイン）からの `fetch` でも通ります。
:::

## ついでにハマった小ネタ

Web Push を最後まで通すのに引っかかった点も、まとめて置いておきます。

- **Admin の default app 初期化（gen2）**: callable 内で `getFirestore()` が `The default Firebase app does not exist`。gen2 ランタイムが名前付きアプリを登録するため `if (!getApps().length)` が誤判定します。**モジュール読み込み時**に初期化しつつ、判定を `getApps().some(a => a.name === "[DEFAULT]")` と厳密化して解決。
- **前面タブでは通知が出ない**: FCM Web は「その origin のタブが 1 つでもフォーカス中だと `onBackgroundMessage`(SW) ではなく `onMessage`(前面) に回す」ので、`onMessage` → `showNotification` を配線しないと無表示になります。同一ブラウザで送信元タブを見ながらテストすると紛らわしいので、**最終確認は実機**（PWA を閉じた状態）で。
- **iOS の制約**: Web Push は **iOS 16.4+ かつ「ホーム画面に追加」した PWA** のみ。通常の Safari タブでは不可。通知許可は**タップ操作が起点**でないと出せません。
- **`fcmregistrations.googleapis.com` の有効化**: FID 登録にはこの API を有効化しておく必要があります。
- **macOS で通知が出ない**: `Notification.permission === "granted"` でも、システム設定 → 通知 → ブラウザが OFF / 集中モードだと表示されません。`registration.showNotification("test", { body: "hi" })` を直接実行して切り分け。

## まとめ

- Firebase の Web Push は **FID ベース**（`register`/`onRegistered` + Admin `{ fids }`）に移行済み。
- **`httpsCallable()` は呼び出しのたびに内部で `messaging.getToken()` を呼び、旧形式登録を作って FID を無効化する**（`messaging/installation-id-not-registered` の隠れた原因）。
- Push 経路の関数呼び出しを **生 `fetch`（Auth ID トークン付き）** にすれば `getToken()` を回避でき、FID のまま送信できる。

「登録は 200 なのに送れない」「トークンなら届く」に遭遇したら、`fcmregistrations` への 2 回目の POST と `httpsCallable` を疑ってみてください。

## 参考

- [Get started with Firebase Cloud Messaging in Web apps](https://firebase.google.com/docs/cloud-messaging/web/get-started)
- [Best practices for FCM registration management](https://firebase.google.com/docs/cloud-messaging/manage-tokens)
