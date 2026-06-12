---
tags:
  - 数论
  - Fourier分析
---
## 有限 Abel 群与特征群

### 基本设定

设 $A$ 是有限 Abel 群。两个最重要的例子：

- $\mathbb{Z}/m\mathbb{Z}$：模 $m$ 剩余类的**加法群**
- $(\mathbb{Z}/m\mathbb{Z})^\times$：模 $m$ 的缩剩余系在**乘法**下的群

### 特征

$A$ 上的一个**特征**是群同态

$$\chi: A \to S^1 = \{z \in \mathbb{C} : |z| = 1\}, \quad \chi(ab) = \chi(a)\chi(b)$$

所有特征构成群 $\hat{A} = \mathrm{Hom}(A, S^1)$，运算为逐点乘法，称为 $A$ 的**特征群**（dual group）。

### 关键事实

- **$|\hat{A}| = |A|$**，即特征群与原群同阶。$\hat{\hat{A}} \cong A$（Pontryagin 对偶）。群的结构由 [[有限 Abel 群与有限域|有限 Abel 群结构定理]] 完全决定。
- 若 $A = \mathbb{Z}/m\mathbb{Z}$（加法群），$\hat{A}$ 同构于所有 $m$ 次单位根构成的群。$\chi$ 由 $\chi(1)$ 完全确定。
- 若 $A = (\mathbb{Z}/m\mathbb{Z})^\times$，由 CRT 和 $m$ 的素因子分解，$A$ 分解为素数幂缩系的直积，$\hat{A}$ 相应分解。

### 特征的正交性

$$\sum_{a \in A} \chi(a)\overline{\psi(a)} = \begin{cases} |A| & \chi = \psi \\ 0 & \chi \neq \psi \end{cases}$$

特别地，$\sum_{a \in A} \chi(a) = |A| \cdot \delta_{\chi, \chi_0}$。

---

## Fourier 变换

### 定义

设 $f: A \to \mathbb{C}$。$f$ 的 **Fourier 变换** $\hat{f}: \hat{A} \to \mathbb{C}$ 定义为

$$\hat{f}(\chi) = \frac{1}{\sqrt{|A|}} \sum_{a \in A} f(a)\overline{\chi(a)}$$

**Fourier 逆变换**：对 $g: \hat{A} \to \mathbb{C}$，

$$\check{g}(a) = \frac{1}{\sqrt{|A|}} \sum_{\chi \in \hat{A}} g(\chi)\chi(a)$$

### 例 1：常数函数

$f \equiv 1$ 的 Fourier 变换为 $\hat{f} = \sqrt{|A|} \cdot \delta_{\chi_0}$。

### 例 2：Dirac 函数

$\delta_{a_0}(a) = \sqrt{|A|} \cdot \mathbf{1}_{a = a_0}$ 的 Fourier 变换为 $\widehat{\delta_{a_0}}(\chi) = \overline{\chi(a_0)}$。

---

## Fourier 反演公式

> **定理**：$f(a) = \frac{1}{\sqrt{|A|}} \sum_{\chi \in \hat{A}} \hat{f}(\chi)\chi(a) = (\hat{f})^\vee(a)$

即先做 Fourier 变换，再做 Fourier 逆变换，回到原函数。

**证明**：按定义展开，利用特征的正交性。

---

## Plancherel 公式（Parseval 等式）

> **定理**：$\|\hat{f}\|_2 = \|f\|_2$，即
> $$\sum_{\chi \in \hat{A}} |\hat{f}(\chi)|^2 = \sum_{a \in A} |f(a)|^2$$

更一般地，$\langle \hat{f_1}, \hat{f_2} \rangle = \langle f_1, f_2 \rangle$。

---

## 卷积

### 定义

设 $f_1, f_2: A \to \mathbb{C}$。卷积定义为

$$(f_1 * f_2)(a) = \frac{1}{\sqrt{|A|}} \sum_{b \in A} f_1(b) f_2(ab^{-1})$$

与实数上的卷积类比：求和代替积分，$ab^{-1}$ 代替 $x-y$。

### 四条基本性质

1. **结合律**：$(f_1 * f_2) * f_3 = f_1 * (f_2 * f_3)$
2. **交换律**：$f_1 * f_2 = f_2 * f_1$（成立的根本原因是 $A$ 是交换群）
3. **分配律**：$(f_1 + f_2) * f_3 = f_1 * f_3 + f_2 * f_3$
4. **核心性质——Fourier 变换将卷积变为逐点乘积**：

$$\widehat{f_1 * f_2}(\chi) = \hat{f_1}(\chi) \cdot \hat{f_2}(\chi)$$

**性质 4 的证明**：展开 Fourier 变换定义，利用 $\overline{\chi(ab^{-1})} = \overline{\chi(a)} \cdot \chi(b)$，交换求和次序。

