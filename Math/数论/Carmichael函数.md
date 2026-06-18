---
tags:
  - 数论
  - 同余
aliases: 卡迈克尔函数
---

Carmichael 函数通常记作 $\lambda(n)$。它描述的是：

$$
\boxed{\text{模 }n\text{ 的所有可逆剩余类，其幂统一回到 }1\text{ 所需的最小正指数}}
$$

更严格地说：

$$
\boxed{\lambda(n)=\min\{m\geq 1:a^m\equiv1\pmod n\text{ 对所有 }(a,n)=1\text{ 成立}\}.}
$$

也就是说，$\lambda(n)$ 是满足

$$
a^{\lambda(n)}\equiv1\pmod n
$$

对每个与 $n$ 互素的整数 $a$ 都成立的最小正整数。

这里的关键词是**可逆剩余类**。如果 $(a,n)=1$，那么 $a$ 在

$$
(\mathbb Z/n\mathbb Z)^\times
$$

中是一个可逆元素；如果 $(a,n)\neq1$，则不能直接套用 Carmichael 函数的降指数结论。

---

## 1. 它和欧拉函数有什么关系？

欧拉定理说：

$$
(a,n)=1
\quad\Longrightarrow\quad
a^{\varphi(n)}\equiv1\pmod n.
$$

所以 $\varphi(n)$ 一定是一个**统一可行指数**：不管取哪个可逆剩余类，把它提升到 $\varphi(n)$ 次幂都会回到 $1$。

Carmichael 函数做的事情更精细：它取所有统一可行指数中最小的那个。

从群论看，$\varphi(n)$ 是群

$$
(\mathbb Z/n\mathbb Z)^\times
$$

的阶，而 $\lambda(n)$ 是这个群的指数。由于每个元素的阶都整除群的阶，所有元素阶的最小公倍数也整除群的阶，因此

$$
\boxed{\lambda(n)\mid \varphi(n)}.
$$

但二者一般不相等。最小的区别来自 $n=8$：

$$
\varphi(8)=4.
$$

模 $8$ 的可逆剩余类是

$$
(\mathbb Z/8\mathbb Z)^\times=\{1,3,5,7\}.
$$

逐个平方可得

$$
1^2\equiv3^2\equiv5^2\equiv7^2\equiv1\pmod 8,
$$

所以

$$
\lambda(8)=2.
$$

这说明：

- $\varphi(n)$：模 $n$ 的可逆剩余类有多少个；
- $\lambda(n)$：这些可逆剩余类的阶的最小公倍数，也就是最小统一回到 $1$ 的指数。

一个有用的判断是：若 $(\mathbb Z/n\mathbb Z)^\times$ 是循环群，则

$$
\lambda(n)=\varphi(n).
$$

因为循环群中存在阶等于整个群阶的元素。若该群不是循环群，则 $\lambda(n)$ 往往会比 $\varphi(n)$ 小。

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
\exp(G)=\operatorname{lcm}\{\operatorname{ord}(g):g\in G\}.
$$

因此

$$
\lambda(n)=
\operatorname{lcm}
\{
\operatorname{ord}_n(a):(a,n)=1
\}.
$$

这里 $\operatorname{ord}_n(a)$ 是 $a$ 模 $n$ 的乘法阶，即最小正整数 $r$，使得

$$
a^r\equiv1\pmod n.
$$

所以 Carmichael 函数不是在研究某一个 $a$ 的循环节，而是在研究所有可逆元素的阶的最小公倍数。

---

## 3. 素数幂情形

Carmichael 函数的计算公式以素数幂为基础。

对于奇素数 $p$：

$$
\boxed{
\lambda(p^k)=\varphi(p^k)=p^{k-1}(p-1).
}
$$

理由是：当 $p$ 为奇素数时，模 $p^k$ 的可逆剩余类群

$$
(\mathbb Z/p^k\mathbb Z)^\times
$$

是循环群，其阶为

$$
\varphi(p^k)=p^k-p^{k-1}=p^{k-1}(p-1).
$$

