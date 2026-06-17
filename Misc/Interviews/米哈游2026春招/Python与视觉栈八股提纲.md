---
tags:
  - 面试
  - 简历
  - Python
  - PyTorch
  - OpenCV
  - YOLO
---

# Python 与视觉栈八股提纲

这一份不只写 `Torch/OpenCV/YOLO`，还把 Python 语言层一起补上。  
原因很现实：只要你简历上写了这些视觉训练栈，面试官很容易顺手问 Python 基础。

---

## 一、Python 语言层主线

建议先守住这几条：

1. 可变 / 不可变对象
2. 引用语义、浅拷贝、深拷贝
3. `list/tuple/dict/set`
4. 迭代器、生成器
5. 装饰器、上下文管理器
6. GIL、线程和多进程

---

## 二、Python 基础高频点

### 1. 可变和不可变

- 常见不可变：`int`、`str`、`tuple`
- 常见可变：`list`、`dict`、`set`

高频追问：

- 为什么默认参数用可变对象会有坑

核心原因是：

- 默认参数在函数定义时求值
- 可变对象会被后续调用共享

### 2. 浅拷贝和深拷贝

- 浅拷贝只复制最外层对象
- 深拷贝会递归复制内部对象

这题经常和数据预处理、配置对象、样本增强管线一起问。

### 3. `list`、`tuple`、`dict`、`set`

你至少要知道：

- `list` 有序可变
- `tuple` 有序不可变
- `dict` 键值映射
- `set` 去重和集合运算

### 4. 迭代器和生成器

- 迭代器实现的是“按需取下一个值”
- 生成器是写法更方便的迭代器构造方式
- `yield` 是高频关键词

### 5. 装饰器

直觉上：

- 不改原函数核心逻辑的前提下，为其增加包装行为
- 常见于日志、计时、权限、缓存

### 6. 上下文管理器

- `with` 语句背后是上下文管理协议
- 核心目的是可靠地获取和释放资源
- 和 C++ 的 RAII 在思想上有共通处

### 7. GIL

高频但容易讲飘。

稳一点的说法：

- CPython 有 GIL
- 它会影响多线程同时执行 Python 字节码
- IO 密集型线程仍然有用
- CPU 密集型任务通常更考虑多进程或把重计算交给 C/CUDA 扩展

---

## 三、Python 并发和工程基础

### 1. 多线程 vs 多进程

- 线程更轻量，适合 IO 场景
- 进程隔离更强，适合 CPU 密集任务
- Python 下讨论线程，常会被追问到 GIL

### 2. 包管理和环境

至少知道：

- `venv/conda`
- `pip`
- requirements 或环境锁定的重要性

视觉训练项目里，环境复现经常是真问题，不是小事。

### 3. Numpy 和张量直觉

即使主战场是 PyTorch，也最好会说：

- `numpy.ndarray` 是 CPU 侧常见基础数组
- `torch.Tensor` 和它很像，但支持 device、autograd、深度学习算子生态

---

## 四、PyTorch / Torch

### 1. 主线

PyTorch 面试高频主线：

1. `Tensor`
2. `autograd`
3. `nn.Module`
4. 训练循环
5. loss / optimizer / scheduler
6. `train()` / `eval()` / `no_grad()`
7. checkpoint
8. DDP / AMP

### 2. Tensor

至少会说：

- `shape`
- `dtype`
- `device`
- `requires_grad`
- `view/reshape`
- `permute/transpose`
- broadcasting

高频坑点：

- `contiguous` 的直觉
- `HWC` 和 `CHW` 转换
- CPU tensor 和 CUDA tensor 的差异

### 3. autograd

- 前向时构建计算图
- `loss.backward()` 回传梯度
- 梯度默认累积在 `.grad`
- `optimizer.zero_grad()` 是为了清掉上一轮累积

### 4. `nn.Module`

- 统一模型结构
- 自动注册参数
- 支持模式切换
- 支持 `state_dict`

### 5. 标准训练流程

1. 准备 `Dataset/DataLoader`
2. `model.train()`
3. 前向
4. 计算 loss
5. `zero_grad`
6. `backward`
7. `step`
8. 验证、记录指标、保存 checkpoint

### 6. `train()` vs `eval()` vs `no_grad()`

- `train()` 让模型处于训练模式
- `eval()` 切换模块行为，影响 `Dropout/BatchNorm`
- `no_grad()` 不记录计算图

