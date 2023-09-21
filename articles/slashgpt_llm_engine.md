---
title: "SlashGPTのプラグインを使って新しいLLMを追加する"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [gpt, ChatGPT, LLM, Tech, OpenAI]
published: true
publication_name: "singularity"
---

[SlashGPT](https://github.com/snakajima/SlashGPT/)は、No CodeでLLMエージェント/LLMアプリを作成、利用できるツールです。
利用できるLLMは事前に用意されているOpenAIのAPI経由でのGPT3.5, GPT4や、palm, llama2 などがありますが、プラグインを書くことにより、簡単に他のLLMを使うことも可能です。

LLM独自の動作部分だけプラグインを記述することにより、SlashGPTのすべての機能をそのLLMで使うことが可能となります。API経由でのLLM利用だけでなく、オープンソースで公開されているLLMモデルをlocal環境で呼び出すことも可能です。

今回はLLMを追加するために、LLMプラグインの作り方と、設定方法を解説します。


# LLMプラグインの追加

今回は[bilingual-gpt-neox-4b-instruction-sft](https://huggingface.co/rinna/bilingual-gpt-neox-4b-instruction-sft/blob/main/README.md)をプラグインとして追加します。

このモデルは、[huggingface](https://huggingface.co/)で配布されていますが、SlashGPTでhuggingfaceを使うプラグインは[plugins/engine/from_pretrained.py](https://github.com/snakajima/SlashGPT/blob/main/plugins/engine/from_pretrained.py) に用意されています。

実装の処理が同じであれば、このプラグインをそのまま使うことができますが、プロンプトの作り方とmodelに渡すパラメーターが異なるので、新しいプラグインとして追加が必要です。

from_pretrained.pyをコピーしてfrom_pretrained2.pyを作ります。
クラス名は、LLMEngineFromPretrained2とします。
[bilingual-gpt-neox-4b-instruction-sft](https://huggingface.co/rinna/bilingual-gpt-neox-4b-instruction-sft)の使い方は、[bilingual-gpt-neox-4b-instruction-sft](https://huggingface.co/rinna/bilingual-gpt-neox-4b-instruction-sft)の[README](https://huggingface.co/rinna/bilingual-gpt-neox-4b-instruction-sft/blob/main/README.md)に書いてありますが、from_pretrained.pyと異なるのは、

- プロンプトのRole
- tokenizer.encodeとmodel.generateに渡すパラメータ

なので、そこを変更します。変更点が気になる方は
```
diff from_pretrained.py from_pretrained2.py
```
で差分を確認をしてください。

変更後のfrom_pretrained2.pyは[こちら](https://github.com/snakajima/SlashGPT/blob/main/plugins/engine/from_pretrained2.py)です。
これでプラグインの実装は終わりです。
ほとんど、bilingual-gpt-neox-4b-instruction-sftのREADMEをコピーするだけです。

動作が異なる新たなプラグインを作る場合でも、LLMEngineBaseを継承したクラスを作り、chat_completion methodを実装するだけです。


# プラグインを利用するための設定追加

作成したLLMプラグインをSlashGPTで利用するには設定の変更が必要です。
LLMの設定ファイルは [config/llm_config.py](https://github.com/snakajima/SlashGPT/blob/main/config/llm_config.py) です。

llm_modelsはSlashGPTのプロンプトで/llmコマンドでllmを指定したときに使われる設定、llm_engine_configsがLLMエンジンの設定です。

llm_modelsは、llmを指定したときに利用するエンジンとそのモデルの設定、llm_engine_configsはエンジンのクラス、もしくはクラスのパスとクラス名を指定します。

さて、今作ったプラグインを追加します。

### llm_engine_configs

エンジン名は`from_pretrained-rinna`として、llm_engine_configsにクラス名をパス(module_name)を追加します。プラグインのパスはplugins.engineなので、それにファイル名を追加したものを指定、class_nameはそのファイルのクラス名を指定します。
```
    "from_pretrained-rinna": {
        "module_name": "plugins.engine.from_pretrained2",
        "class_name": "LLMEngineFromPretrained2",
    },
```

### llm_models

SlashGPTのプロンプトからは
```
/llm rinna
```
で呼び出すとします。

この設定をllm_modelsに追加します。
エンジン名は先程追加した`from_pretrained-rinna`、モデル名は実装時にfrom_pretrainedに渡す値にしているので`rinna/bilingual-gpt-neox-4b-instruction-sft`を指定します。
```
    "rinna": {
        "engine_name": "from_pretrained-rinna",
        "model_name": "rinna/bilingual-gpt-neox-4b-instruction-sft",
        "max_token": 4096,
    },
```

これで設定は完了です。

LLMの動作が同じであれば、model_nameを変えたものをllm_modelsに追加すれば、複数のmodelを試すこともできます。


今回実装したものはGitHubのcommitとして差分を確認できます
https://github.com/isamu/SlashGPT/commit/e471e287f7346cadb7e431668aec1b2b19f24394

### 動作確認

llmをrinnaに指定します。
```
You(dispatcher): /llm rinna
```

初回はモデルデータをダウンロードするので（8Gくらい！！）少し時間がかかります。

(transformersのwarningがでますが、ひとまず気にしない）

サンプルにある質問を試す

```
You(dispatcher): VRはなんですか。
Dispatcher: 「VR」は「Virtual Reality」の略語であり、仮想現実を利用して、現実のような感覚を提供する技術です。VRヘッドセットを着用することで、ユーザーは周囲の世界や仮想的な世界を感じることができます。

VRヘッドセットにはさまざまな種類があり、仮想現実内でさまざまな活動を行うことができます。たとえば、仮想的なオフィス環境を作成して作業したり、仮想的なペットを世話したり、仮想的なゲームで遊んだりすることができます。</s>
```

動きました！！！

# まとめ

今回、SlashGPTのプラグイン機能を使って、localで動かすLLモデル、bilingual-gpt-neox-4b-instruction-sftを追加しました。
簡単に実装ができました！！



### 関連記事

- [SlashGPTの使い方](https://zenn.dev/singularity/articles/slashgpt_tutorial_1)
- [SlashGPTのマニュフェストファイルの解説](https://zenn.dev/singularity/articles/slashgpt_tutorial_2)
- [SlashGPTのFunction callingの動作原理](https://zenn.dev/singularity/articles/slashgpt_spacex)
- [SlashGPTを使ってチョムスキーさんと田原さんを討論させる](https://zenn.dev/singularity/articles/slashgpt_agents)
- [SlashGPTのFunction callingの詳細](https://zenn.dev/singularity/articles/slashgpt_function_calling)

- [SlashGPTで、No-CodeでAIエージェントを作る方法](https://zenn.dev/snakajima/articles/adf436f7f794b3) by 中島さん
- [SlashGPT + Jupyter](https://zenn.dev/singularity/articles/718cba24bf1275) by kozayupapaさん