循环群的指数等于群阶，所以 $\lambda(p^k)=\varphi(p^k)$。

对于 $2$ 的幂，需要单独列出：

$$
\lambda(2)=1,
\qquad
\lambda(4)=2,
$$

而当 $k\ge3$ 时：

$$
\boxed{
\lambda(2^k)=2^{k-2}.
}
$$

这里出现例外，是因为模 $2^k$ 的可逆剩余类群在 $k\ge3$ 时不再是循环群，而有结构

$$
(\mathbb Z/2^k\mathbb Z)^\times
\cong C_2\times C_{2^{k-2}}.
$$

因此它的指数是

$$
\operatorname{lcm}(2,2^{k-2})=2^{k-2}.
$$

可以把素数幂公式记成一张表：

| 模数 | $\lambda$ 的值 |
|---|---:|
| $2$ | $1$ |
| $4$ | $2$ |
| $2^k,\ k\ge3$ | $2^{k-2}$ |
| $p^k,\ p$ 为奇素数 | $p^{k-1}(p-1)$ |

---

## 4. 一般 $n$ 的计算公式

若

$$
n=\prod_{i=1}^r p_i^{\alpha_i},
$$

那么由[[中国剩余定理与Euler定理|中国剩余定理]]，有群同构

$$
(\mathbb Z/n\mathbb Z)^\times
\cong
\prod_{i=1}^r
(\mathbb Z/p_i^{\alpha_i}\mathbb Z)^\times.
$$

直积群的指数是各部分指数的最小公倍数。原因是：

$$
(g_1,\ldots,g_r)^m=e
$$

当且仅当每个分量都满足

$$
g_i^m=e_i.
$$

所以 $m$ 必须同时是每个分量阶的倍数，最小的统一指数就是这些指数的最小公倍数。

因此

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

实际计算时按三步做：

1. 分解素因数：$n=\prod p_i^{\alpha_i}$；
2. 对每个素数幂分别算 $\lambda(p_i^{\alpha_i})$；
3. 对这些值取最小公倍数。

也就是说，Carmichael 函数不是乘法函数，而是通过 CRT 变成“取最小公倍数”的函数。若 $(a,b)=1$，则

$$
\lambda(ab)=\operatorname{lcm}(\lambda(a),\lambda(b)),
$$

而不是一般地等于 $\lambda(a)\lambda(b)$。

---

## 5. 计算例子

### 例 1：$\lambda(100)$

因为

$$
100=4\cdot25,
\qquad (4,25)=1,
$$

所以

$$
\lambda(100)=\operatorname{lcm}(\lambda(4),\lambda(25)).
$$

而

$$
\lambda(4)=2,
\qquad
\lambda(25)=\varphi(25)=20.
$$

因此

$$
\boxed{
\lambda(100)=\operatorname{lcm}(2,20)=20.
}
$$

所以对于任意与 $100$ 互素的 $a$，都有

$$
a^{20}\equiv1\pmod{100}.
$$

这就解释了为什么在计算 $a^N\pmod{100}$ 且 $(a,100)=1$ 时，指数只需要模 $20$。

### 例 2：$\lambda(15)$

$$
15=3\cdot5,
$$

所以

$$
\lambda(15)=\operatorname{lcm}(\lambda(3),\lambda(5))
=\operatorname{lcm}(2,4)=4.
$$

而

$$
\varphi(15)=8.
$$

所以 $\lambda(15)$ 比 $\varphi(15)$ 小一半。

### 例 3：$\lambda(16)$

$$
\lambda(16)=\lambda(2^4)=2^{4-2}=4,
$$

而

$$
\varphi(16)=8.
$$

这来自模 $16$ 的可逆剩余类群不是循环群。

### 例 4：$\lambda(72)$

$$
72=2^3\cdot3^2.
$$

于是

$$
\lambda(72)=\operatorname{lcm}(\lambda(8),\lambda(9))
=\operatorname{lcm}(2,6)=6.
$$

而

$$
\varphi(72)=\varphi(8)\varphi(9)=4\cdot6=24.
$$

这个例子说明 $\lambda(n)$ 可能明显小于 $\varphi(n)$。

