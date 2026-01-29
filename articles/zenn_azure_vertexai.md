---
title: "MulmoCast で Azure OpenAI / Vertex AI を使う"
emoji: "🎬"
type: "tech"
topics: ["mulmocast", "azure", "vertexai", "openai", "gcp"]
published: false
---

# MulmoCast で Azure OpenAI / Vertex AI を使う

MulmoCast はエンタープライズ向けの **Azure OpenAI** と **Google Vertex AI** に対応しています。

本記事では、クラウドのセットアップが完了している前提で、MulmoCast 側で何を設定すればよいかを解説します。

## 切り替え方法の違い

| プロバイダー | 切り替え方法 |
|-------------|-------------|
| Azure OpenAI | **環境変数** で BASE_URL を指定 |
| Vertex AI | **MulmoScript** で `vertexai_project` を指定 |

---

# Azure OpenAI

## ポイント

Azure OpenAI への切り替えは **環境変数のみ** で行います。MulmoScript は OpenAI API と同じ記述のままで動作します。

## 環境変数の設定

`.env` ファイルで `*_OPENAI_BASE_URL` を設定すると、その機能は Azure OpenAI を使用します。

```bash
# 画像生成を Azure に切り替え
IMAGE_OPENAI_API_KEY=<Azure APIキー>
IMAGE_OPENAI_BASE_URL=https://<リソース名>.openai.azure.com/

# TTS を Azure に切り替え
TTS_OPENAI_API_KEY=<Azure APIキー>
TTS_OPENAI_BASE_URL=https://<リソース名>.openai.azure.com/

# LLM（translate, scripting）を Azure に切り替え
LLM_OPENAI_API_KEY=<Azure APIキー>
LLM_OPENAI_BASE_URL=https://<リソース名>.openai.azure.com/
```

**重要**: `BASE_URL` を設定すると Azure モードになります。設定しなければ通常の OpenAI API を使用します。

## MulmoScript

OpenAI API と同じ記述です。変更不要。

```json
{
  "imageParams": {
    "provider": "openai",
    "model": "gpt-image-1.5"
  },
  "speechParams": {
    "speakers": {
      "Presenter": {
        "provider": "openai",
        "voiceId": "alloy",
        "model": "tts"
      }
    }
  }
}
```

## CLI 実行

通常どおり実行するだけです。環境変数に基づいて自動的に Azure を使用します。

```bash
mulmo movie script.json
mulmo audio script.json
mulmo images script.json
mulmo translate script.json -l ja
```

## 注意点

### デプロイメント名 = モデル名

Azure ではデプロイメント名をモデル名と完全一致させてください。MulmoCast は `model` パラメータをそのままデプロイメント名として使用します。

```bash
# OK: デプロイメント名が gpt-image-1.5
--deployment-name "gpt-image-1.5"

# NG: ハイフンに置き換えている
--deployment-name "gpt-image-1-5"
```

### リージョン

画像生成と TTS を両方使う場合は **Sweden Central** を推奨（両方対応）。

| リージョン | 画像生成 | TTS |
|-----------|---------|-----|
| Sweden Central | ✅ | ✅ |
| East US 2 | ✅ | ❌ |
| North Central US | ❌ | ✅ |

---

# Google Vertex AI

## ポイント

Vertex AI への切り替えは **MulmoScript 内** で `vertexai_project` を指定します。

## 前提条件

ADC（Application Default Credentials）を設定済みであること。

```bash
gcloud auth application-default login
```

## MulmoScript の設定

`imageParams` または `movieParams` に `vertexai_project` を追加すると Vertex AI モードになります。

```json
{
  "imageParams": {
    "provider": "google",
    "model": "imagen-4.0-generate-001",
    "vertexai_project": "your-project-id",
    "vertexai_location": "us-central1"
  }
}
```

| パラメータ | 説明 | デフォルト |
|-----------|------|-----------|
| `vertexai_project` | GCP プロジェクト ID | なし（指定すると Vertex AI モード） |
| `vertexai_location` | リージョン | `us-central1` |

## 動画生成（Veo）

```json
{
  "movieParams": {
    "provider": "google",
    "model": "veo-2.0-generate-001",
    "vertexai_project": "your-project-id",
    "vertexai_location": "us-central1"
  }
}
```

## CLI 実行

```bash
mulmo images script.json
mulmo movie script.json
```

## Gemini API との違い

| 項目 | Gemini API | Vertex AI |
|------|-----------|-----------|
| 認証 | `GEMINI_API_KEY` 環境変数 | ADC |
| MulmoScript | `vertexai_project` なし | `vertexai_project` あり |

Gemini API の場合：

```bash
# 環境変数
GEMINI_API_KEY=your_api_key
```

