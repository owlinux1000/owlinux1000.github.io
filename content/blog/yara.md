---
title: "平成のうちに理解しておくYara 入門"
date: 2019-04-29T07:03:53+09:00
draft: false
categories:
- forensics
---

> "平成も終わるし、知ったかぶりしてる技術を令和に持ち越さないようにしような"

## 目次

- Yara とは
- はじめてのYara ルール
- 文字列を用いたYara ルール
- 正規表現を用いたYara ルール
- 複数条件を用いたYara ルール
- Yara の便利機能
- モジュール機能を用いたYara ルール
- Pythonから利用するYara ルール
- まとめ

## Yara とは

Yaraは、「**Yara ルール**」と呼ばれる専用のルールを用いて、マルウェアや悪意あるファイルなどを検知することのできるツールです。Yara ルールでは、文字列による検査や正規表現、論理条件などをサポートしており複雑なルールを構築することもできます。利用例として、あるマルウェアファミリーの特徴をYara ルールとして作成しておくことで、新しいマルウェアが発見されたときに既知のマルウェアファミリーに性質が近いかどうかなどを検知することができます。Yara は、<a href="https://cuckoosandbox.org/" target="_blank">Cuckoo Sandbox</a>など他のセキュリテイツールと連携していることが多く、セキュリテイに関わる人間としては、一度読み書きしておいた方が良いツールの１つです。本記事では、Yara を使ったこと無い人向けに基本的な概念やルールを幅広くご紹介します。  
まずは、Yara を利用するためにインストールしましょう。Ubuntu では、`apt`からインストールすることができます。
```
$ sudo apt install yara
```
また、Macでは`brew`からインストールすることが可能です。
```
$ brew install yara
```

インストールできたら、以下のコマンドを入力して、Yara のバージョンを確認してください。なお本記事では、バージョン3.9.0を対象に執筆しています。
```
$ yara --version
3.9.0
```

## はじめてのYara ルール

では、さっそくはじめてのYara ルールを作成してみましょう。以下に、必ず検知するルールを示します。

```
// hello_rule.yara
rule hello
{
    condition:
        true
}
```

Yaraでは、`rule`キーワードを用いてルールを作成します。ここで`hello`はルール名にあたり、実際の条件が`condition`が`true`のときに検知することを意味しています。本例では、一般的なプログラミング言語と同様に、条件文が必ず真となります。そのため、必ず検知することができます。また、先頭の1行目は、コメント文です。Yara ルールのファイルの拡張子は、`.yara`や`.yar`の場合が多いです。  
では、このルールを用いて実際に検知するか確認してみましょう。そのために検査対象となるファイルを`hello.txt`として以下の内容で作成してください。

```
HelloWorld!
```

以下の以下コマンドを実行してください。

```
$ yara hello_rule.yara hello.txt
hello hello.txt
```

本コマンドでは、`hello.txt`ファイルに対して、`hello_rule.yara`を用いて検査しています。また、`hello`というルールで`hello.txt`が検知していることがわかります。
Yara ルールによる検査は、ファイルの他にディレクトリやPIDを指定することもでき、それらの検査も行うことが可能となっています。  

Yara ルールは、特別なシンタックスで書かれているので、エディタ等のシンタックスハイライトや補完プラグインの利用をおすすめします。Emacs ユーザならば、melpaで`yara-mode`が配布されているので、`package`でインストールすることができます。

## 文字列を用いたYara ルール

先程の例では、無条件に合致するルールを作成していました。次は、`hello.txt`に格納されている`HelloWorld!`という文字列を含んでいるファイルを検知するルールを作成してみましょう。`condition`の条件として、検知したい文字列を設定します。

```
rule hello_string
{
    condition:
        "HelloWorld!"
}
```

本ルールを用いて、先程と同様に検査してみましょう。

```
$ yara hello_string_rule.yara hello.txt
hello_string_rule(5): warning: Using literal string "HelloWorld!" in a boolean operation.
hello_string hello.txt
```

先程と同様に検知できましたが、警告が出ています。これは、文字列リテラルを用いて直接条件を設定しているため出ています。ルールの中で文字列を変数として定義し、その変数を用いることでこの警告は出力されなくなります。以下のようにルールを変更してください

```
rule hello_string
{
    strings:
        $s = "HelloWorld!"
    condition:
        $s
}
```

`strings`キーワードを用いて、変数を定義することができます。再度検査すると警告文が出てきません。

```
$ yara hello_string.yara hello.txt
hello_string hello.txt
```

