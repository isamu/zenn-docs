---
title: "日本語Vision Languageモデル heron-blip-v1をARM Macで動かす"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [turning, llm, VisionLangauge, heron]
published: true
publication_name: "singularity"
---

[シンギュラリティ・ソサエティのAI Innovators Hub](https://singularitysociety.org/activities/aihub/)に参加しています。

完全自動運転車両の開発研究を行うTuring社が公開したマルチモーダルLLMモデルをMacで使ってみます。

https://zenn.dev/turing_motors/articles/00df893a5e17b6

の記事を参考にします。
まずモデルデータや使い方を確認するためにHugging Faceのmodelのページをみます。

https://huggingface.co/turing-motors/heron-chat-blip-ja-stablelm-base-7b-v1

サンプルコードをコピペし、device部分をmpsに指定し、動かしてみたところ、heronがないのでエラーとなりました。

https://github.com/isamu/ai_hub/blob/master/turing/heron-blip-v1.py

heronはpipで用意されてないようなので、まずはheronのインストール。

ソースは以下にあります。

https://github.com/turingmotors/heron/

ソースをclone
```
git clone https://github.com/turingmotors/heron/
```

インストールは Poetry (Recommended) を参考に進めます。
cudaは入ってない環境なので、pyproject.tomlのtorchを変更します。


```
torch = { url = "https://download.pytorch.org/whl/cu121/torch-2.2.0%2Bcu121-cp310-cp310-linux_x86_64.whl"}
```
を
```
torch = "2.2.0"
```
に変更。

python は3.10が入っているので pythonのインストールは除き、

```
pyenv local 3.10

# install packages from pyproject.toml
poetry install

# install local package
pip install --upgrade pip  # enable PEP 660 support
pip install -e .

```

を実行。
問題なければ、heronのインストールは完了します。


先ほどのスクリプトを再度、実行すると

```
File "/hobby/.pyenv/versions/3.10.3/lib/python3.10/site-packages/transformers/modeling_utils.py", line 1613, in _initialize_weights
  self._init_weights(module)
 File "/hobby/turing/heron/heron/models/video_blip/modeling_video_blip.py", line 325, in _init_weights
  module.bias.data.zero_()
AttributeError: 'NoneType' object has no attribute 'data'
```
と、エラー。

エラー内容をググると、ギリアのエンジニアの方の記事で同じ問題に遭遇しているのを発見。

https://note.com/npaka/n/n5018b9465587

こちらを参考にheronのコードにパッチをあてます。

```
--- a/heron/models/video_blip/modeling_video_blip.py
+++ b/heron/models/video_blip/modeling_video_blip.py
@@ -322,8 +322,10 @@ class VideoBlipPreTrainedModel(PreTrainedModel):
             nn.init.trunc_normal_(module.class_embedding, mean=0.0, std=factor)
 
         elif isinstance(module, nn.LayerNorm):
-            module.bias.data.zero_()
-            module.weight.data.fill_(1.0)
+            if module.bias is not None:
+                module.bias.data.zero_()
+            if module.weight is not None:
+                module.weight.data.fill_(1.0)
         elif isinstance(module, nn.Linear) and module.bias is not None:
             module.bias.data.zero_()
```

再度、スクリプトを実行すると、結果が得られました。


>[' 画像の面白い点は、黄色いタクシーの荷台に立っている男性が、タクシーに積まれたアイロン台でアイロンの仕事をしていることだ。このシーンは、アイロンは通常アイロニングやプレスに使われるものであり、日常的なアイテムの典型的な使い方ではないため、興味深く、珍しい。アイロールをしている男性は、この型破りなアイデアを実行する技術と専門知識を披露している。\n<|pad|>']


これで、任意の画像＋テキストを使ってlocalで検証が可能となりました。


