---
title: "PragyanCTF 2019 writeup"
date: 2019-03-10T22:48:37+09:00
draft: false
categories:
- writeup
---

## Welcome

- JPEGが渡されるが、binwalkをかけると中にzipがあるので取り出す
- 取り出したzipを展開すると壊れたbmpとパスワード付きのzipが展開される
- bmpファイルの末尾にはbase64エンコードされたような文字列があるのでデコードするとzipのパスワード
- zipを展開するとpngファイルがでてくる
- stegosolveで適当にぽちぽちやるとFLAG
- `pctf{st3gs0lv3_1s_u53ful}`

## Cookie Monster

- Cookieにflagというフィールドがあり、MD5が設定されている
- どこかのMD5クラックサイトで検索すると`pc`がでてくる
- フラグフォーマットは、`pctf{`なのでひたすらこのCookieのMD5の元を考えれば良いとわかる
- `pctf{c0oki3s_@re_yUm_bUt_tHEy_@ls0_r3vEaL_@_l0t}`

## Feed_me

- バイナリを読むと、渡された乱数3つがチェック用の値になっている
- 入力文字列を10byteずつ分割してatoiに流し込んでいて、以下の式が成り立てばおｋなので、あとは算数
```
x + y = a1
y + z = a2
x + z = a3
```
- 自分が解いたときのケース
```
Can you cook my favourite food using these ingredients :)
-10026 ; -8250 ; -12490 ;
-000007133-000002893-000005357
That's yummy.... Here is your gift:
pctf{p1zz4_t0pp3d_w1th_p1n34ppl3_s4uc3}
```

## Game of Faces

- ページにアクセスすると3色のよくわからん正方形があるが、ソースを見るとファイルアップロードが見える
- 適当にファイルをアップロードするとh1タグでbase64エンコードされた文字列が帰ってくるのでデコードするとファイル名がわかる
- ファイル名で直接アクセスするとフラグ
- `pctf{You_L00K_Wi3Rd_IN_H3R3}`

## Easy RSA

- e, c, nのパラメータが渡されるが、eが著しく大きいので、Wiener's Attackだとわかる
- コード書いて、秘密鍵dを計算すると`12978409760901509356642421072925801006324287746872153539187221529835976408177`だとわかる
- あとは普通にRSAを解読したら、出てきた数値を16進数表記にして1byteずつasciiにすればフラグ
- `pctf{Sup3r_st4nd4rd_W31n3r_4tt4ck}`

## Late PR

- 課題を提出しようとしてたけどパソコンがクラッシュしたという問題文とともにイメージファイルが渡される
- vol.pyでプロセスを見るとchrome.exeがexitしていることがわかり、GoogleChromeCrashHandler.exeというプロセスの存在に気づくので、当該プロセスのメモリダンプを見てみるとフラグ
  - `vol.py -f *.raw --profile Win7SP1x86_23418 pslist`
  - `vol.py -f *.raw --profile Win7SP1x86_23418 memdump --dump-dir .`
  - `strings 468.dmp | grep pctf`
- `pctf{Late_submissions_can_be_good}`

## Mandatory PHP

- 以下のPHPのコードが渡される

```php
<?php 
include 'flag.php'; 
highlight_file('index.php'); 
$a = $_GET["val1"]; 
$b = $_GET["val2"]; 
$c = $_GET["val3"]; 
$d = $_GET["val4"]; 
if(preg_match('/[^A-Za-z]/', $a)) 
die('oh my gawd...'); 
$a=hash("sha256",$a); 
$a=(log10($a**(0.5)))**2; 
if($c>0&&$d>0&&$d>$c&&$a==$c*$c+$d*$d) 
$s1="true"; 
else 
    die("Bye..."); 
if($s1==="true") 
    echo $flag1; 
for($i=1;$i<=10;$i++){ 
    if($b==urldecode($b)) 
        die('duck'); 
    else 
        $b=urldecode($b); 
}     
if($b==="WoAHh!") 
$s2="true"; 
else 
    die('oops..'); 
if($s2==="true") 
    echo $flag2; 
die('end...'); 
?> 
```

- `val1`が`INF`になるので、`val3`と`val4`でINFになるようにデカイ数を渡して、`val2`は10回urldecodeして`WoAHh!`になるようにパラメータを渡せば良い
- `curl 'http://159.89.166.12:14000///index.php?ffval1=0&val2=WoAHh%2525252525252525252521&val3=101&val4=大きい値'`
- `pctf{b3_c4r3fu1_w1th_pHp_f31145}`

## 終了後に解いた問題

### Magic PNGs

- パスワード付きzipとpngファイルが渡されるが、`file`コマンドを叩くと`data`都市てか表示されないためヘッダーなどが間違っていると推測できる
- 案の定Magic numberが間違っているので修正するが、それでも開かない
- PNGは通常IDATチャンクがあるはずだが、そのキーワードが`idat`と小文字になっていたのでこれを修正すると開けて、`h4CK3RM4n`が読み取れる
- ヒントから、この値をmd5してパスワードzipを展開するとフラグ
- `pctf{y0u_s33_m33_n0w!}`
- **ファイルヘッダーとかが違うところに気づいていなかった**

### EXORcism

- 大量の1と0があるファイルを渡される
- 100x100の正方形に整形して、1をスペース、0は黒として描画するとQRコードになる
- QRコードを読み取ると、`160f15011d1b095339595138535f135613595e1a`
- タイトルがXORなので、この文字列となにかをxorする必要があると推測できる
- フラグフォーマットの`pctf{`になるようにxorする値を計算すると`flagf`になるので、`flag`を繰り返してxorすれば良いとわかるので、それで計算するとフラグ
- `pctf{wh4_50_53r1u5?}`
- **そもそもQRコードになるなんて想像ができなかった**

## Save Earth

- USBパケットのデータが渡されて入力した値を求めさせる
- データ内にasciiがないので、間隔つまりモールス信号だと推測する
- USBパケットの末尾のデータ部から4をスペース、1と2をそれぞれモールス信号にするとフラグが得られる
- `CTFS4V3`
 
