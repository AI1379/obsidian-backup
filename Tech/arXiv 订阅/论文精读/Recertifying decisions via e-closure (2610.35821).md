---
tags: [paper, paper/statistics, paper/conformal, paper/multiple-testing, paper/agent, tech/arxiv]
---

# Recertifying decisions via e-closure: uniform improvements of conformal risk control and randomized e-BH

- **arXiv**: [2610.35821](https://arxiv.org/abs/2610.35821)（stat.ME / math.ST / stat.ML / stat.TH, 2026-10-26）
- **作者**: Tian Bai, Aaditya Ramdas（Stanford University）
- **一句话**: 提出 recertification 范式——把有效程序的"未用完风险预算"变成更大的 e-value，从而在不破坏任何保证的前提下把决策做得更不保守；统一改进 conformal risk control 与 randomized e-BH。

## TL;DR
1. 几乎所有风险受控程序都会**剩下一部分风险预算没用**；直接改决策很难知道能推多远，但改 e-value 很容易。
2. Recertification：把程序损失除以预算得到 canonical e-collection，计算它与 1 的差（slack），按条件期望不变的方式加回去 → 更大的 e-value → 反演得到包含原决策、但更激进的认证菜单；可反复迭代。
3. 两个应用都给出"一致改进"：CRC 的 recertified controller 在每个校准样本上都不比 CRC 保守，凸损失下几何收敛到 admissible；randomized e-BH 得到闭式 RU-eBH / RC-eBH，拒绝集逐点包含原方法，期望拒绝数在同预算类中最大。

## 背景：决策、风险预算与 e-closure
- 一般决策问题：数据 X∼P∈P_s，动作 a∈A，损失族 L_t(s,a;X)，预算 α_t(s)。有效性：对所有 s,P,t，E_P[L_t(s,A(X);X)] ≤ α_t(s)。
- **e-collection**：对每个 (s,t) 一列非负统计量 E_{s,t}，满足 E_P[E_{s,t}(X)] ≤ 1。
- **Theorem 2.1**：任何从认证菜单 C_α(E;x) = {a : L_t(s,a;x) ≤ α_t(s) E_{s,t}(x) ∀s,t} 里选出的动作都有效；反之所有有效程序都能被某 e-collection 认证（e-closure 原则，Xu et al. 2026b；Grünwald 等）。
- **关键观察**：有效程序的 e-value 常常期望严格小于 1；如果这个"缺口"可计算（例如在已知条件分布下取条件期望），把它加回去就得到更大的 e-value，仍保持有效性。
- **Proposition 2.2（conditional recertification）**：若 W_{s,t}(X)≥0 且满足相应条件期望约束，则 E_{s,t}+W_{s,t} 仍是 e-collection。Recertification 就是把该构造用到"损失/预算"这个 canonical e-collection 上，反演回决策。

## 应用一：Conformal Risk Control（CRC）
**设定**
- 观测 Z_1..Z_n（校准）+ Z_{n+1}（测试）有限可交换；动作 λ∈Λ（有序，越大越保守），损失 ℓ(z,λ) 有界、对 λ 单调不增、右连续，且 ℓ(z,λ+)≤α；预算 (n+1)α。
- CRC：选最小 λ 使 Σ_{i=1}^n ℓ(Z_i,λ) + B ≤ (n+1)α；envelope CRC 用更紧的 U(λ)≤B。

**精确刻画（Lemma 3.1）**
- 对称控制器 A 在任意有限可交换分布下有效 ⇔ 对每个确定的增广多重集 S=(z_1..z_{n+1})，留一损失和 T_A(S)=Σ_i ℓ(z_i, A(S_{−i})) ≤ (n+1)α。
- 未用预算 Δ_A(S) = (n+1)α − T_A(S) 是精确可计算的 slack。

**Recertification 一步（等分）**
- 新预算 b_i^A(S) = r_i^A(S) + Δ_A(S)/(n+1)；反演：
  (RA)(D) = min{ λ ≤ A(D) : ℓ(z,λ) − ℓ(z,A(D)) ≤ Δ_A(D∪{z})/(n+1) ∀z∈Z }。
- **Theorem 3.2（一致改进）**：RA 仍有效，且 RA ≤ A 逐点；迭代 A^{(k+1)}=R A^{(k)} 满足
  A^{(k+1)}(D) ≤ A^{(k)}(D) ≤ A_env(D) ≤ A_CRC(D)，每轮都控制期望损失。
- **Theorem 3.5（admissible 极限）**：Λ 为区间且损失对 λ 凸时，迭代几何收敛到逐点 admissible 控制器：
  0 ≤ A^{(k)}(D) − A^{(∞)}(D) ≤ ρ^k (A^{(0)}(D) − A^{(∞)}(D))，ρ = n/(n+1)；有效控制器 admissible ⇔ 是 R 的不动点。
- **二维示例**（n=2, α=1/2）：envelope CRC 在 (HH,HL,LL) 上为 (1/2, 2/5, 1/4)，迭代 A(1)=(1/2,23/60,1/5)，A(2)=(1/2,17/45,31/180)，极限 (1/2,3/8,1/8)，与 ρ=2/3 一致；admissible 控制器不唯一（(1/2,2/5,1/10) 也 admissible 但不可比）。

**真实数据：CIFAR-100 层次分类**（ResNet-18 冻结，n=50，300 次配对划分）
- 置信度自适应规则，α=0.15：CRC 风险 14.21% → 反复 recertify 到 14.95%（目标 15%），平均预测集 10.77 → 8.92 个标签（−17.2%）。
- α=0.10、500 轮：correct specificity 67.57% → 68.24%，平均节点大小 19.40 → 17.03；α=0.02 时 CS 提升 1.71 个百分点。
- 共同随机深度族里 envelope CRC 本身已很紧，recertification 增益较小（α=0.15 时 +0.44pp）；置信度自适应族里 envelope CRC=CRC，全部增益来自 recertification。

## 应用二：Randomized e-BH（FDR 控制）
- 背景：e-BH 在任意相依下控制 FDR，但保守；U-eBH 通过除以 U∼Unif(0,1) 提升 power；closed e-BH 通过按 |T| 归一化的 e-collection 自适应真实零假设数。
- **RU-eBH（recertified U-eBH）**：e-values 排序 e[1]≥…≥e[K]，s_r=Σ_{j≥r} e[j]；令 q*_r = min{1,(α/K)(r e[r]+ s_{r+1})}；取 κ*=max({r: U≤q*_r}∪{0})，拒绝最大的 κ* 个。性质（Prop 4.1）：
  (i) FDR ≤ α|T|/K ≤ α；
  (ii) 对每个 e,U，拒绝集包含 U-eBH（从而包含 e-BH）；
  (iii) 在从 E^{eBH} 菜单选择的 boosting 规则中随机意义上拒绝数最大：P(κ≥r|E=e) ≤ P(κ*≥r|E=e)。
- **RC-eBH（recertified closed e-BH）**：用 mean e-collection（按 |T| 归一化）的更大 e-collection；以累积 charge S_s(p)=Σ_{r≥s} c_[r](p) 表述，取 caps β_s=α m_s 的**最大凸下界（greatest convex minorant）** S♯，tail q♯_r=r S♯_r−(r−1)S♯_{r+1}；同样用 U 选择 κ♯。性质（Prop 4.2）：FDR≤α；q♯_r≥q*_r ⇒ 拒绝集包含 RU-eBH/U-eBH/e-BH；在 mean e-collection 认证的**所有随机规则**中期望拒绝数最大。
- **模拟**：K=100，10/50/90 个真替代，Gaussian（均值 1–4），相关系数 ρ=0/0.5/0.9，α=0.1，50,000 次重复。
  - 10 个替代、signal 2.5、独立：RU-eBH power 0.32 vs U-eBH/J-eBH 0.22 vs 确定性方法 ≤0.06；RU 的 FDR 贴近 α|T|/K=0.09，U-eBH 只用到 40–70%。
  - 90 个替代、signal 2、独立：RC-eBH power 0.64 vs RU-eBH 0.52 vs closed e-BH 0.55；RC-eBH 在几乎所有设置最强，最强信号时与最好方法差 <0.005。

## 为什么重要 / 可迁移到 Reliable Agent
- **通用配方**：任何以 e-value / 风险预算认证的决策系统，都可以"找出未用完的预算 → 放大证书 → 得到更激进的决策且保证不变"。这正好对应 Agent 场景里"过于保守的拒答/保守预测"造成的可用性损失。
- **Agent 决策的风险控制**：把"是否采纳某个结论/是否输出预测集/是否在多选题中排除选项"作为 action，损失取错误率或 coverage 损失；用 conformal/e-value 证书控制风险，再用 recertification 在不牺牲保证下提升信息量/召回。
- **与 SuffiCE 的组合**：SuffiCE 给出"证据够不够"的校准置信度，可作为 action 的分数；recertification 提供"在风险预算内尽量激进"的选择规则——一个是"信号校准"，一个是"决策校准"，可拼成 agent 的可靠停止/作答策略。
- **技术要点可复用**：exchangeability 下"每个增广样本一个约束"的精确刻画；可计算 slack 的条件化；凸损失下的几何收敛率 n/(n+1)；多重检验里用 convex minorant 求闭式最优随机规则。

## 局限与注意
- CRC 部分需要**已知损失族/包络**与可交换性；Theorem 3.5 的 admissible 极限要求损失对 λ 凸且动作是区间（网格需用其他 completion）。
- RU-eBH 的最优性是相对 E^{eBH}（按 K 归一化）；当真实零假设少时会被 closed e-BH 超过，故需要 RC-eBH。
- RC-eBH 依赖 mean e-collection，**不适用于 true null 集合随机**的 selective inference 场景（Remark 4.4 明确不延伸）。
- 计算：CRC 每轮要枚举所有可能 z（有限空间可做表）；e-BH 版本只需排序 + 部分和 / O(K) minorant，代价很低。

## 原文链接
- Abs: https://arxiv.org/abs/2610.35821
- PDF: https://arxiv.org/pdf/2610.35821