Yara では、変数の後に様々なキーワードを設定することでより柔軟に検査することができます。例えば、以下のルールでは、検知したい文字列の後に、`nocase`を記載することで、case-insentive に文字列を認識し、検査してくれます。そのため、`hEllOwORLD!` といった文字列も検知することができます。

```
rule hello_string
{
    strings:
        $s = "HelloWorld!" nocase
    condition:
        $s
}
```

`nocase`の他にも、`wide`でUnicode文字列などの場合も検知することが可能となります。もしAscii文字列とUnicode文字列を両方検知したい場合は、`wide` と`ascii` の両方をつけることで可能となります。  
`fullword`を設定すると、検知した文字列が、アルファベット以外で終端してる場合のみ検知することが可能となります。（例：`apple` を検知したい文字列に設定している場合に、`applejuice.com`は検知しないが、`apple.com`は検知することができる）

## 正規表現を用いたYara ルール

Yara では、正規表現を用いた検査を行うことができます。例えば、`a`から始まる文字列を検知する場合を考えてみます。正規表現を用いる場合は、以下のように`"`ではなく、`/`で囲います。

```
rule regexp
{
    strings:
        $s1 = /^a/
    condition:
        $s1
}
```

また、正規表現においても、`nocase`のようなキーワードを付与することもできます。


## 複数条件を用いたYara ルール

今までのルールでは、単一の条件にマッチした場合でしたが、`condition`に、論理演算子を用いることで、より複雑な条件を構築することができます。例として、以下の内容で`hello2.txt`を作成してください。

```
Hello World!
```

そして、`hello2.txt`を先程作成した`hello_string_rule.yara`で検査してみてください。

```
$ yara hello_string_rule.yara hello2.txt
```

結果として検知されず何も表示されません。`hello_string_rule.yara`では、`HelloWorld!`をルールにしていたため、間にスペースがある`hello2.txt`では検知することができませんでした。そこで次に、`Hello World!`も条件として加え、`HelloWorld!` または`Hello World!`の場合に検知するようにルールを作成します。

```
rule hello_string
{
    strings:
        $s1 = "HelloWorld!"
        $s2 = "Hello World!"
    condition:
        $s1 or $s2
}
```

上記ルールでは、文字列として2つ定義し、それらを`or`で接続することで条件を構築しています。以下のように`hello.txt`と`hello2.txt`の両方を検査してみましょう。

```
$ yara hello_string2_rule.yara hello2.txt
hello_string hello2.txt
$ yara hello_string2_rule.yara hello.txt
hello_string hello.txt
```

両方とも検知していることがわかります。  
Yara には、`or`以外にも`and`や`not`など様々な条件に使えるキーワードがあります。詳しくは、<a href="https://yara.readthedocs.io/en/v3.5.0/writingrules.html" target="_blank">公式ドキュメント</a>を参照してください。

## Yara の便利機能

ここでは、個人的に便利そうだなと思った機能をいくつかご紹介します。

### 特別な値を用いたルール

Yara は、主にマルウェアを対象として検知を行うツールなため、標準で文字列マッチ以外にも便利な機能が備わっています。例えば、以下は、対象ファイルのファイルサイズが200KBより大きかった場合に検知するルールの例です。`filesize`は、特別な値で元から検査対象のファイルのサイズが格納されています。

```
rule FileSizeExample
{
    condition:
       filesize > 200KB
}
```

また、後述するモジュール機能を用いることで、より多くの特別な値をルールで利用することができます。  

### Yara ルールのメタデータ

Yara ルールには、メタデータの設定を行うことが可能です。rule中に、`meta`キーワードを用いることで、メタデータを格納することができます。以下に例を示します。

```
rule FileSizeExample
{
    meta:
       description = "File size check"
    condition:
       filesize > 200KB
}
```

このルールでは、`description`というメタデータを設定しています。このメタデータは、`yara` コマンドのオプション`-m` を設定することで表示することができます。

```
$ yara -m elf_rule.yara hello
elf_64 [description="File size check"] hello
```

### Yara ルールのタグ

メタデータと似た機能として、タグの設定を行うことも可能です。ルール名の後に、`:`でタグ名を記載します。複数設定した場合は、スペースで分割して記載します。以下に`hello_rule`にタグを設定した例を示します。

```
rule hello : myfirstrule
{
    condition:
        true
}

rule hello2 : mysecondrule
{
    condition:
        true
}
```

`hello`ルールには、`myfirstrule`、`hello2`ルールには、`mysecondrule`というタグを設定しています。今までのようにこのルールを用いて検査を行うと、以下のように`hello`と`hello2`の両方のルールにマッチしていることがわかります。

```
$ yara hello_rule.yara hello.txt
hello hello.txt
hello2 hello.txt
```

