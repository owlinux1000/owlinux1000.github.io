---
title: "Textlint_introduction"
date: 2021-10-05T20:33:36+09:00
draft: false
---

## とりあえず使ってみる

```sh
$ npm init --yes
Wrote to /Users/chihiro/Desktop/workspace/textlint_template/package.json:

{
  "name": "textlint_template",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

```sh
$ npm install --save-dev \
    textlint \
    textlint-rule-preset-ja-spacing \
    textlint-rule-preset-ja-technical-writing \
    textlint-rule-spellcheck-tech-word
npm install --save-dev \
                                             textlint \
                                             textlint-rule-preset-ja-spacing \
                                             textlint-rule-preset-ja-technical-writing \
                                             textlint-rule-spellcheck-tech-word

added 358 packages, and audited 359 packages in 36s

98 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities
```

`textlint`の設定ファイルを初期化して作成する。

```sh
$ npx textlint --init
.textlintrc is created.
$ cat .textlintrc
{
  "filters": {},
  "rules": {
    "preset-ja-spacing": true,
    "preset-ja-technical-writing": true,
    "spellcheck-tech-word": true
  }
}
```

次に、校正対象の`sample.md`を作成しておく。
```
```


`textlint`は、実行にかなり時間がかかるため、キャッシュを作成することで、その遅さを少し改善できる。本コマンドを実行することで、`.textlintcache`呼ばれるキャッシュファイルが成される。

```sh
$ textlint --cache "content/**/*.md"
$ ls -a
``



## 参考

- [Collection of textlint rule](https://github.com/textlint/textlint/wiki/Collection-of-textlint-rule)
  - 使えるルール一覧が記載されている
