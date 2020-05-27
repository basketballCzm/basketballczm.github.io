---
title: "Zksnark加法乘法同态"
date: 2020-05-27T16:52:36+08:00
draft: false
summary: "zksnark加法和乘法同态理解"
tags: [zksnark]
---

目前，全同态加密（Fully Homomorphic Encryption, FHE）因当前算法复杂度问题，离实用仍有距离。
半同态加密一般指的是：加法同态和乘法同态。

* 加法同态：满足E(X)E(Y)=E(X+Y)。典型的例子为：椭圆曲线加密算法中，$E(x)=g^x$（其中g为椭圆曲线的generator），则$E(x)E(y)=g^xg^y=g^{(x+y)}=E(x+y)$，具有加法同态性。 以及Pedersen Commit也具是加法同态性。
* 乘法同态：满足E(X)E(Y)=E(XY)。典型的例子为：RSA加密算法中，$E(x)=x^e$（其中e为公钥），则$E(x)E(y)=x^ey^e=(xy)^e=E(xy)$，具有乘法同态性。



1. zk-SNARK算法中的加法，可利用椭圆曲线加密算法自身的加法同态性来实现，即E(X+Y)=E(X)E(Y)；
2. zk-SNARK算法中的乘法，可换用椭圆曲线pairing的特征——$e(g^x,g^y)=e(g,g)^{xy}$来实现，即$e(E(x), E(y))=e(g,g)^{xy}$。举例： $W := E(w(s))$，$W’ := E(α w(s))$，则$e(W’, E(1)) = e(E(α w(s)), E(1)) = e(g,g)αw(s) = e(g^{w(s)},g^α) = e(E(w(s)), E(α)) = e(W, E(α))$。这个特征可用于zk-SNARK verfier判断等式成立，即多项式条件成立。
   

The pairing Zcash actually uses is the optimal Ate [pairing](https://www.esat.kuleuven.be/cosic/publications/talk-96.pdf) , which is based on the Tate reduced pairing, and can be computed more efficiently than TateTate.
