---
title: "さいきん流行りの、ぽすとくあんたむあんごう"
tags: ["量子暗号", "quantum cryptography"]
date: 2020-03-19T02:22:58+09:00
---

ポスト？量子暗号ってなんなんだろなああああ〜〜〜。

どうやらlearning with errorとかshortest-vector problemとかLattice-based cryptographyとやらが関係しているらしい。。。

ということでwikipedia<https://en.wikipedia.org/wiki/Learning_with_errors>や<https://doi.org/10.1145%2F1568318.1568324>などを参考に作ってみた。

なんか暗号化の際に、誤差となる数を少しもぐりこませておくらしい？誤差がない場合、復号化がちゃんとできているか検証する際の式は当たり前の式になる。つまり、誤差があるため復号化がたまに失敗する(間違った値を復号してしまう)ことがあるが、そのおかげでナニカが担保される、みたいな感じ。なのではなかろうか。

テータ関数（！）の積分(テータ関数の第一引数をごにょごにょした関数の、[-0.5, 0.5], [0.5, 1.5], ...の積分の数列。巡回する)とかがなんか出てきて辛かった。。。ぱっとみて見た目は正規分布なので近似できそうなきがするが。。。これを効率的に計算できるようにする方法の論文もあるらしい(<https://doi.org/10.1007%2Fs00200-014-0218-3>)。。。

要mpmathライブラリ(pipでインストールできるよ)。

```python
import math
import random
from functools import reduce
from mpmath import jtheta, quad, mp

def prime(a):
    flags = [True] * (a + 1)
    for i in range(2, int(math.sqrt(a) + 0.1) + 1):
        if not flags[i]:
            continue
        for j in range(i*i, a + 1, i):
            flags[j] = False
    lst = []
    for i in range(2, a + 1):
        if flags[i]:
            lst.append(i)
    return lst

def dot(x, y, p):
    return sum(map(lambda x: x[0]*x[1], zip(x, y))) % p

def add(x, y, p):
    return [(x[i] + y[i]) % p for i in range(len(x))]

def add_vectors(matrix, p):
    return reduce(lambda x, y: add(x, y, p), matrix, [0] * len(matrix[0]))

### consts
def genconsts(n = 128):
    p = random.choice(list(filter(lambda x: x >= n*n, prime(2*n*n))))
    m = 5*n

    alp = lambda n: 1/(math.sqrt(n)*math.log(n))
    
    theta = lambda r: jtheta(3, -math.pi*r, math.exp(-alp(n)*alp(n)*math.pi))
    
    theta_quad = lambda i: quad(theta, [(i-0.5)/p, (i+0.5)/p])
    
    print("Generating distribution....")
    x_dist = []
    i = 0
    while True:
        if i % (p // 100) == 0:
            print("%s/%s" % (i, p - 1))
        
        quad_result = theta_quad(i)
        
        x_dist.append(quad_result)

        if quad_result < 1e-10:
            break

        i += 1
    
    print("%s/%s" % (p - 1, p - 1))
    
    x_dist = x_dist + [0]*(p - 2*len(x_dist) + 1) + list(reversed(x_dist[1::]))

    ### priv key
    s = [random.randint(0, p - 1) for i in range(n)]

    ### public key
    a = [[random.randint(0, p - 1) for i in range(n)] for i in range(m)]
    e = random.choices(list(range(p)), weights=x_dist, k=m)
    b = [(dot(a[i], s, p) + e[i]) % p for i in range(m)]

    return (m, a, b, p, s)

### encryption
def encryption(bit, m, a, b, p):
    S = list(filter(lambda x: random.randint(0, 1) == 0, range(m)))

    if bit == 0:
        enc_a = add_vectors([a[i] for i in S], p)
        enc_b = sum([b[i] for i in S]) % p
    else:
        enc_a = add_vectors([a[i] for i in S], p)
        enc_b = ((p // 2) + sum([b[i] for i in S])) % p

    enc = (enc_a, enc_b)

    return enc

### decryption

def decryption(enc, s, p):
    value = (enc[1] - dot(enc[0], s, p)) % p
    distance_zero = min(value, p - value)
    distance_half = abs(value - p//2)
    if distance_zero < distance_half:
        dec = 0
    else:
        dec = 1
    
    return dec

### some test ...

(m, a, b, p, s) = genconsts()
print("key length: %s" % m)
print("public key a: %s" % a)
print("public key b: %s" % b)
print("modulo: %s" % p)
print("secret key: %s" % s)

samples = 1000
ok = 0
error = 0
for i in range(samples):
    bit = random.randint(0, 1)
    enc = encryption(bit, m, a, b, p)
    dec = decryption(enc, s, p)

    if bit == dec:
        ok += 1
    else:
        error += 1

print("ok: %s/%s" % (ok, samples))
print("error: %s/%s" % (error, samples))
```

次回はRing-LWEに挑戦。