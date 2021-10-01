---
title: "新年なのでHugo でGitHub Pages にデプロイ"
date: 2019-01-02T22:27:14+09:00
draft: false
categories:
- misc
---

あけましておめでとうございます。本年もよろしくお願いいたします。   

新年なので、Hugo で技術ブログを作成してみました。私ははてなブログやMedium などいろいろサイトを持っているわけですが、このブログには自分の備忘録的なことを書いて行こうと思います。

## 1. Hugo のインストールと事前準備

まずはHugo をインストールし、使いたいテーマ(今回はoneというテーマ)をsubmoduleとして取り込む。今回は、GitHub に```blog``` というリポジトリを作成(アクセスする際には、```https://owlinux1000.github.io/blog/```) して、GitHubの設定で **masterブランチのdocs/** を公開対象として設定する。  
`config.toml` を編集する際には、`publishDir = "docs"` を追加しておく。これはGitHub 側で公開するディレクトリを`docs` にしているから。


```
$ brew install hugo
$ hugo new site blog
$ cd blog
$ git init
$ cd themes
$ git submodule add https://github.com/resugary/hugo-theme-one one
$ cp one/exampleSite/config.toml ../../../
$ emacs config.toml # 適宜パラメータを修正する
```

## 2. 記事作成

hugo コマンドで記事を作成してまずは、server サブコマンドを使ってPreview を見てみる。大丈夫そうならGitHub にpushする

```sh
$ hugo new posts/helloworld # この記事
$ emacs content/posts/helloworld.md # draft: false に変更
$ hugo # Webページを生成。これでdocsに吐かれる
$ hugo server # アクセスして見れるか確認する
$ git add .
$ git commit -m "1st commit!"
$ git remote add origin git@github.com:owlinux1000/blog.git
$ git push -u origin master
```

## おわりに

Hugo を初めて使ってみた感想として結構めんどくさくなかったと思った。ネット上見てるとCIでがんばったりいろいろ見ていたので、プロジェクトサイトでのデプロイにした。毎月1本程度では継続していきたい。
