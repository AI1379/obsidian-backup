---
tags:
  - ODE
---
## 单调不减时的 Gronwall 不等式

对于积分不等式：

$$
u(t) \leq \beta(t) + \int_{a}^{t} \alpha(s)u(s)\mathrm{d}s
$$
如果满足：

1. $\alpha(t),\ \beta(t),\ u(t)$ 均非负且连续
2. $\beta(t)$ 单调不减

则有不等式：

$$
u(t) \leq \beta(t) \exp\left( \int_{a}^{t} \alpha(s) \mathrm{d}s \right)
$$

### 证明

构造辅助函数 $R(t) = \int_{a}^{t} \alpha(s)u(s) \mathrm{d}s$ ，则有 $R’(t) = \alpha(t)u(t)$ 。不等式两边同乘 $\alpha(t)$ ，则有：

$$
\alpha(t)u(t) \leq \alpha(t)\beta(t) + \alpha(t) R(t)
$$
考虑积分因子 $\mu(t) = \exp\left( -\int_{a}^t\alpha(s)\mathrm{d}s \right)$ ，则有 $\mu’(t) = -\alpha(t)\mu(t)$ 。从而：

$$
\mu(t)\alpha(t)u(t)\leq \mu(t)\alpha(t)\beta(t)+\mu(t)\alpha(t)R(t)
$$

又有 $R’(t) = \alpha(t)u(t)$ 从而：

$$
\mu(t)R’(t) + \mu’(t) R(t) = (\mu(t)R(t))’ \leq \mu(t) \alpha(t)\beta(t)
$$

两边积分

$$
\mu(t)R(t) \leq \int_{a}^t \mu(s)\alpha(s)\beta(s)\mathrm{d}s \leq \beta(t)\int_{a}^t\mu(s)\alpha(s)\mathrm{d}s = \beta(t) (1-\mu(t))
$$

从而：

$$
R(t) \leq \beta(t)\exp\left( \int_{a}^t \alpha(s)\mathrm{d}s \right) - \beta(t)
$$

代入原不等式：

$$
u(t) \leq \beta(t) + R(t) \leq \beta(t)\exp\left( \int_{a}^t \alpha(s)\mathrm{d}s \right)
$$

