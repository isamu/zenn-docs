---
title: "写真をベースにAIでバーチャルアイドルを作る (Stable Diffusion Webui編)"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [stablediffusion, ai, Tech]
published: true
publication_name: "singularity"
---

## epiCRealismとReActorでバーチャルアイドルを作る

Stable Diffusionを使って、人物の顔写真を元にバーチャルアイドルを作成します。


前回の記事で、Stable Diffusion Webuiをインストールし、Stable Diffusion 1.5を動かしてみました。
https://zenn.dev/singularity/articles/stable-diffusion-webui-1

続いて、バーチャルモデルを作ってみます。

AIモデルのデータセットepiCRealismと、元データとなる顔画像を読み込むReActorというツールの２つを、Stable Diffusion Webuiに取り込み、画像データを元ネタにしてオリジナルの画像を生成する動かします。

## epiCRealism

CIVITAIというサイトにある、epiCRealismをダウンロードします。

https://civitai.com/models/25694/epicrealism

![](https://storage.googleapis.com/zenn-user-upload/729da4ec3df3-20240229.png)

`epicrealism_naturalSinRC1VAE.safetensors` というファイルがダウンロードされました。

`models/Stable-diffusion/`に置きます。

Stable Diffusion Webuiを起動します。

```
./webui.sh
```

するとStable Diffusion checkpointにepicrealism_naturalSinRC1VAE.safetensorsが追加されます。
追加されていない場合は`models/Stable-diffusion/`にファイルが置いてあるか確認をしてください。


![](https://storage.googleapis.com/zenn-user-upload/c5b277dd9812-20240229.png)

## ReActor
続いて、ReActorをインストールします。

ブラウザでExtensionsのInstall from URLを選択し、URlを入力。Installをクリックします。

urlは `https://github.com/Gourieff/sd-webui-reactor` を入れます。
![](https://storage.googleapis.com/zenn-user-upload/66b98ab3af1a-20240229.png)

しばらくするとインストールが完了。ここでwebuiを再起動します（一度、webui.shとめて再度webui.shを実行。ブラウザをreload)

GenerateタブにReAcorが追加されています。
![](https://storage.googleapis.com/zenn-user-upload/188f7e4ee353-20240229.png)

ReActorをチェックし、開きます。
![](https://storage.googleapis.com/zenn-user-upload/88a83ed772a5-20240229.png)

ImageでSingle Source Imageに元ネタとなる画像をドロップします。（ここではその画像は割愛）

Textを入れてGenerate。有名な夫人の画像を元に生成した画像が生成されます。
![](https://storage.googleapis.com/zenn-user-upload/db0f0bb829b2-20240229.png)


## まとめ
プラグラム不要で、簡単にできました！

他のmodelやextensionを試すことで、バーチャルアイドルに限らず、Stable Diffusionを使った様々な画像生成AIが使えそうです。

シンギュラリティ・ソサエティは、このように生成AIを使い、AIの検証やサービス、アプリケーションを作るコミュニティです。

https://singularitysociety.org/