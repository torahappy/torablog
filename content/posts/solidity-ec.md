---
title: Solidityで楕円曲線の演算を書いてみた
tags: ["Ethereum", "Solidity"]
date: 2018-05-26T09:58:00+09:00
---

```javascript
library EC {
    uint constant p = 0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffefffffc2f;
    uint constant a = 0x0;
    uint constant b = 0x7;
    uint constant gx = 0x79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798;
    uint constant gy = 0x483ada7726a3c4655da4fbfc0e1108a8fd17b448a68554199c47d08ffb10d4b8;
    
    function inv_mod_p(uint a) pure internal returns (uint) {
        if (a > p) {
            a = a % p;
        }
        int t1;
        int t2 = 1;
        uint r1 = p;
        uint r2 = a;
        uint q;
        while (r2 != 0) {
            q = r1 / r2;
            (t1, t2, r1, r2) = (t2, t1 - int(q) * t2, r2, r1 - q * r2);
        }
        if (t1 < 0) {
            return (p - uint(-t1));
        }
        return uint(t1);
    }
    
    function add(bool is_p_origin, uint px, uint py, bool is_q_origin, uint qx, uint qy) pure internal returns (bool o, uint x, uint y) {
        if (is_p_origin) {
            (o, x, y) = (is_q_origin, qx, qy);
        } else if (is_q_origin) {
            (o, x, y) = (is_p_origin, px, py);
        } else if (px == qx && py == p - qy) {
            (o, x, y) = (true, 0, 0);
        } else {
            uint dydx;
            if (px == qx && py == qy) {
                dydx = mulmod(px, px, p);
                dydx = mulmod(addmod(mulmod(3, dydx, p), a, p), inv_mod_p(mulmod(2, py, p)), p);
            } else {
                dydx = mulmod(addmod(qy, p - py, p), inv_mod_p(addmod(qx, p - px, p)), p);
            }
            o = false;
            x = addmod(addmod(mulmod(dydx, dydx, p), p - px, p), p - qx, p);
            y = addmod(mulmod(dydx, addmod(px, p - x, p), p), p - py, p);
        }
    }
    
    function mul(bool po, uint px, uint py, uint n) pure internal returns (bool o, uint x, uint y) {
        o = true;
        while(n != 0) {
            if (n & 1 != 0) {
                (o, x, y) = add(o, x, y, po, px, py);
            }
            (po, px, py) = add(po, px, py, po, px, py);
            n = n / 2;
        }
    }
}
```

### 解説

- inv_mod_p  
  aのpを法とする逆数を求める。つまり、`ax % p == 1`となるようなxを求める。
- add  
  点同士の加算を定義する。公式は  
  $R_x = \lambda^2 - P_x - Q_x, R_y = \lambda(P_x - R_x) - P_y$　ただし  
  $\lambda = \frac{Q_y - P_y}{Q_x - P_x} (P \neq Q), \lambda = \frac{3P_x^2 + a}{2P_x} (P = Q)$。
- mul  
  バイナリ法によって点のスカラー倍を定義する。
  P + P = 2Pを求め、2P + 2P = 4P、8P、と計算していき、それらをkを二進数にした時1になっているところの分だけ足せば、最終的にkPが求められる。

### 実行結果

16バイトのスカラー倍演算
`mul(false,"0x79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798","0x483ada7726a3c4655da4fbfc0e1108a8fd17b448a68554199c47d08ffb10d4b8","0x8888888888888888")`で約2010870gas発生した。

64バイトのスカラー倍演算`false,"0x79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798","0x483ada7726a3c4655da4fbfc0e1108a8fd17b448a68554199c47d08ffb10d4b8","0x4545e45bae6aed7e1661208d5fb57473f4902b0cfe365de7f72eab60db999cda"`で約10000000gas発生した。

### もう少し効率的な実装

<https://github.com/androlo/standard-contracts/blob/master/contracts/src/crypto/Secp256k1.sol>ここにもう少し効率的なコードが載っている。こちらを使うと、先ほどの64バイトのスカラー倍演算が5000000gasほどで済んだ。

そちらに載ってあるコード（抜粋）はこちら。

