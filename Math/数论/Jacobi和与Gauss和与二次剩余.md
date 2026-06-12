---
tags:
  - 数论
  - 二次剩余
---
## 为什么关心 Gauss 和与 Jacobi 和

有限域 $\mathbb{F}_q$ 上有两套自然的结构——**加法**和**乘法**。它们各自有特征（加法特征 $\psi$ 和乘法特征 $\chi$），各自行使着良好的分析功能。但数论中许多最深刻的结果——二次互反律、Dirichlet $L$ 函数的函数方程、有限域上方程的解数——都依赖于理解这两套结构之间的**交互**。

Gauss 和与 Jacobi 和正是这种交互的两个核心工具：

- **Gauss 和** $g(\chi) = \sum_x \chi(x)\psi(x)$：把一个乘法特征 $\chi$ 放到加法特征 $\psi$ 的框架下做 Fourier 分析。它衡量乘法特征在加法振荡下的抵消强度。
- **Jacobi 和** $J(\chi, \eta) = \sum_x \chi(x)\eta(1-x)$：只显式使用乘法特征，但通过 $x + (1-x) = 1$ 这个加法约束把乘法结构与加法结构联系起来。

两者通过核心公式

$$J(\chi, \eta) = \frac{g(\chi)\,g(\eta)}{g(\chi\eta)}$$

紧密相连。本文从定义出发，逐步建立这个公式——先通过直接计算给出一个简短证明，再通过 Fourier 分析给出一个更深刻的理解。

---

## 背景：有限域上的特征

设 $\mathbb{F}_q$ 是有限域，$q = p^r$。

### 加法特征

加法特征是群同态 $\psi: (\mathbb{F}_q, +) \to \mathbb{C}^\times$，满足 $\psi(x+y) = \psi(x)\psi(y)$。

在 $\mathbb{F}_p$ 上，最常用的非平凡加法特征是 $\psi(x) = e^{2\pi i x/p}$。在一般的 $\mathbb{F}_q$ 上，借助迹映射 $\operatorname{Tr}: \mathbb{F}_q \to \mathbb{F}_p$ 定义 $\psi(x) = e^{2\pi i \operatorname{Tr}(x)/p}$。

### 乘法特征

乘法特征是群同态 $\chi: (\mathbb{F}_q^\times, \cdot) \to \mathbb{C}^\times$，满足 $\chi(xy) = \chi(x)\chi(y)$。

通常将 $\chi$ 扩展到整个 $\mathbb{F}_q$：对非平凡的 $\chi$，令 $\chi(0) = 0$。平凡乘法特征记作 $\varepsilon$，满足 $\varepsilon(x) = 1$（$x \neq 0$），$\varepsilon(0) = 0$。

> 这些概念的更一般框架参见 [[有限Abel群上的Fourier分析]]。

---

## Gauss 和

### 定义

给定乘法特征 $\chi$ 和非平凡加法特征 $\psi$，**Gauss 和**定义为

$$g(\chi, \psi) = \sum_{x \in \mathbb{F}_q} \chi(x)\,\psi(x)$$

当 $\psi$ 固定时简写为 $g(\chi)$。在模素数 $p$ 的情形下，常记作

$$\tau(\chi) = \sum_{a=0}^{p-1} \chi(a)\,e^{2\pi i a/p}$$

### 直觉

Gauss 和把乘法特征 $\chi$ 放到加法 Fourier 框架中：如果把 $\psi$ 看成加法群上的"基频"，那么 $g(\chi)$ 就是 $\chi$ 在这个基频上的 Fourier 系数。因为 $\chi$ 本身在加法群上没有任何规律（它是为乘法结构而生的），所以这个 Fourier 系数的值是一个不平凡的复数——它的大小和相位都携带了 $\chi$ 与加法结构交互的信息。

### 基本性质

