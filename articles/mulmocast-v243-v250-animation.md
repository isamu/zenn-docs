---
title: "MulmoCast v2.4.3〜v2.6.5 — アニメーション・メディア参照・IR可視化"
emoji: "🎬"
type: "tech"
topics: [AI, mulmocast, GraphAI, animation, Puppeteer]
published: true
publication_name: "singularity"
---

# MulmoCast v2.4.3〜v2.6.5 リリースまとめ

MulmoCast v2.4.3〜v2.6.5で追加された主要な機能をまとめて紹介します。アニメーション基盤の強化、メディア参照システム、IR向け可視化チャートなど、多岐にわたるアップデートです。

## 音声・映像の精密同期（v2.4.3）

FFmpegの`trim=duration`が**フレーム境界に切り上げ**される問題で、beatごとに最大0.03秒のズレが累積していました。フレーム単位の精密トラッキング（Bresenham方式）で修正しました。

:::message
この修正は自動的に適用されます。MulmoScript側の変更は不要です。
:::

## image:name URLスキーム（v2.4.8）

`imageParams.images` で定義した画像を、html_tailwindビート内で `image:name` で参照できます。

```json
{
  "imageParams": {
    "images": {
      "logo": {
        "type": "image",
        "source": { "kind": "path", "path": "./assets/logo.png" }
      }
    }
  },
  "beats": [
    {
      "text": "会社紹介",
      "image": {
        "type": "html_tailwind",
        "html": "<div class='p-8'><img src='image:logo' class='w-32' /><h1>Company Name</h1></div>"
      }
    }
  ]
}
```

`src="image:logo"` と書くだけで、ビルド時に自動的にファイルパスに解決されます。

## 相対パスの自動解決（v2.4.6）

html_tailwind内の相対パスが、スクリプトファイルのディレクトリを基準に自動的に`file://`絶対パスに変換されます。

```html
<!-- スクリプトが /project/scripts/demo.json にある場合 -->
<img src="./images/photo.jpg" />
<!-- ↓ 自動変換 -->
<img src="file:///project/scripts/images/photo.jpg" />
```

## キャプション位置調整（v2.4.7）

YouTubeプレーヤーのコントロールUIと重ならないように、キャプションの下端位置を調整できます。

```json
{
  "captionParams": {
    "bottomOffset": 20
  }
}
```

`bottomOffset: 20` で画面下端から20%の位置にキャプションが表示されます。

## CDP Screencastアニメーション録画（v2.4.9）

html_tailwindアニメーションの録画に、Chrome DevTools ProtocolのScreencast APIを利用できるようになりました。

```json
{
  "image": {
    "type": "html_tailwind",
    "html": "<div id='box'>Hello</div>",
    "script": "...",
    "animation": { "movie": true }
  }
}
```

- `"animation": true` — フレームバイフレーム方式（確実、やや遅い）
- `"animation": { "movie": true }` — CDP Screencast（高速）

## data-animation宣言的アニメーション（v2.5.0）

HTMLの`data-*`属性だけでアニメーションを定義できます。JavaScriptを書く必要がありません。

```json
{
  "image": {
    "type": "html_tailwind",
    "html": [
      "<div class='flex items-center justify-center h-full bg-gray-900'>",
      "  <h1 data-animation='animate'",
      "      data-opacity='0,1'",
      "      data-translate-y='30,0'",
      "      data-start='0'",
      "      data-end='2'",
      "      data-easing='easeOut'",
      "      class='text-6xl font-bold text-white'>",
      "    Hello World",
      "  </h1>",
      "</div>"
    ],
    "animation": true,
    "duration": 3
  }
}
```

### 対応アニメーション

| data-animation | 説明 |
|---|---|
| `animate` | プロパティ補間（opacity, translate, scale, rotate） |
| `stagger` | 複数要素の順次アニメーション |
| `typewriter` | タイプライター効果 |
| `counter` | 数値カウントアップ |
| `codeReveal` | コード行の順次表示 |
| `blink` | 点滅 |
| `coverZoom` | Ken Burns風ズーム |
| `coverPan` | Ken Burns風パン |

## Cover Pan / Cover Zoom（v2.5.0）

写真のKen Burns風エフェクトが使えます。

```html
<img id="photo" src="image:landscape"
     data-animation="coverPan"
     data-axis="x"
     data-from="0"
     data-to="100"
     data-start="0"
     data-end="auto"
     data-easing="linear" />
```

