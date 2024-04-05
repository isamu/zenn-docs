---
title: "Claude 3 APIがFunction calling (Tools)に対応したので試してみる"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [claude3, LLM, anthropic, Tech]
published: true
publication_name: "singularity"
---

Claude 3 APIが、OpenAIのFunction callingにあたる機能をToolという名前でパブリックベータとして公開しました。

早速試してみます。

https://docs.anthropic.com/claude/docs/tool-use

公式ドキュメントによれば、

httpのrequest headerに

```
 anthropic-beta: tools-2024-04-04
```

を追加し、request bodyにtoolsとしてスキーマの情報を送れば良いようです。
Node版sdkも対応していて、サンプルも追加されています。

https://github.com/anthropics/anthropic-sdk-typescript/commit/5bcaddbd396fa81e9b65bf2ce3b2917affae5c0a

このサンプルを参考に動かしてみました。

当初、何度かテストをしても
```json
  error: {
    type: 'error',
    error: { type: 'overloaded_error', message: 'Overloaded' }
  }
```

と高負荷な状態のエラーが返ってきました。
モデルを`claude-3-sonnet-20240229`に変更したところ、あっさりと動きました。公開直後なので、しばらくこの状態が続くかもしれませんので、同様のエラーに遭遇した場合には、モデルを変更して試すことをおすすめします。

また、通常と異なるのはmessage apiを呼ぶときに、`anthropic.messages.create` を `anthropic.beta.tools.messages.create` とベータ用のAPIにする必要があります。

npmが0.20.1より以前のものだとcrete関数の引数にtoolsという項目が定義されていないので形エラーになります。かならず最新のバージョンのものを使うようにしましょう。

```typescript
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic({
  apiKey: process.env['ANTHROPIC_API_KEY'], // This is the default and can be omitted
});

async function main() {
  const message = await anthropic.beta.tools.messages.create(
    {
      max_tokens: 1024,
      // model: 'claude-3-opus-20240229',
      model: 'claude-3-sonnet-20240229',
      tools: [
        {
          "name": "get_weather",
          "description": "Get the current weather in a given location",
          "input_schema": {
            "type": "object",
            "properties": {
              "location": {
                "type": "string",
                "description": "The city and state, e.g. San Francisco, CA"
              }
            },
            "required": ["location"]
          }
        }
      ],
      "messages": [
        {
          "role": "user",
          "content": "What is the weather like in San Francisco?"
        }
      ]
    },
  );

  console.log(message.content);
}

main();
```

以下が結果です。

```json
[
  {
    type: 'text',
    text: "Okay, let's get the current weather for San Francisco using the provided tool:"
  },
  {
    type: 'tool_use',
    id: 'toolu_019cy4213qsNVZsFWJEeve6P',
    name: 'get_weather',
    input: { location: 'San Francisco, CA' }
  }
]
```

若干項目の差はありますが、OpenAIのFunction Callingの実装をそのまま変更して使えそうです。