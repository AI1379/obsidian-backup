# 因果推理与 AI 调研报告

> **创建日期**: 2026-04-03
> **标签**: #调研 #因果推理 #机器学习 #LLM

---

## 1. 什么是因果推理

传统机器学习（包括大模型）学习的是**相关性**（correlation），而因果推理回答的是**因果性**（causation）：

| | 相关性 | 因果性 |
|---|---|---|
| 问题 | A 和 B 是否一起出现？ | A 是否导致了 B？ |
| 数学表达 | $P(Y \mid X)$ | $P(Y \mid do(X))$ |
| 可否用于决策？ | 不一定 | 可以 |
| 大模型在哪一层？ | 基本在这层 | 尚在探索 |

**经典例子**：冰淇淋销量和溺水人数正相关。相关性模型会建议少卖冰淇淋来减少溺水，但真正的因是**气温升高**同时导致了两者增加。

---

## 2. 因果之梯（Judea Pearl）

Pearl（图灵奖得主，贝叶斯网络之父）提出三层因果层级：

### 第一层：Association（观察）
- **问题**: "如果我看到了 X，Y 会怎样？"
- 条件概率 $P(Y \mid X)$
- 所有监督学习、大模型基本停在这层

### 第二层：Intervention（干预）
- **问题**: "如果我**做了** X，Y 会怎样？"
- 涉及 **do-calculus**: $P(Y \mid do(X))$
- 与 $P(Y \mid X)$ 本质不同
- 例："如果我吃了药，病会不会好？" vs "吃了药的人病好了吗？"

### 第三层：Counterfactual（反事实）
- **问题**: "如果我当时没做 X，Y 会怎样？"
- 最高层因果推理
- 需要对世界有结构化理解（因果图）

### 核心工具

- **因果图（DAG）**: 有向无环图表示变量间因果关系
- **do-calculus**: 从观察数据计算干预效果，无需随机对照试验
- **结构因果模型（SCM）**: $Y = f(X, U)$，比纯概率模型多一层因果语义

---

## 3. 因果推理 + 深度学习（综述）

### 3.1 Causal Machine Learning: A Survey and Open Problems (2022)

- **链接**: https://arxiv.org/abs/2206.15475
- **核心贡献**: 将因果机器学习（CausalML）分为五大类：
  1. 因果表示学习（Causal Representation Learning）
  2. 因果发现（Causal Discovery）
  3. 因果效应估计（Treatment Effect Estimation）
  4. 反事实推理（Counterfactual Reasoning）
  5. 因果强化学习（Causal Reinforcement Learning）
- **评价**: 最系统的入门综述

### 3.2 Causal Inference Meets Deep Learning: A Comprehensive Survey (2024)

- **链接**: https://spj.science.org/doi/10.34133/research.0467
- **核心贡献**:
  - 从脑启发视角讨论因果推理
  - 对比潜在结果模型（POM）和结构因果模型（SCM）
  - 覆盖大模型时代的最新进展

### 3.3 Causality from Bottom to Top: A Survey (2025)

- **链接**: https://link.springer.com/article/10.1007/s10994-025-06855-5
- **核心贡献**:
  - 因果推断全流程：从因果发现到效应估计
  - 区分因果推断（causal inference）和因果发现（causal discovery）
  - 发表在 Machine Learning 期刊

---

## 4. 因果推理 + 大语言模型（前沿热点）

这是目前最活跃的方向，核心争论：**LLM 是真有因果推理能力，还是只是更精致的统计关联？**

### 4.1 支持派：LLM 有因果推理能力

**Causal Reasoning and Large Language Models: Opening a New Frontier for Causality**
- **会议**: ICLR 2025
- **团队**: 微软 Research
- **链接**: https://openreview.net/forum?id=mqoxLkX210
- **发现**:
  - 对 LLM 做了行为学实验（behavioral study）
  - LLM 在生成因果论证方面**超越传统方法**
  - 涵盖因果发现、反事实推理、事件因果关系
- **意义**: 如果成立，LLM 可以成为因果推理的强大工具

