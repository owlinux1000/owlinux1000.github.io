---
title: "GitHub Action を使ってみる"
date: 2019-12-07T18:47:45+09:00
draft: false
categories: 
- misc
---

## はじめに

リリースされてからまだ[GitHub Action](https://github.com/features/actions)を使っていなかったので、勉強がてら触ってみた。その際に学んだことをまとめておく。

## GitHub Action とは

GitHub Actionは、簡単に言ってしまえばGitHubで使える標準のCI/CDのようなツールだ。独自のワークフローを定義することで、GitHub上の様々なイベントでそれを発火させることができる。設定方法は簡単で、`repository/.github/workflows/[ファイル名].yml` にTravisのようなYMLファイルを作成するだけ。

## 使ってみる

今回試してみたのは、このブログの内容をGitHubにpushしたら、自動的に自分のVPSへデプロイするようなCDとしての機能の実現のために利用してみた。対象リポジトリは以下である。

- [owlinux1000/blog](https://github.com/owlinux1000/blog)

このやりたいことだけ聞くと、Git Hookでもいいと思えるだろうが、今回はGitHub Actionを使ってみるのが目的なので問題無い。この目的のために作成したのが以下の`deploy.yml`だ。

```yaml
name: Deploy my website to the server
on:
  push:
    branches:
      - master
jobs:
  build:
    name: Build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@master
      - name: copy file via ssh password
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          password: ${{ secrets.PASSWORD }}
          port: ${{ secrets.PORT }}
          source: "docs"
          target: "html"
          rm: true
          strip_components: 1
```

フォーマットの様相は、あまり他のCIツールと変わらない。`on`の要素で、`master`ブランチの`push`時に後続の処理を行うような設定にしてある。`on: [push, pull_request]` のような形でpushとpull-request時に実行することも可能だ。また、`schedule`で、cron形式で設定することも可能っぽい。`runs-on`では、動作させる環境を指定する。複数の環境で実行したいときはよくある以下のようなMatrix形式の設定をする。

```yaml
strategy:
  matrix:
    platform: [ubuntu-latest, macos-latest, windows-latest]
```

次に`steps`の子要素で複数の工程を定義することができる。今回は使っていないが、最も基本的なのは`run`っぽい。以下のようにして、bashでスクリプトを実行することもできる。  
```
steps:
  - name: Display the path
    run: echo $PATH
    shell: bash
```
`uses`では、他の人が作成したActionを利用できるっぽい。今回は、当該リポジトリの`docs`フォルダをサーバにscpしたいので、`actions/checkout@master`というActionをまずは実行して、リポジトリをfetchしてcheckoutする。次に、`appleboy/scp-action@master` というscpを実行するActionを利用した。リポジトリのSecrets機能を使ってクレデンシャルを保存してそれを利用する形で設定する。ここで指定してるパラメータはAction依存なもので割愛するが、シンプルで分かりやすかった。  

実際のワークフローの動作している様子は、[ここ](https://github.com/owlinux1000/blog/actions) から確認できる。うまく動作せずに何回も繰り返している様子がわかる。

## おわりに

簡単にだがGitHub Actionを使ってみた。この手のツールは色々あるが、個人的にはGitHub上で簡単に利用できるという点でこれから使っていくのは、GitHub Actionにしようと思った。
