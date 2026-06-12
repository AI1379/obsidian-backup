---
tags:
  - 数论
  - 解析数论
---
## 黎曼 ζ 函数

### 定义与收敛性

对复变量 $s = \sigma + it$（$\sigma = \Re(s) > 1$），定义

$$\zeta(s) = \sum_{n=1}^{\infty} \frac{1}{n^s}$$

> **定理**：对任意 $\delta > 0$，级数在 $\Re(s) \geq 1+\delta$ 上**绝对一致收敛**，因此 $\zeta(s)$ 在半平面 $\Re(s) > 1$ 上解析。

### 欧拉乘积公式

> **定理**：当 $\Re(s) > 1$ 时，
> $$\zeta(s) = \prod_{p} \frac{1}{1-p^{-s}}$$

**证明**：考虑有限部分积 $\prod_{p \leq N} (1-p^{-s})^{-1} = \prod_{p \leq N} \sum_{a=0}^{\infty} p^{-as}$。展开有限乘积，由算术基本定理每一项恰好对应一个素因子均 $\leq N$ 的整数 $n$。因此

$$\sum_{n=1}^{N} \frac{1}{n^s} \leq \prod_{p \leq N} \frac{1}{1-p^{-s}} \leq \zeta(s)$$

令 $N \to \infty$ 即得。

欧拉乘积是算术基本定理的分析等价物。它也给出素数无穷多的又一证明：若素数有限，乘积在 $s = 1$ 处有界，但 $\zeta(s) \to \infty$。

---

## Γ 函数

### 定义

对 $\Re(s) > 0$，

$$\Gamma(s) = \int_0^{\infty} y^{s-1} e^{-y}\, \frac{dy}{y}$$

### 基本性质

1. **解析性**：$\Gamma(s)$ 在 $\Re(s) > 0$ 上解析，可解析延拓到整个复平面（以非正整数为极点）。
2. **函数方程**：
   - $\Gamma(s+1) = s\Gamma(s)$（递推关系）
   - $\Gamma(s)\Gamma(1-s) = \frac{\pi}{\sin(\pi s)}$（余元公式）
   - $\Gamma(s)\Gamma(s+\frac{1}{2}) = 2^{1-2s}\sqrt{\pi}\,\Gamma(2s)$（倍元公式）
3. **特殊值**：$\Gamma(n) = (n-1)!$，$\Gamma(\frac{1}{2}) = \sqrt{\pi}$。

### 解析延拓

当 $\Re(s) > 1$ 时，分部积分给出 $\Gamma(s) = (s-1)\Gamma(s-1)$。定义 $f_1(s) = \Gamma(s+1)/s$，它在 $\Re(s) > -1$（$s \neq 0$）上解析且等于 $\Gamma(s)$。反复此过程可将 $\Gamma$ 延拓到整个复平面（挖掉非正整数）。

---

## θ 函数与 Poisson 求和公式

### θ 函数

对上半平面中的 $s$（$\Re(s) > 0$），

$$\theta(s) = \sum_{n=-\infty}^{\infty} e^{\pi i n^2 s}$$

### 完备 ζ 函数

通过变量替换 $y \mapsto \pi n^2 y$，对 $n$ 求和后得到

$$\pi^{-s/2}\Gamma\!\left(\frac{s}{2}\right)\zeta(s) = \frac{1}{2}\int_0^{\infty}(\theta(iy)-1)\, y^{s/2}\,\frac{dy}{y}$$

定义**完备 ζ 函数**

$$\xi(s) = \pi^{-s/2}\Gamma\!\left(\frac{s}{2}\right)\zeta(s)$$

### θ 函数的对称性

> **关键性质**：对上半平面中的 $\tau$，
> $$\theta\!\left(-\frac{1}{\tau}\right) = \sqrt{\frac{\tau}{i}}\,\theta(\tau)$$

**证明**：对 $f(x) = e^{\pi i \tau x^2}$ 计算 Fourier 变换，通过高斯积分和配方法得 $\hat{f}(\xi) = \frac{1}{\sqrt{-i\tau}} e^{-\pi i\xi^2/\tau}$。然后应用 **Poisson 求和公式** $\sum_{n \in \mathbb{Z}} f(n) = \sum_{k \in \mathbb{Z}} \hat{f}(k)$。

---

## ζ 函数的解析延拓

> **定理**：$\zeta(s)$ 可解析延拓到整个复平面，仅在 $s=1$ 处有**简单极点**，留数为 1。

