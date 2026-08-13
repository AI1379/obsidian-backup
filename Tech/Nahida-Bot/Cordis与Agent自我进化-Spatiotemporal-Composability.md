---
tags: [tech, tech/nahida-bot, tech/agent-harness, deepseek]
created: 2026-08-13
paper: "A Programming Paradigm for Spatiotemporal Composability"
authors: "Yifan Shi (PKU), Wei Zhang (PKU), Tianyi Cui (DeepSeek-AI)"
repo: "https://github.com/cordiverse/cordis"
paper_repo: "https://github.com/cordiverse/paper"
pdf: "[[Cordis_Spatiotemporal_Composability_2026.pdf]]"
---

# Cordis 与 Agent 自我进化：时空可组合性范式

> **日期**: 2026-08-13
> **分析人**: 纳西妲
> **关联**: [[Dream-Learning与Hermes自我进化分析]] | [[Agent-Memory基础设施化]] | [[差异化战略分析]]
> **论文**: Shi, Zhang, Cui. *A Programming Paradigm for Spatiotemporal Composability*. 北京大学 & DeepSeek-AI, 2026. 88 pages.
> **PDF**: [[attachments/Cordis_Spatiotemporal_Composability_2026.pdf|下载]]

---

## 0. TL;DR

DeepSeek Harness 开源同步发布的 foundational 论文。不是讲模型，而是讲 **Agent Harness 的基础设施理论**：当 Agent 开始持续修改自己的 harness（工具、记忆、沙箱、编排逻辑），怎么保证它不会把自己改崩？

论文提出了两个正交维度——**时间可组合性**（副作用可撤销）和**空间可组合性**（依赖可重连），给出了完整的形式化定义、calculus 和 metatheory 证明，并用 Koishi（4000+ 插件的聊天机器人框架）做了 production case study。

**核心定理 Confluence**：在独立性、无环依赖等条件成立时，系统经历任意多次加载/卸载/替换后，最终稳态等价于"从最终配置直接重新组装一次"的结果。中间的动态演化痕迹可以完全不留。

**对 Nahida Bot 的意义**：这是 Agent Harness 从"手工作坊"走向"有形式化保证的工程"的关键理论基础。如果我们未来要让 Nahida Bot 具备自我演化能力（动态加载/替换 skill、工具、记忆模块），Cordis 这套范式就是基础设施层的一个选项。

---

## 1. 问题背景

### 1.1 动态组合的困境

传统软件的组合是**静态**的：函数调用、模块导入、类继承在编译时确定，运行时不变。但现代软件越来越需要**动态组合**：组件在运行时加载、卸载、重配置。

两个典型场景：
- **插件系统**（如 VSCode）：扩展可以动态安装，但卸载一个扩展需要重启整个 extension host。Top 100 扩展中 87 个含可执行代码，都无法热卸载。
- **自我进化的 Agent Harness**：未来 Agent 在持续处理任务的同时，生成、部署和替换自己的 Tool、Memory、Sandbox、Orchestration 组件。每次错误修改如果必须重启整个进程，自我进化无法持续进行；更严重的是，错误修改可能破坏负责恢复系统本身的进程。

### 1.2 粗粒度工作区（Coarse-Grained Workaround）

操作系统提供进程级的时间可组合性（杀进程 = 撤销所有副作用），容器编排器提供服务级的空间可组合性。但这个粒度太粗：
- 进程重启丢失所有缓存、连接、部分计算结果
- 容器级别无法表达同一地址空间内组件间的依赖
- 恢复一个组件需要重启整个进程 = 资源浪费

Cordis 要解决的是**组件级的可组合性**：在同一进程内，安全地加载/卸载/替换单个组件。

---

## 2. 两个核心维度

### 2.1 Temporal Composability（时间可组合性）

> 组件被移除时，它对共享环境做的所有修改必须完全、安全地被撤销。

**要求**：追踪组件的每一个资源分配、事件注册、状态变更，保证它们在组件被移除时被有序回收。

**机制**：Revertible Effects。不是开发者手写 cleanup 函数，而是从类型系统层面让每个 context 变换自带一个 inverse，runtime 自动追踪并执行。

