---
title: Looped Transformer 与 CoT
created: 2026-09-03
source: 与纳西妲对话整理
tags: [tech, dl-theory, llm, architecture]
---

# Looped Transformer 与 CoT

> 2026-09-03 对话：关于 looping transformer（循环权重共享）、它与 Chain-of-Thought 的异同，以及流形/维度视角下的理解。含旅行者提出的"维度压缩猜想"。

## 1. 什么是 Looped Transformer

一个 $L$ 层网络（如 80 层）拆成"一个小网络（如 20 层）× 重复 4 次"，**循环使用同一组参数**（weight tying / weight sharing）。

**谱系**：
- Universal Transformer (Dehghani et al. 2018)：最早的 loop + 动态停机
- ALBERT：跨层共享参数 → 证明层数冗余可观
- Looped Transformer 系列（Alberti et al.）：显式迭代细化
- 2025 热点：Huginn（surprise-driven 循环深度推理）、*Looped Transformers are Better at Learning Learning Algorithms* (Geiping et al. 2025)——证明循环结构能学到真正的迭代式算法，一次前向 = 算法一步
- inference-time recurrence：推理时把浅层多循环几遍，数学准确率提升（测试时加深/扩算力）

## 2. 为什么它能 work（直觉障碍的消解）

**直觉障碍**：假设 80 层 = 80 个不同的算子串行复合，每层做不可替代的事 → 替换成 $T\circ T\circ T\circ T$ 当然损失巨大。

**实证反驳**：
1. **层间相似性高**：CKA 等分析显示中段许多层做的事高度相似，跳过某几层影响不大（layer pruning 文献）。
2. **残差流视角**：每层做 $h \leftarrow h + F(h)$，中段 $F$ 是小修正。80 层 ≈ 缓慢连续形变，而非 80 个质变。
3. 既然相邻层做相似修正，"共享成一个、重复 4 次"自然成立。

**表达力**：共享参数限制了函数类（不能表达 4 段完全不同复合），换来参数效率 + 强先验："同一操作的不动点迭代/流"。

## 3. 流形与动力学视角：为什么合理

### 3.1 循环 = 学习流上的向量场
单次 20 层 = 算子 $T$，循环 4 次即 $T^4$。若 $T = I + \varepsilon F$（接近恒等的 refinement），循环就是在数值积分一个 ODE $dh/dt = F(h)$，步长 $\varepsilon$，走 4 步。
- 与扩散模型逐步去噪、DEQ（Deep Equilibrium Model，无限深 = 不动点 $F(h^*)=h^*$）同族思想。
- 80 层前馈 ≈ Euler 离散化的一条连续轨迹；loop 用同一参数的向量场重新参数化。

### 3.2 几何：表征沿流形连续形变
- token 嵌入 → 语义空间的变换是**连续路径**而非 80 次跳变（"deep networks as discretized flows"、增广神经常微分方程等）。
- 若信息在低维流形上流动，中间 60 层的工作 ≈ "沿流形做平滑搬运 + 逐轮精细化"，这种**同构的重复搬运最适合共享参数**。
- 真正不可共享的是首尾：嵌入理解与最终读出。

### 3.3 算法视角
很多任务（多步推理、状态跟踪）需要**串行步数**，但不需要"每步不同的计算"。循环把"深度（串行步数）"与"参数多样性"解耦；模型想学迭代算法时，loop 是天然参数化，梯度信号重复叠加 k 次反而更稳。

## 4. 旅行者猜想：维度压缩（2026-09-03 对话记录）

**原猜想**：会不会这本质是维度压缩——隐藏层信息其实只在远低于隐藏层维度的子空间上流动；loop 相当于把多个空间做直积，更高效利用隐藏维度。

