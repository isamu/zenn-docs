---
title: "MulmoCast で、しゃべるアバターと効果音つきの動画を作る（初心者向け）"
emoji: "🎙️"
type: "tech"
topics: ["mulmocast", "ai", "tts", "nodejs", "avatar"]
published: false
publication_name: "singularity"
---

## この記事でできること

[MulmoCast](https://github.com/receptron/mulmocast-cli) は、JSON で書いた台本（MulmoScript）から、ナレーション付きの動画を作るツールです。最近、次の 2 つが使えるようになりました。

- **JingleScript**: 効果音やジングルを、台本の中に「楽譜」として書くだけで鳴らせる
- **AvatarScript**: スライドの上で、アバターがナレーションに合わせて口を動かしてしゃべる

この記事では、まったく初めての人向けに、2 本の短い動画を作るところまでを順番に説明します。

**ステップ 1 で作る動画**（効果音つきのスライド）

@[youtube](hxGCeUVFmrA)

**ステップ 2 で作る動画**（しゃべるアバター）

@[youtube](OiESBpJOciw)

![アバターがスライドの上でしゃべっている様子](/images/mulmocast-avatar-jingle-beginner/step2-avatar.png)

## 用意するもの

| もの | 説明 |
|---|---|
| Node.js | 22.22.2 以上（22 系）、24.15.0 以上（24 系）、または 26 以上。`node -v` で確認できます |
| ffmpeg | 動画を組み立てるのに使います。Mac なら `brew install ffmpeg` |
| Gemini の API キー | 読み上げ（TTS）に使います。[Google AI Studio](https://aistudio.google.com/apikey) で作れます |

この記事のサンプルは、読み上げに Gemini を使います。API キーは 1 つだけで動きます。

## ステップ 0: 作業用のフォルダを作る

```bash
mkdir my-mulmo
cd my-mulmo
npm init -y
npm install mulmocast
```

API キーは、同じフォルダに `.env` というファイルを作って書きます。

```bash:.env
GEMINI_API_KEY=ここに自分のキー
```

:::message
新しめの npm では、パッケージのインストールスクリプトが既定で止められ、`npm warn install-scripts … had install scripts blocked` と表示されることがあります。MulmoCast はスライドを画像にするのに Chrome（puppeteer）を使うので、そのときは次のコマンドで Chrome を入れてください。

```bash
npx puppeteer browsers install chrome
```
:::

## ステップ 1: 効果音つきの動画を作る（JingleScript）

### 台本の基本

MulmoScript の台本は、場面（beat）の並びです。1 つの場面には、だいたい次の 3 つを書きます。

- `text`: ナレーションで読み上げる文章
- `image`: その場面で見せる画面（ここではスライド）
- `soundEffects`: その場面で鳴らす効果音（今回のポイント）

最初の場面だけを抜き出すと、こうなります。

```json
{
  "text": "こんにちは。MulmoCast で、効果音つきの動画を作ってみましょう。",
  "image": {
    "type": "slide",
    "slide": { "layout": "title", "title": "効果音つきの動画", "subtitle": "台本に書くだけ。音のファイルは要りません" }
  },
  "soundEffects": [
    {
      "source": {
        "kind": "jinglescript",
        "score": {
          "format": "jinglescript/1",
          "tempo": 110,
          "length": { "seconds": 4 },
          "cues": { "end": { "beats": 4 } },
          "tracks": [
            {
              "instrument": "ukulele",
              "notes": [
                { "at": 0, "chord": "C", "strum": "D-DU-UDU", "vel": 0.7 },
                { "at": "end", "chord": "C", "strum": { "pattern": "down", "spread": 25 }, "vel": 0.85 }
              ]
            }
          ]
        }
      },
      "volume": 0.5
    }
  ]
}
```

### 効果音の「楽譜」の読み方

`source` を `{ "kind": "jinglescript", "score": … }` にすると、`score` に書いた楽譜から、その場で音が作られます。音のファイルを探す必要も、ネットワークもいりません。

| 項目 | 意味 |
|---|---|
| `tempo` | 速さ（1 分間の拍の数）。100〜140 くらいが使いやすい |
| `length` | 音の長さ。最後の音が鳴り終わるまで、少し余裕を持たせる |
| `tracks` | 楽器ごとのパート。いくつでも重ねられる |
| `instrument` | 楽器や効果音の名前（`ukulele`、`marimba`、`pop`、`clock` など） |
| `notes` | 鳴らす音の並び。`at` がいつ（拍の数）、`pitch` が音の高さ（`C5` など）、`chord` が和音（`C`、`Am` など）、`vel` が強さ（0〜1） |
| `cues` | 名前を付けた時刻。`"at": "end"` のように、名前で音を置ける |

効果音そのものの設定は 2 つです。

- `startAt`: 場面が始まってから何秒後に鳴らすか（省略すると場面の最初）
- `volume`: 音量（0〜1）

使える楽器や書き方の詳しい説明は、次のコマンドで読めます。

```bash
npx jinglescript guide
```

:::message
楽譜は JSON なので、Claude などの LLM に「この場面に合う、2 秒の明るい効果音を JingleScript で書いて」と頼むと書いてもらえます。`npx jinglescript guide` の内容を一緒に渡すと、正しい形で書いてくれます。
:::

### 動画を作る

台本の全体を `step1_jingle.json` という名前で保存します（3 つの場面それぞれに、違う効果音が付いています）。

:::details step1_jingle.json（全体）
```json
{
  "$mulmocast": {
    "version": "1.1"
  },
  "lang": "ja",
  "title": "はじめての効果音（JingleScript）",
  "speechParams": {
    "speakers": {
      "Presenter": {
        "provider": "gemini",
        "voiceId": "Leda",
        "displayName": {
          "ja": "ナレーター"
        }
      }
    }
  },
  "beats": [
    {
      "text": "こんにちは。MulmoCast で、効果音つきの動画を作ってみましょう。",
      "image": {
        "type": "slide",
        "slide": {
          "layout": "title",
          "title": "効果音つきの動画",
          "subtitle": "台本に書くだけ。音のファイルは要りません"
        }
      },
      "soundEffects": [
        {
          "source": {
            "kind": "jinglescript",
            "score": {
              "format": "jinglescript/1",
              "title": "Opening strum",
              "tempo": 110,
              "length": {
                "seconds": 4
              },
              "cues": {
                "end": {
                  "beats": 4
                }
              },
              "tracks": [
                {
                  "instrument": "ukulele",
                  "notes": [
                    {
                      "at": 0,
                      "chord": "C",
                      "strum": "D-DU-UDU",
                      "vel": 0.7
                    },
                    {
                      "at": "end",
                      "chord": "C",
                      "strum": {
                        "pattern": "down",
                        "spread": 25
                      },
                      "vel": 0.85
                    }
                  ]
                }
              ]
            }
          },
          "volume": 0.5
        }
      ]
    },
    {
      "text": "ポン、ポン、ポン！ 星が三つ出てきました。",
      "image": {
        "type": "slide",
        "slide": {
          "layout": "bigQuote",
          "quote": "★  ★  ★",
          "author": "ポン、ポン、ポン！"
        }
      },
      "soundEffects": [
        {
          "source": {
            "kind": "jinglescript",
            "score": {
              "format": "jinglescript/1",
              "tempo": 120,
              "length": {
                "seconds": 2
              },
              "tracks": [
                {
                  "instrument": "pop",
                  "reverb": false,
                  "notes": [
                    {
                      "at": 0,
                      "pitch": "E6",
                      "vel": 0.8
                    },
                    {
                      "at": 0.5,
                      "pitch": "G6",
                      "vel": 0.8
                    },
                    {
                      "at": 1,
                      "pitch": "C7",
                      "vel": 0.9
                    }
                  ]
                }
              ]
            }
          },
          "startAt": 0.2
        }
      ]
    },
    {
      "text": "よくできました！",
      "image": {
        "type": "slide",
        "slide": {
          "layout": "title",
          "title": "よくできました！",
          "subtitle": "JingleScript で効果音"
        }
      },
      "soundEffects": [
        {
          "source": {
            "kind": "jinglescript",
            "score": {
              "format": "jinglescript/1",
              "title": "Well done",
              "tempo": 130,
              "length": {
                "seconds": 3
              },
              "cues": {
                "hit": {
                  "seconds": 0.7
                }
              },
              "tracks": [
                {
                  "instrument": "marimba",
                  "notes": [
                    {
                      "at": 0,
                      "pitch": "G4",
                      "vel": 0.8
                    },
                    {
                      "at": 0.5,
                      "pitch": "C5",
                      "vel": 0.85
                    },
                    {
                      "at": "hit",
                      "pitch": [
                        "C5",
                        "E5",
                        "G5",
                        "C6"
                      ],
                      "vel": 1.0
                    }
                  ]
                },
                {
                  "instrument": "marimba",
                  "notes": [
                    {
                      "at": "hit",
                      "pitch": [
                        "C3",
                        "G3"
                      ],
                      "vel": 0.9
                    }
                  ]
                }
              ]
            }
          },
          "startAt": 0.8,
          "volume": 0.8
        }
      ]
    }
  ]
}
```
:::

次のコマンドで動画を作ります。

```bash
npx mulmo movie step1_jingle.json
```

数十秒で `output/step1_jingle_ja.mp4` ができます。

![ステップ 1 の動画の最初の場面](/images/mulmocast-avatar-jingle-beginner/step1-title.png)

## ステップ 2: しゃべるアバターを出す（AvatarScript）

### 追加のインストール

アバターを使うときだけ、2 つのパッケージを追加します。

```bash
npm install avatarscript onnxruntime-node
```

:::message alert
- `onnxruntime-node` は約 290 MB あります
- 最初にアバターを描画するとき、声と文章を合わせるためのモデル（約 635 MB）が一度だけダウンロードされます。最初の 1 回は数分かかりますが、2 回目からは速くなります
:::

### アバターを話者に付ける

アバターは「話者（speaker）」に付けます。`speechParams` に話者を定義し、`avatar.source` にアバターの場所を書きます。

```json
"speechParams": {
  "speakers": {
    "Ani": {
      "provider": "gemini",
      "voiceId": "Leda",
      "avatar": { "source": "https://raw.githubusercontent.com/receptron/mulmocast-media/main/avatars/ani" }
    }
  }
}
```

各場面で `"speaker": "Ani"` と書くと、その場面はアニがしゃべります。

すぐに使えるアバターが 5 体あります（MIT License）。URL の最後の `ani` を入れ替えるだけです。

| 名前 | |
|---|---|
| `ani` | この記事で使っているアバター |
| `marilyn` / `reika` / `seiya` / `ken` | ほかのキャラクター |

元のイラストは [mulmocast-media](https://github.com/receptron/mulmocast-media/tree/main/avatars) で見られます。

### 表情と動きを付ける

場面ごとに `avatarParams` を書くと、表情と動きを変えられます。

```json
{
  "speaker": "Ani",
  "text": "みなさん、こんにちは！ アニです。今日は、スライドの上でしゃべるアバターを紹介します。",
  "avatarParams": {
    "emotion": "happy",
    "motions": [{ "motion": "greet", "at": "こんにちは" }]
  }
}
```

| 項目 | 意味 |
|---|---|
| `emotion` | 表情。`neutral` / `happy` / `sad` / `angry` / `surprised` / `relaxed` |
| `motions` | 動き。`motion` が動きの名前、`at` は「この言葉を話すときに動く」という文章の一部 |
| `position` | 位置と大きさ（`x`、`y`、`scale` を画面に対する % で） |
| `hidden` | `true` でその場面だけアバターを隠す |

動きの名前: `nod`（うなずく）、`tilt`（首をかしげる）、`think`（考える）、`giggle`（くすっと笑う）、`surprise`（驚く）、`shy`（照れる）、`no`（首を振る）、`wink`（ウインク）、`greet`（あいさつ）

口の動きは何も書かなくても、声に合わせて自動で付きます。

### 動画を作る

台本の全体を `step2_avatar.json` として保存します。最後の場面には、ステップ 1 と同じやり方で効果音も付けています。

:::details step2_avatar.json（全体）
```json
{
  "$mulmocast": {
    "version": "1.1"
  },
  "lang": "ja",
  "title": "はじめてのアバター（AvatarScript）",
  "speechParams": {
    "speakers": {
      "Ani": {
        "provider": "gemini",
        "voiceId": "Leda",
        "displayName": {
          "ja": "アニ"
        },
        "avatar": {
          "source": "https://raw.githubusercontent.com/receptron/mulmocast-media/main/avatars/ani"
        }
      }
    }
  },
  "beats": [
    {
      "speaker": "Ani",
      "text": "みなさん、こんにちは！ アニです。今日は、スライドの上でしゃべるアバターを紹介します。",
      "image": {
        "type": "slide",
        "slide": {
          "layout": "title",
          "title": "しゃべるアバター",
          "subtitle": "台本に一行足すだけ。口の動きは声に合わせて自動で"
        }
      },
      "avatarParams": {
        "emotion": "happy",
        "motions": [
          {
            "motion": "greet",
            "at": "こんにちは"
          }
        ]
      }
    },
    {
      "speaker": "Ani",
      "text": "私は、一枚のイラストから作られました。表情や、うなずく動きも、台本で指定できるんです。",
      "image": {
        "type": "slide",
        "slide": {
          "layout": "columns",
          "title": "表情と動き",
          "columns": [
            {
              "title": "emotion",
              "content": [
                {
                  "type": "bullets",
                  "items": [
                    "happy",
                    "relaxed",
                    "surprised"
                  ]
                }
              ]
            },
            {
              "title": "motions",
              "content": [
                {
                  "type": "bullets",
                  "items": [
                    "greet",
                    "nod",
                    "wink"
                  ]
                }
              ]
            }
          ]
        }
      },
      "avatarParams": {
        "emotion": "relaxed",
        "motions": [
          {
            "motion": "nod",
            "at": "台本"
          }
        ]
      }
    },
    {
      "speaker": "Ani",
      "text": "えっ、もう終わり？ 最後まで見てくれて、ありがとう！",
      "image": {
        "type": "slide",
        "slide": {
          "layout": "title",
          "title": "ありがとう！",
          "subtitle": "MulmoCast + AvatarScript"
        }
      },
      "avatarParams": {
        "emotion": "surprised",
        "motions": [
          {
            "motion": "surprise"
          },
          {
            "motion": "wink",
            "at": "ありがとう"
          }
        ]
      },
      "soundEffects": [
        {
          "source": {
            "kind": "jinglescript",
            "score": {
              "format": "jinglescript/1",
              "tempo": 120,
              "length": {
                "seconds": 2
              },
              "tracks": [
                {
                  "instrument": "pop",
                  "reverb": false,
                  "notes": [
                    {
                      "at": 0,
                      "pitch": "C6",
                      "vel": 0.7
                    },
                    {
                      "at": 0.5,
                      "pitch": "G6",
                      "vel": 0.8
                    }
                  ]
                }
              ]
            }
          },
          "startAt": 0.1,
          "volume": 0.6
        }
      ]
    }
  ]
}
```
:::

```bash
npx mulmo movie step2_avatar.json
```

`output/step2_avatar_ja.mp4` ができます。

![最後の場面。表情が「驚き」に変わっている](/images/mulmocast-avatar-jingle-beginner/step2-thanks.png)

## 自分のアバターを作る

[mesh-avatar-studio](https://github.com/shinshin86/mesh-avatar-studio) を使うと、イラスト 1 枚からアバターを作れます。できたフォルダを、`avatar.source` に台本からの相対パスで指定します。

```json
"avatar": { "source": "./avatars/my-avatar" }
```

## うまくいかないとき

| 症状 | 対処 |
|---|---|
| `Avatars need the avatarscript and onnxruntime-node packages` と出て止まる | `npm install avatarscript onnxruntime-node` |
| 最初の 1 回だけ、とても時間がかかる | 声と文章を合わせるモデル（約 635 MB）のダウンロードです。2 回目からは速くなります |
| Chrome が起動しない・見つからない | `npx puppeteer browsers install chrome` |
| 読み上げで API のエラーが出る | `.env` の `GEMINI_API_KEY` を確認。キーの利用上限に達していないかも確認 |
| 台本を直したのに動画が変わらない | 作ったものはキャッシュされます。`-f` を付けて作り直す（`npx mulmo movie -f step2_avatar.json`） |

## まとめ

- 効果音は、台本の `soundEffects` に JingleScript の楽譜を書くだけで鳴らせる。追加のインストールはいらない
- アバターは、`avatarscript` と `onnxruntime-node` を入れて、話者に `avatar.source` を書くだけ。表情と動きは場面ごとに書ける
- どちらも JSON なので、LLM に台本ごと書いてもらうのとも相性がいい

もっと詳しい説明は、MulmoCast のドキュメントにあります。

- [しゃべるアバター（docs/avatar.md）](https://github.com/receptron/mulmocast-cli/blob/main/docs/avatar.md)
- [効果音（docs/sound_and_voice.md）](https://github.com/receptron/mulmocast-cli/blob/main/docs/sound_and_voice.md)
