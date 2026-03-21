---
title: "MulmoCast v2.4.3〜v2.5.0 — アニメーション基盤の大幅強化"
emoji: "🎬"
type: "tech"
topics: [AI, mulmocast, GraphAI, animation, Puppeteer]
published: true
publication_name: "singularity"
---

# MulmoCast v2.4.3〜v2.5.0 アニメーション基盤の大幅強化

MulmoCast v2.4.3〜v2.5.0で、アニメーション関連の機能が大幅に強化されました。この記事では主要な変更点と、MulmoScriptでの具体的な使い方を解説します。

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

:::message alert
macOS 15.4以降でScreencastが0バイトのmp4を出力する問題が確認されています。現在はフレーム方式に自動フォールバックします。
:::

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

---

📦 **npm**: `npx mulmocast@2.5.0`
🔗 **GitHub**: https://github.com/receptron/mulmocast-cli
