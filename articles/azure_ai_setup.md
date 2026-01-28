---
title: "Azure OpenAI / Azure Speech Service セットアップガイド"
emoji: "☁️"
type: "tech"
topics: ["azure", "openai", "tts", "typescript"]
published: true
publication_name: "singularity"
---

# Azure OpenAI / Azure Speech Service セットアップガイド

Node.js/TypeScriptでAzure OpenAI（Chat、画像生成、TTS）およびAzure Speech Serviceを使用するための設定ガイドです。

## 概要

| サービス | 用途 | リージョン制限 |
|---------|------|---------------|
| Azure OpenAI (Chat/LLM) | テキスト生成 | 多くのリージョンで利用可能 |
| Azure OpenAI (Image) | 画像生成 (DALL-E) | 多くのリージョンで利用可能 |
| Azure OpenAI (TTS) | 音声合成 | **North Central US** または **Sweden Central** のみ |
| Azure Speech Service | 音声合成 (代替) | 多くのリージョンで利用可能 |

## 前提条件

- Azure サブスクリプション
- Azure CLI (`az`) インストール済み
- Node.js パッケージ: `openai` v6.x 以上

```bash
brew install azure-cli
az login
```

## リソース作成

### 1. Azure OpenAI リソース (Chat/Image用)

```bash
# リソースグループ作成（既存の場合はスキップ）
az group create --name <resource-group> --location eastus

# Azure OpenAI リソース作成
az cognitiveservices account create \
  --name <resource-name> \
  --resource-group <resource-group> \
  --kind OpenAI \
  --sku S0 \
  --location eastus \
  --yes

# カスタムドメイン設定（API アクセスに必要）
az cognitiveservices account update \
  --name <resource-name> \
  --resource-group <resource-group> \
  --custom-domain <resource-name>
```

### 2. Azure OpenAI リソース (TTS用)

**重要**: TTSは `northcentralus` または `swedencentral` でのみ利用可能

```bash
az cognitiveservices account create \
  --name <tts-resource-name> \
  --resource-group <resource-group> \
  --kind OpenAI \
  --sku S0 \
  --location northcentralus \
  --yes

az cognitiveservices account update \
  --name <tts-resource-name> \
  --resource-group <resource-group> \
  --custom-domain <tts-resource-name>
```

### 3. Azure Speech Service (TTS代替)

Azure OpenAI TTSが使えないリージョンでの代替:

```bash
az cognitiveservices account create \
  --name <speech-resource-name> \
  --resource-group <resource-group> \
  --kind SpeechServices \
  --sku F0 \
  --location eastus \
  --yes
```

## モデルのデプロイ

### 利用可能なモデル確認

```bash
# LLMモデル
az cognitiveservices model list --location eastus -o json | \
  jq '[.[] | .model | {name, version}] | unique_by(.name) | .[] | select(.name | test("gpt|o1|o3"; "i"))'

# TTSモデル（North Central US）
az cognitiveservices model list --location northcentralus -o json | \
  jq '[.[] | .model | {name, version}] | unique_by(.name) | .[] | select(.name | test("tts"; "i"))'

# 画像生成モデル
az cognitiveservices model list --location eastus -o json | \
  jq '[.[] | .model | {name, version}] | unique_by(.name) | .[] | select(.name | test("dall"; "i"))'
```

### モデルデプロイ

```bash
# Chat モデル (例: gpt-4o)
az cognitiveservices account deployment create \
  --name <resource-name> \
  --resource-group <resource-group> \
  --deployment-name gpt-4o \
  --model-name gpt-4o \
  --model-version "2024-11-20" \
  --model-format OpenAI \
  --sku-capacity 1 \
  --sku-name Standard

# 画像生成モデル (DALL-E 3)
az cognitiveservices account deployment create \
  --name <resource-name> \
  --resource-group <resource-group> \
  --deployment-name dall-e-3 \
  --model-name dall-e-3 \
  --model-version "3.0" \
  --model-format OpenAI \
  --sku-capacity 1 \
  --sku-name Standard

# TTSモデル（North Central US リソースで）
az cognitiveservices account deployment create \
  --name <tts-resource-name> \
  --resource-group <resource-group> \
  --deployment-name tts \
  --model-name tts \
  --model-version "001" \
  --model-format OpenAI \
  --sku-capacity 1 \
  --sku-name Standard
```

## APIキーの取得

```bash
# Azure OpenAI
az cognitiveservices account keys list \
  --name <resource-name> \
  --resource-group <resource-group> \
  --query "key1" -o tsv

# エンドポイント確認
az cognitiveservices account show \
  --name <resource-name> \
  --resource-group <resource-group> \
  --query "properties.endpoint" -o tsv
```

## 環境変数の設定

`.env.azure` ファイルを作成:

```bash
# Azure OpenAI (Chat/Image) - East US
AZURE_OPENAI_KEY=<your-api-key>
AZURE_OPENAI_ENDPOINT=https://<resource-name>.openai.azure.com/

# Azure OpenAI TTS - North Central US
AZURE_OPENAI_TTS_KEY=<your-tts-api-key>
AZURE_OPENAI_TTS_ENDPOINT=https://<tts-resource-name>.openai.azure.com/
AZURE_OPENAI_TTS_DEPLOYMENT=tts

# Azure Speech Service (代替TTS)
AZURE_SPEECH_KEY=<your-speech-key>
AZURE_SPEECH_REGION=eastus
```