**平凡特征**：若 $\chi = \varepsilon$ 且 $\psi$ 非平凡，则 $g(\varepsilon, \psi) = \sum_{x \neq 0} \psi(x) = -1$（因为 $\sum_{x \in \mathbb{F}_q} \psi(x) = 0$）。

**绝对值公式**：若 $\chi$ 非平凡，则

$$|g(\chi, \psi)| = \sqrt{q}$$

更精确地，$g(\chi, \psi)\,g(\chi^{-1}, \psi) = \chi(-1)\,q$。

Gauss 和的大小恰好是 $\sqrt{q}$——这正是 [[有限Abel群上的Fourier分析|Plancherel 公式]] 的推论（$\chi$ 延拓到加法群上的 $L^2$ 范数为 $\varphi(q) = q - 1$，而 Fourier 变换保持 $L^2$ 范数）。但 Gauss 和的**相位**——即 $g(\chi)$ 的确切值——远非显然，确定它往往是深刻定理的关键。

---

## Jacobi 和

### 定义

给定两个乘法特征 $\chi, \eta$，**Jacobi 和**定义为

$$J(\chi, \eta) = \sum_{x \in \mathbb{F}_q} \chi(x)\,\eta(1-x)$$

换一种写法，$J(\chi, \eta) = \sum_{a+b=1} \chi(a)\,\eta(b)$。所以 Jacobi 和是在加法约束 $a + b = 1$ 下考察两个乘法特征的相关性。

注意 Jacobi 和的定义中没有显式出现加法特征 $\psi$——它只用了乘法特征。但 $a + b = 1$ 这个约束本身就是加法结构的体现。

### 为什么取 $a + b = 1$

$1$ 本身没有神秘的特殊性。更一般地，令 $J_c(\chi, \eta) = \sum_{a+b=c} \chi(a)\eta(b)$。若 $c \neq 0$，做变量替换 $a = cx$, $b = c(1-x)$：

$$J_c(\chi, \eta) = (\chi\eta)(c)\,J(\chi, \eta)$$

因此所有非零的加法条件 $a + b = c$ 本质上都等价，只差一个简单的乘法因子 $(\chi\eta)(c)$。将右边规范化为 $1$ 是最自然的选择。

$c = 0$ 的情形确实不同：$\sum_{a+b=0} \chi(a)\eta(b) = \eta(-1)\sum_a (\chi\eta)(a)$，它退化为乘法特征在整个域上的总和（非平凡时为 $0$），不再给出 Jacobi 和。

### 基本性质

若 $\chi, \eta, \chi\eta$ 都非平凡，则 $|J(\chi, \eta)| = \sqrt{q}$。

特殊情形：

- $J(\chi, \chi^{-1}) = -\chi(-1)$（$\chi$ 非平凡）
- $J(\varepsilon, \eta) = J(\eta, \varepsilon) = -1$（$\eta$ 非平凡）

---

## 核心公式：$J(\chi, \eta) = \frac{g(\chi)\,g(\eta)}{g(\chi\eta)}$

这是 Gauss 和与 Jacobi 和之间最重要的关系。我们给出两个证明：一个简短的直接计算，一个基于 Fourier 分析。

### 证明一：直接计算

从乘积出发：

$$g(\chi)\,g(\eta) = \sum_{u, v \in \mathbb{F}_q} \chi(u)\,\eta(v)\,\psi(u+v)$$

令 $t = u + v$。当 $t \neq 0$ 时，写 $u = tx$，$v = t(1-x)$。利用乘法特征的乘性：

$$\chi(u)\,\eta(v) = \chi(tx)\,\eta(t(1-x)) = (\chi\eta)(t)\,\chi(x)\,\eta(1-x)$$

因此

$$g(\chi)\,g(\eta) = \left(\sum_x \chi(x)\,\eta(1-x)\right)\left(\sum_t (\chi\eta)(t)\,\psi(t)\right) = J(\chi, \eta)\,g(\chi\eta)$$

（这里需要验证 $t = 0$ 的贡献为 $0$，当 $\chi, \eta$ 都非平凡时确实如此。）于是

