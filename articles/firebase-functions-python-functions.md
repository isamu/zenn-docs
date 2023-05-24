---
title: "Firebase Functionsで新たにサポートされたPythonをSetupし、更にTypeScriptと共存させる"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Firebase, Python, JavaScript, Typescript]
published: true
publication_name: "singularity"
---

Google I/O 2023でFirebaseの新機能としてFunctionsのPythonのサポートが発表されました。
https://www.youtube.com/watch?v=emIxn-f9bK0

早速、試してみたいと思います。
まず、Pythonを含むFunctionsをデプロイするにはfirebase-toolsのバージョンを上げる必要があります (ChangeLogを追ったが、どのバージョンからPythonをサポートしているかは不明でした。)

```
npm install -g firebase-tools
```

で最新にupdate.

続いて、firebaseの初期をします。

```
firebase init 
```

言語の選択肢。Pythonがあります！

```
? What language would you like to use to write Cloud Functions? 
  JavaScript 
  TypeScript 
❯ Python
```

Pythonを選択するとfunctions以下にPythonのファイルが作成されます。

```
? What language would you like to use to write Cloud Functions? Python
✔  Wrote functions/requirements.txt
✔  Wrote functions/.gitignore
✔  Wrote functions/main.py
```

見てると、作成されていました。

```
$ ls functions 
main.py           requirements.txt  venv/
```

ファイルを見てみると、
```
# Welcome to Cloud Functions for Firebase for Python!
# To get started, simply uncomment the below code or create your own.
# Deploy with `firebase deploy`

from firebase_functions import https_fn
from firebase_admin import initialize_app

# initialize_app()
#
#
# @https_fn.on_request()
# def on_request_example(req: https_fn.Request) -> https_fn.Response:
#     return https_fn.Response("Hello world!")

```
です。コメントアウト部分を外してデプロイするとPythonのFunctionsがデプロイされました。

コンソールで確認すると

![Functions console](https://storage.googleapis.com/zenn-user-upload/5627e82342df-20230524.png)
と、v2のFunctionsでデプロイされています。

firebase.json を見た所、
```
  "functions": [
    {
      "source": "functions",
      "codebase": "default",
      "ignore": [
        "venv",
        ".git",
        "firebase-debug.log",
        "firebase-debug.*.log"
      ]
    }
  ]    
```
Funcitonsの設定がArrayになっていました！
早速調べてみた所、[複数の関数を整理する](https://firebase.google.com/docs/functions/organize-functions?hl=ja) とcodebaseで複数のFunctionsをデプロイできるようです。これを使えば、PythonとTypeScriptを１つのprojectで共存できそうです。
早速こちらも試してみます。


firebaseの初期をします。

```
firebase init 
```

Functionsを選択したところ、既存のcodebaseがあるので

```
? Would you like to initialize a new codebase, or overwrite an existing one?  
❯ Initialize 
  Overwrite
```
と聞かれたので、上書きではなく新たに作るを選択します

```
? What should be the name of this codebase? typescript
```
codebaseの名前はtypescriptとします

```
? In what sub-directory would you like to initialize your functions for codebase typescript? typescript
```
sub-directory(通常はfunctions)はtypescriptとします。

```
? What language would you like to use to write Cloud Functions?
  JavaScript
❯ TypeScript
  Python
```
言語もTypeScriptを選択

```
? What language would you like to use to write Cloud Functions? TypeScript
? Do you want to use ESLint to catch probable bugs and enforce style? Yes
✔  Wrote functions/package.json
✔  Wrote functions/.eslintrc.js
✔  Wrote functions/tsconfig.json
✔  Wrote functions/tsconfig.dev.json
✔  Wrote functions/src/index.ts
? File functions/.gitignore already exists. Overwrite? (y/N) 
```
諸々のファイルが作成される。

これで完了です。

firebaseの設定を見てみると

```
 more firebase.json 
{
  "functions": [
    {
      "source": "functions",
      "codebase": "default",
      "ignore": [
        "venv",
        ".git",
        "firebase-debug.log",
        "firebase-debug.*.log"
      ]
    },
    {
      "source": "typescript",
      "codebase": "typescript",
      "ignore": [
        "node_modules",
        ".git",
        "firebase-debug.log",
        "firebase-debug.*.log"
      ],
      "predeploy": [
        "npm --prefix \"$RESOURCE_DIR\" run lint",
        "npm --prefix \"$RESOURCE_DIR\" run build"
      ]
    }
  ]
}
```

Functionsが２つ設定されています。

```
$ ls 
firebase.json  functions/     typescript/
```

dirも指定したtypescriptが作成されています。
typescript以下には、いつものtypescriptのデプロイに必要なファイルが作成されています。


デプロイすると

```
i functions: creating Node.js 16 function test(asia-northeast1)...
i functions: creating Python 3.11 function python:on_request_example(us-central1)...
```
と、２つそれぞれの言語でデプロイされています！！

![Console](https://storage.googleapis.com/zenn-user-upload/9387f910dd28-20230524.png)

既存のprojectでも、firebase.jsonのfunctions部分をarrayに変更し、codebaseを追加、funcitonsの設定を増やし各codebaseのdirを作れば同様にデプロイできることが確認できました。
機械学習を含むプロジェクトでFirebase FunctionでPythonを使うことが簡単になりました！