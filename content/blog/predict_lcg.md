---
title: "線形合同法で生成される乱数値を予測する"
date: 2019-04-01T22:33:26+09:00
draft: false
categories:
- cryptography
---

## はじめに

線形合同法（LCG）とは、至極一般的かつ簡易に実装することができる乱数生成手法の1つです。先日行われたVolgaCTF 2019では、この線形合同法により生成される乱数値を予測する問題（問題名LG）が出題されました。本問題ではx0 ~ x6に相当する6つの乱数が表示され、それを元に次の値を算出する問題です。

## 線形合同法

線形合同法は、以下の式で定義されます。

```
x1 = (a * x0 + c) % p
```

このとき、x0は乱数のseed値、a, c, pがパラメータとなり適切な値を選びます。今回の問題では、x0 ~ x6までの値が与えられ、その次の値を入力するといったものでした。「a, c, pがわからないと無理じゃない？ｗ」と思ってしまったのですが、どうやら行列演算することで「a, c, p」を算出可能らしいです。

## 線形合同法のパラメータの算出

まずはじめに、各種x0 ~ x6を用いて以下の4つの行列を作成します。それらの行列式を用いて、各行列式の最大公約数(GCD)を算出します。これが、「p」に相当します。その後、(x3 - x4) と(x2 - x3)の法pにおける逆元を乗算し、再度pで割ると「a」になります。最後に、`(x4 - a * x3) % p`で割った際の剰余がcになる。以上の計算を行うコードを以下に示す。

```python
#!/usr/bin/env python
#coding: utf-8

import math
import numpy as np

x0 = 64302589647963933737451564
x1 = 23099347408308738343740115
x2 = 60779187967701597680605077
x3 = 41531243105709646792416331
x4 = 71461317334046189800115379
x5 = 50094315434186546595562390
x6 = 27719142972686291997765807

# なんかnp.linalg.detがエラー吐くので作成
def det(matrix):
    return matrix[0][0] * matrix[1][1] - matrix[0][1] * matrix[1][0]

# https://stackoverflow.com/questions/4798654/modular-multiplicative-inverse-function-in-python
def egcd(a, b):
    if a == 0:
        return (b, 0, 1)
    else:
        g, y, x = egcd(b % a, a)
        return (g, x - (b // a) * y, y)
def modinv(a, m):
    g, x, y = egcd(a, m)
    if g != 1:
        raise Exception('modular inverse does not exist')
    else:
        return x % m

matrix0 = np.array([
    [x1 - x0, x2 - x1],
    [x2 - x0, x3 - x1]
])
matrix1 = np.array([
    [x2 - x0, x3 - x1],
    [x3 - x0, x4 - x1]
])
matrix2 = np.array([
    [x3 - x0, x4 - x1],
    [x4 - x0, x5 - x1]
])
matrix3 = np.array([
    [x4 - x0, x5 - x1],
    [x5 - x0, x6 - x1]
])

p0 = math.gcd(det(matrix0), det(matrix1))
p1 = math.gcd(p0, det(matrix2))
p = math.gcd(p1, det(matrix3))

a = ((x3 - x4) * modinv(x2 - x3, p)) % p
c = (x4 - a * x3) % p

assert x1 == (a * x0 + c) % p, "Not eqaul"
assert x2 == (a * x1 + c) % p, "Not eqaul"
assert x3 == (a * x2 + c) % p, "Not eqaul"
assert x4 == (a * x3 + c) % p, "Not eqaul"
assert x5 == (a * x4 + c) % p, "Not eqaul"
assert x6 == (a * x5 + c) % p, "Not eqaul"
print((a * x6 + c) % p) #=> 54571278391299526410540376
```

## おわりに

本記事では、線形合同法における乱数値の予測方法についてまとめた。パラメータが不明な場合でも本手法を用いることで解読なので覚えておきたい。