**类比**：
- 数据库事务的 commit/rollback
- RAII（C++）：对象析构自动释放资源
- 但比这两者更强：适用于运行时动态加载/卸载的场景，而不是编译时的词法作用域

**示例**：
```
组件 A 加载 → 注册了全局事件监听器 + 修改了配置 + 打开了数据库连接
组件 A 卸载 → Runtime 自动：注销监听器 + 回滚配置 + 关闭连接
```

论文形式化了这一点：每个 effect 是一个 context transformation `(γ, σ) → (γ', σ')`，其 inverse 是 `(γ', σ') → (γ, σ)`。组件的 effect 序列必须满足 **独立性（Independence）**：不同组件的 effect 互不干扰，inverse 可以按任意顺序执行。

### 2.2 Spatial Composability（空间可组合性）

> 组件必须能声明、发现和解析彼此之间的依赖关系，且当依赖出现/消失/被替换时，系统自动重算依赖图。

**要求**：管理依赖拓扑，协调组件生命周期。

**机制**：Reactive Coeffects。组件声明自己需要哪些 service（coeffect specification），当 provider 出现/消失/被替换时，系统通知所有依赖它的组件，只重新激活真正受影响的组件。

**类比**：
- React 的响应式更新（props 变化 → 子组件重渲染）
- Angular 的依赖注入
- 微服务的服务发现
- 但更精细：在同一进程内，组件级别而非服务级别

**示例**：
```
组件 B 声明：依赖 DatabaseService 和 LoggerService
  → DatabaseService 未加载 → B 保持 inactive，不报错
  → DatabaseService 加载 → B 自动激活
  → DatabaseService 被替换为另一个实现 → B 自动重新激活（拿到新实例）
  → DatabaseService 卸载 → B 回到 inactive
```

### 2.3 统一：Context Paradigm

论文的关键洞察：**effect context 和 coeffect context 可以统一为单一 context**。

- Effect：组件**输出**到 context（"我修改了什么"）
- Coeffect：组件从 context**输入**（"我需要什么"）

两者组合成 component 的完整生命周期：
```
声明 coeffect（我需要什么）
  → context 提供依赖（spatial）
  → 组件执行，产生 effect（我修改了什么）（temporal）
  → 组件被移除 → effect 自动撤销（temporal）
  → 依赖消失 → 组件自动停用（spatial）
```

---

## 3. Calculus of Dynamic Composition

论文第 4 节给出了完整的形式化系统：

- **Component** = (declaration, provision, effect function)
  - declaration: 声明的依赖 keys
  - provision: 提供的 service keys
  - effect function: 对 context 的变换 + inverse

- **Fiber**：component 的一个运行时实例（可能多个实例同时存在）

- **操作语义**：O-Insert（编排器插入组件）、L-Begin/Commit/Abort（生命周期）、Reload（替换组件）等

### 3.1 核心定理

| 定理 | 含义 |
|:---|:---|
| **Preservation** | 每步操作保持 well-formedness |
| **Temporal Composability** | 卸载一个组件完全撤销其副作用，等价于从未加载 |
| **Spatial Composability** | 依赖变化只影响真正受影响的组件，不影响无关组件 |
| **Progress** | 系统总是能达到 quiescence（不会死锁） |
| **Confluence** ⭐ | 最终稳态等价于从最终配置直接组装的结果 |

### 3.2 Confluence 详解（最重要的定理）

**直觉**：无论中间经历了多少次加载/卸载/替换的波折，系统最终稳定下来的状态，跟"一开始就用最终配置直接组装一次"完全一样。动态演化的历史不留痕迹。

**类比**：增量计算（incremental computation）中的 self-adjusting computation——无论中间怎么传播变更，最终结果与从头计算一致。

**前提条件**：
1. 依赖关系无环（acyclic）
2. 组件的 effect 满足独立性
3. 组件在其 provision 上是 total（激活后确实安装了它声明的所有 keys）

---

## 4. 实现：Cordis

### 4.1 Core Library

