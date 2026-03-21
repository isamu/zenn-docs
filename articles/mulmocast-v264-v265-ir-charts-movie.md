---
title: "MulmoCast v2.6.4〜v2.6.5 — IR向け可視化チャート & 動画生成の参照画像対応"
emoji: "📊"
type: "tech"
topics: [AI, mulmocast, GraphAI, chartjs, Veo]
published: true
publication_name: "singularity"
---

# MulmoCast v2.6.4〜v2.6.5 IR向け可視化チャート & 動画生成の参照画像対応

MulmoCast v2.6.4〜v2.6.5で、決算・IRプレゼンテーション向けの可視化チャートと、動画生成時の参照画像サポートが追加されました。

## Chart.jsプラグイン自動ロード（v2.6.4）

チャートタイプに応じて、サンキー図（`chartjs-chart-sankey`）とツリーマップ（`chartjs-chart-treemap`）のCDNが自動的に読み込まれます。

### サンキー図

貸借対照表のような資金フロー可視化に最適です。

```json
{
  "image": {
    "type": "chart",
    "title": "",
    "chartData": {
      "type": "sankey",
      "data": {
        "datasets": [{
          "data": [
            { "from": "cash", "to": "current_a", "flow": 3000 },
            { "from": "receivable", "to": "current_a", "flow": 2000 },
            { "from": "current_a", "to": "total_a", "flow": 5000 },
            { "from": "total_a", "to": "liabilities", "flow": 3000 },
            { "from": "total_a", "to": "equity", "flow": 2000 }
          ],
          "column": {
            "cash": 0, "receivable": 0,
            "current_a": 1,
            "total_a": 2,
            "liabilities": 3, "equity": 3
          },
          "labels": {
            "cash": "Cash\n$3,000M",
            "receivable": "Receivables\n$2,000M",
            "current_a": "Current Assets\n$5,000M",
            "total_a": "Total Assets\n$5,000M",
            "liabilities": "Liabilities\n$3,000M",
            "equity": "Equity\n$2,000M"
          },
          "colorFrom": {
            "cash": "#555", "receivable": "#555",
            "current_a": "#555", "total_a": "#555",
            "liabilities": "#DC2626", "equity": "#16A34A"
          },
          "colorTo": {
            "current_a": "#555", "total_a": "#555",
            "liabilities": "#DC2626", "equity": "#16A34A"
          },
          "colorMode": "gradient",
          "nodeWidth": 20,
          "nodePadding": 24
        }]
      }
    }
  }
}
```

**ポイント**:
- `column`: ノードの列位置を明示指定（砂時計型レイアウト）
- `priority`: 列内の縦順を制御
- `labels`: `\n` で改行した複数行ラベル
- `colorFrom` / `colorTo`: ノード名→色のマッピング（JSONでは関数が使えないため、オブジェクト形式で指定するとランタイムで自動変換）

### ツリーマップ

事業ポートフォリオの構成比を面積で可視化します。

```json
{
  "image": {
    "type": "chart",
    "title": "Business Portfolio",
    "chartData": {
      "type": "treemap",
      "data": {
        "datasets": [{
          "tree": [
            { "name": "Cloud", "value": 420 },
            { "name": "Enterprise", "value": 350 },
            { "name": "Consumer", "value": 200 }
          ],
          "key": "value",
          "backgroundColor": ["#3B82F6", "#6366F1", "#10B981"],
          "labels": {
            "display": true,
            "color": "white",
            "font": { "size": 14, "weight": "bold" }
          }
        }]
      }
    }
  }
}
```

:::message
ツリーマップの`backgroundColor`配列は、ランタイムでscriptable関数に自動変換されます。JSONで直感的に色を指定できます。
:::

## ウォーターフォールチャート（v2.6.4）

Slide DSLに`waterfall`レイアウトが追加されました。営業利益ブリッジなど、増減要因の分析に最適です。

```json
{
  "image": {
    "type": "slide",
    "slide": {
      "layout": "waterfall",
      "title": "FY2025 Operating Profit Bridge",
      "subtitle": "YoY +$15M (+12.5%)",
      "items": [
        { "label": "FY2024\nOp. Profit", "value": 120, "isTotal": true },
        { "label": "Revenue\nGrowth", "value": 35 },
        { "label": "COGS\nImprovement", "value": 12 },
        { "label": "Personnel\nCosts", "value": -18 },
        { "label": "R&D\nInvestment", "value": -10 },
        { "label": "FX\nImpact", "value": -4 },
        { "label": "FY2025\nOp. Profit", "value": 135, "isTotal": true }
      ],
      "unit": "$M",
      "callout": {
        "text": "Revenue growth drove profit growth",
        "label": "Summary"
      }
    }
  }
}
```