> 从 Fourier 变换的角度看，卷积并不"奇怪"——它正是逐点乘法在对偶空间中的像。

### 卷积环

$A$ 上的函数空间在逐点加法和卷积下构成一个环（不同于逐点加法和逐点乘法构成的标准函数环）。

---

## 平移算子

### 定义

给定 $a_0 \in A$，平移算子 $T_{a_0}$ 定义为

$$(T_{a_0} f)(b) = f(a_0^{-1}b)$$

### 平移与卷积

$$T_{a_0} f = f * \delta_{a_0}$$

### 平移的 Fourier 变换

$$\widehat{T_{a_0} f}(\chi) = \hat{f}(\chi) \cdot \overline{\chi(a_0)}$$

---

## Gauss 和的重新理解

### Dirichlet 特征延拓到加法群

取 $A = \mathbb{Z}/m\mathbb{Z}$（加法群）。设 $\kappa$ 是模 $m$ 的 Dirichlet 特征（$(\mathbb{Z}/m\mathbb{Z})^\times$ 上的乘法特征），延拓为加法群上的函数

$$\tilde{\kappa}(a) = \begin{cases} \kappa(a) & (a,m)=1 \\ 0 & \text{otherwise} \end{cases}$$

### Gauss 和作为 Fourier 变换

$$\hat{\tilde{\kappa}}(\chi_k) = \frac{1}{\sqrt{m}} \cdot g_k(\kappa)$$

其中 $g_k(\kappa) = \sum_{(a,m)=1} \kappa(a)\overline{\zeta_m^{ak}}$ 是 Gauss 和。

> **Gauss 和就是延拓后的 Dirichlet 特征的 Fourier 变换**（乘以 $\sqrt{m}$）。

### Gauss 和的模长

利用 Plancherel 公式：

$$\|\tilde{\kappa}\|_2^2 = \sum_{(a,m)=1} |\kappa(a)|^2 = \varphi(m)$$

由 $\|\hat{\tilde{\kappa}}\|_2^2 = \|\tilde{\kappa}\|_2^2$，得 $\sum_k |g_k(\kappa)|^2 = m \cdot \varphi(m)$。

对本原特征 $\kappa$，当 $(k,m)=1$ 时 $|g_k(\kappa)| = \sqrt{m}$，当 $(k,m) \neq 1$ 时 $g_k(\kappa) = 0$。因此 $|\tau(\kappa)| = |g_1(\kappa)| = \sqrt{m}$ 成为 Plancherel 公式的直接推论。

关于 Gauss 和的更多讨论，参见 [[Jacobi和与Gauss和与二次剩余]]。

---

## 应用：$x^N = a$ 在 $\mathbb{F}_p^\times$ 中的解

### 问题

设 $p$ 为素数，$N$ 为正整数，$a \in \mathbb{F}_p^\times$。定义

$$f(a) = \#\{x \in \mathbb{F}_p^\times : x^N = a\}$$

### Fourier 分析方法

对乘法群 $A = \mathbb{F}_p^\times$ 上的函数 $f$，计算 Fourier 变换：

$$\hat{f}(\psi) = \frac{1}{\sqrt{p-1}} \sum_{x \in \mathbb{F}_p^\times} \overline{\psi(x^N)} = \frac{1}{\sqrt{p-1}} \sum_{x} \overline{\psi^N(x)}$$

由正交关系：

$$\hat{f}(\psi) = \frac{1}{\sqrt{p-1}} \cdot \begin{cases} p-1 & \psi^N = \psi_0 \\ 0 & \text{otherwise} \end{cases}$$

### 结果

由 Fourier 反演：

$$\boxed{f(a) = \sum_{\substack{\psi \in \widehat{\mathbb{F}_p^\times} \\ \psi^N = \psi_0}} \psi(a)}$$

即解的个数等于所有满足 $\psi^N = \psi_0$ 的乘法特征在 $a$ 处取值之和。

### 特例：$N = 2$（Legendre 符号）

满足 $\psi^2 = \psi_0$ 的特征有两个：$\psi_0$ 和 Legendre 符号 $(\cdot/p)$。因此

$$f(a) = 1 + \left(\frac{a}{p}\right)$$

$a$ 是二次剩余时有 2 个解，非剩余时无解。经典结果作为特例自然出现。

---

## Fourier 变换的谱分析

固定同构 $A \xrightarrow{\sim} \hat{A}$ 后，Fourier 变换成为 $A$ 上函数空间到自身的线性映射。

- 两次 Fourier 变换给出反射：$\widehat{\hat{f}}(a) = f(a^{-1})$
- 四次 Fourier 变换回到自身：$\mathcal{F}^4 = \mathrm{Id}$

因此 Fourier 变换的特征值只能是 $\{\pm 1, \pm i\}$。