$$\boxed{J(\chi, \eta) = \frac{g(\chi)\,g(\eta)}{g(\chi\eta)}}$$

### 证明二：Fourier 分析的视角

直接计算给出了公式，但没解释**为什么**这个公式应该成立。Fourier 分析提供了更深刻的理解。

回顾 [[有限Abel群上的Fourier分析|有限 Abel 群上的 Fourier 变换]]：在加法群 $(\mathbb{F}_q, +)$ 上，固定非平凡加法特征 $\psi$，对函数 $f: \mathbb{F}_q \to \mathbb{C}$ 定义 Fourier 变换

$$\hat{f}(t) = \sum_{x \in \mathbb{F}_q} f(x)\,\psi(tx)$$

**Gauss 和是 Fourier 变换在 $t = 1$ 处的值。** 把乘法特征 $\chi$ 延拓到整个 $\mathbb{F}_q$（令 $\chi(0) = 0$），则

$$\hat{\chi}(t) = \sum_x \chi(x)\,\psi(tx) = \chi^{-1}(t)\,g(\chi) \quad (t \neq 0)$$

做变量替换 $y = tx$ 即得。这意味着乘法特征的整个 Fourier 变换由一个 Gauss 和完全控制。

**Jacobi 和是加法卷积在 $1$ 处的值。** 定义加法卷积 $(f * h)(c) = \sum_a f(a)\,h(c-a)$，则

$$J(\chi, \eta) = (\chi * \eta)(1)$$

Fourier 分析的核心规则是**卷积变为乘积**：$\widehat{f * h} = \hat{f} \cdot \hat{h}$。因此

$$\widehat{\chi * \eta}(t) = \hat{\chi}(t)\,\hat{\eta}(t) = (\chi\eta)^{-1}(t)\,g(\chi)\,g(\eta)$$

另一方面，$\widehat{\chi\eta}(t) = (\chi\eta)^{-1}(t)\,g(\chi\eta)$。比较两者：

$$\widehat{\chi * \eta}(t) = \frac{g(\chi)\,g(\eta)}{g(\chi\eta)}\,\widehat{\chi\eta}(t)$$

由于 Fourier 变换是可逆的，原函数也满足 $\chi * \eta = \frac{g(\chi)\,g(\eta)}{g(\chi\eta)}\,(\chi\eta)$。在 $1$ 处取值，利用 $(\chi\eta)(1) = 1$，即得 $J(\chi, \eta) = \frac{g(\chi)\,g(\eta)}{g(\chi\eta)}$。

> 这个 Fourier 分析的视角把公式的本质揭示得非常清楚：Jacobi 和是卷积在 $1$ 处的值，而 Fourier 变换把卷积变成乘积，所以 Jacobi 和由 Gauss 和的乘除关系决定。

---

## 二次 Gauss 和

现在看最重要的特例：$q = p$ 为奇素数，$\chi$ 取为二次特征（Legendre 符号 $\chi(a) = \left(\frac{a}{p}\right)$）。

二次 Gauss 和为

$$g(\chi) = \sum_{a=0}^{p-1} \left(\frac{a}{p}\right) e^{2\pi i a/p}$$

### 核心公式

$$g(\chi)^2 = \chi(-1)\,p = \left(\frac{-1}{p}\right) p$$

这可以由一般公式 $g(\chi)\,g(\chi^{-1}) = \chi(-1)\,q$ 和 $\chi = \chi^{-1}$（二次特征是其自身的逆）直接得到。

因为 $\chi(-1) = 1$（$p \equiv 1 \pmod{4}$）或 $-1$（$p \equiv 3 \pmod{4}$），所以在标准取法下

$$g(\chi) = \begin{cases} \sqrt{p}, & p \equiv 1 \pmod{4} \\ i\sqrt{p}, & p \equiv 3 \pmod{4} \end{cases}$$

### 深层含义

