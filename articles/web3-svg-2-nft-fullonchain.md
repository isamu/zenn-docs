---
title: "Web上で画像からフルオンチェーンのNFTを簡単に作成するツール"
emoji: "🍣"
type: "tech"
topics: [Web3, Ethereum, SmartContract, solidity]
published: true
publication_name: "singularity"
---


従来、フルオンチェーンのNFTを作る場合は、SVGの画像を元にスマートコントラクトのコードにSVGを直接埋め込んだり、SVGを生成するようなコードを記述する必要がありました。手軽にNFTを作成するためにWebブラウザ上でSVGの画像を選択するだけで、Ethereum上でフルオンチェーンのNFTを生成するツールのα版リリースしました。

α版の為、Ethereumのmain netではなくGoerli Testnetで動作します。Goerli TestnetのETHは無料で入手できる為、費用をかけないで動作検証をすることが可能です。


## 使い方

1. Goerli TestnetのETHを入手する
  - https://goerlifaucet.com/ に行き、自身のGoerliのwalletのアドレスを入力すると入手できます。
  - Alchemyのユーザ登録は必要です。(MetaMaskは必要です）

2. SVGの画像を用意する
 - NFTにしたい画像をSVG形式で用意します。
 - SVG画像がない場合には以下にいくつかサンプルがあるのでそちらも利用可能です。変換可能な画像は後述する制約があります。
   - https://github.com/Cryptocoders-wtf/generative/tree/main/src/svgs/emoji
   - https://fullyonchain.xyz/svgtest

3. NFTに変換する
  - 今回リリースしたMintサイトに行きます。
    - https://fullyonchain.xyz/svg
  - 2で用意した画像を選択します。
  - Uploadする画像とNFTに変換後の画像のプレビューが表示されるので、NFTに変換後の画像が表示されていれば、おおよそ変換は可能です。

![](/images/SVG2NFT/imageperview.png)


  - ファイルサイズが大きい場合などは変換に失敗する可能性があります。mintボタンを押してしばらくして、下に画像が追加されれば成功です。
  - 「Sorry, this image is not supported」とエラーが出た場合には失敗です。
    - 失敗した場合は、変換できなかった画像を報告いただければ出来る範囲で原因を調査して、変換できるようにしていきたいと思います。

![](/images/SVG2NFT/sorry.png)


## 変換するSVG画像の制約
- NFTで表示される画像は、path要素のd属性のみで構成されてものです。
- 本ツールではSVGファイルを解析し、path要素のd属性のみの画像に変換します。
  - d 属性に変換されるのはcircle, ellips, line, rect, polygon, polylineのみ。それ以外の要素は無視します。
  - g 要素はフラットに展開されます。入れ子構造になっている場合、元の入れ子の順番通りには展開されないので、レイヤーの重なりが壊れる（入れ替わる）可能性があります。
  - transformで移動や拡縮/回転は変換されません。transformを無視して読み込みや変換をするので、変換に成功しても正しく表示されない可能性があります。
  - fillはそのまま維持されます。
  - stroke, stroke-widthはサポートします。ただし、stroke-linecap:round;stroke-linejoin:roundが必ず指定される。widthは正規化されます。
  - 画像のサイズはviewBox、もしくはsvg propertiesのwidth, heightの大きさを元に1024*1024に正規化されます。
- 画像データをブロックチェーンに書き込むため、ファイルサイズには敏感です。極力小さなファイルサイズにしてください。大きいと失敗します。

画像ツールに詳しくないため、Adobe Illustratorやinkspaceで上記変換に対応できるファイルの書き出し方のノウハウを募集します。よろしくおねがいします。