しかしながら、タグを設定することで、特定のタグのルールのみを表示するといったフィルターのような処理を行うことができます。オプション`-t` に、`myfirstrule`を設定することで、`hello`ルールのみを表示することが可能です。

```
$ yara -t myfirstrule hello_rule.yara hello.txt
hello hello.txt
```

### Yara ルールを分割管理

1ファイルに多くのYara ルールを記載することは、管理上の観点からも好ましくありません。そこで、Yara では`include`文を利用することで分割されたファイルを読みこむことが可能となります。以下の例では、最初に作った`hello_rule`を読み込んでいます。

```
include "./hello_rule"
rule regexep
{
    strings:
        $s1 = /^a/
    condition:
        $s1
}
```

## モジュール機能を用いたYara ルール

Yara には、モジュール機能があります。特に、PE モジュールは、マルウェア解析者にとってよく使うモジュールの1つです。PE モジュールの他にも、ELF モジュールやCuckoo モジュール、Hash モジュール、Magic モジュールなどがあります。モジュールは、`import` 文を用いて読み込みます。今回は、ELF モジュールを用いた例を見てみましょう。以下は、ELFバイナリがx64向けの場合検知するルールです。

```
import "elf"

rule elf_64
{
    condition:
        elf.machine == elf.EM_X86_64
}
```

本ルール検証のために、`Hello World!`と表示するELFバイナリを作成します。以下のC言語のファイルを作成してください。

```c
#include <stdio.h>
void main() {
  printf("Hello World!");
}
```

以下のようにコンパイルして、実行ファイルを作成してください。

```
$ gcc hello.c -o hello
```

作成したELFバイナリに対して検査をすると検知していることがわかります。
```
$ yara elf_rule.yara hello
elf_64 hello
```

次に、IOCなどでもよく使われるハッシュ値を用いたルールを作成してみましょう。ハッシュ値を求める処理は、Hash モジュールを読み込むことで利用できます。Hash モジュールでは、md5やsha1、 sha256の代表的なハッシュアルゴリズムによるハッシュ値が求められます。まずは、検査対象となる`hello.txt`のsha256ハッシュ値を求めてみましょう。

```
$ shasum -a 256 hello.txt
209a3f843d2a572c2d66457dd7c8a6120fa308949867a5ebed5f4dca08fe4920  hello.txt
```

では、このハッシュ値を用いて、読み込んだファイルのハッシュ値と等しかったら検知させるルールを作成してみましょう。
```
import "hash"
rule sha256
{
     condition:
         hash.sha256(0, filesize) == "209a3f843d2a572c2d66457dd7c8a6120fa308949867a5ebed5f4dca08fe4920"
}
```

Hash モジュールのハッシュ値を求める機能では、2つの引数を設定できます。本例における前者の`0`は、ファイルのオフセットを示しています。つまり、ファイルの先頭からのハッシュ値を求めようとしています。ファイルオフセットを適切に設定することで、生の攻撃ペイロードの不要部分を飛ばし、適切な位置からハッシュ値を求めることも可能となります。また、2つ目の引数は、サイズを示しており、今回の例だと`filesize`つまり、ファイル全体を示しています。そのため、第1引数に`0`、第2引数に`filesize`を設定することで、ファイル全体のハッシュ値を算出することができます。本機能における戻り値のハッシュ値は、lowercase で帰ってくるため、比較する際には気をつけてください。

## Pythonから利用するYara ルール

Yara ルールのファイルは、`yara`コマンドだけでなく、Pythonから利用することも可能です。Python から利用することで、よりプログラマブルにYara を利用することができるため、システムに組み込むことが容易になります。利用するためには、`yara-python`というライブラリをインストールしてください。

```
$ pip install yara-python
```

では、2つ目に作成した`hello_string_rule`のYara ルールファイルを読み込んで検査してみましょう。以下の検査スクリプトを`hello.py`として作成してください。

```python
import yara
rules = yara.compile('hello_rule.yara')
result = rules.match('hello.txt')
print(result)
```

上記スクリプトを実行してください。
```
$ python hello.py
[hello_string]
```

実行すると予定通り検知していることがわかります。また、検知したファイルがない場合は、空の配列が帰ってくるため注意してください。

## まとめ

今回、マルウェアなどを検知するルールベースのツールYara を見てきました。Yara は、文字列やファイルサイズ、バイナリの情報などを様々な値を用いて柔軟にルールを作成することができます。また、Pythonから利用するなどプログラマブルな使い方も可能となっています。ぜひYara を用いて様々な悪意ある挙動をするファイルを検知していきましょう。
