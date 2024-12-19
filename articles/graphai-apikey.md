---
title: "GraphAIでのAPIKEYの扱いについて"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

:::message
GraphAI記事の一覧は[こちら](https://zenn.dev/singularity/articles/graphai-index)
:::


# APIキー管理の重要性

GraphAIに限らず、APIキーの適切な管理は非常に重要です。APIキーには **公開可能なもの** と **公開してはいけないもの** の2種類があります。

---

## 公開可能なAPIキー  
公開可能なAPIキーはWebサイトに埋め込んでも問題のないキーです。  
具体例として以下のものが挙げられます：  
- Google MapsやFirebaseの公開キー  
- StripeやShopifyの公開キー  

これらのキーは公開されていても安全な理由：  
- **ブラウザのReferrer制限** によりアクセス範囲を制限できる  
- 単なる **識別子** として使用され、データの書き込みなどの操作ができない  

---

## 公開してはいけないAPIキー  
一方で、**StripeやShopifyのSecret Key** など、他者に絶対に見せてはいけないAPIキーも存在します。  
これらが漏洩した場合、以下の対応が必要です：  
- サービスの即時停止  
- APIキーの無効化と新しいキーへの速やかな更新  

---

## GraphAIにおけるAPIキーの取り扱い  

GraphAIでは、OpenAIのAgentなど、APIキーが必要なAgentが存在します。また、GraphAIのAgentは **サーバ** と **ブラウザ** の両方で動作することが可能です（技術的制約がない限り）。  
このため、APIキーの **公開可否** と **GraphAI上での安全な取り扱い** を理解する必要があります。


### OpenAI Agentの例
GraphAIのOpenAI Agentでは、以下2つの方法でAPIキーを渡すことができます：

-  **環境変数** を利用してnpmパッケージに直接APIキーを渡す（サーバのみ）
-  **params** を指定してAPIキーを渡す（サーバ・ブラウザ両方で使用可能）

```Javascript
{ params: { apiKey: "xxx" } }
```

---

## 注意点: ブラウザ環境と環境変数  
VueやReactのCLI環境では、APIキーを環境変数に設定してブラウザに渡す方法が一般的ですが、以下に注意が必要です：  
- **環境変数はサーバサイドでのみ安全に扱える** ものです。  
- ブラウザ側にAPIキーを渡すと、ビルド後のJavaScriptにハードコードされてしまい、キーが露出するリスクがあります。  
  **これは、コードに直接APIキーを書き込むのと同じ危険性** があります。  

---

## 利用シナリオごとのAPIキー管理方法  

### 1. 運営側で用意したAPIキーを使い、不特定多数のユーザーが利用するサービス  
- **環境変数** を使い、サーバ側でAPIキーを管理します。  
- APIの利用料金は運営側が負担するため、ユーザーごとの利用制限を検討する必要があります。  

### 2. 不特定多数のユーザーが個別のAPIキーを使うWebサービス  
- 運営側はAPIキーを用意せず、ユーザーに個別のAPIキーを用意してもらいます。  
- ユーザーがブラウザ上でAPIキーを入力し、**Local Storage** などに保存します。  
- **ブラウザから直接OpenAIへアクセス** するため、運営側のサーバにはAPIキーを保存しません。  
  **ブラウザ側のセキュリティが保たれていれば安全です。**  

### 3. 個人向けソフトウェアや開発環境（React/Vueをブラウザで動作させる場合）  
- **params** に直接APIキーを埋め込む方法が利用できます。  
ただし、**APIキーがコードにハードコードされる** ことになるため、第三者に渡らない状況でのみ使用することが推奨されます。  

### 4. 個人向けデスクトップアプリケーション（Electronなど）  
- アプリの設定値として、ユーザー自身にAPIキーを入力してもらいます（シナリオ2と同様）。  
- **サービス運営側がAPIキーを提供する場合** は、ユーザーごとに異なるAPIキーを発行することが推奨されます。  
- APIキーが流出して他で転用される可能性も考慮し、適切な対策を講じる必要があります。  

---

## まとめ  
APIキー管理は、シナリオに応じて適切な方法で取り扱うことが不可欠です。特に、公開・非公開の区別や利用者ごとの責任範囲を明確にし、安全な運用を徹底しましょう。


---

# The Importance of API Key Management

Proper management of API keys is crucial not only for GraphAI but for any service. API keys can be classified into two types: **publicly embeddable keys** and **keys that must remain private**.

---

## Publicly Embeddable API Keys  
Publicly embeddable API keys can be safely embedded in websites.  
Examples include:  
- Google Maps and Firebase keys  
- Stripe and Shopify public keys  

These keys are considered safe to expose because:  
- Access can be restricted using **browser referrer settings**.  
- They act as **identifiers only** and do not allow operations like data modification.  

---

## API Keys That Must Remain Private  
On the other hand, **Stripe's secret key** or **Shopify's secret key** are examples of API keys that must never be exposed.  
If these keys are leaked, the following actions are required:  
- Immediate suspension of the service  
- Revocation of the compromised API key and issuance of a new one  

---

## Handling API Keys in GraphAI  

In GraphAI, certain agents (e.g., OpenAI agents) require API keys to function.  
Agents in GraphAI can operate on both the **server** and **browser** unless there are specific technical constraints (e.g., accessing the file system).  
This dual functionality requires a clear understanding of:  
1. Whether an API key can be exposed.  
2. How API keys are managed within GraphAI.

---

## Key Management Methods in OpenAI Agents  

GraphAI’s OpenAI agent supports the following two methods for passing API keys:  

1. Passing API keys via **environment variables** (server-side only).  
2. Passing API keys via **params** (usable on both server and browser).  

```Javascript
{ params: { apiKey: "xxx" } }
```
---

## Browser Environments and Environment Variables  
In CLI environments such as Vue or React, it is common to use environment variables to pass API keys to the browser. However, caution is required:  
- **Environment variables are safe only for server-side usage.**  
- When passing keys to the browser, they become embedded in the built JavaScript code.  
  **This is effectively the same as hardcoding the API key into the source code**, creating a significant exposure risk.  

---

## API Key Management Scenarios  

### 1. Service Using API Keys Provided by the Operator for Public Use  
- Use **environment variables** to manage the API key on the server side.  
- Since the service operator bears the API usage cost, consider implementing user-level usage limits.  

### 2. Public Web Services Where Users Provide Their Own API Keys  
- The operator does not provide API keys. Users must supply their own keys.  
- Users input their API key in the browser and store it in **Local Storage** or similar.  
- The browser directly accesses the API (e.g., OpenAI) without storing the API key on the server.  
  **This approach is safe as long as the browser's security is maintained.**  

### 3. Personal Software or Development Environments (Running React/Vue Locally)  
- Directly embed the API key in **params** or pass it via environment variables.  
- Note that embedding keys in code exposes them, so this method should only be used in isolated or personal environments.  

### 4. Desktop Applications for Personal Use (e.g., Electron)  
- Prompt users to input their API keys as application settings (similar to scenario 2).  
- **If the operator provides API keys**, generate unique keys for each user to mitigate unauthorized reuse.  
- Consider the risk of key leakage and implement proper safeguards where necessary.  

---

## Summary  
API key management must be tailored to each use case to ensure security.  
Clearly distinguish between keys that can be public and those that must remain private, and establish appropriate practices to manage and safeguard API keys effectively.
