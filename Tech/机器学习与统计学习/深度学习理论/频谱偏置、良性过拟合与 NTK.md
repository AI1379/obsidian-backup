---
title: 频谱偏置、良性过拟合与 NTK
created: 2026-09-03
source: 与纳西妲对话整理
tags: [tech, dl-theory, ml-theory]
---

# 频谱偏置、良性过拟合与 NTK

> 三个假说本质上是同一个数学结构——梯度下降在线性化动力学下的谱分析——在不同问题上的投影。

## 1. 频谱偏置（Spectral Bias）

**现象**：深度网络学习目标函数时，倾向于**先学低频分量，后学高频分量**。用傅里叶视角看，训练误差按频率从低到高逐步消除。

- Rahaman et al. (2019) 经典实验：对 1D 函数和图像，低频分量收敛速度显著快于高频。
- 直觉：网络先拟合轮廓/平滑结构，最后拟合细节和噪声。

**理论解释（NTK 路线）**：NTK 的特征值谱在高频方向衰减，而梯度下降收敛速度正比于特征值大小 → 特征值小（高频）的方向学得慢。

**实际意义**：
- 解释为什么网络不立刻记住噪声；隐式正则化偏向平滑解。
- positional encoding（NeRF 的 Fourier features）为何有效：把高频信息编码进低频通道，绕开偏置。

**反直觉推论**：对纯粹高频目标（如图像细节），宽网络可能收敛极慢——这是"spectral bias 是福也是祸"的一面。

## 2. 良性过拟合（Benign Overfitting）

**现象**：模型在训练集上插值（训练误差 0，完美记住噪声），但**测试泛化依然好**。经典 bias-variance 权衡预言插值应当灾难性过拟合，深度网络/核方法却经常不崩（Belkin et al. 2019 的 double descent 也是同族现象）。

**理论刻画**（线性回归/线性核设定，Bartlett et al. 2020, Tsigler-Bartlett 2020）：
- 泛化误差可精细分解。
- 良性发生的条件大致是：**噪声能量被摊薄到特征谱的许多小特征值方向上**——模型把噪声"记"在不影响预测主方向的维度里，如同把垃圾扫进不常用的抽屉。

**与频谱偏置的关系**：一体两面。
- 频谱偏置描述"先学什么"（低频优先）。
- 良性过拟合描述"把噪声塞到哪些慢学的方向"（噪声集中在小特征值方向 = 隐式正则化）。

## 3. NTK：lazy regime vs feature learning

**NTK（Neural Tangent Kernel, Jacot et al. 2018）**：无限宽网络在梯度下降下的动力学等价于**固定核**上的核回归。

### Lazy regime（惰性 / 核机制）
- 初始化处线性化：$f(\theta) \approx f(\theta_0) + \langle \nabla f(\theta_0), \theta - \theta_0 \rangle$
- 训练中权重几乎不动（$\|\theta-\theta_0\| \to 0$），只有输出端的等效线性组合在变。
- 网络行为等价于固定 NTK 的核回归。宽网络（宽度→∞）在适当缩放下自动落入此机制。
- 优点：可分析。缺点：**表达能力受限，本质没学新特征**。

### Feature learning regime（特征学习 / rich 机制）
- 权重显著移动，中间层表征被实质改变。
- 真正"学到特征"，是实践中深度网络成功、迁移学习强大的来源。NTK 分析在此失效。

### 关键：宽网络 ≠ 一定 lazy
- Chizat & Bach (2019)：**参数化方式（scaling）决定落入哪种机制**。同样的网络，输出缩放不同，无穷宽极限走完全不同的路。
- 这是 μP / Tensor Programs（Yang & Hu）的理论起点：调好缩放，超参才能从小模型迁移到大模型（hyperparameter transfer）。

## 4. 统一视角（与其它笔记的连接）

在 NTK/lazy 视角下，训练动力学由核的谱决定 → 核谱高频衰减 → 频谱偏置 → 噪声滞留慢方向 → 良性过拟合。

- 流形版本：数据在低维流形上时，"谱"应换成流形上的 Laplace 谱 → [[流形假说与 LLM 表征几何]]
- 动力学外推：当网络不再 lazy（真的在学特征），谱分析失效，需要新的工具。

## 关键文献

- Jacot, Gabriel, Hongler (2018): Neural Tangent Kernel
- Chizat & Bach (2019): On Lazy Training in Differentiable Programming
- Rahaman et al. (2019): On the Spectral Bias of Neural Networks
- Woodworth et al. (2020): Kernel and Rich Regimes in Overparametrized Models
- Bartlett, Long, Lugosi, Tsigler (2020): Benign overfitting in linear regression
- Tsigler & Bartlett (2020): Benign overfitting in ridge regression
- Yang & Hu: Tensor Programs / μP

## 相关笔记

- [[深度学习理论 索引]]
- [[流形假说与 LLM 表征几何]]
- [[Looped Transformer 与 CoT]]