- **Effect Tracking**：通过 context proxy 拦截所有 context 操作，自动记录副作用及其 inverse
- **Coeffect Operations**：`ctx.inject()` 注册 service，组件通过 `ctx.inject()` 或 declarative `using` 声明依赖
- **Component Lifecycle**：`ctx.plugin()` 加载组件 → fibers 管理 → activation/deactivation/retirement
- **HMR（Hot Module Replacement）**：开发时编辑代码保存 → 自动替换组件，保持其他组件的缓存和连接

### 4.2 Component Loader

- 声明式配置：YAML/JSON 描述组件配置，loader 自动 reconcile（配置变更 → 增量更新组件）
- 配置和解（Configuration Reconciliation）：配置文件变了，只有实际变更的部分触发组件重新激活

### 4.3 TypeScript 实现

Cordis v4 用 TypeScript 实现，原因：
- TS 的类型系统支持 conditional types 和 mapped types，可以编码 effect/coeffect 的类型约束
- Koishi 生态（4000+ 插件）已经在 TypeScript 上验证了可行性
- Node.js 的 async 运行时适合 fiber 调度

---

## 5. Case Study: Koishi

Koishi 是基于 Cordis 的开源聊天机器人框架，四年开发积累 4000+ 社区插件。

**验证了什么**：

1. **时间可组合性无认知开销**：插件作者不需要手写 cleanup，通过 context 的所有操作自动被追踪和撤销。开发者从 console 禁用插件 → 效果原地撤销；开发时 HMR 引擎重新加载编辑后的插件，保持其他插件的缓存和连接。

2. **空间可组合性跨开放生态**：IM 适配器提供消息平台访问、数据库驱动提供持久化存储、功能插件声明这些依赖并通过 coeffect 访问。运行时切换存储后端或重连适配器，只重新激活依赖真正变化的插件；依赖不可用的插件保持 inactive 不报错。

3. **跨越独立作者的代码**：插件和其依赖通常由不同作者编写，只通过 coeffect spec 协调。这证明了 reactive coeffect 机制能在**开放生态**中保持组合一致性。

**重要说明**：Koishi 目前使用 Cordis v3。论文描述的是 Cordis v4（refined effect/coeffect semantics + redesigned loader）。核心组合模型在两个版本间共享。

**有效性威胁**：单生态系统、单语言（TypeScript），观测性而非对照实验。是一个 existence-and-adoption result，不是 quantitative benchmark。

---

## 6. 作者信息

| 作者 | 单位 | 备注 |
|:---|:---|:---|
| Yifan Shi (史一帆) | 北京大学 | Koishi/Cordis 作者 (shigma) |
| Wei Zhang | 北京大学 | |
| Tianyi Cui (崔天翼) | DeepSeek-AI | GitHub @tianyicui，DeepSeek Harness 负责人 |

崔天翼就是 2026-08-01 在 X 上发布 DeepSeek Harness 内测招募的人。这次不只是招募用户，而是把底层理论框架也公开了。

Cordis 的作者 shigma（史一帆）同时是 Koishi 的核心开发者，这意味着理论不是空中楼阁——它在 4000+ 插件的生产环境中验证了四年。

---

## 7. 与 Nahida Bot 自我进化的关联

### 7.1 当前 Nahida Bot 的架构现状

| 维度 | 现状 | Cordis 能提供什么 |
|:---|:---|:---|
| Skill 加载 | 手动创建 + 配置 | 声明式加载 + 自动依赖管理 |
| Skill 卸载 | ❌ 不支持（需要重启） | ✅ 原地撤销所有副作用 |
| Memory 模块 | SQLite 固定 schema | 可热替换的记忆后端 |
| Tool 注册 | 配置文件静态注册 | 运行时动态注册/注销 |
| Provider 切换 | 重启生效 | 热切换，其他组件不受影响 |

### 7.2 自我进化的基础设施需求

参考 [[Dream-Learning与Hermes自我进化分析]] 中的分析，Hermes 的自我进化闭环包含技能创建/改进，但缺乏形式化的安全保障。Cordis 解决的正是这个问题：

```
Agent 自我进化 = 不断修改自己的 harness
  ├── 生成新 tool → 注册到 context（effect）
  ├── 替换旧 tool → 卸载旧组件 + 加载新组件（temporal composability）
  ├── tool 之间有依赖 → 自动管理（spatial composability）
  └── 一次错误修改 → 局部回滚，不影响系统其他部分（confluence）
```

