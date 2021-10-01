---
title: "Powershell学習サイトUnder The Wireの問題を解く"
date: 2019-12-02T23:00:00+09:00
draft: false
---

## はじめに

本記事は、[彌冨研 Advent Calendar 2019](https://adventar.org/calendars/4371) の3日目の記事です。ちなみに2018年に彌冨研究室を卒業しました。卒業研究は、[One-dimensional convolutional neural networks for Android malware detection](https://ieeexplore.ieee.org/document/8368693) でした。  
みなさんは、Powershellはお好きですか？僕は嫌いでした。主にWindowsでしか使わない点やよくわからない構文など勉強したいと思う意欲すら浮かびませんでした。しかしながら、セキュリティ界隈だと特にペネトレーションテストなどでは、Windows に標準的に導入されているPowershellは非常に有効なツールです。またマルウェアなどもPowershellを使うケースが非常に多いためセキュリティ業界ではPowershellの理解は必須となるツールの1つです。そこで重い腰を上げて勉強しようとした際にとあるサイトを見つけました。それが[Under The Wire](https://underthewire.tech/) です。このサイトは、CTF形式でPowershellについて学ぶことができるサイトです。本記事では、それの最初のレベルであるCenturyの解き方を紹介していきます。このサイトで提示されている環境へSSHでログインするとPowershellのプロンプトが表示されます。次の問題に進むためには、最初の問題を解く必要があり、その問題の答えが次の問題へのSSHパスワードになっています。

## Century0（事前準備）

- Slackのチームに参加して、指定されているチャンネルにログインすると、`username:password` を得ることができる

## Century1

### 問題概要

- 当該システムのビルドバージョンを求める

### 解法

- `PSVersionTable` という変数に様々なバージョン情報が格納されている
- 下記結果より、答えは`10.0.14393.3053`

```
PS C:\Users\century1\documents> $PSVersionTable

Name                           Value
----                           -----
PSVersion                      5.1.14393.3053
PSEdition                      Desktop
PSCompatibleVersions           {1.0, 2.0, 3.0, 4.0...}
BuildVersion                   10.0.14393.3053
CLRVersion                     4.0.30319.42000
WSManStackVersion              3.0
PSRemotingProtocolVersion      2.3
SerializationVersion           1.1.0.1
```

## Century2

### 問題概要

- PowerShellでwgetのような挙動をするコマンドレット名（小文字）にデスクトップ上にあるファイルのファイル名を連結したものを求めよ

### 解法

- Invoke-WebRequest がwgetのような動きをするコマンドレット
- デスクトップに移動すると443というファイル名のファイルが存在する
- 答えは、`invoke-webrequest443`

## Century3

### 問題概要

- デスクトップ上にあるファイル数を求めよ

### 解法

- Get-ChildItem で対象ディレクトリの中身の一覧を取得
- Where-Object を使ってファイルのみを抽出
    - `$_` が、パイプラインで渡されたオブジェクトを示す
    - `PSIsContainer` は、ディレクトリの場合`True`を返す
- Measure でパイプラインで渡されたオブジェクトの数をカウント
- 答えは、`123`

```
PS C:\Users\century3\documents> Get-ChildItem ..\Desktop | Where-Object { !$_.PSIsContainer} | Measure


Count    : 123
Average  :
Sum      :
Maximum  :
Minimum  :
Property :
```

## Century4

### 問題概要

- デスクトップにある空白が含まれたディレクトリの中にあるファイル名を求めよ
- 答えは、`61580`

### 解法

- Get-ChildItem で表示するだけ
    - ディレクトリ名に空白が含まれている場合は、`'`で囲う

```
PS C:\Users\century4\Documents> Get-ChildItem '..\Desktop\Can You Open Me'


    Directory: C:\Users\century4\Desktop\Can You Open Me


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        8/30/2018   3:29 AM             24 61580
```

## Century5

### 問題概要

- ドメイン名(TLDを除く)とデスクトップ上にあるファイルを連結したものを求めよ

### 解法

- 環境変数USERDOMAINという値にドメイン名は格納されている
- 答えは、`underthewire3347`

```
PS C:\Users\century6\documents> Get-ChildItem -Path env: | Where-Object {$_.NAME -eq "USERDNSDOMAIN"}

Name                           Value
----                           -----
USERDNSDOMAIN                  UNDERTHEWIRE.TECH
```

## Century6

### 問題概要

- デスクトップ上のフォルダの数を求めよ

### 解法

- Century3とほぼ同様
    - `()`でくくって、Countプロパティにアクセスすれば値のみを表示できる
- 答えは、`197`

```
PS C:\Users\century6\documents> (Get-ChildItem ..\Desktop | Where-Object { $_.PSIsContainer} | Measure).Count
197
```

## Century7

### 問題概要

- ユーザプロファイル配下のどこかにあるreadmeファイルの中に記載されている値を求めよ

### 解法

- 数が少ないので愚直にできるが、再帰的な検索方法を用いた
- Get-ChildItem の`-Recurse` オプションを利用することで再帰的に表示可能
- 中を見てみると、答えは`7points`

```
Get-ChildItem -Recurse ..


    Directory: C:\Users\century7


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-r---        7/16/2016   1:23 PM                Desktop
d-r---        8/30/2018   3:10 AM                Documents
d-r---       10/28/2019   5:50 PM                Downloads
d-r---        7/16/2016   1:23 PM                Favorites
d-r---        7/16/2016   1:23 PM                Links
d-r---        7/16/2016   1:23 PM                Music
d-r---        7/16/2016   1:23 PM                Pictures
d-----        7/16/2016   1:23 PM                Saved Games
d-r---        7/16/2016   1:23 PM                Videos


    Directory: C:\Users\century7\Downloads


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        8/30/2018   3:29 AM              7 Readme.txt
```

## Century8

### 問題概要

- デスクトップにあるファイルの中のユニークな文字列の数を求めよ

### 解法

- Get-Content でcatが可能
    - 元からエイリアス貼られているので`cat`でもOK
- ユニーク数を取るためには、sortしてuniqすれば良い
    - sortはSort-Objectのエイリアス
    - Get-Unique でユニークな値を抽出
- 答えは、`696`

```
PS C:\Users\century8\documents> Get-Content ..\Desktop\unique.txt | sort | Get-Unique | measure


Count    : 696
Average  :
Sum      :
Maximum  :
Minimum  :
Property :
```

## Century9

### 問題概要

- デスクトップにあるファイルの161番目の単語を求めよ
    - 空白で区切られた単語が並んでいる

### 解法

- `()`にして文字列として、Splitメソッドで空白で区切り添字160番目の要素にアクセス
- 答えは、`pierid`

```
PS C:\Users\century9\documents> (cat ..\Desktop\Word_File.txt).Split(" ")[160]
pierid
```

## Century10

### 問題概要

- Windows Updateサービスの説明の10番目、8番目の単語とデスクトップ上のファイルのファイル名をつなげた値を求めよ

### 解法

- WMI のWin32_Serviceクラスから説明を得る
    - `Get-Service` でサービス情報を取得できるが、説明の部分は保持していなかった
- 説明は英文なため空白で区切られているのでSplitして10番目と8番目を抜き出してくる
- デスクトップ上のファイルのファイル名をつなげて答えは、`windowsupdates110`

```
PS C:\Users\century10\documents> word=(get-wmiobject win32_service -filter "name='wuauserv'").Description.Split(" ")
PS C:\Users\century10\documents> $word[9] + $word[7]
Windowsupdates
```

## Century11

### 問題概要

- ユーザプロファイル配下の隠しファイルのファイル名を求めよ

### 解法

- Get-ChildItemの`-Hidden`パラメータで表示できる
    - 権限なくて見れないフォルダが多く出力されるが一つ怪しいのがある
- 答えは、`secret_sauce`

```
PS C:\Users\century11\documents> Get-ChildItem -Hidden -Recurse ..
    Directory: C:\Users\century11\Downloads


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
--rh--        8/30/2018   3:34 AM             30 secret_sauce
```

## Century12

### 問題概要

- 所属するドメインコントローラの説明とデスクトップ上にあるファイルのファイル名をつなげたものを求めよ

### 解法

- まずは、ドメインコントローラをGet-ADDomainControllerで確認する
    - `Name`が`UTW`だと判明する
- Get-ADComputerを用いて、Descriptionを表示する
- デスクトップのファイルのファイル名とつなげて答えは、`i_authenticate_things`

```
PS C:\Users\century12\documents> Get-ADDomainController


ComputerObjectDN           : CN=UTW,OU=Domain Controllers,DC=underthewire,DC=tech
DefaultPartition           : DC=underthewire,DC=tech
Domain                     : underthewire.tech
Enabled                    : True
Forest                     : underthewire.tech
HostName                   : utw.underthewire.tech
InvocationId               : 09ee1897-2210-4ac9-989d-e19b4241e9c6
IPv4Address                : 192.99.167.156
IPv6Address                :
IsGlobalCatalog            : True
IsReadOnly                 : False
LdapPort                   : 389
Name                       : UTW
NTDSSettingsObjectDN       : CN=NTDS Settings,CN=UTW,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=underthewire,DC=tech
OperatingSystem            : Windows Server 2016 Standard
OperatingSystemHotfix      :
OperatingSystemServicePack :
OperatingSystemVersion     : 10.0 (1439
PS C:\Users\century12\documents> Get-ADComputer UTW -Properties Description


Description       : i_authenticate
DistinguishedName : CN=UTW,OU=Domain Controllers,DC=underthewire,DC=tech
DNSHostName       : utw.underthewire.tech
Enabled           : True
Name              : UTW
ObjectClass       : computer
ObjectGUID        : 5ca56844-bb73-4234-ac85-eed2d0d01a2e
SamAccountName    : UTW$
SID               : S-1-5-21-758131494-606461608-3556270690-1000
UserPrincipalName :
```

## Century13

### 問題概要

- デスクトップにあるファイルの中の単語数を求めよ

### 解法

- ファイルの中身を空白でSplitしてMeasureにパイプラインで渡す
- 答えは、`755`

```
PS C:\Users\century13\documents> (cat ..\Desktop\countmywords).Split(" ") | measure


Count    : 755
Average  :
Sum      :
Maximum  :
Minimum  :
Property :
```

## Century14

### 問題概要

- デスクトップ上にあるファイルの中で、`polo`という単語の出現回数を求めよ

### 解法

- 中身の抽出は今まで通り行い、Select-Stringで`polo`を検索してMeasureにパイプラインで渡す
    - 完全一致させる必要があるため`"^polo$"` を指定する必要がある
- 答えは、`153`

```
PS C:\Users\century14\documents> (cat ..\Desktop\countpolos).Split(" ") | Select-String "^polo$" | measure


Count    : 153
Average  :
Sum      :
Maximum  :
Minimum  :
Property :
```

## おわりに

これをやる前はあまり好きになれなかったPowershellもやってるうちに好きになることができました。このサイトではまだ次のレベルがあるので取り組んでみようと思います。もしセキュリティ系の職種に付きたい方は、Powershellわかるとなにかと便利なので勉強しておくことをおすすめします。