- **自動色分け**: 正の値 → 緑、負の値 → 赤、`isTotal: true` → 青
- **`unit`**: 値の横に表示される単位（`$M`、`億円` 等）
- **Running total**: バーの位置が自動的に累積計算

## チャートの背景画像/スタイル（v2.6.4）

chartプラグインに`backgroundImage`と`style`が追加されました。

```json
{
  "image": {
    "type": "chart",
    "title": "Revenue",
    "backgroundImage": {
      "source": { "kind": "url", "url": "https://example.com/bg.jpg" },
      "size": "cover",
      "opacity": 0.3
    },
    "style": "h1 { color: white; }",
    "chartData": { "type": "bar", "data": { "..." } }
  }
}
```

## 動画生成の参照画像対応（v2.6.4）

Veo 3.1とReplicateモデルで、動画生成時に参照画像を渡せるようになりました。

### firstFrameImageName — 最初のフレーム

`imageRefs`の画像をそのまま動画の最初のフレームとして使用します。`imagePrompt`による新規画像生成が不要です。

```json
{
  "moviePrompt": "Cherry blossoms swirling in wind",
  "movieParams": {
    "firstFrameImageName": "start_frame"
  }
}
```

### lastFrameImageName — 最終フレーム（補間）

最初のフレームと最終フレームの間を動画で補間します。

```json
{
  "moviePrompt": "Transition from day to sunset",
  "movieParams": {
    "firstFrameImageName": "day_scene",
    "lastFrameImageName": "sunset_scene"
  }
}
```

### referenceImages — スタイル/アセット参照

動画の生成時にスタイルやアセットの参照画像を指定します（Veo 3.1のみ）。

```json
{
  "moviePrompt": "A car driving through a city",
  "movieParams": {
    "referenceImages": [
      { "imageName": "car_photo", "referenceType": "ASSET" },
      { "imageName": "anime_style", "referenceType": "STYLE" }
    ]
  }
}
```

- **ASSET**: 被写体・オブジェクト・キャラクターの参照（最大3つ）
- **STYLE**: 色調・ライティング・テクスチャの参照（最大1つ）

:::message alert
**排他制約**: `referenceImages` は `firstFrameImageName` / `lastFrameImageName` と同時に使えません。両方指定した場合、`referenceImages`は自動的に無視されます。
:::

### モデル別対応状況

| Feature | Veo 2.0 | Veo 3.0 | Veo 3.1 | seedance | pixverse | hailuo-02 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **firstFrame** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **lastFrame** | — | ❌ | ✅ | ✅ | ✅ | ✅ |
| **referenceImages** | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |

非対応モデルで指定した場合、**警告メッセージ**が表示され、該当オプションは無視されます。

### 8秒超の動画（Video Extension）

Veo 3.1のVideo Extension（8秒超の動画生成）では、`firstFrameImageName`のみ有効です。`lastFrame`と`referenceImages`はVideo Extensionと互換性がありません。

## 画像生成の参照画像（v2.6.5）

`imageParams.images`の`imagePrompt`で、他の画像を参照しながら新しい画像を生成できます。

### referenceImageName — 他の画像キーを参照

```json
{
  "imageParams": {
    "images": {
      "original": {
        "type": "image",
        "source": { "kind": "path", "path": "./photo.jpg" }
      },
      "anime_version": {
        "type": "imagePrompt",
        "prompt": "Same scene in anime style",
        "referenceImageName": "original"
      }
    }
  }
}
```

### referenceImage — 直接ソース指定

```json
{
  "anime_version": {
    "type": "imagePrompt",
    "prompt": "Transform to watercolor style",
    "referenceImage": { "kind": "url", "url": "https://example.com/photo.jpg" }
  }
}
```

### チェーン参照

画像A → 画像B（Aを参照） → 画像C（Bを参照）のような連鎖も可能です。

```json
{
  "imageParams": {
    "images": {
      "base": {
        "type": "imagePrompt",
        "prompt": "A mountain landscape"
      },
      "with_snow": {
        "type": "imagePrompt",
        "prompt": "Add snow caps to the mountains",
        "referenceImageName": "base"
      },
      "final": {
        "type": "imagePrompt",
        "prompt": "Add aurora in the night sky",
        "referenceImageName": "with_snow"
      }
    }
  }
}
```

内部的に2段階で解決されます：
1. Stage 1: 参照なしの画像を生成
2. Stage 2: `referenceImageName`付きの画像を生成（Stage 1の結果を利用）

---

📦 **npm**: `npx mulmocast@2.6.5`
🔗 **GitHub**: https://github.com/receptron/mulmocast-cli
📖 **テストスクリプト**: `scripts/test/test_ir_visualizations.json`, `scripts/test/test_movie_references.json`, `scripts/test/test_image_prompt_reference.json`