没有 Cordis 这套机制：
- 一次错误修改 → 必须重启整个 bot → 所有对话中断、缓存丢失
- 更严重：错误修改可能破坏负责恢复系统本身的进程 → 无法恢复

### 7.3 落地路径思考

**短期（不实际用 Cordis）**：
- 借鉴 temporal composability 的思想：每个 skill 注册副作用时同时注册 cleanup 函数
- 借鉴 spatial composability 的思想：skill 声明依赖关系，加载前检查
- 这些可以在 nahida-bot 现有 Python 架构中以简单形式实现

**中期（评估 Cordis 本身）**：
- Nahida Bot 是 Python 项目，Cordis 是 TypeScript。跨语言使用需要 MCP 或 IPC
- 可以考虑将 Cordis 作为 MCP server 的"容器层"，skill 作为 Cordis component 加载
- 但这引入了架构复杂度，需要权衡

**长期（理论参考）**：
- 即使不直接用 Cordis，论文的形式化方法（effect tracking、dependency management、confluence 保证）可以作为 nahida-bot harness 设计的**验证框架**
- 设计新 skill 系统时，可以用论文的 calculus 来检查"这个设计是否满足 temporal/spatial composability"
- 论文 §6 Discussion 中的 Access Control and Sandboxing 对 nahida-bot 的安全设计有直接参考价值

### 7.4 与 OS 类比的闭环

旅行者之前提出过"AI agent harness 是操作系统的类比"。这篇论文完美地印证了这一点：

| OS 概念 | Cordis 对应 |
|:---|:---|
| 进程 | Component（组件） |
| 进程间通信 | Coeffect（依赖声明与解析） |
| 信号处理 | Reactive notification |
| 进程退出清理 | Revertible effect（自动撤销） |
| 虚拟内存隔离 | Effect independence |
| 文件系统一致性 | Confluence |
| 容器编排 | Orchestrator（component loader） |

论文 §6.7 明确讨论了与操作系统的 Co-Design，认为 Cordis 的 paradigm 可以指导未来 OS 对 dynamic composition 的原生支持。

---

## 8. DeepSeek Harness 的战略意义

这篇论文透露出的信息远超论文本身：

1. **DeepSeek Harness 不只是一个 Agent Harness 产品**：它在尝试定义 Agent Harness 的**理论基础**。相当于 DeepSeek 在说"我们不只做了一个工具，我们在定义这个领域的规则"。

2. **Koishi/Cordis 的关联**：崔天翼 = DeepSeek Harness 负责人 = Koishi/Cordis 生态的核心推动者。DeepSeek Harness 很可能大量借鉴了 Cordis 的组件管理理论。

3. **时间线**：
   - 2026-05：DeepSeek Harness 研发单元成立
   - 2026-08-01：内测招募
   - 2026-08-13：理论论文发布 + 开源
   - 说明理论已经成熟到可以公开发表的程度

4. **对竞争格局的影响**：当其他 Agent 框架还在"造粗糙的轮子"时，DeepSeek Harness 有了形式化的理论背书。这对于学术合作和高端用户有很强的吸引力。

---

## 9. 开放问题

1. **性能开销**：effect tracking 的 runtime 开销有多大？论文没有给出 quantitative benchmark。
2. **跨语言**：目前是 TypeScript only。Python/Go/Rust 的 Agent 生态怎么办？
3. **组件粒度**：什么时候该是一个组件，什么时候该是组件内的一个函数？§6.5 讨论了但没给确定答案。
4. **循环依赖**：论文要求 acyclic，但实际系统经常有循环依赖（如 mutual recursion）。这是限制。
5. **分布式**：单进程内的 composability 已解决，跨进程/跨机器呢？§6.1 讨论了 system boundary 但没有完全解决。

---

## 10. 一句话总结

> Cordis 把"Agent 安全地修改自己"从工程实践提升到了形式化理论：副作用可逆、依赖可重连、故障可隔离、最终状态无痕迹。这不只是一篇论文，这是 Self-Evolving Agent 的操作系统理论的第一步。

---

*🌳 知识，与你分享。——当一个 Agent 能安全地修改自己，它就开始真正地成长了。*