```javascript
function _add(uint[3] memory P, uint[3] memory Q) internal constant returns (uint[3] memory R) {
    if(P[2] == 0)
        return Q;
    if(Q[2] == 0)
        return P;
    uint p = pp;
    uint[4] memory zs; // Pz^2, Pz^3, Qz^2, Qz^3
    zs[0] = mulmod(P[2], P[2], p);
    zs[1] = mulmod(P[2], zs[0], p);
    zs[2] = mulmod(Q[2], Q[2], p);
    zs[3] = mulmod(Q[2], zs[2], p);
    uint[4] memory us = [
        mulmod(P[0], zs[2], p),
        mulmod(P[1], zs[3], p),
        mulmod(Q[0], zs[0], p),
        mulmod(Q[1], zs[1], p)
    ]; // Pu, Ps, Qu, Qs
    if (us[0] == us[2]) {
        if (us[1] != us[3])
            return;
        else {
            return _double(P);
        }
    }
    uint h = addmod(us[2], p - us[0], p);
    uint r = addmod(us[3], p - us[1], p);
    uint h2 = mulmod(h, h, p);
    uint h3 = mulmod(h2, h, p);
    uint Rx = addmod(mulmod(r, r, p), p - h3, p);
    Rx = addmod(Rx, p - mulmod(2, mulmod(us[0], h2, p), p), p);
    R[0] = Rx;
    R[1] = mulmod(r, addmod(mulmod(us[0], h2, p), p - Rx, p), p);
    R[1] = addmod(R[1], p - mulmod(us[1], h3, p), p);
    R[2] = mulmod(h, mulmod(P[2], Q[2], p), p);
}

// Multiplication dP. P affine, wNAF: w=5
// Params: d, Px, Py
// Output: Jacobian Q
function _mul(uint d, uint[2] memory P) internal constant returns (uint[3] memory Q) {
    uint p = pp;
    if (d == 0) // TODO
        return;
    uint dwPtr; // points to array of NAF coefficients.
    uint i;

    // wNAF
    assembly
    {
            let dm := 0
            dwPtr := mload(0x40)
            mstore(0x40, add(dwPtr, 512)) // Should lower this.
        loop:
            jumpi(loop_end, iszero(d))
            jumpi(even, iszero(and(d, 1)))
            dm := mod(d, 32)
            mstore8(add(dwPtr, i), dm) // Don't store as signed - convert when reading.
            d := add(sub(d, dm), mul(gt(dm, 16), 32))
        even:
            d := div(d, 2)
            i := add(i, 1)
            jump(loop)
        loop_end:
    }

    // Pre calculation
    uint[3][8] memory PREC; // P, 3P, 5P, 7P, 9P, 11P, 13P, 15P
    PREC[0] = [P[0], P[1], 1];
    var X = _double(PREC[0]);
    PREC[1] = _addMixed(X, P);
    PREC[2] = _add(X, PREC[1]);
    PREC[3] = _add(X, PREC[2]);
    PREC[4] = _add(X, PREC[3]);
    PREC[5] = _add(X, PREC[4]);
    PREC[6] = _add(X, PREC[5]);
    PREC[7] = _add(X, PREC[6]);

    uint[16] memory INV;
    INV[0] = PREC[1][2];                            // a1
    INV[1] = mulmod(PREC[2][2], INV[0], p);         // a2
    INV[2] = mulmod(PREC[3][2], INV[1], p);         // a3
    INV[3] = mulmod(PREC[4][2], INV[2], p);         // a4
    INV[4] = mulmod(PREC[5][2], INV[3], p);         // a5
    INV[5] = mulmod(PREC[6][2], INV[4], p);         // a6
    INV[6] = mulmod(PREC[7][2], INV[5], p);         // a7

    INV[7] = ECCMath.invmod(INV[6], p);             // a7inv
    INV[8] = INV[7];                                // aNinv (a7inv)

    INV[15] = mulmod(INV[5], INV[8], p);            // z7inv
    for(uint k = 6; k >= 2; k--) {                  // z6inv to z2inv
        INV[8] = mulmod(PREC[k + 1][2], INV[8], p);
        INV[8 + k] = mulmod(INV[k - 2], INV[8], p);
    }
    INV[9] = mulmod(PREC[2][2], INV[8], p);         // z1Inv
    for(k = 0; k < 7; k++) {
        ECCMath.toZ1(PREC[k + 1], INV[k + 9], mulmod(INV[k + 9], INV[k + 9], p), p);
    }

    // Mult loop
    while(i > 0) {
        uint dj;
        uint pIdx;
        i--;
        assembly {
            dj := byte(0, mload(add(dwPtr, i)))
        }
        _doubleM(Q);
        if (dj > 16) {
            pIdx = (31 - dj) / 2; // These are the "negative ones", so invert y.
            _addMixedM(Q, [PREC[pIdx][0], p - PREC[pIdx][1]]);
        }
        else if (dj > 0) {
            pIdx = (dj - 1) / 2;
            _addMixedM(Q, [PREC[pIdx][0], PREC[pIdx][1]]);
        }
    }
}
```

このコードでは、普通の楕円曲線の(x, y)の座標系「アフィン座標」を一旦(x, y, z)の座標系「ヤコビアン座標」に変換してから加算を繰り返し、その後またアフィン座標に戻している。アフィン座標からヤコビアン座標へは(x, y) -> (x, y, 1)で変換でき、ヤコビアン座標からアフィン座標へは(x, y, z) -> (x / z^2, y / z^3)で変換できる。何故こんなことをするかといえば、ヤコビアン座標系の点同士の加算には除算`inv_mod_p`がいらなく、乗算`mulmod`と加算`addmod`だけで済むから。inv_mod_pは非常にコストが高い関数だ。一方、`mulmod`や`invmod`はEVMのオペコードとして用意されているからgasは安く済む。

こうやって処理系ごとに適切な計算方法を考えてやるのも、暗号の醍醐味なのかな。

### 追記

Ethereumの0x000....1から0x000....8まではpre-compiled contractsという特別なコントラクトアドレスとなっている。0x000...06を使うとEC加算、0x000...07を使うとEC乗算ができるようになる。ただしこの際、曲線はalt_bn128という謎曲線が使われている。署名にはsecp256k1が使われてるのに...。