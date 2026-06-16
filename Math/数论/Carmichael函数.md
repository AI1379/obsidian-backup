---
tags:
  - 数论
---
Carmichael 函数通常记作 (\lambda(n))。它描述的是：

$$
\boxed{\text{模 }n\text{ 的所有可逆剩余类，其幂统一回到 }1\text{ 所需的最小指数}}
$$

更严格地说：

$$
\lambda(n)=\min\{m\ge 1: a^m\equiv 1 \pmod n \forall (a,n)=1\quad a\in \mathbb{Z}\}
$$

也就是说，(\lambda(n)) 是满足

$$
a^{\lambda(n)}\equiv1\pmod n
$$

对每个与 (n) 互素的 (a) 都成立的最小正整数。

---

## 1. 它和欧拉函数有什么关系？

欧拉定理说：

$$
(a,n)=1
\quad\Longrightarrow\quad
a^{\varphi(n)}\equiv1\pmod n.
$$

所以 $\varphi(n)$ 一定是一个可行指数。但它未必是最小的。

Carmichael 函数就是把这个统一指数缩到最小：

$$
\boxed{\lambda(n)\mid \varphi(n)}.
$$

例如 (n=8)：

$$
\varphi(8)=4.
$$

但对任意奇数 (a)，都有

$$
a^2\equiv1\pmod8.
$$

因此

$$
\lambda(8)=2,
$$

而不是 $4$。

所以可以把两者理解为：

- $\varphi(n)$：模 (n) 的可逆剩余类有多少个；
    
- $\lambda(n)$：这些可逆剩余类的阶的最小公倍数。

---

## 2. 从群论角度定义

模 $n$ 的可逆剩余类构成乘法群

$$
(\mathbb Z/n\mathbb Z)^\times.
$$

Carmichael 函数就是这个有限群的指数：

$$
\boxed{
\lambda(n)=\exp\bigl((\mathbb Z/n\mathbb Z)^\times\bigr).
}
$$

有限群 $G$ 的指数定义为：

$$
\exp(G)=\operatorname{lcm}{\operatorname{ord}(g):g\in G}.
$$

因此

$$
\lambda(n) = 
\mathrm{lcm}
\{
\mathrm{ord}_n(a):(a,n)=1
\}.
$$

这里 $\operatorname{ord}_n(a)$ 是 (a) 模 (n) 的乘法阶，即最小正整数 (r)，使得

$$
a^r\equiv1\pmod n.
$$

所以 Carmichael 函数不是在研究某一个 (a) 的循环节，而是在研究所有可逆元素的阶的最小公倍数。

---

## 3. 素数幂情形

Carmichael 函数的计算公式以素数幂为基础。

对于奇素数 (p)：

$$
\boxed{
\lambda(p^k)=\varphi(p^k)=p^{k-1}(p-1).
}
$$

对于 (2) 的幂：

$$
\lambda(2)=1,
\qquad
\lambda(4)=2,
$$

而当 (k\ge3) 时：

$$
\boxed{
\lambda(2^k)=2^{k-2}.
}
$$

这里出现了例外，因为模 (2^k) 的可逆剩余类群在 (k\ge3) 时不是循环群。

---

## 4. 一般 (n) 的计算公式

若

$$
n=\prod_{i=1}^r p_i^{\alpha_i},
$$

那么由中国剩余定理，

$$
(\mathbb Z/n\mathbb Z)^\times
\cong
\prod_{i=1}^r
(\mathbb Z/p_i^{\alpha_i}\mathbb Z)^\times.
$$

直积群的指数是各部分指数的最小公倍数，所以

$$
\boxed{
\lambda(n)
=
\operatorname{lcm}
\left(
\lambda(p_1^{\alpha_1}),
\dots,
\lambda(p_r^{\alpha_r})
\right).
}
$$

---

## 5. 例如计算 $\lambda(100)$

因为

$$
100=4\cdot25,
\qquad (4,25)=1,
$$

所以

$$
\lambda(100)
=
\operatorname{lcm}(\lambda(4),\lambda(25)).
$$

而

$$
\lambda(4)=2,
$$

$$
\lambda(25)=\varphi(25)=20.
$$

因此

$$
\boxed{
\lambda(100)=\operatorname{lcm}(2,20)=20.
}
$$

所以对于任意与 (100) 互素的 (a)，都有

$$
a^{20}\equiv1\pmod{100}.
$$

这就是上一题中为什么指数只需要模 (20)。

---

## 6. 为什么奇素数幂时等于欧拉函数？

对于奇素数 (p)，群

$$
(\mathbb Z/p^k\mathbb Z)^\times
$$

是循环群，阶为

$$
\varphi(p^k)=p^{k-1}(p-1).
$$

循环群中存在一个元素，其阶等于整个群的阶。因此所有元素阶的最小公倍数也等于群阶：

$$
\lambda(p^k)=\varphi(p^k).
$$

但对于 (2^k)，当 (k\ge3) 时，

$$
(\mathbb Z/2^k\mathbb Z)^\times
\cong C_2\times C_{2^{k-2}},
$$

因此它的指数是

$$
\operatorname{lcm}(2,2^{k-2})

2^{k-2}.
$$

这比群的阶

$$
\varphi(2^k)=2^{k-1}
$$

小一半。

---

## 7. 一个很重要的使用条件

只有当

$$
(a,n)=1
$$

时，才能直接说

$$
a^{\lambda(n)}\equiv1\pmod n.
$$

因此对于

$$
a^N\pmod n,
$$

若 ((a,n)=1)，则指数可以模 (\lambda(n)) 降低：

$$
a^N\equiv a^{N\bmod \lambda(n)}\pmod n.
$$

严格来说，如果 (N\bmod\lambda(n)=0)，右侧应理解为 (a^0=1)，这没有问题，因为底数可逆。

若 ((a,n)\ne1)，则不能直接这样做，通常要先把 (n) 分解为素数幂，再分别处理。

一句话概括：

$$
\boxed{
\lambda(n)\text{ 是模 }n\text{ 的可逆剩余类群的指数。}
}
$$