最常见翻车点：

- 把 `eval()` 说成“关闭梯度”

### 7. optimizer 和 scheduler

至少知道：

- `SGD`
- `Adam`
- `AdamW`
- `StepLR`
- `Cosine`
- warmup

### 8. checkpoint

断点续训通常不只存模型参数，还会存：

- optimizer 状态
- scheduler 状态
- epoch / step
- 最优指标

### 9. DDP 和 AMP

- `DDP` 是更主流的多卡方案
- `AMP` 目标是提速和省显存
- `GradScaler` 常用于降低半精度梯度下溢风险

### 10. 项目排障口径

训练不正常时，优先排查：

1. 数据和标签是否正确
2. loss 是否合理下降
3. 学习率是否离谱
4. `train/eval` 是否切对
5. 是否忘了 `zero_grad`
6. 是否误用增强或前处理不一致

---

## 五、OpenCV

### 1. 基础点

至少要稳住：

- 图像本质上是矩阵
- OpenCV 默认 `BGR`
- 灰度图是单通道
- 常见输入形状是 `HWC`

### 2. 高频操作

- `imread/imwrite`
- `resize`
- `crop`
- `cvtColor`
- normalize
- threshold

### 3. 插值

- 最近邻
- 双线性
- 双三次
- `INTER_AREA`

### 4. 滤波

- 均值
- 高斯
- 中值
- 双边

### 5. 边缘与形态学

- `Canny`
- 腐蚀
- 膨胀
- 开运算
- 闭运算

### 6. 轮廓和几何

- `findContours`
- 外接框
- 最小外接矩形
- 仿射变换
- 透视变换

### 7. 项目里怎么说

- “OpenCV 在我的项目里主要负责图像读写、预处理、颜色转换、resize、可视化和简单后处理。”
- “如果是传统 CV 流程，我会用滤波、边缘、形态学和轮廓做辅助处理。”

---

## 六、YOLO

### 1. 主线

1. one-stage 检测器
2. bbox 回归和分类一起做
3. NMS 后处理
4. precision / recall / mAP / IoU
5. 数据增强、误检漏检分析、部署一致性

### 2. one-stage vs two-stage

- two-stage：先 proposal，再分类回归
- one-stage：直接做密集预测

YOLO 常被说快，核心就是链路更直接。

### 3. 必答概念

- bbox 表示：`x1,y1,x2,y2` 或 `cx,cy,w,h`
- IoU：框重叠程度
- NMS：压掉重复框
- precision / recall / mAP：检测效果评价

### 4. 数据增强

常见：

- flip
- random crop
- scale
- color jitter
- mosaic
- mixup

高频追问：

- 检测任务做增强时，bbox 标签也要同步变换

### 5. 误检和漏检

误检多常看：

- 阈值太低
- 负样本不够
- 背景模式学歪
- 标注噪声

漏检多常看：

- 小目标
- 遮挡
- 输入分辨率不够
- 数据覆盖不足
- 阈值太高

### 6. 部署口径

- 前处理一致性
- 输入尺寸
- 阈值和 NMS 参数
- ONNX / TensorRT 导出
- 离线和线上数值一致性

---

## 七、把 Python、Torch、OpenCV、YOLO 串成一条链

你可以统一这样说：

- “我的典型链路是：用 Python 组织训练脚本和数据流程，用 OpenCV 做图像读写和基础预处理，用 PyTorch 完成训练验证和 checkpoint 管理，用 YOLO 负责检测任务本身，最后再关注推理部署和线上效果一致性。”

这句话的好处是：

- 逻辑完整
- 不装底层专家
- 听起来像真的做过完整流程

---

## 八、容易翻车的点

1. Python 默认参数可变对象坑
2. GIL 只会背名字，不知道它对线程意味着什么
3. 把 `eval()` 说成“关闭梯度”
4. 不知道 checkpoint 为什么要存 optimizer 状态
5. 不知道 OpenCV 默认 `BGR`
6. 说不清 NMS 和 mAP
7. 训练效果不好时，只会说“调参”，没有排障顺序

---

## 九、最小防守口径

- Python 语言层我能讲清常见容器、拷贝语义、生成器、上下文管理器和 GIL 的基本影响。
- PyTorch 我能讲清训练和验证主流程、常见模块和排障思路。
- OpenCV 和 YOLO 我能讲清常见图像处理、检测任务核心概念和工程落地问题。
