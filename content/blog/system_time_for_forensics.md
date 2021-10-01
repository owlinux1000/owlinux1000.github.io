---
title: "フォレンジックのためのシステム時刻入門"
date: 2019-02-23T20:37:24+09:00
draft: false
categories:
- forensics
---

## はじめに

フォレンジックなどをやっていると様々な時刻形式と直面することがあります。いざ調べてみると時刻形式は、プラットフォームによって異なる場合が多いことに気づきました。時刻の情報はタイムライン作成では、とても重要な要因となってくるので、適切に時刻を解釈することが必要です。本記事では、よく見る3つの時刻表記についてご紹介します。

## UNIX time

- UNIX timeはDFIRだけに限らず多くの場面でよく見かける時刻形式の1つです。**UTC（協定世界時）の1970年1月1日0時0分0秒からの経過秒数**で表す時刻形式です。主要なプログラミング言語の代表的な時刻を扱うライブラリなどでは、UNIX timeをサポートしています。例えばRubyでは以下の様に、UNIX timeを扱うことができます。

```ruby
require 'time'
JST_OFFSET = 3600 * 9
puts Time.parse("1970-01-01 00:00:00").to_i + JST_OFFSET #=> 0
puts Time.at(0) #=> 1970-01-01 09:00:00 +0900
```

## Apple Cocoa Core Data timestamp

- Apple Cocoa Core Data timestampは、主にMac OSやiOSで見かける時刻形式です。Cocoaとは、macOS用のフレームワークです。また、**Mac absolute time**と表記されることもあります。本時刻形式は、**UTCの2001年1月1日0時0分0秒からの経過秒数**で表す時刻形式です。UNIX timeとの差分は、**978307200**なので、これを考慮すればUNIX timeからすぐ算出することができます。

## WebKit/Chrome time

- WebKit/Chrome timeは、Google Chrome、OperaやSafariなどのデータで使われている時刻形式です。本時刻形式は、**UTCの1601年1月1日0時0分0秒からの経過マイクロ秒**で表す時刻形式です。UNIX timeなどと異なりマイクロ秒なので値が大きいという特徴があります。

## 便利なツールやサイト

以下の2つは、システム時刻変換をする際に、使いやすかったツールです。

- [Epoch Converter](https://www.epochconverter.com/)
  - 本記事で説明した時刻形式などは概ねカバーしているツールです
- [DCode](https://www.digital-detective.net/dcode/)
  - Epoch Converterと同様の機能を有しているWindowsアプリケーション

## 参考文献

- [Practical Mobile Forensics - Third Edition](https://www.packtpub.com/networking-and-servers/practical-mobile-forensics-third-edition)
