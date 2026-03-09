---
tags:
  - 组合数学
  - 数论
---
## Catalan 数的定义

## 利用 Catalan 数估计 $\pi(x)$ 

首先有对 $\pi(x)$ 的估计：$\pi(x) \sim \frac{x}{\log x}$  。事实上有更强的结论：二者在 $x\to \infty$ 时二者为等价无穷小。在这里我们只给出最简单的阶的估计。

考虑函数 $\theta(x) = \sum_{1\leq p\leq x} \log p$ ，其中 $p$ 为素数。于是有引理：

> **引理**：$\theta(x) < 4 \log_{2} x$ 当 $x$ 充分大

有：

$$
2^{2n} = (1+1)^{2n} = \sum_{i=0}^{2n} C_{2n}^i > C_{2n}^n
$$
考察所有 $n<p<2n$ ，有 $p | C_{2n}^n$ ，于是又有