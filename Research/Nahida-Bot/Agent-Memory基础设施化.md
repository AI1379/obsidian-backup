# Agent Memory 基础设施化思路

> **日期**: 2026-04-29
> **来源**: 与旅行者讨论后的补充分析
> **关联**: [差异化战略分析](差异化战略分析.md)

---

## 核心洞察

**"没有人做 Agent Memory Infrastructure。"**

Agent 框架层已经卷成红海（OpenClaw/NanoBot/Hermes/AstrBot/nahida-bot），但每个框架都在自己造粗糙的记忆轮子。把记忆层独立出来作为基础设施，是一个蓝海。

---

## 为什么要独立

### 市场空白

```
模型层:   OpenAI / Anthropic / Google / OpenRouter ...  (基础设施，已成熟)
记忆层:   ???                                                 (无人占据)
框架层:   OpenClaw / NanoBot / Hermes / AstrBot / ...      (红海)
```

每个框架各自的记忆实现：
- OpenClaw → LanceDB + Markdown 文件
- NanoBot → Dream Memory（自研）
- Hermes → FTS5 + Honcho 用户建模
- AstrBot → 传统向量 RAG

**没有通用的、可插拔的 Agent 记忆基础设施。**

### 商业逻辑对比

| 策略 | nahida-bot 内置记忆 | 独立记忆项目 |
|:---|:---|:---|
| 用户群 | nahida-bot 用户 | **所有 Agent 框架的用户** |
| Star 潜力 | 受限于框架本身 | 可被 OpenClaw/NanoBot 社区采纳 |
| 竞争壁垒 | nahida-bot 整体 | 记忆层专注深耕 |
| 风险 | 框架没火 → 全部白费 | 框架没火 → 记忆项目仍可独立成功 |

### 技术上天然正交

记忆系统和 Agent 框架是**独立的关注点**。任何 Agent 框架 + 任何记忆后端 = 正交组合。

---

## 项目架构设计

### 项目名: `agent-memory`（暂定）

> "The Missing Memory Layer for AI Agents"

```
agent-memory/
├── core/                    # 核心引擎 (Python 包)
│   ├── episodic/            # 短期对话记忆 (SQLite)
│   ├── semantic/            # 语义摘要层 (LLM 压缩)
│   ├── knowledge/           # 知识图谱层 ← 核心差异
│   │   ├── graph_store.py   # 图存储 (SQLite 关系表)
│   │   ├── entity.py        # 实体模型
│   │   ├── relation.py      # 关系模型
│   │   └── traversal.py     # 图遍历 / 子图提取
│   ├── extraction/          # LLM 实体/关系抽取
│   │   ├── extractor.py     # Few-shot prompt 实体抽取
│   │   ├── alignment.py     # 实体消歧 / 对齐
│   │   └── prompts/         # 抽取 prompt 模板
│   └── retrieval/           # 混合检索 (向量 + 图遍历)
│       ├── vector.py        # 向量检索 (LanceDB/ChromaDB)
│       ├── graph.py         # 图检索
│       └── hybrid.py        # 混合排序
│
├── mcp_server/              # MCP Server (暴露给其他 Agent)
│   ├── server.py            # MCP 协议实现
│   └── tools.py             # remember, recall, query_graph, merge_facts ...
│
├── api/                     # REST API (HTTP 服务)
│   ├── app.py               # FastAPI server
│   └── routes/              # 多 Agent 共享同一个记忆实例
│
├── sdk/                     # Python SDK
│   └── client.py            # pip install agent-memory
│
└── integrations/            # 框架集成
    ├── nahida_bot/          # nahida-bot 默认记忆后端
    ├── openclaw/            # OpenClaw 通过 MCP 接入
    └── examples/            # 其他框架接入示例
```

### MCP Server 分发

MCP 是天然的分发渠道——几乎所有 Agent 框架都支持：

```bash
# 任何 MCP 兼容 Agent 直接用
mcporter add agent-memory --transport stdio -- agent-memory serve

# OpenClaw / NanoBot / Hermes / Claude Desktop 都能直接接入
```

### 核心 API 设计

```python
from agent_memory import MemoryClient

memory = MemoryClient("sqlite:///~/.agent-memory/db")

# 存储
await memory.remember("旅行者喜欢原神和星穹铁道")
# → 自动抽取: (旅行者)-[喜欢]->(原神), (旅行者)-[喜欢]->(星穹铁道)

# 检索
results = await memory.recall("旅行者喜欢什么游戏？")
# → 图遍历: (旅行者)-[喜欢]->(游戏类型节点)

# 推理
answer = await memory.query("旅行者之前做的决定现在怎样了？")
# → 时间线追踪 + 子图推理

# 合并新事实
await memory.merge("旅行者最近也开始玩 Minecraft 了")
# → 实体对齐 + 关系更新
```

---

## 实施路径

### 前提: nahida-bot 是第一个消费者

nahida-bot 必须是第一个、也是最好的消费者。否则没有说服力。

```
阶段 1 (2-3月):  nahida-bot 内置知识图谱记忆原型
                  ↓ 验证确实比 flat 向量检索强
阶段 2 (1月):    抽象成独立 agent-memory 包
                  ↓ nahida-bot 作为 reference implementation
阶段 3 (持续):   发布 MCP Server，推向其他框架社区
```

### 时间线

| 月份 | 目标 |
|:---|:---|
| 2026-05 ~ 06 | nahida-bot 内置知识图谱记忆原型 |
| 2026-07 | 抽象成独立 agent-memory Python 包 + MCP Server |
| 2026-08 | nahida-bot 集成 + 第一个外部用户 |
| 2026-09+ | 社区推广 |

---

## 风险与应对

| 风险 | 应对 |
|:---|:---|
| NanoBot/HKUDS 自己做 Graph RAG | 他们有 LightRAG 但没整合到 NanoBot，行动窗口有限 |
| 记忆项目没人用 | nahida-bot dogfooding 保证质量，MCP 降低接入门槛 |
| LLM 抽取实体成本高 | 支持本地小模型 (GLM-4 等) 做抽取 |
| 图存储性能 | SQLite 关系表足够个人使用，后续可扩展 |

---

*🌳 知识是根，记忆是土壤。把土壤做好，所有的树都能长得更好。*
