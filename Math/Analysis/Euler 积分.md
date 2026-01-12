---
tags:
  - 分析
---

## Beta 函数

Beta 函数的定义如下：

$$
B(p,q) = \int_{0}^{1} x^{p-1}(1-x)^{q-1}\mathrm{d}x
$$
显然定义域是 $[0,+\infty) \times [0, +\infty)$ 。

### 性质

Beta 函数具有对称性：$B(p,q)=B(q,p)$ 。这是显然的。

Beta 函数有递推公式：

$$
B(p,q) = \frac{q-1}{p+q-1} B(p,q-1)
$$
这可以通过分部积分整理得到。

### 常见变形

 做变量代换 $x = \cos^2t$ 有：

$$
B(p,q) = 2\int_{0}^{\pi/2} \cos^{2p-1}t \sin^{2q-1}t \mathrm{d}t
$$
从而可以得到：

$$
B\left( \frac{1}{2}, \frac{1}{2} \right)=\pi
$$

---

做变量代换 $x = \frac{1}{1+t}$ 则有：

$$
B(p,q) = \int_{0}^{+\infty}\frac{t^{q-1}}{(1+t)^{p+q}} \mathrm{d}t
$$

将这个积分分为 $[0,1]$ 和 $[1,+\infty)$ 两段，并作变量代换 $t = \frac{1}{u}$ 从而可以得到：

$$
B(p,q) = \int_{0}^{1} \frac{t^{p-1}+t^{q-1}}{(1+t)^{p+q}} \mathrm{d}t
$$

## Gamma 函数

$$
\Gamma(s) = \int_{0}^{+\infty} x^{s-1} e^{ -x } \mathrm{d}x
$$

### 性质

可以证明它连续可导。它有重要的递推公式：

$$
\Gamma(s+1) = s\Gamma(s)
$$
这说明它事实上是阶乘的延拓。它的定义域是 $[0, +\infty)$ 。然而，利用这个递推关系，可以将它推广到除了 $0$ 和负整数以外的整个实轴上。

### 常见变形

1. 做变量代换 $x=t^2$ ，从而有：
	$$
	\Gamma(s) = 2 \int_{0}^{+\infty} t^{2s-1} e^{ -t^2 } \mathrm{d}t
	$$
	从而可以推出：
	$$
	\Gamma\left( \frac{1}{2} \right) = 2 \int_{0}^{+\infty}e^{ -t^2 }\mathrm{d}t = \sqrt{ \pi }
	$$
2. 做变量代换 $x = \alpha t$ 可得：
	$$
	\Gamma(s) = \alpha^s \int_{0}^{+\infty}t^{s-1} e^{ -\alpha t }\mathrm{d}t
	$$

### 重要公式

Beta 函数和 Gamma 函数关系密切，因为可以证明如下关系：

$$
B(p,q) = \frac{\Gamma(p)\Gamma(q)}{\Gamma(p+q)}
$$

此外，关于 Gamma 函数，还有一些重要公式：

>[!important] Legendre 公式
> $$
> \Gamma(s)\Gamma\left( s+\frac{1}{2} \right) = \frac{\sqrt{ \pi }}{2^{2s-1}}\Gamma(2s),\quad s>0
> $$

>[!important] 余元公式
> $$
> \Gamma(s)\Gamma(1-s) = \frac{\pi}{\sin \pi s},\quad \forall s \not \in \mathbb{Z}
> $$

> [!important] Stirling 公式
> $$
> \Gamma(s+1) = \sqrt{ 2\pi s }\left( \frac{s}{e} \right) ^s e^{ \theta/12s },\quad s>0
> $$
> 是关于 $\Gamma(s)$ 的一个渐进估计，其中 $0<\theta<1$

这其中余元公式尤为重要。
