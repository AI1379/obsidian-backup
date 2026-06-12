---
tags:
  - 组合数学
  - 数论
---
## Catalan 数的定义

**Catalan 数**定义为

$$C_n = \frac{1}{n+1}\binom{2n}{n} = \frac{(2n)!}{(n+1)!\,n!}$$

前几项为 $1, 1, 2, 5, 14, 42, 132, \ldots$。

Catalan 数出现在许多组合问题中：$n$ 对括号的合法匹配数、$n+1$ 个节点的二叉树个数、凸 $(n+2)$ 边形的三角剖分数等。

它满足递推关系

$$C_{n+1} = \sum_{i=0}^{n} C_i C_{n-i}, \quad C_0 = 1$$

以及显式递推

$$C_{n+1} = \frac{2(2n+1)}{n+2}\,C_n$$

渐近行为：

$$C_n \sim \frac{4^n}{n^{3/2}\sqrt{\pi}}$$

---

## 素数计数函数 $\pi(x)$

定义 $\pi(x)$ 为不超过 $x$ 的素数个数。素数定理（Prime Number Theorem）给出了其渐近行为：

> **素数定理**（Hadamard, de la Vallée-Poussin, 1896）：$\displaystyle\pi(x) \sim \frac{x}{\log x}$，即
> $$\lim_{x \to \infty} \frac{\pi(x)}{x/\log x} = 1$$

更强的结论是 $\pi(x) = \mathrm{Li}(x) + O(xe^{-c\sqrt{\log x}})$，其中 $\mathrm{Li}(x) = \int_2^x \frac{dt}{\log t}$。在 [[Riemann zeta函数与解析数论基础|RH]] 成立时，误差项可改进为 $O(\sqrt{x}\log x)$。

以下用初等方法给出 $\pi(x)$ 的阶的估计。

---

## Chebyshev 函数与初等估计

### $\theta(x)$ 和 $\psi(x)$

定义两个 **Chebyshev 函数**：

$$\theta(x) = \sum_{p \leq x} \log p$$

$$\psi(x) = \sum_{\substack{p^k \leq x \\ p \text{ 素数}}} \log p = \sum_{n \leq x} \Lambda(n)$$

其中 $\Lambda$ 是 von Mangoldt 函数：$\Lambda(n) = \log p$ 若 $n = p^k$，否则 $0$。

两者的关系为 $\theta(x) \leq \psi(x) \leq \theta(x) + O(\sqrt{x}\log x)$，因此它们有相同的渐近行为。

### $\theta(x)$ 的上界

> **定理**：$\theta(x) < (2\log 2)\, x$ 对所有 $x \geq 1$ 成立（即 $\theta(x) < 1.387\,x$）。

**证明**：考虑 $\binom{2n}{n} = \frac{(2n)!}{(n!)^2}$。

一方面，$\binom{2n}{n} \leq \sum_{i=0}^{2n} \binom{2n}{i} = 2^{2n} = 4^n$。

另一方面，所有满足 $n < p \leq 2n$ 的素数 $p$ 整除 $(2n)!$ 但不整除 $(n!)^2$（因为 $p$ 在 $(2n)!$ 中出现一次，在 $n!$ 中不出现），因此

$$\prod_{n < p \leq 2n} p \;\Big|\; \binom{2n}{n}$$

于是 $\prod_{n < p \leq 2n} p \leq 4^n$，取对数得

$$\theta(2n) - \theta(n) = \sum_{n < p \leq 2n} \log p \leq n \log 4$$

对任意 $x$，取 $n = \lfloor x/2^k \rfloor$ 逐层累加：

$$\theta(x) \leq \sum_{k=0}^{\infty} (\theta(x/2^k) - \theta(x/2^{k+1})) \leq \sum_{k=0}^{\infty} \frac{x}{2^k} \log 4 = 2x\log 4 \cdot \frac{1}{2} = (2\log 2)\,x$$

### $\theta(x)$ 的下界

> **定理**：$\theta(x) \geq c\,x$ 对某个常数 $c > 0$ 和充分大的 $x$ 成立。

**证明**：考虑 $N = \binom{2n}{n}$，则 $\log N = \log(2n)! - 2\log(n!)$。由 Stirling 公式，

$$\log N = 2n\log 2 - \frac{1}{2}\log n + O(1)$$

因此 $\log N \sim 2n\log 2$。

另一方面，$N = \prod_p p^{\alpha_p}$，其中 $\alpha_p = \sum_{k=1}^{\infty}\left(\left\lfloor\frac{2n}{p^k}\right\rfloor - 2\left\lfloor\frac{n}{p^k}\right\rfloor\right)$。对 $p > \sqrt{2n}$，$\alpha_p \leq 1$，所以

$$\log N \leq \psi(2n) = \theta(2n) + \sum_{p \leq \sqrt{2n}} \log p \cdot O\!\left(\frac{\log n}{\log p}\right)$$

由于 $\psi(2n) \leq \theta(2n) + O(\sqrt{n}\log^2 n)$，结合 $\log N \sim 2n\log 2$，得 $\theta(2n) \geq (2\log 2 - \varepsilon)n$。

### 从 $\theta(x)$ 到 $\pi(x)$

由 $\theta(x) = \sum_{p \leq x} \log p$ 及分部求和（Abel 求和），$\theta(x)$ 与 $\pi(x)$ 的渐近行为等价：

$$\pi(x) \sim \frac{x}{\log x} \iff \theta(x) \sim x$$

Chebyshev 的初等方法给出了

$$c_1 \,\frac{x}{\log x} \leq \pi(x) \leq c_2 \,\frac{x}{\log x}$$

其中 $c_1 \approx 0.92$，$c_2 \approx 1.10$。这虽然不能直接给出素数定理，但证明了 $\pi(x)$ 的阶确实是 $x/\log x$。

---

## Catalan 数与 $\pi(x)$ 的联系

Catalan 数 $C_n = \frac{1}{n+1}\binom{2n}{n}$ 在上述估计中自然出现。核心观察是：

$\binom{2n}{n}$ 的素因子恰好是所有 $n < p \leq 2n$ 的素数（以及 $\leq \sqrt{2n}$ 的素数的适当幂次），因此对 $\binom{2n}{n}$ 的精确分析可以直接给出 $\theta(2n) - \theta(n)$ 的界。

利用 Catalan 数的渐近公式 $C_n \sim 4^n/(n^{3/2}\sqrt{\pi})$，可以给出更精细的估计。

---

## 素数定理的解析证明路线

初等方法难以证明 $\pi(x) \sim x/\log x$。解析证明的核心路线（详见 [[Riemann zeta函数与解析数论基础]]）：

1. $\zeta(s)$ 的欧拉乘积 $\zeta(s) = \prod_p (1 - p^{-s})^{-1}$ 将素数信息编码为解析对象。
2. 对数导数 $-\zeta'(s)/\zeta(s) = \sum_{p,k} \frac{\log p}{p^{ks}}$ 直接联系 $\psi(x)$ 与 $\zeta$ 的零点分布。
3. **关键**：$\zeta(s)$ 在 $\mathrm{Re}(s) = 1$ 上无零点（这是最核心的一步）。
4. 由 Wiener-Ikehara 定理或直接积分推导 $\psi(x) \sim x$，进而 $\pi(x) \sim x/\log x$。

> Erdős 和 Selberg 在 1949 年各自给出了素数定理的**初等证明**（不依赖复分析），但证明并不简单。