### 4.2 质疑派：LLM 只是伪装的关联

**Unveiling Causal Reasoning in Large Language Models: Reality or Mirage?**
- **会议**: NeurIPS 2024
- **链接**: https://proceedings.neurips.cc/paper_files/paper/2024/hash/af2bb2b2280d36f8842e440b4e275152-Abstract-Conference.html
- **发现**:
  - 构建 **CausalProbe2024** 基准测试
  - LLM 在新基准上性能大幅下降
  - **结论**: LLM 主要做第一层因果推理（association），达不到第二层（intervention）
  - 核心发现：大模型的"因果推理"可能是精致的统计关联

### 4.3 方法改进

**Causal-aware Large Language Models**
- **会议**: IJCAI 2025
- **链接**: https://dl.acm.org/doi/10.24963/ijcai.2025/478
- **方向**: 让 LLM 具备因果意识，用于复杂决策

**Large Language Models for Causal Discovery**
- **会议**: IJCAI 2025
- **链接**: https://www.ijcai.org/proceedings/2025/1186.pdf
- **方向**: 用 LLM 从数据中自动推断因果结构（因果图发现）

**Eliciting and Improving the Causal Reasoning Abilities of LLMs**
- **期刊**: Computational Linguistics, 2025
- **链接**: https://direct.mit.edu/coli/article/51/2/467/127461
- **方向**: 如何激发和提升 LLM 的因果推理能力（反事实、溯因推理）

### 4.4 综述

**Causal Inference with Large Language Model: A Survey**
- **会议**: NAACL 2025 Findings
- **链接**: https://aclanthology.org/2025.findings-naacl.327.pdf
- **内容**: LLM 做因果推断的方法、局限和未来方向

---

## 5. 争论总结

| 立场 | 代表工作 | 核心论点 |
|---|---|---|
| **LLM 有因果能力** | 微软 ICLR 2025 | LLM 在因果任务上超越传统方法 |
| **LLM 只是关联** | NeurIPS 2024 CausalProbe | 换测试集就暴露，本质是 pattern matching |
| **中间立场** | 多篇 | LLM 在第一层很强，第二三层需外部工具辅助 |

---

## 6. 基础阅读推荐

| 书/论文 | 难度 | 备注 |
|---|---|---|
| **《The Book of Why》** (Pearl) | ⭐ | 科普书，中文版《为什么》 |
| **《Causal Inference in Statistics: A Primer》** | ⭐⭐ | 薄薄一本数学入门 |
| **《Causality》** (Pearl) | ⭐⭐⭐⭐ | 圣经级，完整的数学理论 |
| **Bernhard Schölkopf 因果表示学习系列论文** | ⭐⭐⭐ | Max Planck 所，因果+DL 结合 |

---

## 7. 与现有研究方向的关系

### 7.1 因果推理 × RL（强化学习）

- 因果强化学习：用因果图加速策略学习
- Offline RL 中的混淆偏差（confounding bias）问题
- 与我之前给旅行者讲的 RL 内容相关（策略梯度/PPO 的因果视角）

### 7.2 因果推理 × 对齐（AI Alignment）

- 因果推理可以帮助 AI 理解人类意图的**原因**
- 反事实推理是安全对齐的基础：AI 需要理解"如果我做 X，会怎样"
- 与 RLHF 的关系：因果关系 vs 奖励模型的关联性

### 7.3 因果推理 × 数学

- do-calculus 的完备性证明本身就是数学定理
- 因果发现涉及图论、条件独立性检验
- 作为数院学生，这是一个**数学功底有天然优势**的 AI 方向

---

## 8. 开放问题

1. **LLM 是否真的能做因果推理**，还是只是更灵活的模式匹配？
2. 如何将因果图知识**注入** LLM，而非期望 LLM 自己学会？
3. 因果表示学习能否学到**可迁移**的因果特征？
4. 反事实推理在 LLM 中的机制是什么？
5. 因果推理能否解决 LLM 的幻觉问题？

---

*本报告基于 2022-2025 年的论文和综述整理，作为初步调研参考。*
