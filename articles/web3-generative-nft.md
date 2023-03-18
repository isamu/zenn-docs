---
title: "Fully-On-Chain.solで作るフルオンチェーンNFT"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Web3, NFT, Solidity, Ethereum, Tech]
published: true
publication_name: "singularity"
---

Fully-On-Chain.solは[Satoshi Nakajima](https://github.com/snakajima)が開発しているフルオンチェーンのスマートコントラクト用のグラフィックライブラリです。スマートコントラクト上で、少ないデータ量、少ない計算処理でSVGの画像を生成するためのツールです。主な用途フルオンチェーンのNFTのSVG画像の処理に使われます。

https://github.com/Cryptocoders-wtf/generative/tree/main/contract/contracts/packages/graphics

公式サイトにいくつか使い方の例があるのですが、公式の例を元にシンプルなNFTを作成しました。今回紹介するNFTのスマートコントラクトは、そのままEthereumのmainnetにデプロイ可能です。自由に書き換えて、フルオンチェーンのNFTの作成に利用してください。
本稿では、localでhardhatのnodeを動かし、localへデプロイすることを前提としています。


# 準備

[https://github.com/Cryptocoders-wtf/generative](https://github.com/Cryptocoders-wtf/generative)

から最新のコードを取得
```
git clone git@github.com:Cryptocoders-wtf/generative.git
```

`generative/contract`以下で

```
npm install
```
をして必要なnpmパッケージをインストール。

`generative/contract/tools`以下でも同様に

```
npm install
```
を実行。

`generative/contract`以下で

```
 npx hardhat node 
```

を実行して、ローカルでEthereumノードを動かします。
これはデーモンなのでテスト中は起動した状態にする必要があります。

# ソースについて

`generative/contract/contracts` [以下](https://github.com/Cryptocoders-wtf/generative/tree/main/contract/contracts/)がスマートコントラクトのソースです。ここに[フルオンチェーンのサイト](https://fullyonchain.xyz/)にデプロイ済のNFTのコードがあります。本ドキュメントを読み終えた後は、他のコードを読むことでさらに理解が深まります。

本稿で使用する[サンプル](https://github.com/Cryptocoders-wtf/generative/tree/main/contract/contracts/samples)は`generative/contract/contracts/samples/` にあります。全てのNFTの規格であるERC721を使ったスマートコントラクトです。NFTはほとんどの処理が共通で、画像の関する処理のみ異なります。Fully-On-Chain.solを使うSVGの画像処理は全てgeneratePathの関数内にあります。

## SimpleSVGToken
SimpleSVGToken.solは[SVGライブラリ](https://github.com/Cryptocoders-wtf/generative/tree/main/contract/contracts/packages/graphics#svg-elements)を使うサンプルです。


SVGライブラリを使うと、SVGをメソッドチェーンを使って記述することが可能です。


```
  function generateSVGPart(uint256 _tokenId) public pure returns (string memory svgPart) {
     SVG.Element memory path = SVG.rect(256, 256, 512, 512)
         .fill("yellow")
         .stroke("blue", 10)
         .transform("rotate(30 512 512)");
         
     svgPart = string(path.svg());
  }
```

生成されるSVGは
```
<rect x="256" y="256" width="512" height="512" fill="yellow" stroke="blue" stroke-width="10" transform="rotate(30 512 512)"/>
```
です。

`generative/contract`以下で

```
 npx hardhat run scripts/deploy_simpleSVG.ts  
```

とするとデプロイできます。

[ドキュメント](https://github.com/Cryptocoders-wtf/generative/tree/main/contract/contracts/packages/graphics#svg-elements)を参考に、様々なSVGを作ることができます。SVG.groupを使うと複数のSVGを組み合わせることもできます。

## SimplePathToken
SimplePathToken.solは、[VectorとPath](https://github.com/Cryptocoders-wtf/generative/tree/main/contract/contracts/packages/graphics#path)を使うサンプルです。

Vectorは２次元のベクトルを扱うライブラリで、ベクトルの加減乗除と回転ができます。
PathはSVGのべジュエ曲線を扱うライブラリで、イラストレータで書くようななめらかな曲線を描くことができます。
VectorとPathはセットで使い、Vectorでポイントを指定してPathで線をつなぎます。

```
  function generatePath(Props memory _props) public pure returns (bytes memory path) {
    uint count = _props.count;
    int radius = 511; // We want to fill the whole viewbox (1024 x 1024)
    int length = int(_props.length);
    Vector.Struct memory center = Vector.vector(512, 512); // center of the viewbox
    uint[] memory points = new uint[](count * 2);
    for (uint i = 0; i < count * 2; i += 2) {
      int angle = (Vector.PI2 * int(i)) / int(count) / 2;
      Vector.Struct memory vector = Vector.vectorWithAngle(angle, radius);
      points[i] = Path.roundedCorner(vector.add(center));
      points[i + 1] = Path.sharpCorner(vector.mul(length).div(100).rotate(Vector.PI2 / int(count) / 2).add(center));
    }
    path = points.closedPath();
  }
```

生成されるSVGは
```
<path d="M833 505 C937 508 937 514 833 518 L644 524 828 566 C930 589 929 594 826 578 L639 549 812 625 C908 667 906 671 808 636 L630 572 785 679 C871 738 868 743 778 690 L616 594 748 729 C821 803 817 807 739 738 L598 612 703 769 C761 856 756 859 693 776 L578 626 651 801 C691 897 685 899 639 806 L555 637 593 822 C614 924 609 925 581 825 L530 643 533 832 C534 936 528 936 520 832 L505 644 472 830 C453 933 447 932 459 828 L480 640 412 817 C374 914 369 913 400 813 L456 632 356 793 C300 881 295 879 345 787 L435 620 306 758 C234 834 230 831 296 750 L415 603 262 715 C177 776 174 772 255 705 L400 583 229 664 C134 709 131 704 223 653 L388 561 204 608 C102 633 101 628 201 596 L381 537 192 548 C87 554 87 549 191 536 L379 512 191 487 C87 473 87 467 192 474 L381 486 201 427 C101 394 102 389 204 415 L388 462 223 370 C131 319 134 314 229 359 L400 440 255 318 C174 250 177 246 262 308 L415 420 296 273 C230 191 234 187 306 264 L435 403 345 236 C295 144 300 141 356 230 L456 391 400 210 C369 109 374 108 412 206 L480 383 459 194 C447 90 453 89 472 192 L505 379 520 190 C528 86 534 86 533 191 L530 380 581 197 C609 96 614 97 593 200 L555 386 639 216 C685 122 691 125 651 222 L578 397 692 246 C755 163 760 166 702 253 L598 411 739 285 C817 215 821 219 748 294 L616 429 778 332 C868 278 871 283 785 343 L629 451 807 386 C905 350 908 355 812 398 L639 474 826 444 C929 427 930 433 828 457 L644 499 833 505 " id="1"/>
```

です。

`generative/contract` 以下で
```
 npx hardhat run scripts/deploy_simplepath.ts  
```

とするとデプロイできます。

## SimpleVectorToken

SimpleVectorToken.solは,SVG画像を取り込むためのサンプルです。イラストレータなどで作成したSVGデータを圧縮しNFT化できます。
具体的にはSVGのPathのd要素のデータを圧縮し、スマートコントラクトで解凍するサンプルです。SVGデータをそのままスマートコントラクトに乗せるのはデータサイズの問題で現実的でないので、データを正規化、圧縮してスマートコントラクトに乗せる方法を用意しました。

まずはSVGの画像データを圧縮します。元となるSVGファイルは、

```
<svg viewBox="0 0 1024 1024"  xmlns="http://www.w3.org/2000/svg">
<g transform="scale(0.25)"> 
<path d="M2947.77 1754.38c40.72,-272.26 -166.56,-418.61 -450,-516.24l91.95 -368.8 -224.5 -55.94 -89.51 359.09c-59.02,-14.72 -119.63,-28.59 -179.87,-42.34l90.16 -361.46 -224.36 -55.94 -92 368.68c-48.84,-11.12 -96.81,-22.11 -143.35,-33.69l0.26 -1.16 -309.59 -77.31 -59.72 239.78c0,0 166.56,38.18 163.05,40.53 90.91,22.69 107.35,82.87 104.62,130.57l-104.74 420.15c6.26,1.59 14.38,3.89 23.34,7.49 -7.49,-1.86 -15.46,-3.89 -23.73,-5.87l-146.81 588.57c-11.11,27.62 -39.31,69.07 -102.87,53.33 2.25,3.26 -163.17,-40.72 -163.17,-40.72l-111.46 256.98 292.15 72.83c54.35,13.63 107.61,27.89 160.06,41.3l-92.9 373.03 224.24 55.94 92 -369.07c61.26,16.63 120.71,31.97 178.91,46.43l-91.69 367.33 224.51 55.94 92.89 -372.33c382.82,72.45 670.67,43.24 791.83,-303.02 97.63,-278.78 -4.86,-439.58 -206.26,-544.44 146.69,-33.83 257.18,-130.31 286.64,-329.61l-0.07 -0.05zm-512.93 719.26c-69.38,278.78 -538.76,128.08 -690.94,90.29l123.28 -494.2c152.17,37.99 640.17,113.17 567.67,403.91zm69.43 -723.3c-63.29,253.58 -453.96,124.75 -580.69,93.16l111.77 -448.21c126.73,31.59 534.85,90.55 468.94,355.05l-0.02 0z" fill="#F7931A" />
</g>
</svg>
```

このデータを元に圧縮データを作ります。圧縮するツールは`generative/contract/tools/src/sample.ts`にあります。TypeSciptで記述しています。

上記SVGデータのpathエレメントのd要素内のデータを`const bitcoin`に代入し、
```
const data = solidityString(compressPath(normalizePath(bitcoin, 1024 * 4), 1024));
```
で正規化、圧縮、スマートコントラクト用に形式に変換しています。
グラフィックライブラリのスマートコントラクトでは、1024の正規化されたデータを必要とします。正規化するために normalizePath に、元画像の縦横の最大サイズを渡す必要があります。サンプル画像はSVGは1024サイズで、pathのデータがtransformで0.25 = 1/4しているので、元々のデータは 1024 * 4のサイズを指定しています(`1024*4`)

```
npx ts-node ./src/sample.ts   
```
で実際に変換スクリプトを動かすことができます。

生成されてデータをSimpleVectorToken.solで利用しています。
`bytes constant bitcoin`に、上記スクリプトで生成した圧縮データが入っています。デコードして表示する部分はシンプルです。
また、基本的にはSVGのライブラリを使用するので、解凍したデータをfillやrotateなどSVGライブラリで扱うことが可能です。
```
  function generateSVGPart(uint256 _tokenId) public pure returns (string memory svgPart) {
      SVG.Element memory path = SVG.path(Path.decode(bitcoin)).fill("#F7931A");
      
     svgPart = string(path.svg());
  }
```

本サンプルは１つのpath dを圧縮、利用していますが、このサンプルを元に複数のデータを用意すれば、複雑な画像もフルオンチェーンに変換できます。
また、扱えるデータはpathのd要素のみなのですが、circle,polygon,line,rectなどは全てpathに変換可能なので、一般的な画像は全て扱えます。


`generative/contract` 以下で
```
 npx hardhat run scripts/deploy_simplevector.ts  
```

とするとデプロイできます。


## SimpleFontToken
SimpleFontToken.solは、ブロックチェーン上のフォントデータを使ってSVG画像に文字を書くサンプルです。
フォントデータのある他のスマートコントラクトを呼び出す必要があるため、mainnetやtestnetなどの実際のEthreum上で動かす必要があります。
hardhatでは、mainnet forkingと呼ばれるmainnetのコピー環境でテストができるため、実際のEthreumと同等の環境でEthを使わないで動作させることができます

alchemyでapikeyを取得し、
```
 run npx hardhat node --fork https://eth-mainnet.alchemyapi.io/v2/xxxxxx"
```
のようにローカルでEthereumノードを動かします。

Fontを表示するコードは以下です。現在、英数字のフォントのみなので英語の文章を渡しています。
改行はできないので１行ずつSVG.textで作成し、２行目以降はtransformを使って移動させます。フォント表示もSVGライブラリの一部なので、SVGライブラリを使って表示の制御が可能です

```
  function generateSVGPart(uint256 _tokenId) public view returns (string memory svgPart) {
      SVG.Element memory path = SVG.group([
                                           SVG.text(font, "hello"),
                                           SVG.text(font, "world").transform('translate(0 1024)')
                                           ]).transform('scale(0.4)');
 
     svgPart = string(path.svg());
  }
```

SVGの出力は長いので省略。フォントデータなので、上記で紹介した単純なSVGと比較してデータ量は多いです。


`generative/contract` 以下で
```
 npx hardhat run scripts/deploy_simplefont.ts  
```

とするとデプロイできます。


デプロイ時にデプロイスクリプトで

```
const lonrinaFont = addresses.londrina_solid["mainnet"];
const contract = await factory.deploy("SimpleFontToken", "Font", lonrinaFont);
```

と、lonrinaFontのブロックチェーンのアドレスを渡しています。このアドレスを元に、SimpleFontTokenからlonrinaFontの入ったスマートコントラクトを呼び出すことができます。goerliなどのテストネットで動かす場合、ここのアドレスを変更してgoerliにデプロイしてください。各環境のスマートコントラクトのアドレスは `src/utils/addresses` を参照してください。


# まとめ

これらのフルオンチェーンのグラフィックライブラリを組み合わせて使えば多彩なフルオンチェーンのNFTが作成可能です
[フルオンチェーン](https://fullyonchain.xyz/)のWebサイトに有るNFTのほぼ全ては、この仕組みを使って作成されたフルオンチェーンNFTです。

NFTではありませんが、これらの要素を盛り込んだコントラクトの実践的なサンプルが [test](https://github.com/Cryptocoders-wtf/generative/tree/main/contract/contracts/test)にあります。こちらのSVGと名前についたスマートコントラクトも参考にどうぞ。

# 今回作成したNFT

今回作成したNFTをgoerliにデプロイしています。opensea, etherscanのアドレスは以下です。

SVG Token
https://testnets.opensea.io/ja/assets/goerli/0x02893397ff87e8a07b24b6aed1d2dc4d39a96ab0/1
https://goerli.etherscan.io/address/0x02893397Ff87e8A07B24B6Aed1D2Dc4d39a96Ab0#code

SVG Path

https://testnets.opensea.io/ja/assets/goerli/0x61c6d58fbe47b103a8eb8104ab2250f9cfffcc2d/1
https://goerli.etherscan.io/address/0x61c6d58fbe47b103a8eb8104ab2250f9cfffcc2d#code

SVG Vector

https://goerli.etherscan.io/address/0x7902593f20414894F8eC647b2BE540320e080425#code
https://testnets.opensea.io/ja/assets/goerli/0x7902593f20414894F8eC647b2BE540320e080425/1

SVG Font
https://goerli.etherscan.io/address/0xeE9FBCb10527EA23c3eCe3536022d41F682D4536#code
https://testnets.opensea.io/ja/assets/goerli/0xeE9FBCb10527EA23c3eCe3536022d41F682D4536/1