**讨论后的修正**：
- ✅ 对的部分：信息确实在低维子空间流动（本征维数测量、LoRA 低秩修正均为证据）；"深度=时间=新自由度"，中间结果写回同一块 d 维内存复用——像一台只有 d 个寄存器、靠多步程序完成超单步能力计算的机器。
- ❌ 需修正："直积"表述。循环**不是**把空间做成直积 $R^d \otimes R^d \otimes \cdots$，而是**拒绝**直积式扩容：轨迹被约束为离散动力系统轨道 $\{h, T(h), T^2(h), T^3(h)\}$——只有 1 个生成元，而非 4 个自由选择。
- 更贴切对应：**Kolmogorov-Arnold / 深度复合表示论**——复合低元函数可在不增加环境维数下表达高复杂度函数；循环是极端版（复合同一个函数）。
- 一句话：循环是用强约束换参数效率，用"时间维度上复用低维内存"高效利用维度，而非维度扩张。

**要处理的数学问题**：
1. **表达力下界**：共享权重回路（类比电路复杂度）严格弱于不共享深度电路（TC⁰ 附近）。80 层独立可表达但 $T^4$ 不可表达的函数类存在——需要"同一生成元迭代可模拟"这一真实算法论约束。
2. **迭代动力学稳定性**：输出 Jacobian ≈ $J^k$，$T$ 须在"不收缩太快（未处理完就塌缩到不动点）"与"不发散"之间；流形视角帮忙：谱控制只需对切空间负责，法向收缩无害。
3. **步数与精度交换**：收敛率决定 $k$ 需求；adaptive computation（动态停机）为此设计。
4. **可辨识性/阶段对齐**：共享权重下无法硬性规定"第 i 次循环做第 i 类工作"；要么纯 refinement，要么时间步嵌入引导。

## 5. Looped Transformer vs CoT

**共同本质**：时间即算力。两者都突破"一次前向 = 固定深度电路"（TC⁰ 附近天花板），把串行步数作为新自由度。
- Merrill & Sabharwal：CoT 长度 $O(n)$ → transformer 能模拟多项式时间（TC⁰ → P）
- Geiping et al. 2025：looped transformer 能学迭代算法

**关键区别：中间状态住在哪里**

| 维度 | CoT | Looped |
|---|---|---|
| 中间状态介质 | **外化为 token**（离散瓶颈：经过输出头 → 词表分布 → 采样） | **连续隐向量**（内部迭代，不过词表） |
| 中间记忆容量 | 被"可被词表表达的字符串"限制（0.7381 有舍入） | 连续向量，流形上信息无损传递 |
| 步数 | 原则上无上界，可自适应（o1/R1 test-time scaling） | 结构固定（4 次就 4 次）；只能动态停机做有限自适应 |
| 中间语义 | 被训练目标强制为可读推理文本（但显示 CoT 未必忠实） | 裸计算轨迹，不解释自己 |
| 训练来源 | 从"预测下一个 token"自监督中**涌现** | 架构师**显式设计**（循环次数、权重绑定） |
| 错误传播 | 离散采样不可逆误差，一步错步步错；但人可读可干预 | 连续迭代温和漂移；无人类干预接口 |

**统一视角**：CoT 是"退化的循环"——它的 $T$ = "整个模型 + 自回归条件化"，迭代函数每步被新 token 扰动；loop 是纯净的 $T^4$。两者是同一骨架（离散动力系统轨道）的两种介质：
- CoT 的"低维内存" = token 序列（外存、离散、可无限扩展）
- Loop 的"低维内存" = 隐藏状态（内存、连续、容量固定 d）

**后继谱系**：Coconut (2024, Chain of Continuous Thought) 把最后隐状态喂回输入、不生成 token → "latent reasoning" 系列在探索中间态从纯语言滑向纯向量的整个谱系。

## 关键文献

- Dehghani et al. (2018): Universal Transformers
- Lan et al. (2019): ALBERT
- Bai et al. (2019): Deep Equilibrium Models
- Geiping et al. (2025): Looped Transformers are Better at Learning Learning Algorithms
- Merrill & Sabharwal: The Expressive Power of Transformers with Chain of Thought
- Hao et al. (2024): Coconut（Training LLMs to Reason in a Continuous Latent Space）
- 增广 Neural ODE 系列（discretized flow 视角）

## 相关笔记

- [[深度学习理论 索引]]
- [[流形假说与 LLM 表征几何]]
- [[频谱偏置、良性过拟合与 NTK]]
- [[因果推断]]（MIF/MCF 引发讨论）
