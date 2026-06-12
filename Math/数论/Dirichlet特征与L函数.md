---
tags:
  - 数论
  - 解析数论
---
## Dirichlet 特征

### 定义

> Dirichlet 特征是有限 Abel 群特征的特例，更一般的理论参见 [[有限Abel群上的Fourier分析]]。特征的线性无关性由 [[域|Artin 引理]] 保证。

固定正整数 $m$。**模 $m$ 的 Dirichlet 特征**是从 $(\mathbb{Z}/m\mathbb{Z})^\times$ 到 $\mathbb{C}^\times$ 的群同态

$$\chi: (\mathbb{Z}/m\mathbb{Z})^\times \to \mathbb{C}^\times, \quad \chi(ab) = \chi(a)\chi(b)$$

将其延拓为 $\mathbb{Z} \to \mathbb{C}$ 上的函数：

$$\chi(n) = \begin{cases} \chi(n \bmod m) & (n,m)=1 \\ 0 & (n,m) > 1 \end{cases}$$

延拓后 $\chi$ 对**所有**整数满足完全乘性：$\chi(ab) = \chi(a)\chi(b)$。

### 基本性质

- $\chi$ 的取值落在单位圆上（有限群的元素阶有限，故 $\chi(a)$ 是单位根）。
- **主特征** $\chi_0$：$\chi_0(a) = 1$ 对所有 $(a,m)=1$。

### 诱导特征与本原特征

若 $d \mid m$，存在自然投射 $(\mathbb{Z}/m\mathbb{Z})^\times \to (\mathbb{Z}/d\mathbb{Z})^\times$。将模 $d$ 的特征复合此投射得到的特征称为**诱导特征**。

若 $\chi$ 不能由任何真因子 $d \mid m$（$d < m$）诱导，则称 $\chi$ 为**本原的**（primitive）。

---

## Dirichlet L 函数

### 定义

给定模 $m$ 的 Dirichlet 特征 $\chi$，定义

$$L(s, \chi) = \sum_{n=1}^{\infty} \frac{\chi(n)}{n^s}$$

### 三大核心性质

**性质 1：欧拉乘积**（$\Re(s) > 1$）

$$L(s, \chi) = \prod_{p} \frac{1}{1 - \chi(p)p^{-s}}$$

证明与 ζ 函数的欧拉乘积完全类似，利用 $\chi$ 的完全乘性。

**性质 2：解析延拓**

- 若 $\chi = \chi_0$（主特征），$L(s, \chi_0)$ 与 ζ 函数只差有限个欧拉因子：$L(s, \chi_0) = \zeta(s) \prod_{p \mid m} (1 - p^{-s})$，在 $s=1$ 处有**简单极点**。
- 若 $\chi \neq \chi_0$，$L(s, \chi)$ 在 $s=1$ 处**解析**（整个复平面上解析）。

**性质 3：函数方程**

完备 L 函数 $\Lambda(s, \chi)$ 满足

$$\Lambda(s, \chi) = \epsilon(\chi) \cdot \Lambda(1-s, \bar{\chi})$$

其中 $\epsilon(\chi)$ 是绝对值为 1 的常数。

---

## 完备 L 函数

### 特征的奇偶性

$\chi(-1) \in \{1, -1\}$。定义

$$\varepsilon_\chi = \begin{cases} 0 & \chi(-1) = 1 \text{（偶特征）} \\ 1 & \chi(-1) = -1 \text{（奇特征）} \end{cases}$$

### 完备化

$$\Lambda(s, \chi) = \left(\frac{m}{\pi}\right)^{(s+\varepsilon_\chi)/2} \Gamma\!\left(\frac{s + \varepsilon_\chi}{2}\right) L(s, \chi)$$

- 偶特征：$\Gamma(s/2)$（与 ζ 函数情形相同）。
- 奇特征：$\Gamma((s+1)/2)$。

---

## θ 函数的变换公式

### 带 Dirichlet 特征的 θ 函数

$$\theta_\chi(\tau) = \sum_{n \in \mathbb{Z}} \chi(n)\, e^{\pi i n^2 \tau / m}$$

$\chi$ 的出现破坏了普通 θ 函数的对称性。为恢复对称性，按 mod $m$ 的余数对 $n$ 分类，引入更基本的单元

$$\Theta_\varepsilon(a, b; \tau) = \sum_{k \in \mathbb{Z}} (a+k)^\varepsilon\, e^{\pi i (a+k)^2 \tau + 2\pi i b(a+k)}$$

### 基本变换公式

$$\Theta_\varepsilon(a, b; -1/\tau) = \text{const} \cdot \tau^{1/2+\varepsilon} \cdot e^{2\pi i ab} \cdot \Theta_\varepsilon(-b, a; \tau)$$

通过 Poisson 求和公式和高斯积分证明。

### θ 函数的变换公式

$$\theta_\chi(i/y) = C(\chi) \cdot y^{1/2+\varepsilon_\chi} \cdot \theta_{\bar{\chi}}(iy)$$

常数 $C(\chi)$ 涉及高斯和 $\tau(\chi)$。

---

## 高斯和

### 定义

对模 $m$ 的本原 Dirichlet 特征 $\chi$，

$$\tau(\chi) = \sum_{a=0}^{m-1} \chi(a)\, e^{2\pi i a/m}$$

### 关键性质

$$|\tau(\chi)| = \sqrt{m}$$

