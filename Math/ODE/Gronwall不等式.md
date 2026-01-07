---
tags:
  - ODE
---
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

