---
title: "Terraform でConoha のインスタンスを立ててみる"
date: 2019-05-18T12:24:26+09:00
draft: false
categories: 
- Infrastructure
tags:
- Terraform
---

## はじめに

最近(?)、自分の中で触ってみたかった技術の1つにTerraformがある。そこで今回は、Terraform でConoha のインスタンスを立ててみる。自分の中での理解をまとめているので間違いあれば教えてください。

## Terraform とは

Terraform は、Vagrant などで有名な[HashiCorp ](https://www.hashicorp.com/)が開発しているツールで、クラウドサービスやOpenStack を対象に、独自の定義ファイルを用いてインスタンスの管理（構築・削除）を行うことができる。定義ファイルには、`HCL` を用いて主に`tf` ファイルに記載していく。私の第一印象としては、Ruby のDSL のような雰囲気を感じるが、`Vagrantfile` ほどRuby 感はない。どちらかというと`json` などに近い雰囲気だ。

## Terraform のインストール

今回は、MacBook Proで行っているので、`brew` 経由でインストールする。

```
$ brew install terraform
```

また、`HCL`を書いていくためエディタ側にもプラグイン等をインストールしてあげよう。渡しの場合は、Emacsの`hcl-mode`を導入してみた。

## 定義ファイルの作成

今回は、`main.tf` というファイル名で以下の内容を作成した。以下の内容は、ほぼ[参考記事](https://qiita.com/kaminchu/items/d0776c381213d54a3a69) と同一である。今回は、勉強も兼ねているので、これを丁寧に読み解いていく。  
まずはじめに、Provider の設定だ。

```
provider "openstack" {
  user_name   = "hgoehoge"
  password    = "hagehage"
  tenant_name = "fugafuga"
  auth_url    = "https://identity.tyo1.conoha.io/v2.0"
}
```

Provider の設定では、`provider` キーワードを用いてブロックを定義する。次に、実際のProvider 名を設定するのだが、Conoha がOpenStack を使っているため、Provider に`openstack` を設定する。ここには、`aws` や`azure` といった様々なクラウドサービスなどを指定することができる。今回の`openstack`の場合は、API用のユーザ名やパスワード、テナントを名、認証用URLを設定する。  
次に、Keypairについて見ていく。

```
resource "openstack_compute_keypair_v2" "keypair" {
  name       = "terraform-keypair"
  public_key = "ssh-rsa hogehoge"
}
```

ここでは、構築したインスタンスにssh する際に利用する公開鍵の設定をしている。`resource` ブロックを用いて、リソースを定義する。`resource` キーワードの次に、リソースの種類、リソース名を設定する。このブロックの中では、一意に識別するための`name` や公開鍵を貼り付ける`public_key` を設定している。

次に、実際にインスタンス自体の設定について見ていく。

```
resource "openstack_compute_instance_v2" "basic" {
  name        = "basic"                             # 好きな名前
  image_name  = "vmi-ubuntu-18.04-amd64-20gb"
  flavor_name = "g-512mb"
  key_pair    = "terraform-keypair"
  security_groups = [
    "gncs-ipv4-all",
  ]
}
```

こちらも、先程と同様に`resource` ブロックを使って定義する。インスタンス自体の設定に関しては、`openstack_compute_instance_v2` で設定できるようだ。`image_name` や`flavor_name` は、ConohaのAPIを使って設定できる値がわかるので、その値を設定する。今回は最安値プランのVPS構成だ。また、`key_pair` には先程定義した`name` 設定する。最後に、`security_groups` では、通信許可に関する設定だが、今回はIPv4における全開放を指定している。

今回は、設定ファイルにクレデンシャル情報をハードコードしているが、実際には環境変数で渡したり別ファイルに定義したりなどを行うように注意する。

## Terraform の実行

Terraform では、まずはじめに`init` サブコマンドを実行する。ここで、Provider用のプラグインや各種初期化処理などが走る。次に、`plan` サブコマンドを用いて、実行計画の作成など実際に行われる処理の確認ができる。そして、`apply` サブコマンドを用いて、実際に処理を実行させる。また、削除した場合は、`destroy` サブコマンドを利用する。

```
$ terraform init
...(skip)
$ terraform plan
...(skip)
$ terraform apply
```

`apply` サブコマンドが成功すると結果が出力されるので、そのIP アドレスに対して、ssh すると構築されたサーバにアクセスできることがわかる。また、ssh が確認できたら`destroy` サブコマンドを実行してみるとインスタンスが削除されている。ここらへんは、逐次実行しながらConoha のコンパネを見ると具体的なイメージが湧きやすい。

## おわりに

今回は、既存の記事を参考にTerraform を使ってConoha にVPSを構築してみた。若干~元記事に騙されて時間をとかしたが、実際に構築することができた。これがあれば、また新しくVPS を立てたいときやイベントを行う際などには便利だと思った。もうちょっと使い道を模索して慣れていきたい。
