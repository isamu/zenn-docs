---
title: "GraphAIで複雑な依存関係のあるタスクを最適な順に実行する"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [nodejs, Async, GraphAI, gpt]
published: true
publication_name: "singularity"
---


## 複雑な異存のある非同期タスク

非同期は難しいです。非同期は難しいです。非同期は難しいです。

プログラムではなく日常のタスクをサンプルに考えてみます。

朝起きてから家を出るまでのタスク例として書き出します。
左の()内がタスクのid. 右の()は、依存するタスクidとそれぞれの実行にかかる時間です。
コーヒーに着目すると、(1)でお湯を沸かし、(5)でドリップし、(10)でのみ終え、(11)で食器を洗います。(8)の朝食も(5)に依存しています。
この順番は入れ替えることもできませんし、前のタスクの終了を待たないと次のタスクは実行できません。お湯が湧いたら適切な温度ですぐにドリップしたいですし、のんびりしていると遅刻してしまうので、前のタスクが終わり次第、できる限りすぐに次にタスクを実行したいです。

このように日常はタスクが複雑に依存し、それぞれが非同期に動いています。

#### 起きたらすぐ実行するタスク

- (1) コーヒーのお湯をわかす (４分)
- (2) 子供を起こす
  - 3分おきの5回呼ぶと起きる。ただし、3分以内の呼んだり、5分以上間隔が開くと、最初からやり直し
- (3) 朝食をつくる (15分)
- (4) いつも遅れる新聞をとりにいく
  - 5分以上間隔をあけて４回見に行くと、新聞はきている。ただし5分未満で見に行くと 最初からやりなおし

#### 依存タスク

- (5) お湯が湧いたらコーヒーをいれる(1)	(5分)
- (6) 子供が起きたら着替える(2)	(6分)
- (7) 子供が着替えおわったら洗濯機をまわす(6) (13分)
- (8) 朝食ができる && コーヒーができる&& こども着替えおわるの条件を満たすと朝食を食べる(3)(5)(6) (９分)
- (9) 洗濯機終了後、洗濯を干す(7) (5分)
- (10) コーヒーができていて、洗濯が終わっていたらコーヒーを飲み終える(5)(9) (５分)
- (11) 朝食終了 && コーヒーを飲み終えると食器を洗う(10)(8) (４分)
- (12) 朝食終了後、子供が用意する(8) (３分)
- (13) 新聞到着していたら新聞が読む(4) (8分)
- (14) 13までのタスクが全て終わっていたら家出発(1~13)

## Promise.allを使う

これらのタスクをプログラム化し、非同期で実行しようと考えます。

Twitterのスレッドをみると、Promise化して、async/awaitで処理する、まとめたい部分はPromise.allで待つ、という意見が多いです。

https://twitter.com/snakajima/status/1784773464030949547

しかし、タスク(1)(2)(3)(4)をPromise.allで待つと、(4)のタスクが20分以上かかりますが、その間４分で終わった(1)のタスクがブロックされます。

仮に、Promise.allで待ったとして、次に実行できるタスクは、(5),(6),(13)のみです。

このようにPromise.allを使って、可能なタスクを順次実行していくと、確実に遅刻してしまうことがわかります。

タスクが終わったらcallbackさせ、タスク管理をして次に動かすものを判定するとできそうですが、直感的に大変そうなのがわかります。


#### ChatGPTに聞いてみる

チャットGPTに相談してみましたが、なかなか最適なコードはつくれませんでした。
課題を整理して、新しいモデルで試せば完璧なものができるかもしれません。だれか試してください。

https://chatgpt.com/share/88e6f773-eea0-4954-82cf-34192c13cd76?oai-dm=1


### GraphAI

そこでGraphAIの登場です。
GraphAIは、各タスクを１つ１つの独立したTypeScriptのコードのAgentとして用意し、その依存関係をyaml/json/dataで定義すれば、依存関係を考慮して順次並列に実行するTypeScriptのエンジンです。

https://github.com/receptron/graphai/


タスクは単純にsleep関数を用意してn秒だけ待つように実装しています（分を秒に読み替えている）

```TypeScript
const sleep = async (milliseconds: number) => {
  return await new Promise((resolve) => setTimeout(resolve, milliseconds));
};

const task1 = async (time: number) => {
  console.log("start task 1");
  await sleep(time * 1000)
  console.log("end task 1");
};
```

それらの各タスクの依存関係を定義します。今回はTypeScriptで書きましたが、yamlやjsonで書くことも可能です。その場合は、Angetも別の方法で渡します（別記事で解説予定）

各タスクで実行するプログラムをagentに渡し、依存があるタスクはinputsでその依存するタスクの名前(object/dictonaryのkey)を指定します。

```TypeScript
{
  task1: {
    agent: task_1_coffee_water,
  },
  task2: {
    agent: task_2_wakeup,
  },
  task3: {
    agent: task_3_cooking,
  },
  task4: {
    agent: task_4_newspaper,
  },
  task5: {
    agent: task_5_cooking_drip,
    inputs: [":task1"],
  },
  task6: {
    agent: task_6_change,
    inputs: [":task2"],
  },
  task7: {
    agent: task_7_wash,
    inputs: [":task6"],
  },
  task8: {
    agent: task_8_breakfirst,
    inputs: [":task3", ":task5", ":task6"],
  },
  task9: {
    agent: task_9_laundry,
    inputs: [":task7"],
  },
  task10: {
    agent: task_10_coffee,
    inputs: [":task5", ":task9"],
  },
  task11: {
    agent: task_11_wash_dish,
    inputs: [":task8", ":task10"],
  },
  task12: {
    agent: task_12_ready_to_go_out,
    inputs: [":task8",],
  },
  task13: {
    agent: task_13_read_news_paper,
    inputs: [":task4",],
  },
  task14: {
    agent: task_14_go_out,
    inputs: [":task1", ":task2",":task3",":task4",":task5",":task6",":task7",":task8",":task9",":task10",":task11",":task12",":task13",],
  }
```

こちらに完成品をおいておきますが、各タスクを関数として用意します。
https://github.com/isamu/graphai_doc/blob/main/samples/morning/tasks.ts


これを
```
npx ts-node tasks.ts
```
で実行すると、非同期に依存を考慮して実行される様子が見えます。

朝の複雑なタスクも、人間が実行しているのと同様に効率よく非同期に実行することが可能となりました。

### まとめ

GraphAIを使うことで各タスクは独立した関数として定義でき（なので、テストも簡単）、依存するタスクはデータとして簡単に定義が可能になります。

今回のサンプルにはありませんが、Agent同士で結果を入力として受け渡すことも可能です。
