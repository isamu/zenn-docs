---
title: "__name__ == \"__main__\" のNode.js版"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [TypeScript]
published: true
publication_name: "singularity"
---


Pythonで直接呼ばれた場合はmain関数を実行するがmoduleとして呼ばれた場合には実行しない

```Python
if __name__ == "__main__":
```

と書くが、これのNode(TypeScript)版は

```TypeScript
if (process.argv[1] === __filename) {
```

でいけるはず。nodeとnpm経由で問題なく動く。