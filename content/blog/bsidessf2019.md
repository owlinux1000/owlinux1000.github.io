---
title: "BesidesCTF 2019 writeup"
date: 2019-03-10T22:43:52+09:00
draft: false
categories:
- writeup
---

## blink

- apkが渡されるのでunzipして、`classes.dex`をdex2jarでjarファイルに変換
- jarファイルをunzipして、classファイルをjadで一括逆コンパイル
- 適当にファイルを眺めていると`/com/example/blink/r2d2.jad`にbase64でエンコードされた文字列があるのでデコード
- `CTF{PUCKMAN}`と書かれた画像が出てくる

## zippy

- pcapngが渡されるのでFollo TCP Streamをすると`supercomplexpassword`がzipのパスワードだとわかる
- パケットからzipファイルを通信してるパケットを見つけて抽出
- パスワードを使ってunzipすると中のflag.txtに`CTF{this_flag_is_your_flag}`

##  futurella

- HTMLのソースを見ると`CTF{bring_it_back}`

## kookie

- `cookie/monster`でログインできるのでログインする
- その際のCookieを見ると、`cookie`になっているので、これを`admin`に変更
- その後reloadすると`CTF{kookie_cookies}`