## コード例

### OpenAI SDK バージョンについて

`openai` パッケージ v6.x では `AzureOpenAI` クラスが標準で提供されています。

```typescript
// v6.x ではこれだけでOK
import { AzureOpenAI } from "openai";
```

**注意**: 古いバージョン (v3/v4) で使用されていた `import "openai/shims/node"` は v6.x では不要です（エクスポートされていません）。

### Azure OpenAI TTS

```typescript
import { AzureOpenAI } from "openai";

const client = new AzureOpenAI({
  apiKey: process.env.AZURE_OPENAI_TTS_KEY,
  endpoint: process.env.AZURE_OPENAI_TTS_ENDPOINT,
  apiVersion: "2025-04-01-preview",
});

const response = await client.audio.speech.create({
  model: process.env.AZURE_OPENAI_TTS_DEPLOYMENT || "tts",
  voice: "alloy",  // alloy, ash, coral, echo, fable, onyx, nova, sage, shimmer
  input: "Hello, world!",
});

const buffer = Buffer.from(await response.arrayBuffer());
```

### Azure Speech Service TTS

```typescript
import * as sdk from "microsoft-cognitiveservices-speech-sdk";

const speechConfig = sdk.SpeechConfig.fromSubscription(
  process.env.AZURE_SPEECH_KEY,
  process.env.AZURE_SPEECH_REGION
);
speechConfig.speechSynthesisVoiceName = "ja-JP-NanamiNeural";

const synthesizer = new sdk.SpeechSynthesizer(speechConfig);

synthesizer.speakTextAsync(
  "こんにちは",
  (result) => {
    if (result.reason === sdk.ResultReason.SynthesizingAudioCompleted) {
      console.log("Success");
    }
    synthesizer.close();
  }
);
```

## 利用可能モデル一覧 (2025-01時点)

### LLM

| モデル | 備考 |
|--------|------|
| gpt-5, gpt-5.1, gpt-5.2 | 最新 |
| gpt-5-mini, gpt-5-nano | 軽量版 |
| gpt-4.1, gpt-4.1-mini, gpt-4.1-nano | |
| gpt-4o, gpt-4o-mini | |
| o4-mini, o3, o3-mini, o1, o1-mini | reasoningモデル |
| gpt-35-turbo | レガシー |

### 画像生成

| モデル | 備考 |
|--------|------|
| dall-e-3 | 推奨 |
| dall-e-2 | レガシー |

### TTS (North Central US / Sweden Central のみ)

| モデル | 備考 |
|--------|------|
| tts | 標準 (= tts-1) |
| tts-hd | 高品質 (= tts-1-hd) |

### Azure OpenAI TTS ボイス

| ボイス名 | 備考 |
|----------|------|
| alloy | |
| ash | |
| coral | |
| echo | |
| fable | |
| onyx | |
| nova | |
| sage | |
| shimmer | |

### Azure Speech Service 日本語ボイス

| ボイス名 | 性別 |
|----------|------|
| ja-JP-NanamiNeural | 女性 |
| ja-JP-KeitaNeural | 男性 |
| ja-JP-AoiNeural | 女性 |
| ja-JP-DaichiNeural | 男性 |

## トラブルシューティング

### DNS解決エラー

カスタムドメイン設定後、DNS反映に数分かかる場合があります。

```bash
# DNS確認
nslookup <resource-name>.openai.azure.com
```

### DeploymentNotFound エラー

- deployment名が正しいか確認
- モデルがデプロイされているか確認

```bash
az cognitiveservices account deployment list \
  --name <resource-name> \
  --resource-group <resource-group> \
  -o table
```

### TTS が 404 エラー

- リソースが `northcentralus` または `swedencentral` にあるか確認
- `tts` または `tts-hd` モデルがデプロイされているか確認
- 他のリージョンでは Azure OpenAI TTS は利用不可（Azure Speech Service を使用）

### OpenAI vs AzureOpenAI クラス

Azure OpenAI を使用する場合は `AzureOpenAI` クラスを使用する必要があります。
通常の `OpenAI` クラスに `baseURL` を設定しても動作しません。

```typescript
// NG: OpenAI クラスでは Azure は使えない
import OpenAI from "openai";
const client = new OpenAI({ baseURL: "https://xxx.openai.azure.com/" }); // 動作しない

// OK: AzureOpenAI クラスを使用
import { AzureOpenAI } from "openai";
const client = new AzureOpenAI({ endpoint: "https://xxx.openai.azure.com/" }); // OK
```

## 参考リンク

- [Azure OpenAI Service ドキュメント](https://learn.microsoft.com/azure/ai-services/openai/)
- [Azure Speech Service ドキュメント](https://learn.microsoft.com/azure/ai-services/speech-service/)
- [Azure OpenAI TTS クイックスタート](https://learn.microsoft.com/azure/ai-services/openai/text-to-speech-quickstart)
