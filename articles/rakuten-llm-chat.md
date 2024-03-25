---
title: "楽天が公開した日本語LLM,RakutenAI-7B-chatをARM Macで動かす"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [rakuten, llm, ai_innovators_hub]
published: true
publication_name: "singularity"
---

[シンギュラリティ・ソサエティのAI Innovators Hub](https://singularitysociety.org/activities/aihub/)に参加しています。
楽天が公開した日本語LLMをMacで動かします。
３つモデルが公開されていますが、Chat用のモデルを試します。

https://huggingface.co/Rakuten/RakutenAI-7B-chat

Usageにあるコードをコピーして実行。このコードはcudaに限定して書かれてないので期待。

```
TypeError: BFloat16 is not supported on MPS
```
となる。

```
torch_dtype=torch.float16
```
に変更したところ動いた！！

```
USER:
「馬が合う」はどう言う意味ですか
ASSISTANT:
『馬が合う』とは、価値観が合い、気がよいという意味です。元々は馬の性格をあらわす言葉で、気性、気質が同じ、すなわち「気が合う」ことに由来する言葉です。
その訳を、以下のようになった説明があります。
馬は草食動物ではありますが、昔から戦や荷役などの作業に使う動物として飼育されていました。そのため、人に飼育されると、どんな馬も人間に恐怖を抱き、警戒してしまいます。その馬が、人が近くにいても穏やかに草を食べているような状態のときに、馬が「合っている」と表現されます。
そういう馬は、人だけではなく、他の馬とでも「合っている」状態です。
そこから転じて、人や物事に対して、気遣いから自然な態度でいることが出来ることを『馬が合う』と表現するようになりました。
例えば、相棒になるような友人や、同僚に対して使う場合に用います。
「A子はどんな場面でも、おっとりした態度で、誰に対しても馬が合う性格だ」など。
```

完成したコードはこちら。

https://github.com/isamu/ai_hub/blob/master/rakuten/episode1.py