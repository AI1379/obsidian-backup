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
