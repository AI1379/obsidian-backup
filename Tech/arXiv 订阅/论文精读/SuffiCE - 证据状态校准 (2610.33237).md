---
tags: [paper, paper/llm, paper/calibration, paper/agent, tech/arxiv]
---

# Each Insufficient in Its Own Way: Calibrating Language Models Across Varying Evidence States

- **arXiv**: [2610.33237](https://arxiv.org/abs/2610.33237)（cs.CL, 2026-10-23）
- **作者**: Jiayi Li, Dennis L. Shung, Bradly C. Stadie（Northwestern University / Mayo Clinic）
- **一句话**: 提出 evidence-state calibration（证据状态校准）与 SuffiCE，训练模型在**作答前**给出"当前证据够不够回答"的校准置信度。

## TL;DR
1. 现有校准把输入当固定，只检验 confidence 是否等于 accuracy，却不检验"证据变化时 confidence 是否跟着变"。作者把后者形式化为 evidence-state calibration。
2. SuffiCE 不额外加分类头，直接从模型 next-token 分布里读 A（Answer）/ S（Seek）两个 token 的 softmax 作为置信度，用蒙特卡洛估计的"该证据状态下的成功率"做软标签 BCE 训练。
3. 在 4 个模型（Kimi K2.6、Qwen3.8-27B、Qwen3.5-9B/4B）× 3 个数据集（MIMIC、MedQA、MuSiQue）上全面最优：Brier 比 base logit 降 20–60%、比 Platt 降 5–28%，AUROC 最多 +0.16；且置信度随"决定性证据"上升，而不是随"证据数量"上升。

## 问题设定
- 问题 i：question q_i、证据池 E_i、金标答案 y_i*。证据状态 x=(q_i, I)，I⊆E_i。
- 固定回答策略 π0，成功率 p*(x)=Pr_{ŷ∼π0(·|x)}[g(ŷ,y_i*)=1]。
- **sufficiency 由"模型能否答对"定义，而不是"输入是否包含关键事实"**——同一证据状态对不同模型可以充分/不充分。
- 交互场景（追问、检索、开检查）每一步都要决定：现在答，还是继续找证据。需要**作答前**的校准信号。

## 核心定义：evidence-state calibration
要求 cθ(x)∈[0,1] 满足：
1. **Calibration**：E[p*(x) | cθ(x)=c] = c（同置信度水平的平均成功率等于该置信度）。
2. **Evidence sensitivity**：cθ(x_{i,t}) − cθ(x_{i,s}) ≈ p*(x_{i,t}) − p*(x_{i,s})（证据变化引起的置信度变化 ≈ 成功率变化）。

**Proposition 1（误差分解）**：
E[(cθ−p*)²] = E_i[(c̄θ,i−p̄*i)²] + E_i[ 1/(2m_i²) Σ_{s,t} δ_st² ]，
第一项是每个问题平均成功率上的误差，第二项是问题**内部**跨证据状态的误差；只做 calibration 无法约束第二项（把每个状态都预测成问题平均成功率即可满足 calibration，但第二项仍大）。→ 必须在多个证据状态上训练。

## 方法 SuffiCE（三阶段）
1. **构造证据状态**（每个问题多个状态，共享问题与金标，只变证据子集）
   - MIMIC：急诊腹部四选一诊断；证据分 history / physical exam / imaging / other tests（lab、pathology），状态为不同子集。
   - MedQA：USMLE 题，按 AgentClinic 结构化临床病例；状态为逐步加入检查/病史。
   - MuSiQue：3-hop 多跳 QA（chain 链式 / branching 分支式）；证据单元 e1,e2,e3；状态含 question only、distractors only、单跳、两跳、全部；其余用 distractor 补齐到 3 段，避免"段数/位置"泄露状态。
2. **蒙特卡洛估计成功率**：冻结回答策略 π0，每个状态采样 K=10 个答案，p̂(x)=正确比例（对 p*(x) 无偏，Bernoulli 均值）。
3. **训练估计器**：prompt 不真正答题，只问"现有信息是否足以正确回答"；定义两个动作 token A（answer）/S（seek）；cθ(x)=softmax(ℓ_A,ℓ_S)；损失为软标签 BCE：
   L(x) = −p̂(x)log cθ(x) − (1−p̂(x))log(1−cθ(x))。
   BCE 是 strictly proper scoring rule，且对目标线性 ⇒ 期望损失在 cθ=p* 最小化；**Corollary 1**: E[(cθ−p*)²] ≤ ½(E[L]−L*)，训练损失同时控制 Proposition 1 的两项。

## 实验
- 模型：Kimi K2.6、Qwen3.8-27B、Qwen3.5-9B、Qwen3.5-4B；数据集 MIMIC / MedQA / MuSiQue；3 seeds；K=10。
- 指标：Brier↓、AUROC↑（置信度对"答对/答错"的排序能力）、ρΔ↑ = Corr(c_{i,t}−c_{i,s}, p̂_{i,t}−p̂_{i,s})（跨状态变化的相关，直接对应 evidence sensitivity）。
- 主要结果（Table 1，摘几个）：
  - Qwen3.8-27B · MedQA：Brier 0.143→0.091，AUROC 0.744→0.826，ρΔ 0.413→0.489
  - Qwen3.8-27B · MuSiQue：Brier 0.182→0.092，AUROC 0.738→0.830，ρΔ 0.274→0.717
  - Kimi K2.6 · MIMIC：Brier 0.219→0.105，AUROC 0.686→0.844
  - SuffiCE 在所有设置所有指标均最优；Platt 只能缩放不能重排，AUROC/selective accuracy 不变，SuffiCE 能提升排序。
- **Selective answering**（只答最自信的 10%/30%/50%）：SuffiCE 精度全面领先，如 MIMIC 10% 时 100.0%，MedQA 10% 时 99.1%，MuSiQue 10% 时 68.9%。
- **消融**：只用 full state 训练（不看缺失证据）在所有指标上差于跨状态训练 → 收益来自"跨证据状态"。
- **机制分析**：
  - 置信度随**决定性证据**而非证据数量上升：MuSiQue chain 中 {e1,e2} 时 base logit 已 0.71 而真实成功率仅 0.06，SuffiCE 只有 0.16，直到 e3 出现才升；branching 中单个分支不升。
  - 固定证据数量、改变组成：SuffiCE 能区分哪种组合可答（含 e3 才高）。
  - 医学案例（Hirschsprung 病）：X-ray 只能提示梗阻，biopsy 才能确诊；SuffiCE 在 {1,4} 保持 0.5 左右，加入 biopsy（{1,3,4}）才升到 0.97；base logit 各状态都接近 1。
  - **跨策略估计**：固定 Kimi K2.6 为回答策略、训练 Qwen 估计器；27B 估计器在 Brier 上匹配甚至优于 Kimi 自估，说明可从证据状态本身预测另一模型的成功率。

## 局限
1. 证据状态是**预构造**的，不是真实在线多轮交互；是受控实验。
2. 每个状态只采样 K=10 个答案，目标有噪声（增大 K 可更准）。
3. 未在真实临床场景验证，不应替代医生判断。

## 与我的研究（Reliable Agent + 统计保证）的关系
- **Agent 停止准则**：cθ 可直接作为"继续检索/调用工具 vs 现在作答"的校准信号；比用原始 logit 或 verbalized confidence 更可靠。
- **评测指标**：ρΔ 提供"证据获取策略是否真的让置信度跟上成功率"的度量，补足静态 ECE/Brier。
- **可组合**：把 cθ 当分数做阈值选择时，可再用 conformal/风险控制给出错误率保证（与另一篇 recertification 思路天然互补）。
- **可迁移**：工具调用场景下，把"证据 = 工具返回内容"，训练 sufficiency estimator 作为 tool-use policy 的 stopping rule；或用于 multi-agent 中判断"队友给的信息是否足够"。

## 原文链接
- Abs: https://arxiv.org/abs/2610.33237
- PDF: https://arxiv.org/pdf/2610.33237
