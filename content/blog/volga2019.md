---
title: "VolgaCTF 2019 writeup"
date: 2019-04-03T08:07:43+09:00
draft: false
categories:
- writeup
---

## Shadow cat

- `/etc/shadow`と`encrypted.txt`が渡される
- `/etc/shadow`には、1文字のユーザ名とその暗号化されたパスワードなどが乗っており、`encrypted.txt`にはその1文字で構成される文字列が含まれており、単一替え字暗号だとわかる
- `/etc/shadow`のパスワードを総当たりで解き、得れた替え字表を使って置換するコードを書いて実行したらFLAG

```ruby
#!/usr/bin/env ruby

require 'unix_crypt' # gem install unix-crypt
org = []
ans = []
lines = File.read("./shadow.txt").split("\n")
lines.each do |line|
  org << line.split(":")[0]
  enc = line.split(":")[1]
  salt = line.split("$")[2]
  
  0x1f.upto(0x7e) do |i|
    hashpass = UnixCrypt::SHA512.build(i.chr, salt)
    if hashpass == enc
      ans << i.chr
      puts "Passed"
      break
    end
  end
end

encrypted = File.read("./encrypted.txt").chomp
puts "VolgaCTF{#{encrypted.tr(org.join(""), ans.join(""))}}"
```
- **VolgaCTF{pass_hash_cracking_hashcat_always_lurks_in_the_shadows}**
