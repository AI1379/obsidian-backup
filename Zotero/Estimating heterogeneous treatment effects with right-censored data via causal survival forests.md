---
tags:
  - paper
citekey: "cuiEstimatingHeterogeneousTreatment2023"
---

# Estimating heterogeneous treatment effects with right-censored data via causal survival forests

## 📒 引用

```plaintext
Cui, Yifan, Michael R. Kosorok, Erik Sverdrup, Stefan Wager, and Ruoqing Zhu. “Estimating Heterogeneous Treatment Effects with Right-Censored Data via Causal Survival Forests.” _Journal of the Royal Statistical Society Series B: Statistical Methodology_ 85, no. 2 (2023): 179–211. [https://doi.org/10.1093/jrsssb/qkac001](https://doi.org/10.1093/jrsssb/qkac001).
```

Cite Key:

```plaintext
cuiEstimatingHeterogeneousTreatment2023
```

URL：[https://doi.org/10.1093/jrsssb/qkac001](https://doi.org/10.1093/jrsssb/qkac001)

## 📌 摘要

Forest-based methods have recently gained in popularity for non-parametric treatment effect estimation. Building on this line of work, we introduce causal survival forests, which can be used to estimate heterogeneous treatment effects in survival and observational setting where outcomes may be right-censored. Our approach relies on orthogonal estimating equations to robustly adjust for both censoring and selection effects under unconfoundedness. In our experiments, we find our approach to perform well relative to a number of baselines.

## 💡 我的思考

- [ ] 本文与我的课题关联点：_________
- [ ] 可复现的实验：_________

# Assistant

可以。你现在缺的不是数学能力，而是这篇论文背后的几套“统计语言”。它们本身并不神秘，只是统计学喜欢把几个很直观的问题压缩成大量术语。

先把整篇论文压成一句话：

> 对每个患者，我们只观察到他接受了一种治疗，而且生存时间还可能因为研究结束或失访而没有完整观测；论文利用因果推断纠正治疗选择偏差，利用生存分析纠正删失，再利用随机森林学习治疗效果怎样随患者特征变化。

它的完整数据流可以写成：

$$
\boxed{
\text{患者特征 }X
\longrightarrow
\text{治疗 }W
\longrightarrow
\text{生存时间 }T
}
$$

但真实观察到的是：

$$
\boxed{
(X,W,U,\Delta),\qquad
U=\min(T,C),\quad
\Delta=\mathbf 1{T\le C}
}
$$

其中 $C$ 是删失时间。论文最终想学习的是：

$$
\boxed{
\tau(x)
=
E[y(T(1))-y(T(0))\mid X=x]
}
$$

也就是“对于特征为 $x$ 的患者，治疗 1 相对于治疗 0 能改善多少生存结局”。论文正式使用的也是这个数据结构和目标。

下面按依赖关系讲。

---

# 一、先理解普通监督学习在做什么

假设有数据：

$$
(X_i,Y_i),\qquad i=1,\dots,n.
$$

其中 $X_i\in\mathbb R^p$ 是特征，$Y_i$ 是结果。回归问题通常希望学习：

$$
f(x)=E[Y\mid X=x].
$$

例如：

- $X$：年龄、体重、血液指标；
    
- $Y$：一年后的某个健康指标；
    
- $f(x)$：这类患者平均会得到什么结果。
    

训练模型时，通常最小化平方误差：

$$
\hat f
=
\arg\min_f
\frac1n\sum_{i=1}^n(Y_i-f(X_i))^2.
$$

这里有一个关键前提：每个样本的 $Y_i$ 是一个完整、明确的标签。

第二篇论文同时破坏了这个简单设定的两点：

1. 研究目标不是预测 $Y$，而是估计**治疗造成的增量效果**；
    
2. 生存时间 $T$ 还不一定能完整观测。
    

因此它需要因果推断和生存分析两套额外工具。

---

# 二、什么是生存时间和生存分析？

## 1. 生存时间不一定指“死亡时间”

在生存分析里，$T$ 一般表示某个事件发生前的等待时间，例如：

- 患者死亡；
    
- 癌症复发；
    
- 病情恶化；
    
- 设备故障；
    
- 用户流失；
    
- 借款违约。
    

“生存”只是历史名称。数学对象就是一个非负随机变量：

$$
T\ge 0.
$$

最基本的函数是**生存函数**：

$$
S_T(t)=P(T>t),
$$

表示事件在时间 $t$ 之后才发生的概率。

例如：

$$
S_T(1000)=0.8
$$

表示有 80% 的对象在 1000 天后仍未发生目标事件。

与它对应的分布函数是：

$$
F_T(t)=P(T\le t)=1-S_T(t).
$$

---

## 2. 什么是 hazard function？

风险率或 hazard function 描述：

> 已经存活到 $t$ 的前提下，紧接着发生事件的瞬时速率。

形式上：

$$
\lambda_T(t)
=
\lim_{\Delta t\to0}
\frac{
P(t\le T<t+\Delta t\mid T\ge t)
}{
\Delta t
}.
$$

它不是“在 $t$ 时死亡的概率”，而是一个条件瞬时速率。

生存函数和累计风险之间满足：

$$
S_T(t)
=
\exp\left(
-\int_0^t\lambda_T(s),ds
\right).
$$

这篇论文里还会出现 **censoring survival function** 和 **censoring hazard**：

$$
S_w^C(t\mid x)
=
P(C\ge t\mid X=x,W=w),
$$

$$
\lambda_w^C(t\mid x)
=
-\frac{d}{dt}\log S_w^C(t\mid x).
$$

注意上标 $C$：它们描述的是“尚未被删失”的概率，不是患者尚未死亡的概率。

---

# 三、什么是右删失？

这是读论文最先要真正吃透的概念。

假设患者真实生存时间为：

$$
T=1200\text{ 天}.
$$

但研究在第 1000 天结束，那么我们不知道患者究竟在第 1200 天还是第 2000 天发生事件，只知道：

$$
T>1000.
$$

这种数据叫做**右删失**：我们知道真实值位于观测值的右边。

定义删失时间 $C$，实际观察到：

$$
U=\min(T,C),
$$

以及指标：

$$
\Delta=\mathbf 1{T\le C}.
$$

其中：

- $\Delta=1$：事件在删失前发生，真实 $T$ 被观察到；
    
- $\Delta=0$：先失访或研究结束，只知道 $T>C$。
    

举三个例子：

|患者|真实事件时间 $T$|删失时间 $C$|观察值 $U$|$\Delta$|知道什么|
|---|--:|--:|--:|--:|---|
|A|400|1000|400|1|事件发生于第 400 天|
|B|1200|1000|1000|0|只知道 $T>1000$|
|C|800|250|250|0|只知道 $T>250$|

对 B 和 C 来说，把 $U$ 当成真实生存时间是错的。

例如患者 C 在第 250 天失访，不代表他在第 250 天死亡。相反，我们至少知道：

$$
T>250.
$$

因此被删失样本不是“完全没有标签”，而是具有一个不等式形式的部分标签。

---

## 为什么不能只删除被删失者？

假设病情更严重的患者更容易失访，或者某类治疗副作用更强，导致患者更早退出。

如果只分析未删失患者，那么剩余样本不再代表原始患者群体。

即使删失在给定 $X,W$ 后是随机的，删除被删失样本也会：

- 浪费“至少存活到 $U$”的信息；
    
- 改变不同类型患者的相对比例；
    
- 降低统计效率；
    
- 若删失概率与特征相关，还会产生偏差。
    

论文正是因此先讨论 IPCW，再提出更完整的 doubly robust correction。作者明确指出，IPCW 会丢弃被删失者仍然包含的信息，而且对删失模型的估计误差敏感。

---

# 四、为什么论文引入有限时间范围 $h$？

假设研究最多随访三年，那么你通常无法可靠估计：

$$
E[T],
$$

因为某些患者三年后究竟还能活多久完全没有数据支持。

所以论文选择一个时间范围 $h$，只研究在 $h$ 之前的结局。

## 1. 限制平均生存时间 RMST

定义：

$$
y(T)=T\wedge h=\min(T,h).
$$

于是：

$$
E[T\wedge h]
$$

叫作 restricted mean survival time。

它可以理解为：

> 在时间范围 $[0,h]$ 内，一个人平均存活了多长时间。

例如 $h=1000$ 天：

- 若患者在第 400 天死亡，$T\wedge h=400$；
    
- 若患者活过第 1000 天，$T\wedge h=1000$。
    

因此治疗效果可以定义为：

$$
\tau_h(x)
=
E[T(1)\wedge h-T(0)\wedge h\mid X=x].
$$

假如：

$$
\tau_h(x)=30\text{ 天},
$$

意思是：对于这类患者，在前 $h$ 天内，治疗 1 相比治疗 0 平均增加约 30 个生存日。

---

## 2. 固定时间点生存概率

也可以选择：

$$
y(T)=\mathbf 1{T\ge h}.
$$

那么：

$$
E[y(T)\mid X=x]
=
P(T\ge h\mid X=x).
$$

治疗效果就是：

$$
P(T(1)\ge h\mid X=x)
-
P(T(0)\ge h\mid X=x).
$$

例如结果是 $0.08$，表示治疗将活到 $h$ 时刻的概率提高约 8 个百分点。

论文理论上允许一般的 $y(T)$，只要求它在 $h$ 后不再变化；软件实现主要支持 RMST 和固定时间点生存概率。

---

## 3. 为什么有限时间范围还能缓解删失？

若某患者在第 1200 天失访，而 $h=1000$，虽然我们不知道他的真实 $T$，但已经知道：

$$
T\wedge1000=1000.
$$

所以对这个 estimand 来说，他其实是完整观测的。

论文定义有效非删失指标：

$$
\Delta_i^h
=
\mathbf 1{T_i\wedge h\le C_i}.
$$

只要患者被追踪到 $h$ 之后，就可以视为对 $y(T)$ 完整观测。论文在 ACTG175 中选择 $h=1000$ 天，正是为了避免尾部所有长生存时间都被删失、无法识别的问题。

---

# 五、从预测转向因果：潜在结果是什么？

这是整篇论文的第二块基石。

假设每个患者可能接受两种治疗：

$$
W\in{0,1}.
$$

为每个患者定义两个**潜在结果**：

$$
T(0)=\text{若接受治疗 0 的生存时间},
$$

$$
T(1)=\text{若接受治疗 1 的生存时间}.
$$

个体治疗效果理想上是：

$$
T(1)-T(0).
$$

但一个患者实际上只能接受一种治疗，因此我们只能观察：

$$
T=T(W).
$$

若 $W=1$，观察 $T(1)$，但 $T(0)$ 永远不可见；反之亦然。

这叫作因果推断的 fundamental problem：

> 对同一个对象，无法同时观察事实和反事实。

---

## ATE 与 CATE

总体平均治疗效果是：

$$
\operatorname{ATE}
=
E[y(T(1))-y(T(0))].
$$

而论文关心的是条件平均治疗效果：

$$
\operatorname{CATE}(x)
=
\tau(x)
=
E[y(T(1))-y(T(0))\mid X=x].
$$

两者区别是：

- ATE：治疗平均有没有用；
    
- CATE：治疗对哪类人更有用。
    

例如总体上：

$$
\operatorname{ATE}=10\text{ 天},
$$

但可能：

$$
\tau(x)=
\begin{cases}
40\text{ 天},&\text{年轻且 CD4 较低},\
-10\text{ 天},&\text{年长且 CD4 较高}.
\end{cases}
$$

那么只看平均效果会掩盖重要异质性。

论文的目标正是估计：

$$
x\longmapsto \tau(x).
$$

---

# 六、怎样从观察数据识别因果效果？

潜在结果只是定义了目标，还没有说明它能否从数据中算出来。

论文使用几条标准识别假设。

## 1. Consistency / potential outcomes

实际观察结果等于实际接受治疗对应的潜在结果：

$$
T=T(W).
$$

也就是说，接受治疗 1 时观察到的是 $T(1)$。

这还隐含治疗版本定义清楚、对象之间没有未建模干扰等要求。

---

## 2. Ignorability / unconfoundedness

论文假设：

$$
{T(0),T(1)}\perp W\mid X.
$$

读作：

> 给定协变量 $X$ 后，治疗选择与潜在结果独立。

直觉上，$X$ 已经包含了所有同时影响治疗选择和结局的混杂因素。

例如医生更倾向于给重症患者使用治疗 1，同时重症患者本来生存率就较低。若没有记录疾病严重程度，那么直接比较：

$$
E[T\mid W=1]-E[T\mid W=0]
$$

会把“治疗效果”和“患者原本更严重”混在一起。

若严重程度已包含在 $X$ 中，并且条件于 $X$ 后治疗近似随机，则可以比较同类患者。

这是假设，不是随机森林能够从数据中证明的。

---

## 3. Overlap

定义 propensity score：

$$
e(x)=P(W=1\mid X=x).
$$

要求存在 $\eta_e>0$，使：

$$
\eta_e\le e(x)\le1-\eta_e.
$$

意思是每一类患者都有非零概率接受两种治疗。

如果某种患者必然接受治疗 1：

$$
P(W=1\mid X=x)=1,
$$

那么数据中没有相似患者接受治疗 0，无法知道治疗 0 的反事实结果。

这和插值、外推的区别很像：

> 因果估计只能在两组数据有共同支持的区域中可靠进行。

---

## 4. Ignorable censoring

论文还假设：

$$
T\perp C\mid X,W.
$$

即给定患者特征和治疗后，删失时间与真实生存时间独立。

这不是说删失完全随机。删失可以依赖年龄、病情和治疗，只要这些都已包含在 $X,W$ 中。

例如：

- 年龄大的人更容易失访：可以，只要年龄在 $X$ 中；
    
- 治疗 1 的人更容易退出：可以，只要条件于 $W$；
    
- 患者因未记录的病情恶化而退出：可能违反假设。
    

---

## 5. Censoring positivity

还要求在关注的范围 $h$ 内，每类患者都有足够概率不被删失：

$$
P(C\ge h\mid X=x,W=w)>0.
$$

否则某类患者永远在很早的时候消失，就不可能知道他们在 $h$ 时刻的结局。

论文正式列出的正是 potential outcomes、ignorability、overlap、conditional independent censoring 和 censoring positivity。

---

# 七、决策树是什么？

现在进入机器学习部分。

假设要预测：

$$
Y=f(X),
\qquad X=(X_1,\dots,X_p).
$$

一棵回归树会递归地把特征空间切成矩形区域。

例如：

1. 先判断年龄是否小于 50；
    
2. 在年龄小于 50 的患者中，再判断 CD4 是否小于 300；
    
3. 在每个最终区域，即叶节点中，用样本平均值作预测。
    

形式上，它构造一个分片常数函数：

$$
\hat f(x)
=
\sum_{\ell=1}^L
\hat\mu_\ell
\mathbf 1{x\in R_\ell},
$$

其中 $R_\ell$ 是叶节点区域。

回归树通常选择能最大程度降低平方误差的分裂：

$$
\text{父节点误差}
>
\text{两个子节点误差}.
$$

---

## 决策树为什么容易过拟合？

单棵树对数据扰动非常敏感。

一个样本稍微变化，最佳第一次分裂可能从：

$$
\text{age}<50
$$

变成：

$$
\text{CD4}<280,
$$

后续整棵树都随之改变。

从算法角度看，它是高方差估计器。

---

# 八、随机森林是什么？

随机森林训练很多棵彼此有差异的树，然后平均：

$$
\hat f_{\mathrm{RF}}(x)
=
\frac1B\sum_{b=1}^B
\hat f_b(x).
$$

树之间的随机性通常来自：

- 每棵树使用不同的子样本；
    
- 每次分裂只考虑随机选取的部分特征；
    
- 不同随机种子产生不同树结构。
    

直觉上类似 ensemble：

$$
\text{多个有噪声但不完全相关的估计}
\quad\xrightarrow{\text{平均}}\quad
\text{降低方差}.
$$

它的优势是：

- 自动处理非线性；
    
- 自动发现变量交互；
    
- 不要求预先指定函数形式；
    
- 对特征缩放通常不敏感；
    
- 可适应复杂异质性。
    

---

# 九、把随机森林看成“自适应最近邻”

这是理解论文最重要的一步。

普通 $k$-nearest neighbours 对预测点 $x$，找到欧氏距离最近的训练样本并平均。

随机森林定义了一种由数据学习出来的相似度：

> 若样本 $X_i$ 经常和 $x$ 落入同一个叶节点，就认为它们相似。

论文定义森林权重：

$$
\alpha_i(x)
=
\frac1B
\sum_{b=1}^B
\frac{
\mathbf 1{X_i\in N_b(x)}
}{
|N_b(x)|
},
$$

其中 $N_b(x)$ 是第 $b$ 棵树中包含 $x$ 的叶节点。

于是：

$$
\alpha_i(x)\ge0,\qquad
\sum_i\alpha_i(x)=1.
$$

预测可以写成：

$$
\hat f(x)
=
\sum_i\alpha_i(x)Y_i.
$$

这说明随机森林本质上像一个**数据自适应核方法**：

- 普通核回归用固定距离定义邻近；
    
- 随机森林用树的分裂结构学习哪些方向重要。
    

如果年龄很重要，叶节点会沿年龄方向切得很细；若体重几乎没用，沿体重方向可能很宽。论文正是以 adaptive nearest-neighbour / local kernel weight 的方式理解 generalized random forest。

---

# 十、普通随机森林、随机生存森林和因果森林有什么区别？

这三个名字很像，但目标不同。

## 1. 普通随机森林

学习：

$$
E[Y\mid X=x].
$$

目标是预测结果。

---

## 2. 随机生存森林

学习：

$$
S_T(t\mid X=x)
=
P(T>t\mid X=x),
$$

或者 hazard、RMST 等生存结果。

它会使用能够处理删失的分裂准则，例如 log-rank splitting。

目标仍然是：

> 预测某类患者的生存情况。

它不直接估计治疗的因果效果。

---

## 3. 因果森林

学习：

$$
\tau(x)
=
E[Y(1)-Y(0)\mid X=x].
$$

分裂的目标不是找到“结局不同”的人群，而是找到“治疗效果不同”的人群。

例如：

- 年轻患者生存时间普遍更长，是 prognostic effect；
    
- 治疗只对年轻患者有效，是 treatment effect heterogeneity。
    

普通生存森林容易主要学到第一种；因果森林专门针对第二种。

论文指出，已有随机生存森林等方法，但它们没有直接针对观察性研究中删失结局下的 heterogeneous treatment effect。

---

# 十一、propensity score 是什么？

定义：

$$
e(x)=P(W=1\mid X=x).
$$

它表示具有特征 $x$ 的患者接受治疗 1 的概率。

例如：

$$
e(x)=0.8
$$

表示这类患者有约 80% 接受治疗 1。

propensity score 的作用是描述治疗分配机制。

若治疗是完全随机分配的：

$$
e(x)=\frac12.
$$

若是观察性数据，$e(x)$ 通常随病情、年龄、医生决策等变化。

因果推断中需要把这种选择机制从结果中剥离，否则会把 selection bias 当成 treatment effect。

---

# 十二、outcome regression $m(x)$ 是什么？

论文定义：

$$
m(x)=E[y(T)\mid X=x].
$$

注意这里暂时没有条件于治疗，它是对患者总体结果的平均预测。

它和 propensity score 都不是最终目标，最终目标是 $\tau(x)$。因此统计学把它们叫作 **nuisance functions**。

“nuisance”不是说它们没用，而是：

> 它们是为了估计目标参数而需要估计的中间函数。

论文中还需要：

$$
S_w^C(s\mid x),
\qquad
\lambda_w^C(s\mid x),
\qquad
Q_w(s\mid x).
$$

其中：

$$
Q_w(s\mid x)
=
E[y(T)\mid X=x,W=w,T\wedge h>s].
$$

它表示一个特征为 $x$、接受治疗 $w$、并且已经存活到 $s$ 的患者，最终 $y(T)$ 的条件期望。

---

# 十三、Robinson residualization 在做什么？

先忽略删失，假设 $Y=y(T)$ 完整观测。

定义：

$$
\mu_w(x)=E[Y\mid X=x,W=w].
$$

治疗效果是：

$$
\tau(x)=\mu_1(x)-\mu_0(x).
$$

同时：

$$
m(x)
=
E[Y\mid X=x]
=
e(x)\mu_1(x)+(1-e(x))\mu_0(x).
$$

可以验证：

$$
E[Y-m(X)\mid X,W]
=
(W-e(X))\tau(X).
$$

因此有近似“残差回归”形式：

$$
Y-m(X)
=
\tau(X)(W-e(X))
+
\varepsilon,
$$

其中：

$$
E[\varepsilon\mid X,W]=0.
$$

这特别漂亮，因为它把因果问题转化成：

- 结果残差：$Y-m(X)$；
    
- 治疗残差：$W-e(X)$；
    
- 两者之间的局部回归系数：$\tau(X)$。
    

若 $\tau$ 是常数，可以通过估计方程：

$$
\sum_i
(W_i-\hat e(X_i))
\left[
Y_i-\hat m(X_i)
-\tau(W_i-\hat e(X_i))
\right]
=0
$$

求解。

论文使用的 complete-data score 正是：

$$
\psi_\tau^{(c)}
=
[W-\hat e(X)]
\left[
y(T)-\hat m(X)-\tau(W-\hat e(X))
\right].
$$

---

## 为什么残差化很重要？

假设重症患者更容易接受治疗 1。

那么 $W$ 自身包含“患者有多严重”的信息，直接回归 $Y$ 对 $W$ 会混杂。

而：

$$
W-e(X)
$$

表示实际治疗选择相对于“根据患者特征本来有多可能接受治疗”的偏离。

类似地：

$$
Y-m(X)
$$

表示实际结果相对于“根据患者特征本来会有怎样结果”的偏离。

比较这两个残差，能够消除大量由 $X$ 导致的系统变化。

这和 Frisch–Waugh–Lovell theorem 的思想非常接近：先对协变量残差化，再估计关心的系数。

---

# 十四、因果森林怎样使用这个估计方程？

如果 $\tau(x)$ 随 $x$ 变化，就不能只估计一个全局常数。

对目标点 $x$，因果森林使用森林权重：

$$
\alpha_i(x),
$$

求解局部估计方程：

$$
\sum_i
\alpha_i(x)
\psi_{\tau(x),i}^{(c)}
=0.
$$

其显式解类似：

$$
\hat\tau(x)
=
\frac{
\sum_i\alpha_i(x)
(W_i-\hat e_i)(Y_i-\hat m_i)
}{
\sum_i\alpha_i(x)(W_i-\hat e_i)^2
}.
$$

这可以看成一个**局部加权残差回归**。

随机森林负责回答：

> 哪些训练患者与目标患者 $x$ 在治疗效果意义下相似？

估计方程负责回答：

> 在这些相似患者中，治疗残差与结果残差的关系是多少？

论文就是把 forest weights 和 Robinson score 组合起来估计 CATE。

---

# 十五、树怎样专门寻找治疗效果异质性？

普通回归树的分裂依据是 $Y$ 是否不同。

因果森林希望找到 $\tau(x)$ 是否不同，但每个个体的真实治疗效果不可观测，因此不能直接把：

$$
T_i(1)-T_i(0)
$$

当标签。

解决方式是构造 **pseudo-outcome**。

粗略地说，利用当前节点中的 score 和 estimating equation 的导数，可以为每个样本构造一个量：

$$
\rho_i
\approx
\text{样本 }i\text{ 对局部治疗效果估计的影响}.
$$

然后像回归树一样寻找能让两个子节点的 pseudo-outcome 均值差异最大的分裂。

所以 pseudo-outcome 不是每个患者的真实个体治疗效果，而是一个用于近似局部 treatment-effect variation 的统计构造。论文在式 (14) 后把分裂转化为对这些 pseudo-outcome 的回归树分裂。

---

# 十六、删失出现以后，为什么原始因果森林失效？

原始 score 需要：

$$
y(T_i).
$$

但删失样本只知道：

$$
T_i>U_i.
$$

所以不能直接计算：

$$
y(T_i)-m(X_i).
$$

论文首先给出一个简单修复：IPCW。

---

# 十七、IPCW 是什么？

IPCW 是 inverse probability of censoring weighting，逆删失概率加权。

设：

$$
S_w^C(t\mid x)
=
P(C\ge t\mid W=w,X=x).
$$

对于一个在 $T\wedge h$ 前未被删失的样本，它被完整观察的概率是：

$$
S_W^C(T\wedge h\mid X).
$$

因此给它权重：

$$
\frac{1}{
S_W^C(T\wedge h\mid X)
}.
$$

完整样本越难被观察到，权重越大。

---

## IPCW 为什么能纠正偏差？

设 $g(T,X,W)$ 是一个我们想平均的量。考虑：

$$
\frac{\Delta^h}{
S_W^C(T\wedge h\mid X)
}
g(T,X,W).
$$

在给定 $T,X,W$ 后：

$$
E[\Delta^h\mid T,X,W]
=
S_W^C(T\wedge h\mid X).
$$

因此：

$$
E\left[
\frac{\Delta^h}{
S_W^C(T\wedge h\mid X)
}
g(T,X,W)
\middle|
T,X,W
\right]
=
g(T,X,W).
$$

分母恰好抵消“这个样本能够被完整观察到”的概率。

这和 survey sampling 或 missing-data 中的 Horvitz–Thompson estimator 是同一个思想。

---

## IPCW 的缺点

第一，被删失样本本身没有直接进入结果项。

假设患者在 800 天时失访，我们至少知道他活过了 800 天，但简单 IPCW 没有充分利用这条信息。

第二，如果：

$$
S_W^C(T\wedge h\mid X)
$$

很小，逆权重会极大，方差爆炸。

第三，它依赖删失模型估计准确。

论文的 IPCW causal forest 只用有效未删失样本，并把逆删失概率作为 sample weight；作者随后正是因为效率和稳健性问题转向 augmented correction。

---

# 十八、什么是 doubly robust correction？

这是全文最难、也最重要的统计部分。

先讲基本思想，不急着硬啃式 (11)。

假设一个样本的结果缺失。我们有两种恢复总体平均值的方式：

### 方法 A：建模缺失机制

估计该样本被观察的概率，然后逆概率加权。

这就是 IPCW。

### 方法 B：建模结果

根据已知特征预测缺失结果：

$$
\hat Q(X).
$$

Augmented estimator 把二者结合：

$$
\text{结果模型预测}
+
\text{逆概率加权的预测残差}.
$$

在最经典的缺失数据问题中，它形如：

$$
\hat Q(X)
+
\frac{\Delta}{\hat\pi(X)}
\left[
Y-\hat Q(X)
\right].
$$

直觉是：

- 所有人先获得一个模型预测；
    
- 对真正观察到结果的人，再用逆概率加权残差校正预测错误。
    

若结果模型很好，残差很小；若观察概率模型很好，加权残差能修正结果模型偏差。

---

## 在生存数据中为什么更复杂？

因为被删失者不是简单的“结果完全缺失”。

若在时刻 $U$ 被删失，我们知道：

$$
T>U.
$$

因此结果模型应使用：

$$
Q_w(s\mid x)
=
E[y(T)\mid X=x,W=w,T\wedge h>s].
$$

也就是：

> 已知患者已经存活到 $s$，其最终结局的条件期望是多少？

论文的 correction 包含：

- 对完整结果的 IPCW 项；
    
- 对删失患者使用 $Q_w(U\mid X)$ 的补偿；
    
- 沿时间对 censoring hazard 的积分校正；
    
- treatment residualization。
    

完整公式很长，但结构不是魔法：

$$
\boxed{
\text{因果残差化}
+
\text{逆删失概率加权}
+
\text{基于生存到当前时刻的结果预测}
+
\text{删失过程校正}
}
$$

---

# 十九、“双重稳健”到底稳健在哪里？

直观上，它利用两类信息：

1. 删失机制模型：  
    $$
    S_w^C,\lambda_w^C;
    $$
    
2. 生存结局模型：  
    $$
    Q_w.
    $$
    

如果只使用 IPCW，删失模型有误就会直接影响结果。

加入 outcome augmentation 后，两部分误差会发生抵消，使估计对 nuisance estimation error 不那么敏感。

不过要避免一种过度理解：

> doubly robust 不等于随便哪个模型都可以胡乱估计。

论文的正式渐近结果仍对多个 nuisance functions 给出一致性和收敛速度条件。它的关键优势是这些误差以乘积或二阶形式进入最终误差，而不是一阶线性进入。

---

# 二十、什么是 Neyman orthogonality？

可以把最终估计量写成估计方程：

$$
E[\psi(Z;\tau,\eta)]=0,
$$

其中：

- $\tau$ 是目标参数；
    
- $\eta$ 是 nuisance functions 的集合。
    

例如：

$$
\eta=(e,m,S^C,\lambda^C,Q).
$$

若 score 对 nuisance 在真实值附近满足：

$$
\left.
\frac{d}{dr}
E[\psi(Z;\tau,\eta_0+r(\eta-\eta_0))]
\right|_{r=0}
=0,
$$

就叫 Neyman orthogonal。

这表示：

> nuisance function 出现一个小误差时，目标方程的一阶变化为零。

假设 nuisance 估计误差为：

$$
|\hat\eta-\eta_0|=O_p(n^{-1/4}).
$$

普通非正交 plug-in 可能直接产生 $n^{-1/4}$ 级误差；正交化后，误差往往以乘积出现：

$$
O_p(n^{-1/4})\cdot O_p(n^{-1/4})
=
O_p(n^{-1/2}).
$$

于是最终低维目标仍可能达到：

$$
n^{-1/2}
$$

级别。

论文对全局常数治疗效果的结果正是：在 cross-fitting 下，若 nuisance components 达到约四次根速率，就可以得到根 $n$ 收敛和渐近正态性。

---

# 二十一、什么是 cross-fitting？

假设你用同一批数据训练一个非常灵活的随机森林 $\hat m$，然后又在同一批数据上计算：

$$
Y_i-\hat m(X_i).
$$

模型可能对训练样本过拟合，使残差人为偏小，并且残差与模型估计误差存在复杂依赖。

Cross-fitting 的做法是：

1. 把数据分为两部分 $I_1,I_2$；
    
2. 用 $I_1$ 训练 nuisance models；
    
3. 在 $I_2$ 上计算 score；
    
4. 用 $I_2$ 训练 nuisance models；
    
5. 在 $I_1$ 上计算 score；
    
6. 合并两部分。
    

于是每个样本的 score 都使用“没见过这个样本”的 nuisance model。

这类似 out-of-fold prediction：

$$
\hat\eta_{-i}(X_i).
$$

它的作用是削弱过拟合造成的自相关，让复杂机器学习模型可以安全地作为 nuisance estimator 使用。论文附录明确将 cross-fitting 描述为用交叉折估计避免 overfitting bias。

---

# 二十二、什么是 honesty？

Honest forest 是另一种样本分离思想。

普通树可能用同一组样本：

- 决定在哪里分裂；
    
- 估计叶节点中的参数。
    

这会产生 selection bias：树专门选择了在当前样本上看起来差异最大的分裂。

Honesty 通常把一棵树使用的样本再分成：

- splitting sample：只决定树结构；
    
- estimation sample：只估计叶节点中的参数。
    

可以类比成：

- 先用训练集选择模型结构；
    
- 再用独立数据估计模型参数。
    

这降低自适应分裂造成的偏差，也是随机森林能够进行统计推断的重要技术条件。论文渐近理论明确假设树具有 honesty，并对随机分裂和 subsampling 作出条件。

---

# 二十三、什么是 generalized random forest？

普通随机森林通常输出叶节点平均值。

Generalized Random Forest，GRF，把森林的作用抽象成产生局部权重：

$$
\alpha_i(x).
$$

然后用这些权重求解任意局部估计方程：

$$
\sum_i
\alpha_i(x)
\psi_{\theta(x)}(Z_i)
=0.
$$

不同的 score $\psi$ 对应不同任务：

- 均值回归；
    
- 分位数回归；
    
- 因果效果；
    
- 工具变量；
    
- 本文的删失生存因果效果。
    

因此 GRF 可以理解为：

$$
\boxed{
\text{森林学习局部邻域}
+
\text{估计方程定义局部参数}
}
$$

Causal Survival Forest 就是：

- 用森林寻找具有相似 treatment effect 的患者；
    
- 用删失校正后的 doubly robust score 定义局部 CATE。
    

---

# 二十四、整篇算法可以怎样理解？

把论文的算法压成伪代码：

```text
输入：患者特征 X、治疗 W、观察时间 U、删失指标 Δ

1. 用交叉拟合估计：
   - 治疗概率 e(x)
   - 总体结果回归 m(x)
   - 删失生存函数 S^C_w(s|x)
   - 删失风险率 λ^C_w(s|x)
   - 条件未来结局 Q_w(s|x)

2. 利用这些 nuisance models 构造删失校正后的正交 score。

3. 根据 score 构造 pseudo-outcome。

4. 训练 honest generalized random forest：
   - 分裂特征空间；
   - 寻找 treatment effect 不同的区域。

5. 对目标患者 x，森林生成权重 α_i(x)。

6. 解局部估计方程：
      Σ_i α_i(x) ψ_i(τ(x)) = 0

7. 输出：
   - CATE 估计 τ̂(x)
   - 方差估计
   - 点态置信区间
```

最核心的一行就是：

$$
\boxed{
\sum_i\alpha_i(x)\psi_{\hat\tau(x),i}=0
}
$$

森林决定“谁和 $x$ 相似”，score 决定“怎样从这些人的删失观察中恢复因果效果”。

---

# 二十五、什么是一致性和渐近正态性？

## 1. 一致性

一致性指：

$$
\hat\tau_n(x)
\xrightarrow{P}
\tau(x).
$$

也就是样本量越来越大时，估计值以高概率接近真实值。

它回答的是：

> 算法最终有没有学对？

---

## 2. 收敛速度

如果：

$$
\hat\tau_n(x)-\tau(x)
=
O_p(r_n),
$$

表示典型误差量级约为 $r_n$。

参数模型常见：

$$
r_n=n^{-1/2}.
$$

非参数函数估计通常更慢，因为要在高维空间中学习一个函数。

---

## 3. 渐近正态性

论文证明在一定条件下：

$$
\frac{
\hat\tau(x)-\tau(x)
}{
\sigma_n(x)
}
\Rightarrow N(0,1).
$$

这意味着大样本下可以构造：

$$
\hat\tau(x)
\pm
1.96,\hat\sigma_n(x)
$$

作为近似 95% 点态置信区间。

但“点态”很重要：它是在固定 $x$ 处的置信区间，不是同时覆盖整个函数 $x\mapsto\tau(x)$ 的置信带。

---

# 二十六、为什么个体 CATE 的置信区间会很宽？

因为 $\tau(x)$ 是一个高维非参数函数。

若 $x\in\mathbb R^{15}$，要在目标点附近找到足够多的相似患者，同时还要保证两种治疗都有样本、删失不严重，这非常困难。

这就是维数灾难。

论文也指出：

- 点态非参数推断本身很难；
    
- 区间可能很宽；
    
- 理论要求 undersmoothing；
    
- 偏差是否足够小在实践中不易判断。
    

所以 $\hat\tau(x)$ 更准确地说是“局部患者群体平均效果”，而不是某个患者确定性的命运预测。

---

# 二十七、什么是 Best Linear Projection？

既然完整的 $\tau(x)$ 很复杂、逐点置信区间不稳定，可以用一个低维线性模型概括它：

$$
\tau(X)
\approx
\beta_0+A^\top\beta,
$$

其中 $A$ 可以只包含年龄、性别等少数变量。

Best Linear Projection 定义为：

$$
(\beta_0^\ast,\beta^\ast)
=
\arg\min_{\beta_0,\beta}
E[
(\tau(X)-\beta_0-A^\top\beta)^2
].
$$

例如只取年龄：

$$
\tau(X)\approx\beta_0+\beta_1\cdot\text{Age}.
$$

那么：

- $\beta_1>0$：治疗效果平均随年龄增加；
    
- $\beta_1<0$：治疗效果平均随年龄减弱；
    
- 区间包含 0：没有足够证据认为年龄系统性修改疗效。
    

BLP 不声称真实 $\tau(x)$ 是线性的，它只是复杂函数的最优线性摘要。论文使用它来获得更稳健、可解释的异质性推断。

---

# 二十八、这篇论文最重要的概念关系图

你现在可以把它看成以下六层：

$$
\boxed{\text{层 1：生存数据}}
$$

$$
T,C,U,\Delta,\quad
S_T,\lambda_T,\quad
T\wedge h
$$

解决：真实结局没有完整观测。

$$
\Downarrow
$$

$$
\boxed{\text{层 2：因果目标}}
$$

$$
T(0),T(1),\quad
ATE,\ CATE
$$

解决：想预测的不是结局，而是治疗造成的差值。

$$
\Downarrow
$$

$$
\boxed{\text{层 3：识别假设}}
$$

$$
\text{ignorability, overlap, censoring independence, positivity}
$$

解决：为什么不可观察的反事实能够从观察数据恢复。

$$
\Downarrow
$$

$$
\boxed{\text{层 4：正交估计}}
$$

$$
e,m,S^C,\lambda^C,Q,\quad
\psi_\tau
$$

解决：怎样纠正治疗选择和删失，同时允许使用灵活 ML 模型。

$$
\Downarrow
$$

$$
\boxed{\text{层 5：森林}}
$$

$$
\alpha_i(x),\quad
\sum_i\alpha_i(x)\psi_i=0
$$

解决：怎样学习 treatment effect 随 $x$ 的非线性变化。

$$
\Downarrow
$$

$$
\boxed{\text{层 6：统计推断}}
$$

$$
\text{consistency, asymptotic normality, CI, BLP}
$$

解决：怎样评价估计的不确定性，而不只是输出一个数。

---

# 二十九、你精读前真正需要补到什么程度？

不需要先学完整本科统计学。建议按下面顺序补：

## 第一组：必须先懂

1. 条件期望：  
    $$
    E[Y\mid X]
    $$
    
2. 回归与平方损失；
    
3. 随机变量独立、条件独立；
    
4. 大数定律与中心极限定理；
    
5. 决策树、随机森林的基本结构；
    
6. 生存函数和右删失；
    
7. 潜在结果、ATE、CATE；
    
8. propensity score 和 overlap。
    

这些懂了，就可以读 introduction 和第 2.1 节。

## 第二组：读方法主体时补

1. estimating equation；
    
2. residualization；
    
3. IPCW；
    
4. influence function 的直觉；
    
5. doubly robust estimator；
    
6. Neyman orthogonality；
    
7. cross-fitting；
    
8. generalized random forest。
    

这些懂了，就可以读第 2.2、2.3 节。

## 第三组：读理论证明时补

1. $O_p,o_p$；
    
2. 一致性和渐近正态性；
    
3. U-statistics 与随机森林渐近理论；
    
4. influence function linearization；
    
5. subsampling；
    
6. honesty；
    
7. bootstrap of little bags；
    
8. 半参数统计中的 nuisance-rate conditions。
    

这些不必在第一次读正文前全部学完。

---

# 三十、第一次读论文时哪些公式先跳过？

第一次不要正面硬啃完整的式 (9)、(11) 和附录证明。

先抓住五个关键公式：

### 目标

$$
\tau(x)
=
E[y(T(1))-y(T(0))\mid X=x].
$$

### 完整数据 score

$$
\psi_\tau^{(c)}
=
(W-e(X))
\left[
y(T)-m(X)-\tau(W-e(X))
\right].
$$

### 森林权重

$$
\alpha_i(x)
=
\frac1B
\sum_b
\frac{
\mathbf 1{X_i\in N_b(x)}
}{
|N_b(x)|
}.
$$

### IPCW 原理

$$
\frac{\Delta^h}{
S_W^C(T\wedge h\mid X)
}.
$$

### 最终局部方程

$$
\sum_i
\alpha_i(x)
\psi_{\hat\tau(x),i}
=0.
$$

只要这五个公式之间的逻辑串起来，整篇论文的骨架就已经掌握了。

最合理的下一步是按论文的公式顺序，具体推导式 (2) 的 estimand、式 (3) 的 Robinson score、式 (8) 的 IPCW，再逐项拆解最难的式 (11)。