**证明**：将 $\xi(s)$ 的积分拆为 $(0,1)$ 和 $(1,\infty)$ 两部分。

$(1,\infty)$ 部分对所有 $s$ 绝对收敛（$\theta(iy)-1$ 当 $y \to \infty$ 时指数衰减）。

对 $(0,1)$ 部分，变量替换 $y \mapsto 1/y$ 并利用 $\theta(i/y) = \sqrt{y}\,\theta(iy)$。最终得到

$$\xi(s) = \frac{1}{s(s-1)} + \frac{1}{2}\int_1^{\infty}\left(\theta(iy) - 1\right)\left(y^{s/2} + y^{(1-s)/2}\right)\frac{dy}{y}$$

右边的积分对所有 $s \in \mathbb{C}$ 绝对收敛。

---

## 函数方程

> **定理**：$\xi(s) = \xi(1-s)$，即
> $$\pi^{-s/2}\Gamma\!\left(\frac{s}{2}\right)\zeta(s) = \pi^{-(1-s)/2}\Gamma\!\left(\frac{1-s}{2}\right)\zeta(1-s)$$

**证明**：在 $\xi(s)$ 的表达式中，$\frac{1}{s(s-1)}$ 和 $y^{s/2} + y^{(1-s)/2}$ 在 $s \mapsto 1-s$ 下不变。

---

## 平凡零点

在函数方程中，$\Gamma(s/2)$ 在 $s = -2, -4, -6, \ldots$ 处有极点。由于 $\xi(s)$ 整体解析，必须有

$$\zeta(-2m) = 0, \quad m = 1, 2, 3, \ldots$$

这些是 ζ 函数的**平凡零点**。

---

## 黎曼假设

ζ 函数的所有**非平凡零点**都位于临界线 $\Re(s) = 1/2$ 上。

这是数学中最重要的未解决问题之一。已验证前 $10^{13}$ 个非平凡零点全部位于临界线上，超过 1000 个数学问题的解决依赖于黎曼假设。

---

## ζ 函数在 $s=1$ 附近的行为

$$\zeta(s) = \frac{1}{s-1} + \gamma + O(s-1)$$

其中 $\gamma$ 为 Euler-Mascheroni 常数。

### 对数导数

对欧拉乘积取对数求导：

$$-\frac{\zeta'(s)}{\zeta(s)} = \sum_p \sum_{k=1}^{\infty} \frac{\log p}{p^{ks}}$$

当 $s \to 1^+$ 时，主要发散项来自 $\sum_p (\log p)/p^s$。

---

## L 函数与朗兰兹纲领

### L 函数的三大性质

ζ 函数满足的三条核心性质可推广到更一般的 **L 函数**：

1. **欧拉乘积**：在 $\Re(s)$ 充分大时可表示为局部因子的乘积。
2. **解析延拓**：可延拓到整个复平面（可能有限个极点）。
3. **函数方程**：具有对称性 $\Lambda(s) = \epsilon\,\overline{\Lambda(1-\bar{s})}$。

### 朗兰兹纲领

建立**自守表示**（偏分析）与 **Galois 表示**（偏代数）之间的对应，通过 L 函数的相等来刻画：$L(s, \pi) = L(s, \rho)$。

这可以看作 [[Galois 理论思想发展|Galois 理论]] 的极大推广——Galois 对应建立的是域扩张与群之间的字典，而 Langlands 纲领建立的是表示论与分析对象之间的字典。

- 最简单的自守 L 函数就是 Riemann ζ 函数。
- Dirichlet 特征给出 Dirichlet L 函数，是下一个自然情形。
- 二次互反律是这个框架的特例。

---

## Dirichlet 密度

### 定义

设 $S$ 是素数集合的子集。$S$ 的**Dirichlet 密度**定义为

$$\delta(S) = \lim_{s \to 1^+} \frac{\sum_{p \in S} p^{-s}}{\sum_p p^{-s}}$$

### 基本性质

- 全体素数的 Dirichlet 密度为 1。
- 不交并的密度可加。

### Dirichlet 密度定理

> **定理**：设 $(a,m)=1$，则 $\{p : p \equiv a \pmod{m}\}$ 的 Dirichlet 密度为 $1/\varphi(m)$。

即素数在模 $m$ 的各个可逆同余类中是等分布的。证明需要 Dirichlet L 函数，详见 [[Dirichlet特征与L函数]]。
