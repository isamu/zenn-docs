---
title: "MulmoCast v2.6.0〜v2.6.3 — 宣言的アニメーション要素 & メディア参照システム"
emoji: "🎨"
type: "tech"
topics: [AI, mulmocast, GraphAI, animation, video]
published: true
publication_name: "singularity"
---

# MulmoCast v2.6.0〜v2.6.3 宣言的アニメーション要素 & メディア参照システム

MulmoCast v2.6.0〜v2.6.3で、JSONだけでアニメーションを定義できるSwipe風要素と、動画・画像の参照システムが追加されました。

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

---

📦 **npm**: `npx mulmocast@2.6.3`
🔗 **GitHub**: https://github.com/receptron/mulmocast-cli

