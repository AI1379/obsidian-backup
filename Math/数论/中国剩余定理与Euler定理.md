---
tags:
  - 数论
  - 同余
---
## $\mathbb{Z}/p\mathbb{Z}$ 是整环

> 代数背景：$\mathbb{Z}/p\mathbb{Z}$ 是整环等价于 $p\mathbb{Z}$ 是 $\mathbb{Z}$ 的素理想，参见 [[域]] 中素理想与极大理想的讨论。

给定正整数 $M$，模 $M$ 剩余类的集合 $\mathbb{Z}/M\mathbb{Z}$ 在加法和乘法下构成环。当 $M = p$ 为素数时，这个环是**整环**。

> **定理**：若 $p$ 为素数，$\bar{a} \cdot \bar{b} = \bar{0}$ 在 $\mathbb{Z}/p\mathbb{Z}$ 中成立，则 $\bar{a} = \bar{0}$ 或 $\bar{b} = \bar{0}$。

证明：翻译成同余语言即 $p \mid ab \Rightarrow p \mid a$ 或 $p \mid b$，这是素数的定义。

**推论**：$(\mathbb{Z}/p\mathbb{Z})[x]$ 也是整环。设 $f = a_n x^n + \cdots$，$g = b_m x^m + \cdots$ 为两个非零多项式，则 $fg$ 的最高次项系数为 $a_n b_m \neq 0$（因为 $\mathbb{Z}/p\mathbb{Z}$ 是整环），故 $fg \neq 0$。

这个事实在高斯引理（多项式版本）和 Eisenstein 判别法的证明中是关键的。

---

## 中国剩余定理

> 一般环上的 CRT 参见 [[中国剩余定理]]（取 $R = \mathbb{Z}$ 即得本节的情形）。

### 环论表述

> **定理（CRT）**：设 $\gcd(m,n)=1$，则映射
> $$\varphi: \mathbb{Z}/mn\mathbb{Z} \to \mathbb{Z}/m\mathbb{Z} \times \mathbb{Z}/n\mathbb{Z}, \quad [a]_{mn} \mapsto ([a]_m, [a]_n)$$
> 是**环同构**。

### 经典表述（孙子定理）

设 $\gcd(m,n)=1$，则同余方程组

$$x \equiv a \pmod{m}, \quad x \equiv b \pmod{n}$$

在 $\mathbb{Z}/mn\mathbb{Z}$ 中有唯一解。

### 证明

**良定性**：若 $a' \equiv a \pmod{mn}$，则 $a' \equiv a \pmod{m}$ 且 $a' \equiv a \pmod{n}$。

