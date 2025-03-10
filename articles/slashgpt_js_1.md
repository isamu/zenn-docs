---
title: "SlashGPT jsの紹介"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [gpt, ChatGPT, LLM, Tech, SlashGPT]
published: true
publication_name: "singularity"
---

# SlashGPTとは

[SlashGPT](https://github.com/snakajima/SlashGPT/)は[中島聡](https://twitter.com/snakajima)が開発したChatGPTなどのLLMエージェントを手軽に開発するためのツールです。SlashGPTを使えば、jsonファイルを記述するだけでChatGPTを使ったLLM AI エージェントやチャットアプリを手軽に、簡単につくることができます。

# SlashGPT jsとは

オリジナルのSlashGPTはPythonで作られています。SlashGPT jsはそれをTypeScriptに移植したものです。LLMエージェントの動作を定義するManifestやFunction callingのための定義ファイル、APIに互換性があります。

Node.jsでサーバサイドでの動作と、ブラウザでの動作をサポートしています。

https://github.com/isamu/slashgpt-js/

https://www.npmjs.com/package/slashgpt

# インストール

npmで提供されています。npmコマンドやyarnコマンドなどでインストールします。

```sh
yarn install slashgpt
```

# 動作

サンプルのmanifestファイルと、function callingの動作を定義しているファイルをダウンロードします。
- [paper.yml](https://github.com/isamu/slashgpt-js/blob/main/tests/paper.yml)
- [paper.json](https://github.com/isamu/slashgpt-js/blob/main/tests/paper.json)

以下のTypeScriptをファイルに保存し、そのファイルと同じディレクトリーにこの２つのファイルをおきます。
このTypeScriptは、manifestを簡単に動かすためのsimple_clientを使ってmanifestを実行してます。

```typescript
import { justRun } from "slashgpt/lib/simple_client";

const main = async () => {
  const file_path = path.resolve(__dirname) + "/paper.yml";
  const session = await justRun(file_path);
  console.log(session);
};
main();
```
justRunの関数で、manifestファイルを読み込み、manifestファイルのsampleを実行するようになっています。
ts-nodeやtypescriptが入ってない場合は、インストールしてください。もし、入っていない場合は実行時にエラーとなります。
以下の、シェルスクリプトで実行します。OPENAI_API_KEYは、各自で用意が必要です。

```
OPENAI_API_KEY=sk-xxxxxx npx ts-node simple_client.ts
```


```
    {
      role: 'function',
      content: 'title: 注意はすべてです\n' +
        'keywords: sequence transduction, neural networks, attention mechanism, machine translation, BLEU score\n' +
        'issues: 従来の複雑な再帰型または畳み込みニューラルネットワークを使用したシーケンス変換モデルの問題を解決し、より高品質で並列化が可能なモデルを提案する\n' +
        'methods: 再帰と畳み込みを全く使用せず、注意機構のみに基づくTransformerという新しいネットワークアーキテクチャを提案\n' +
        'results: WMT 2014の英独翻訳タスクで28.4 BLEU、英仏翻訳タスクで41.8 BLEUを達成し、文献中の最良モデルよりも優れた成績を示す',
      name: 'paper_summary'
    }
```

のような結果が表示されれば成功です。

上記と同じ動作をjustRunではなく、APIを使って実装して動かします。動作については上の例と同じです。


```typescript
import fs from "fs";
import path from "path";
import YAML from "yaml";

import { readManifestData } from "slashgpt/lib/file_utils";
import { callback } from "slashgpt/lib/simple_client";

import { ChatSession } from "slashgpt";
import { ChatConfig } from "slashgpt";

const main = async () => {
  const fileName = path.resolve(__dirname) + "/paper.yml";
  const manifest_file = fs.readFileSync(fileName, "utf8");
  const manifest = YAML.parse(manifest_file);

  const config = new ChatConfig("./");

  const session = new ChatSession(config, manifest);

  session.append_user_question(manifest.sample);
  await session.call_loop(callback);

  console.log(session.history);
};

main();
```

先ほどと同様の結果が得られれば成功です。
このサンプルを参考に、サーバ等に組み込むことができます。

## まとめ

今回、Python版のSlashGPTをTypeScriptに移植したSlashGPT jsを紹介しました。
SlashGPT jsはNode.jsとブラウザの両方で動作するので、サーバに組み込むことも、フロントエンドで使うことも可能です。
manifestファイルは、Python版含めすべてのバージョンで互換性があるので、Pythonで作成したものをNode.jsのサーバに組み込むことなどが簡単にできます。

これは[シンギュラリティ・ソサエティのAI Innovators Hub](https://singularitysociety.org/activities/aihub/)で開発したプロダクトです。