---

## 6. 为什么奇素数幂时等于欧拉函数？

对奇素数 $p$，群

$$
(\mathbb Z/p^k\mathbb Z)^\times
$$

是循环群，阶为

$$
\varphi(p^k)=p^{k-1}(p-1).
$$

循环群中存在一个生成元 $g$，它的阶等于整个群的阶：

$$
\operatorname{ord}_{p^k}(g)=\varphi(p^k).
$$

由于 $\lambda(p^k)$ 是所有元素阶的最小公倍数，它至少要被这个最大阶元素“撑到” $\varphi(p^k)$；另一方面所有元素阶都整除群阶，所以又有 $\lambda(p^k)\mid\varphi(p^k)$。因此

$$
\lambda(p^k)=\varphi(p^k).
$$

但对 $2^k$，当 $k\ge3$ 时，

$$
(\mathbb Z/2^k\mathbb Z)^\times
\cong C_2\times C_{2^{k-2}},
$$

因此它的指数是

$$
\operatorname{lcm}(2,2^{k-2})=2^{k-2}.
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

若 $(a,n)=1$，则指数可以模 $\lambda(n)$ 降低。写成标准形式：设

$$
N=q\lambda(n)+r,
\qquad 0\le r<\lambda(n),
$$

则

$$
a^N=a^{q\lambda(n)+r}
=(a^{\lambda(n)})^q a^r
\equiv a^r\pmod n.
$$

也就是

$$
\boxed{a^N\equiv a^{N\bmod\lambda(n)}\pmod n\qquad ((a,n)=1).}
$$

如果 $N\bmod\lambda(n)=0$，右侧就是 $a^0=1$，这没有问题，因为此时底数 $a$ 在模 $n$ 下可逆。

若 $(a,n)\neq1$，则不能直接这样做。通常要先把 $n$ 分解为素数幂，再分别处理每个模 $p^k$ 的情况，最后用 CRT 合并。

---

## 8. 和 Euler 降指数的比较

Euler 定理给出的是

$$
a^{\varphi(n)}\equiv1\pmod n,
\qquad (a,n)=1.
$$

Carmichael 函数给出更强的版本：

$$
a^{\lambda(n)}\equiv1\pmod n,
\qquad (a,n)=1.
$$

因为 $\lambda(n)\mid\varphi(n)$，所以 Carmichael 降指数至少不比 Euler 降指数差。

例如 $n=72$ 时，

$$
\varphi(72)=24,
\qquad
\lambda(72)=6.
$$

因此若 $(a,72)=1$，则

$$
a^6\equiv1\pmod{72},
$$

比用 $a^{24}\equiv1\pmod{72}$ 更精细。

---

## 9. 常见易错点

1. $\lambda(n)$ 只控制模 $n$ 的可逆剩余类。若 $(a,n)\neq1$，不能直接把指数模 $\lambda(n)$。
2. $\lambda(n)$ 一般不是乘法函数。互素合成时取的是最小公倍数：

$$
\lambda(ab)=\operatorname{lcm}(\lambda(a),\lambda(b))
\qquad ((a,b)=1).
$$

3. 对奇素数幂 $p^k$，$\lambda(p^k)=\varphi(p^k)$；但对 $2^k$，当 $k\ge3$ 时只有

$$
\lambda(2^k)=2^{k-2},
$$

而不是 $\varphi(2^k)=2^{k-1}$。

4. $\lambda(n)$ 是“所有可逆元素统一回到 $1$ 的最小指数”，不是某个单独元素的阶。单个元素 $a$ 的阶 $\operatorname{ord}_n(a)$ 总是整除 $\lambda(n)$。

---

## 10. 一句话概括

$$
\boxed{
\lambda(n)\text{ 是模 }n\text{ 的可逆剩余类群 }(\mathbb Z/n\mathbb Z)^\times\text{ 的指数。}
}
$$

它是 Euler 定理中 $\varphi(n)$ 的最小统一指数版本。计算时先分解素数幂，再对各部分的 $\lambda$ 取最小公倍数。