**单射**：$\varphi([a]) = \varphi([a'])$ 推出 $m \mid (a-a')$ 且 $n \mid (a-a')$。因 $\gcd(m,n)=1$，$mn \mid (a-a')$。

**满射（构造性）**：由 Bézout 定理，取 $mu + nv = 1$。令 $e_1 = nv$，$e_2 = mu$，则

$$e_1 \equiv 1 \pmod{m}, \quad e_1 \equiv 0 \pmod{n}$$
$$e_2 \equiv 0 \pmod{m}, \quad e_2 \equiv 1 \pmod{n}$$

对任意 $(a,b)$，取 $x = a \cdot e_1 + b \cdot e_2$。这里 $e_1, e_2$ 起到"基"的作用。

**保持环运算**：直接验证 $\varphi$ 保持加法和乘法。

### 推广

对 $m = m_1 m_2 \cdots m_r$ 且 $\gcd(m_i, m_j) = 1$（$i \neq j$），有环同构

$$\mathbb{Z}/m\mathbb{Z} \cong \prod_{i=1}^{r} \mathbb{Z}/m_i\mathbb{Z}$$

### 例题

解 $x \equiv 2 \pmod{3}$，$x \equiv 3 \pmod{5}$，$x \equiv 2 \pmod{7}$。

找"基"元素：$e_1 = 70$（模 3 余 1，模 5,7 余 0），$e_2 = 21$（模 5 余 1），$e_3 = 15$（模 7 余 1）。

解为 $x = 2 \times 70 + 3 \times 21 + 2 \times 15 = 233 \equiv 23 \pmod{105}$。

---

## 广义中国剩余定理（EXCRT）

普通 CRT 要求模数两两互素。若模数不互素，就进入常说的 **EXCRT** 情形。

> 环论视角下，EXCRT 对应于理想不必互素时的相容性条件，参见 [[中国剩余定理]]。

### 两个同余的判据

> **定理（广义 CRT，两个模数）**：同余方程组
> $$x \equiv a \pmod{m}, \quad x \equiv b \pmod{n}$$
> 有解当且仅当
> $$a \equiv b \pmod{\gcd(m,n)}.$$
> 若有解，则解在模 $\operatorname{lcm}(m,n)$ 意义下唯一。

### 成立性证明

设 $d = \gcd(m,n)$。

**必要性**：若存在 $x$ 使

$$x \equiv a \pmod{m}, \quad x \equiv b \pmod{n},$$

则 $m \mid (x-a)$ 且 $n \mid (x-b)$，从而 $d \mid (x-a)$ 且 $d \mid (x-b)$。两式相减得

$$d \mid (a-b),$$

即

$$a \equiv b \pmod d.$$

**充分性**：现设 $a \equiv b \pmod d$，即 $d \mid (b-a)$。写成

$$b-a = dt.$$

再设

$$m = dm_1,\quad n = dn_1,\quad \gcd(m_1,n_1)=1.$$

令

$$x = a + mk,$$

则第一个同余自动满足。要使第二个同余也成立，只需

$$a + mk \equiv b \pmod n,$$

即

$$mk \equiv b-a \pmod n.$$

代入上式分解，得到

$$dm_1k \equiv dt \pmod{dn_1}.$$

约去 $d$，化为

$$m_1k \equiv t \pmod{n_1}.$$

由于 $\gcd(m_1,n_1)=1$，$m_1$ 在模 $n_1$ 意义下可逆，因此此同余必有解。取一组解 $k$，便得到

$$x = a + mk$$

是原方程组的一组解。

### 为什么唯一到最小公倍数

若 $x_1,x_2$ 都是解，则

$$x_1 \equiv x_2 \pmod m,\quad x_1 \equiv x_2 \pmod n.$$

于是 $m \mid (x_1-x_2)$ 且 $n \mid (x_1-x_2)$，故

$$\operatorname{lcm}(m,n) \mid (x_1-x_2).$$

所以所有解构成一个模 $\operatorname{lcm}(m,n)$ 的剩余类。

### 与算法中的 excrt 的关系

算法里的 `excrt` 本质上就是反复使用上面的两模合并：

把

$$x \equiv a_1 \pmod{m_1}, \quad x \equiv a_2 \pmod{m_2}$$

先合并成

$$x \equiv A \pmod{\operatorname{lcm}(m_1,m_2)},$$

再与第三个同余继续合并。每一步都要检查新旧余数之差是否被当前模数的最大公因数整除；若不整除，则整个方程组无解。

> 也就是说，`excrt` 不是一个和 CRT 完全不同的定理，而是“非互素模数情形下的 CRT + 迭代合并算法”。

### 例子

例如

$$x \equiv 5 \pmod{12}, \quad x \equiv 11 \pmod{18}.$$

这里 $\gcd(12,18)=6$，而 $11-5=6$ 可被 $6$ 整除，所以有解。并且解唯一到模

$$\operatorname{lcm}(12,18)=36.$$

令 $x = 5 + 12k$，代入第二式：

$$12k \equiv 6 \pmod{18}.$$

约去 $6$，得

$$2k \equiv 1 \pmod 3,$$

所以 $k \equiv 2 \pmod 3$，于是

$$x = 5 + 12(2+3t) = 29 + 36t.$$

即

$$x \equiv 29 \pmod{36}.$$

---

## 欧拉函数

### 定义

**欧拉函数** $\varphi: \mathbb{Z}_{>0} \to \mathbb{Z}_{\geq 0}$ 定义为

$$\varphi(m) = |(\mathbb{Z}/m\mathbb{Z})^\times|$$

即 $1$ 到 $m$ 中与 $m$ 互素的正整数的个数。

### 乘性

由 CRT 的环同构，若 $\gcd(m,n)=1$，则

$$(\mathbb{Z}/mn\mathbb{Z})^\times \cong (\mathbb{Z}/m\mathbb{Z})^\times \times (\mathbb{Z}/n\mathbb{Z})^\times$$

因此 $\varphi(mn) = \varphi(m) \cdot \varphi(n)$。

### 素数幂上的值

$$\varphi(p^k) = p^k - p^{k-1} = p^k\left(1 - \frac{1}{p}\right)$$

### 一般公式

设 $m = p_1^{k_1} p_2^{k_2} \cdots p_r^{k_r}$，则

$$\varphi(m) = m \prod_{i=1}^{r} \left(1 - \frac{1}{p_i}\right)$$

---

## 欧拉定理

> 代数视角：Euler 定理是 [[群|Lagrange 定理]] 在群 $(\mathbb{Z}/m\mathbb{Z})^\times$ 上的直接推论——元素阶整除群阶。

> **定理**：设 $\gcd(a,m)=1$，则 $a^{\varphi(m)} \equiv 1 \pmod{m}$。

### 证明

由于 $\gcd(a,m)=1$，$a$ 是 $(\mathbb{Z}/m\mathbb{Z})^\times$ 中的可逆元。考虑序列 $a^1, a^2, \ldots$，由有限性存在 $i < j$ 使 $a^i \equiv a^j \pmod{m}$，乘以 $a^{-i}$ 得 $a^{j-i} \equiv 1$。设 $n_0$ 为使得 $a^{n_0} \equiv 1 \pmod{m}$ 的最小正整数。

在 $(\mathbb{Z}/m\mathbb{Z})^\times$ 上定义等价关系：$b \sim c \iff \exists\, k \geq 0,\, b = c \cdot a^k$。每个等价类恰好有 $n_0$ 个元素 $\{b, ba, \ldots, ba^{n_0-1}\}$。等价类划分群，故 $n_0 \mid \varphi(m)$，从而 $a^{\varphi(m)} \equiv 1$。

---

## 费马小定理

> **推论**：设 $p$ 为素数，则 $a^p \equiv a \pmod{p}$。

若 $p \mid a$，两边 $\equiv 0$。若 $p \nmid a$，由欧拉定理 $a^{p-1} \equiv 1 \pmod{p}$，乘以 $a$ 即得。

---

## Frobenius 自同态

> Frobenius 自同态在有限域理论中是核心工具，参见 [[有限 Abel 群与有限域]] 中更一般的讨论。

在 $\mathbb{F}_p[x]$ 上定义 $\mathrm{Frob}_p: f \mapsto f^p$（对系数取 $p$ 次方，将 $x$ 替换为 $x^p$）。

**核心性质**：$(f+g)^p = f^p + g^p$。

证明：二项式展开 $(f+g)^p = \sum_{k=0}^{p} \binom{p}{k} f^{p-k}g^k$。对 $0 < k < p$，$\binom{p}{k}$ 被 $p$ 整除，故在 $\mathbb{F}_p$ 中为零。

这是特征 $p$ 的环中特有的性质，在一般整数环中不成立。Frobenius 自同态在代数数论中具有根本性的重要地位。