- `coverZoom`: ズームイン/アウト（`data-zoom-from="1"` `data-zoom-to="1.3"`）
- `coverPan`: パン（`data-axis="x"` or `"y"`、`data-from`/`data-to` で位置%指定）

## 画像ごとのcanvasSize（v2.5.0）

参照画像ごとに異なるアスペクト比を指定できます。

```json
{
  "imageParams": {
    "images": {
      "portrait_bg": {
        "type": "imagePrompt",
        "prompt": "A vertical portrait photo",
        "canvasSize": { "width": 720, "height": 1280 }
      }
    }
  }
}
```

## Swipe風宣言的アニメーション要素（v2.6.0）

`html`フィールドの代わりに`elements`配列でアニメーションを宣言的に定義できます。HTMLやJavaScriptを書く必要がありません。

```json
{
  "image": {
    "type": "html_tailwind",
    "elements": [
      {
        "tag": "div",
        "text": "Hello World",
        "style": {
          "x": 100, "y": 200,
          "w": 400, "h": 100,
          "fontSize": 48,
          "color": "#ffffff"
        },
        "loop": { "type": "bounce", "speed": 2 },
        "transitions": [
          {
            "prop": "opacity",
            "from": 0, "to": 1,
            "startFrame": 0, "endFrame": 30
          }
        ]
      }
    ],
    "animation": true,
    "duration": 5
  }
}
```

### ループアニメーション

| type | 説明 |
|---|---|
| `wiggle` | 左右に揺れる |
| `vibrate` | 細かい振動 |
| `bounce` | 上下にバウンス |
| `pulse` | 拡大縮小の脈動 |
| `blink` | 点滅 |
| `spin` | 回転 |
| `shift` | 左右にシフト |

### トランジション

`transitions`配列で、フレーム範囲ごとにプロパティを補間します。

```json
{
  "transitions": [
    { "prop": "opacity", "from": 0, "to": 1, "startFrame": 0, "endFrame": 15 },
    { "prop": "translateX", "from": -100, "to": 0, "startFrame": 0, "endFrame": 20 }
  ]
}
```

## movie:name 動画参照（v2.6.1）

`imageParams.images`で動画ファイルを参照定義し、html_tailwindやmarkdown内で`movie:name`で利用できます。

```json
{
  "imageParams": {
    "images": {
      "bg_video": {
        "type": "movie",
        "source": { "kind": "path", "path": "./assets/background.mp4" }
      }
    }
  },
  "beats": [
    {
      "text": "動画背景のスライド",
      "image": {
        "type": "html_tailwind",
        "html": "<video src='movie:bg_video' autoplay muted loop class='absolute inset-0 w-full h-full object-cover'></video><div class='relative z-10 p-12'><h1 class='text-white text-6xl'>Title</h1></div>"
      }
    }
  ]
}
```

Puppeteerでの`<video>`要素レンダリングも対応。フレームバイフレーム録画時にはビデオの各フレームが正確に同期されます。

## Markdownでのimage:name参照（v2.6.1）

markdownプラグインでも`image:name`参照が使えるようになりました。

```json
{
  "image": {
    "type": "markdown",
    "markdown": "# My Slide\n![Logo](image:company_logo)\n\nWelcome to our presentation"
  }
}
```

## beat単位のローカルメディア参照（v2.6.3）

グローバルの`imageParams.images`に加えて、beat単位で`images`を定義できます。そのbeatでのみ使用する画像・動画をローカルに定義できます。

```json
{
  "beats": [
    {
      "text": "このbeatでのみ使う画像",
      "images": {
        "local_bg": {
          "type": "image",
          "source": { "kind": "url", "url": "https://example.com/bg.jpg" }
        }
      },
      "image": {
        "type": "html_tailwind",
        "html": "<div style='background-image: url(image:local_bg)'>...</div>"
      }
    }
  ]
}
```

- ローカル定義はグローバル定義より優先されます
- 同名のキーがある場合、ローカルが上書き

## アニメーション付きbeatのPDF出力修正（v2.6.3）

アニメーション付きhtml_tailwindビートでPDFを生成する際、全アニメーションが完了した状態（最終フレーム）の高品質静止画が生成されるようになりました。

以前はアニメーションの初期状態（opacity: 0等）でキャプチャされ、PDF上でコンテンツが見えない問題がありました。

```bash
# アニメーション付きスクリプトでもPDFが正しく生成される
mulmocast pdf script.json
```

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
