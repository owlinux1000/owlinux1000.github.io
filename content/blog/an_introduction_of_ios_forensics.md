---
title: "iOSデバイスフォレンジック入門"
date: 2019-02-28T21:00:00+09:00
draft: false
categories:
- forensics
tags:
- mobile
---

## はじめに

本記事では、**iOSフォレンジックをやったことない**人向けに基本的な情報をまとめてみました。取っ掛かりとしては簡単に読める内容だと思います。主にiOSデバイスのバックアップデータやその解析ツールについてご紹介しています。

## 準備

本記事では主に以下のデバイスやソフトウェアを使って検証しています。当該デバイスやバージョンでない場合同じ結果や解釈にならない可能性もありますのでご了承ください。  

- iPad mini4 (iOS 12.1.1 非Jailbreak)
- MacBookPro (High Sierra 10.13.6)
- iTunes (12.8.3)

## iOS 概要

iOSは、iPhoneやiPad、Apple Watch等で利用されているOSです。iOSは大きく分けて、以下の4つのシステムから構成されています。

- Cocoa Touch
- Media
- Cocoa Service
- Core OS

Cocoa Touchは、名前からもわかるようにタッチ操作を始めとするユーザーインタフェースのシステムです。Mediaは、動画像や音楽などMedia用のシステムです。Cocoa Serviceは、アプリケーションにとって必要な基本的なシステムを提供しています。最後のCore OSは、ハードウェアに近いより低レイヤーなネットワークやメモリ管理、スレッドの機能などを提供しています。  

次に、iOSで用いられているファイルシステムについて簡単にご紹介いたします。近年のiOSではAPFS（Apple File System）と呼ばれるファイルシステムが利用されています。従来はHFS、HFS+が使われていましたが、2017年のiOS 10.3からAPFSが導入されました。APFSの特徴としては、inodeが64bitに拡張されたためより多くのファイルが扱えるようになったり、CoW（Copy on Write）のサポート、タイムスタンプがナノ秒単位まで記録するようになったりなど従来のファイルシステムに比べ大きく変わっています。より詳細な情報としては以下のWebページなどが参考になると思います。

- [Apple File System](https://ja.wikipedia.org/wiki/Apple_File_System)
- [ファイルシステムがAPFSになった事による変更点](https://cyberforensic.focus-s.com/knowledge/articles_detail/356/)

## iTunesバックアップ

iOSデバイスにおけるデータを抽出する方法としては、物理と論理の2通りの方法があります。しかしながら、物理デバイスから情報を抽出するためには、機材が必要であったり、論理面でも有償のツールが必要なことが多く入門には不向きです。そこで本記事では、iTunesのバックアップデータを元にiOSデバイスのフォレンジック調査に役立つ情報をまとめていこうと思います。もしすでにホストマシンにiTunesを用いてバックアップをとっている人はそのデータをお使いできます。もしなければ、iOSデバイスを接続し、iTunesの画面よりバックアップをとってください。

iOSデバイスのバックアップデータは、以下の場所に格納されます。  

* Mac：`/User/ユーザ名/Library/Application Support/MobileSync/Backup`
* Windows：`\AppData\Roaming\Apple Computer\MobileSync\Backup\`
* Windowsストアアプリ経由でiTunesを入れた場合：`%USERPROFILE%\Apple\MobileSync\Backup`

バックアップを行っている状態で、上記フォルダにアクセスするとハッシュ値が名前のフォルダがあると思います。それがバックアップデータの本体です。バックアップデータのフォルダの中には、主に以下のファイルやフォルダが格納されていると思います。

- Manifest.plist
- Manifest.db
- Info.plist
- Status.plist
- 大量のフォルダ

Manifest.plistは、主にバックアップの内容について記載されています。例えばバックアップした日時、バックアップを暗号化しているかどうか、インストールしたアプリケーション一覧などがあげられます。Manifest.dbは、SQLiteのデータベースファイルで、バックアップデータに含まれるファイルやフォルダの情報が格納してあります。`fileID`カラムには、SHA1が格納されており、これはファイル名を表しています。そのため、バックアップフォルダの中で、このハッシュ値を使って検索したりします。  
以下の図は、DB Browser for SQLiteでManifest.dbを読み込んだときの図です。CUIのsqlite3コマンドなどでも良いのですが、フォレンジック業務をやるときには、フィルターやソート、検索などが手軽に使えるほうが効率が良いので、こういったGUIツールを使っています。

![Manifest.db](https://imgur.com/download/2PzjpVu)

Info.plistは、主にバックアップ対象のデバイス情報について記載されています。例えば、IMEIやシリアルナンバー、最後にバックアップした日などが挙げられます。Status.plistは、主にバックアップ状況について記載されています。  

バックアップフォルダの中に格納されている主要なアーティファクトは、SANSの公開資料の[DFIR-Smartphone-Forensics-Poster.pdf](https://digital-forensics.sans.org/media/DFIR-Smartphone-Forensics-Poster.pdf) に掲載されています。とても便利なのでダウンロードしておくことをおすすめします。

## iTunesバックアップ解析ツール

今まで見てきたバックアップデータなどを解析する際に役立つツールをご紹介します。

### plutil

拡張子plistのバイナリファイルの中身を解析する際には、`plutil`コマンドが便利です。Macの場合は標準でインストールされているのですぐに使えます。Windowsでは、[plist Editor Pro for Windows](https://www.icopybot.com/plist-editor.htm)というものが存在するので、それを利用できそうです（未検証）。

本来`plutil`コマンドはいろいろな操作ができますが、今回はフォレンジックでよく使うplistファイルを別のファイルに変換する操作をご紹介します。以下は、`Info.plist`ファイルをxmlファイルに変換する場合の例です。plistファイルはバイナリファイルなので、こういった変換を行うことが必要となってきます。

```
$ plutil -t convert xml1 Info.plist -o Info.xml
```

`-o`オプションを忘れてしまうとplistファイル自体の中身が書き換わってしまうので注意してください。  
その他Pythonが利用できる環境であれば、`plistlib`ライブラリが利用できるので、これを用いてプログラマブルに解析することも可能です。

### iExplorer

![iExplorer](https://imgur.com/download/Aogfu9Z)

iExplorerは、高機能なiOSデバイスファイルブラウザーです。[iExplorer](https://macroplant.com/iexplorer) よりダウンロードできます。iExplorerは、Manifest.dbなどの情報を自動でパースし、より人間にわかりやすい形で表示してくれます。正直これがあれば、バックアップデータの閲覧には困らないと思います。全体的に眺めたい際などは、こちらを使って足りない場合、前述した`plutil`コマンドなどを使って解析します。同様のツールとして`iBackupBot`や`iBackup Viewer`などがあるので、使い比べてみて自分にあったものを使うと良いでしょう。

## おわりに

今回は、iOSの基本構造やバックアップデータを用いた解析方法などについて簡単にご紹介しました。次は、気が向けば個々のアーティファクトについてより詳細にとりあげて記事を今度書こうかなと思っています。

## 参考資料

- [Practical Mobile Forensics - Third Edition](https://www.packtpub.com/networking-and-servers/practical-mobile-forensics-third-edition)
