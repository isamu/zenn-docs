---
title: "Stable Diffusion Webui をMacで動かす"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [stablediffusion]
published: true
publication_name: "singularity"
---

Stable Diffusionは、画像生成AIです。
オープンソースなので自分のPCで動かすことができます。

Stable Diffusion Webuiは、Web UIを使ってStable Diffusionを動かすことができるツールです。Stable Diffusion単体では、自分でプログラムをして動かす必要がありますが、Stable Diffusion Webuiを使えばブラウザから操作することができます。
NVIDIAのチップを積んでいないMacでも動かせるので早速試してみます。

AUTOMATIC1111/stable-diffusion-webui がMacでもインストールでき、使いやすいバージョンです。

レポジトリはこちら
https://github.com/AUTOMATIC1111/stable-diffusion-webui

M1 Macでインストールする方法があります。
https://github.com/AUTOMATIC1111/stable-diffusion-webui/wiki/Installation-on-Apple-Silicon

私の環境では brew + 各種ツールが入っているのでスキップしますが、開発環境が入っていない場合にはbrewをインストールし、

```
brew install cmake protobuf rust python@3.10 git wget
```

で、開発ツールを入れる必要があります。

次にソースをclone(ダウンロード)します。

```
git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui
```

modelを選んでダウンロードする必要があります。
Stable Diffusion 1.5を試します。(Stable Diffusion 2.1を試してみましたが、私の環境ではエラーが発生して動きませんでした。)

1.5のckptをダウンロードする
https://huggingface.co/runwayml/stable-diffusion-v1-5/resolve/main/v1-5-pruned-emaonly.ckpt

先ほどcloneしたディレクトリの配下に移動します。

```
stable-diffusion-webui/models/Stable-diffusion
```
そして、stable-diffusion-webuiのdirectoryで

```
./webui.sh
```
を実行。初回は、必要なツールのダウンロード/インストールがされるので、少し時間がかかります。

```
Running on local URL:  http://127.0.0.1:7860
```

と表示されると起動成功です。
ブラウザで http://127.0.0.1:7860 を開くと

![起動時の画面](https://storage.googleapis.com/zenn-user-upload/13cfd64fb348-20240228.png)

と表示されます。
Stable Diffusion checkpointには先程ダウンロードしたモデルが表示されれいると思います。


text2imageのテキスト入力部分に表示させたい内容を書いてGenerateを押します。しばらくして問題がなければ画像が生成されます。

![](https://storage.googleapis.com/zenn-user-upload/ed86cf4e072b-20240228.png)