$g(\chi)^2 = (-1)^{(p-1)/2}\,p$ 这个看似简单的公式蕴含着深刻的信息。它说明 $g(\chi) = \pm\sqrt{p^*}$（其中 $p^* = (-1)^{(p-1)/2}\,p$），因此 $g(\chi) \in \mathbb{Q}(\sqrt{p^*})$。

但 $g(\chi)$ 又是分圆域 $\mathbb{Q}(\zeta_p)$ 中的元素（因为它是 $p$ 次单位根的线性组合）。所以 $g(\chi)$ 同时属于 $\mathbb{Q}(\sqrt{p^*})$ 和 $\mathbb{Q}(\zeta_p)$——它给出了二次域 $\mathbb{Q}(\sqrt{p^*})$ 到分圆域 $\mathbb{Q}(\zeta_p)$ 的嵌入。

这正是 [[二次剩余与二次互反律|Gauss 和证明二次互反律]] 的基础，也是 [[Galois 理论思想发展|类域论]] 的最初萌芽。

### 例子：$p = 5$

模 $5$ 的二次剩余是 $1, 4$，非二次剩余是 $2, 3$。令 $\zeta = e^{2\pi i/5}$，则

$$g(\chi) = \zeta^1 - \zeta^2 - \zeta^3 + \zeta^4 = \sqrt{5}$$

这与 $5 \equiv 1 \pmod{4}$ 一致（$g(\chi) = \sqrt{p}$）。

---

## 二次 Jacobi 和

仍令 $\chi$ 为二次特征。因为 $\chi^2 = \varepsilon$，由 Jacobi 和的基本性质 $J(\chi, \chi^{-1}) = -\chi(-1)$，得

$$J(\chi, \chi) = -\chi(-1) = \begin{cases} -1, & p \equiv 1 \pmod{4} \\ 1, & p \equiv 3 \pmod{4} \end{cases}$$

### 例子：$p = 5$

$5 \equiv 1 \pmod{4}$，故 $\chi(-1) = 1$，$J(\chi, \chi) = -1$。

直接验证：$J(\chi, \chi) = \sum_{x \in \mathbb{F}_5} \chi(x)\,\chi(1-x)$，逐项计算

| $x$ | $\chi(x)$ | $\chi(1-x)$ | $\chi(x)\chi(1-x)$ |
|:---:|:---:|:---:|:---:|
| 0 | 0 | 1 | 0 |
| 1 | 1 | 0 | 0 |
| 2 | $-1$ | 1 | $-1$ |
| 3 | $-1$ | $-1$ | 1 |
| 4 | 1 | $-1$ | $-1$ |

总和 $0 + 0 - 1 + 1 - 1 = -1$，与公式一致。

---

## 总结

| | Gauss 和 $g(\chi)$ | Jacobi 和 $J(\chi, \eta)$ |
|---|---|---|
| **定义** | $\sum_x \chi(x)\psi(x)$ | $\sum_x \chi(x)\eta(1-x)$ |
| **出现的特征** | 一个乘法 + 一个加法 | 两个乘法（加法约束隐含在 $a+b=1$ 中） |
| **Fourier 理解** | 乘法特征的加法 Fourier 系数 | 乘法特征加法卷积在 $1$ 处的值 |
| **大小** | $|g(\chi)| = \sqrt{q}$ | $|J(\chi, \eta)| = \sqrt{q}$ |
| **核心公式** | $g(\chi)\,g(\chi^{-1}) = \chi(-1)\,q$ | $J(\chi, \eta) = g(\chi)\,g(\eta)\,/\,g(\chi\eta)$ |

**应用**：
- 证明[[二次剩余与二次互反律|二次互反律]]和高次互反律
- 研究 Dirichlet $L$ 函数的函数方程（$g(\chi)$ 出现在根数中，详见 [[Dirichlet特征与L函数]]）
- 研究有限域上方程的解数（如 $x^m + y^m = 1$ 的解数由 Jacobi 和给出）
- 研究椭圆曲线和代数曲线在有限域上的点数
