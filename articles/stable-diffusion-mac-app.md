---
title: "Stable Diffusionのデスクトップアプリを作る"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [stablediffusion, python, 機械学習, mac, アプリ]
published: true
publication_name: "singularity"
---

[シンギュラリティ・ソサエティのAI Innovators Hub](https://singularitysociety.org/activities/aihub/)に参加しています。

# はじめに

PythonでAIツールを作り他の人に使ってもらおうとすると、Webサービス化する、もしくはPythonのソースを共有しパソコンやGoogle colabで動かしてもらう必要があります。

Webサービス化するのは、AI系の処理は時間がかかる事とサーバのコスト問題で、現状ではあまり良い方法ではありません。Pythonのコードを使う場合は、使う側にある程度の知識が要求されます。

そこで、Pythonで書いたツールをアプリ化し、インストールするだけで使えるようにしたいと思います。

アプリ化には

- OSのnaviteのUIから使えるようにする
- パッケージ化して配布する

が必要となります。

## UIツール

MacでPythonを使ってUIを作るために、いくつからライブラリーがあります。

- Tkinter
  - Tcl/TkのPythonパッケージ。古いので非推奨
- PyQt5
  - Qtのライブラリ。GPL
- PySide2
  - Qtのライブラリ。LGPL。

PyQt5とPySide2はほぼ同じAPIで互換性があるようです。今回は情報が多いのでPyQt5を使うことにします。

## パッケージ化

pyinstallerやpy2appといったツールでPythonのスクリプトを、実行環境込みで配布できるアプリ化できます。

両方を検証したところpy2appは今回のケースではパッケージ化に失敗したのでpyinstallerを使います。

## 作り方

ByteDanceのSDXL-Lightningを使って、テキストから画像を生成するアプリを作ります。

https://huggingface.co/ByteDance/SDXL-Lightning

完成品はこちら。
QTでUIを作り、入力されたテキストをStableDiffusionに渡して画像生成、その画像を/tmp以下に保存しつつpreviewとして表示させます。


https://github.com/isamu/ai_hub/blob/master/stable-diffusion/ui2.py

必要に応じてパッケージを追加します。
```
pip install pyinstaller PyQt5 diffusers
```

```
python ui2.py
```
で起動させ、動作確認をします。問題なければ


```
pyinstaller ui.py --noconsole
```
でパッケージ化します

```
--onefile
```
オプションをつけるとファイルがまとめられる＆パッケージが小さくなるのですが起動に時間がかかるので外しています。
今回のスクリプトだとファイルサイズは300M/1Gと、かなり大きな違いがあります。

dist以下にアプリが作成されます。Macから見るとアプリなのですが、uploadしようとするとfolderであることがわかります。zipで圧縮して人に渡すとよいです。
![](https://storage.googleapis.com/zenn-user-upload/ce870a70898d-20240314.png)

普通にPythonスクリプトのように作ってパッケージ化すると

- 起動が遅い
- 多重起動する

の問題がありました。

起動が遅い問題に関しては
 - 起動時にtransformarの初期化処理は走らせない
 - (Pythonの問題ではないですが) pyinstallerでパッケージ化するときにonefileをつけない

多重起動の問題については
 - lockファイルを作って多重起動を防止する

として回避しています。

配布されたアプリを実行したイメージはこちらです。
![](https://storage.googleapis.com/zenn-user-upload/2e1048fc90e1-20240314.png)

pyinstallerでbuildされたアプリをzipでまとめることにより、他の人に使ってもらうことが容易になりました。