```json
{
  "imageParams": {
    "provider": "google",
    "model": "gemini-2.5-flash-image"
  }
}
```

Vertex AI の場合は `vertexai_project` を追加するだけ：

```json
{
  "imageParams": {
    "provider": "google",
    "model": "imagen-4.0-generate-001",
    "vertexai_project": "your-project-id"
  }
}
```

## 利用可能なモデル

**画像生成**:
- `imagen-4.0-generate-001`（標準）
- `imagen-4.0-ultra-generate-001`（高品質）
- `imagen-4.0-fast-generate-001`（高速）

**動画生成**:
- `veo-2.0-generate-001`
- `veo-3.0-generate-001`
- `veo-3.1-generate-preview`

---

# まとめ

| プロバイダー | 設定場所 | 設定内容 |
|-------------|---------|---------|
| Azure OpenAI | 環境変数 | `*_OPENAI_BASE_URL` を設定 |
| Vertex AI | MulmoScript | `vertexai_project` を追加 |

```bash
# Azure: 環境変数で切り替え
IMAGE_OPENAI_BASE_URL=https://xxx.openai.azure.com/

# Vertex AI: MulmoScript で指定
"vertexai_project": "my-project-id"
```

---

# 補足: クラウドのセットアップ

## Azure OpenAI のセットアップ

### 1. Azure CLI のインストールとログイン

```bash
# macOS
brew install azure-cli

# ログイン
az login
```

### 2. リソースの作成

```bash
# リソースグループ作成
az group create --name my-rg --location swedencentral

# Azure OpenAI リソース作成
az cognitiveservices account create \
  --name my-openai-resource \
  --resource-group my-rg \
  --kind OpenAI \
  --sku S0 \
  --location swedencentral \
  --yes

# カスタムドメイン設定（必須）
az cognitiveservices account update \
  --name my-openai-resource \
  --resource-group my-rg \
  --custom-domain my-openai-resource
```

### 3. モデルのデプロイ

```bash
# 画像生成モデル
az cognitiveservices account deployment create \
  --name my-openai-resource \
  --resource-group my-rg \
  --deployment-name "gpt-image-1.5" \
  --model-name gpt-image-1.5 \
  --model-version "2025-12-16" \
  --model-format OpenAI \
  --sku-capacity 1 \
  --sku-name GlobalStandard

# TTSモデル
az cognitiveservices account deployment create \
  --name my-openai-resource \
  --resource-group my-rg \
  --deployment-name "tts" \
  --model-name tts \
  --model-version "001" \
  --model-format OpenAI \
  --sku-capacity 1 \
  --sku-name Standard

# LLMモデル
az cognitiveservices account deployment create \
  --name my-openai-resource \
  --resource-group my-rg \
  --deployment-name "gpt-4o" \
  --model-name gpt-4o \
  --model-version "2024-11-20" \
  --model-format OpenAI \
  --sku-capacity 1 \
  --sku-name Standard
```

### 4. API キーの取得

```bash
# API キー
az cognitiveservices account keys list \
  --name my-openai-resource \
  --resource-group my-rg \
  --query "key1" -o tsv

# エンドポイント確認
az cognitiveservices account show \
  --name my-openai-resource \
  --resource-group my-rg \
  --query "properties.endpoint" -o tsv
```

### 5. 利用可能なモデルの確認

```bash
# 画像生成モデル
az cognitiveservices model list --location swedencentral -o json | \
  jq '[.[] | .model | {name, version}] | unique_by(.name) | .[] | select(.name | test("gpt-image|dall"; "i"))'

# TTSモデル
az cognitiveservices model list --location swedencentral -o json | \
  jq '[.[] | .model | {name, version}] | unique_by(.name) | .[] | select(.name | test("tts"; "i"))'
```

---

## Google Vertex AI のセットアップ

### 1. gcloud CLI のインストールとログイン

```bash
# macOS
brew install google-cloud-sdk

# ログイン
gcloud auth login

# プロジェクト一覧
gcloud projects list

# プロジェクト設定
gcloud config set project YOUR_PROJECT_ID
```

### 2. API の有効化

```bash
# Vertex AI API
gcloud services enable aiplatform.googleapis.com

# Generative AI API
gcloud services enable generativelanguage.googleapis.com
```

### 3. ADC の設定

```bash
# Application Default Credentials を設定（ブラウザが開きます）
gcloud auth application-default login

# 確認
gcloud auth application-default print-access-token
```

### 4. IAM 権限の付与（必要に応じて）

```bash
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="user:YOUR_EMAIL" \
  --role="roles/aiplatform.user"
```

### 5. リージョンの確認

Vertex AI は以下のリージョンで利用可能です：

- `us-central1`（推奨）
- `us-east1`
- `us-west1`
- `europe-west1`
- `asia-northeast1`（東京）