$\tau(\chi)$ 起到联系 $\chi$ 与其复共轭 $\bar{\chi}$ 的桥梁作用。在变换公式和函数方程中，$\tau(\chi)$ 出现在根数 $\epsilon(\chi)$ 中。

关于 Gauss 和的更多讨论，参见 [[Jacobi和与Gauss和与二次剩余]] 和 [[有限Abel群上的Fourier分析]]。

---

## L 函数的解析延拓与函数方程

### 解析延拓

**证明思路**：

1. 将 $\Lambda(s, \chi)$ 写成 $\theta_\chi$ 的积分表示。
2. 拆分为 $(0,1)$ 和 $(1,\infty)$。
3. $(1,\infty)$ 部分对所有 $s$ 绝对收敛（指数衰减）。
4. $(0,1)$ 部分做变量替换 $y \mapsto 1/y$，利用 $\theta_\chi$ 的变换公式。

### 函数方程

$$\Lambda(s, \chi) = \epsilon(\chi) \cdot \Lambda(1-s, \bar{\chi})$$

在积分表达式中将 $s$ 换为 $1-s$，利用变换公式的对称性即得。证明方法与 [[Riemann zeta函数与解析数论基础|ζ 函数]] 完全平行。

### 非本原特征的情形

若 $\chi$ 由模 $d$ 的本原特征 $\chi_1$ 诱导，则

$$L(s, \chi) = L(s, \chi_1) \cdot \prod_{\substack{p \mid m \\ p \nmid d}} \frac{1}{1 - \chi_1(p)p^{-s}}$$

差别仅是有限乘积的解析函数。

---

## $L(1, \chi) \neq 0$

这是证明 Dirichlet 密度定理最核心的输入。

### 复特征情形

**反证法**：假设复特征 $\chi$（$\chi \neq \bar{\chi}$）满足 $L(1, \chi) = 0$。则 $L(1, \bar{\chi}) = \overline{L(1, \chi)} = 0$。

考虑乘积 $\prod_\chi L(s, \chi)$，其 Dirichlet 级数系数 $a_n$ 非负。$L(s, \chi_0)$ 在 $s=1$ 有简单极点（发散到 $+\infty$），但 $\chi$ 和 $\bar{\chi}$ 的零点消去了极点。因此乘积在 $s=1$ 处有界。

然而 $a_n \geq 0$ 且某些 $a_n > 0$，乘积不可能在 $s \to 1^+$ 时趋向零，矛盾。

### 实特征情形

设 $\chi$ 为实特征（$\chi = \bar{\chi}$，即二次特征），考虑辅助函数

$$f(s) = \zeta(s) \cdot L(s, \chi) = \sum_{n=1}^{\infty} \frac{a_n}{n^s}$$

其中 $a_n = (1 * \chi)(n) = \sum_{d \mid n} \chi(d)$。

**系数非负**：对 $n = p^k$，
- $\chi(p) = 1 \Rightarrow a_{p^k} = k+1 \geq 1$
- $\chi(p) = -1 \Rightarrow a_{p^k} \in \{0, 1\}$
- $\chi(p) = 0 \Rightarrow a_{p^k} = 1$

由乘性知所有 $a_n \geq 0$。

**导出矛盾**：若 $L(1, \chi) = 0$，则 $f(s)$ 是整函数。$f(2) = \sum a_n/n^2 > 0$。在 $s = 2$ 处 Taylor 展开并利用系数非负性，可以导出矛盾。

---

## Dirichlet 密度定理

> **定理**：设 $(a,m)=1$，则 $\{p : p \equiv a \pmod{m}\}$ 的 Dirichlet 密度为 $1/\varphi(m)$。

### 证明路线

**第一步**：利用特征的正交性提取同余类

$$\frac{1}{\varphi(m)} \sum_\chi \bar{\chi}(a) \chi(p) = \begin{cases} 1 & p \equiv a \pmod{m} \\ 0 & \text{否则} \end{cases}$$

因此

$$\sum_{\substack{p \text{ 素数} \\ p \equiv a \pmod{m}}} \frac{1}{p^s} = \frac{1}{\varphi(m)} \sum_\chi \bar{\chi}(a) \sum_p \frac{\chi(p)}{p^s}$$

**第二步**：利用对数导数

$$-\frac{L'(s,\chi)}{L(s,\chi)} = \sum_p \sum_{k=1}^{\infty} \frac{\chi(p)^k \log p}{p^{ks}}$$

**第三步**：取极限 $s \to 1^+$：

- $\chi = \chi_0$：$-L'(s,\chi_0)/L(s,\chi_0)$ 发散（极点贡献），主要项为 $1/(s-1)$。
- $\chi \neq \chi_0$：$L(1,\chi) \neq 0$ 保证 $-L'(s,\chi)/L(s,\chi)$ 在 $s=1$ 附近有界。

只有主特征的贡献发散，除以 $-\log(s-1)$ 后其他特征的贡献消失，最终得到密度 $1/\varphi(m)$。

---

## 自然密度与 Dirichlet 密度

**自然密度**：$d(S) = \lim_{x \to \infty} \frac{|\{p \leq x : p \in S\}|}{|\{p \leq x : p \text{ 素数}\}|}$。

自然密度存在 $\Rightarrow$ Dirichlet 密度存在且相等。反之不成立。

**例子**（Benford 定律）：$\{p : p \text{ 的十进制首位为 } 1\}$ 的 Dirichlet 密度和自然密度都为 $\log_{10} 2 \approx 0.301$。
