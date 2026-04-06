# OpenClaw 核心架构深度分析报告

> **分析日期**: 2026-04-06  
> **目标**: 提炼 OpenClaw 的核心设计模式，作为新项目架构设计的参考输入。

---

## 目录

1. [Workspace 设计](#1-workspace-设计)
2. [Gateway-Node 架构](#2-gateway-node-架构)
3. [插件与 Channel 系统](#3-插件与-channel-系统)
4. [Agent 系统](#4-agent-系统)
5. [跨领域设计模式总结](#5-跨领域设计模式总结)
6. [可简化项与必保留项](#6-可简化项与必保留项)

---

## 1. Workspace 设计

### 1.1 核心概念

Workspace 是 Agent 的「家」——唯一的文件工作目录，同时也是 Agent 的持久化记忆存储。它**不是硬沙箱**，而是默认的 cwd（当前工作目录），工具以 workspace 为基准解析相对路径。

```
~/.openclaw/workspace/           ← Agent Workspace（可 Git 备份）
├── AGENTS.md                    ← 操作指令与行为规范
├── SOUL.md                      ← 人格、语气、边界
├── USER.md                      ← 用户档案
├── IDENTITY.md                  ← Agent 身份（名称/表情/风格）
├── TOOLS.md                     ← 本地工具笔记（不影响工具可用性）
├── HEARTBEAT.md                 ← 心跳检测清单（可选）
├── BOOT.md                      ← 启动检查清单（可选）
├── BOOTSTRAP.md                 ← 首次运行仪式（一次性）
├── MEMORY.md                    ← 整理后的长期记忆
├── memory/
│   └── YYYY-MM-DD.md            ← 每日记忆日志
├── skills/                      ← 工作区级技能（最高优先级）
└── canvas/                      ← Canvas UI 文件（可选）

~/.openclaw/                     ← 状态目录（不可提交到 Git）
├── openclaw.json                ← 配置文件
├── credentials/                 ← 渠道/提供商凭据
├── agents/<agentId>/
│   ├── agent/auth-profiles.json ← 模型认证配置
│   └── sessions/                ← 会话 JSONL 文件
├── skills/                      ← 托管技能
└── nodes/                       ← 节点配对状态
```

### 1.2 上下文注入机制

**数据流**：Workspace 文件 → Bootstrap 注入 → System Prompt → 模型上下文

1. **会话首次 turn**：OpenClaw 读取 workspace 中的 bootstrap 文件（AGENTS.md、SOUL.md、USER.md 等）
2. **空文件跳过**，大文件截断（默认上限 20000 字符/文件，总计 150000 字符）
3. **缺失文件**注入 "missing file" 标记，不中断运行
4. 文件内容直接嵌入 System Prompt，作为模型的"长期知识"

### 1.3 技能加载优先级

```
workspace/skills/          → 最高优先级
project agent skills/      → 项目级
personal agent skills/     → 个人级
~/.openclaw/skills/        → 托管级
bundled skills/            → 内置级
extraDirs                  → 额外目录
```

### 1.4 可复用设计模式

| 模式 | 描述 | 级别 |
|------|------|------|
| **约定优于配置的文件结构** | 固定文件名（AGENTS.md/SOUL.md/USER.md）映射固定语义，无需额外配置 | ⭐ 必须保留 |
| **文件即记忆** | Markdown 文件作为持久化存储，Git 友好、可审计、可迁移 | ⭐ 必须保留 |
| **降级容错** | 文件缺失不崩溃，注入标记继续运行；大文件截断而非报错 | ⭐ 必须保留 |
| **分层优先级解析** | 技能/配置从最高到最低优先级逐级覆盖 | ⭐ 必须保留 |
| **Workspace 与 State 分离** | 可版本控制的内容 vs 敏感凭据/会话分离存储 | ⭐ 必须保留 |
| **可选沙箱** | 默认非沙箱，可按需启用 sandbox + workspaceAccess 控制 | 🔧 可简化 |

---

## 2. Gateway-Node 架构

### 2.1 核心拓扑

```
                    ┌──────────────────────────────────┐
                    │         Gateway (Daemon)          │
                    │  单进程、长运行、所有消息面的Owner   │
                    │                                   │
                    │  ┌─────────────────────────────┐  │
                    │  │   WebSocket Server (:18789)   │  │
                    │  │   + HTTP API (同端口)         │  │
                    │  └──────┬──────┬──────┬────────┘  │
                    │         │      │      │           │
                    │    ┌────┘  ┌───┘  ┌───┘           │
                    └────┼──────┼──────┼───────────────┘
                         │      │      │
              ┌──────────┘      │      └──────────┐
              ▼                 ▼                  ▼
        ┌──────────┐    ┌──────────┐      ┌──────────┐
        │ Operator │    │ Operator │      │   Node   │
        │ (CLI/UI) │    │ (macOS)  │      │(iOS/Droid)│
        └──────────┘    └──────────┘      └──────────┘
              │                                    │
         role: operator                      role: node
         scopes: [read,write,admin]           caps: [camera,canvas,...]
                                              commands: [camera.snap,...]
```

**关键约束**：每个主机**仅一个 Gateway**；它是唯一拥有 WhatsApp 等渠道会话的进程。

### 2.2 通信协议

#### 传输层
- **WebSocket**，文本帧，JSON 负载
- 默认绑定 `ws://127.0.0.1:18789`（loopback-first）
- 同端口复用 HTTP API + WS + Canvas Host

#### 帧类型

```json
// 请求
{ "type": "req", "id": "...", "method": "...", "params": {} }

// 响应
{ "type": "res", "id": "...", "ok": true, "payload": {} }
{ "type": "res", "id": "...", "ok": false, "error": {} }

// 事件（服务端推送）
{ "type": "event", "event": "...", "payload": {}, "seq": 1, "stateVersion": 42 }
```

#### 握手流程

```
Gateway ──→ Client:  event:connect.challenge { nonce, ts }
Client  ──→ Gateway:  req:connect { role, scopes, caps, auth, device }
Gateway ──→ Client:  res:hello-ok { protocol, policy, auth.deviceToken }
```

- **首帧必须是 `connect`**，非 JSON 或非 connect 帧直接断开
- 客户端必须签名 `connect.challenge` nonce（v3 签名绑定 platform + deviceFamily）
- 协议版本协商：`minProtocol` / `maxProtocol`

#### 角色与权限模型

| 角色 | 职责 | 认证方式 |
|------|------|---------|
| **Operator** | 控制面客户端（CLI/UI/自动化） | token / password / Tailscale / trusted-proxy |
| **Node** | 能力宿主（摄像头/屏幕/Canvas） | 设备配对 + deviceToken |

Operator 作用域：`read`, `write`, `admin`, `approvals`, `pairing`, `talk.secrets`  
Node 声明：`caps`（能力类别）, `commands`（命令白名单）, `permissions`（细粒度开关）

### 2.3 节点发现与配对

#### 配对流程

```
Node ──connect──→ Gateway (role:node, device identity)
         │
         ▼
   Gateway 创建 pending request
         │
         ▼
   广播 node.pair.requested 事件
         │
         ▼
   Operator 审批 (CLI/UI)
         │
         ▼
   Gateway 签发 deviceToken
         │
         ▼
   Node 使用 token 重连 → paired
```

**关键特性**：
- 每设备密钥对身份（`device.id` = 指纹）
- 新设备需审批（loopback 可自动审批）
- 5分钟过期，幂等请求
- 审批始终生成新 token，token 不可扩展权限范围
- 2026.3.31+：**配对未完成则命令全部禁用**

#### 节点命令调用

```
Gateway ──node.invoke──→ Node (command + params)
Node    ──result──────→ Gateway (node.invoke.result)
```

- 网关维护全局命令策略（`allowCommands` / `denyCommands`）
- `system.run` 需额外审批（`operator.admin`）
- 离线节点使用持久化队列（`node.pending.enqueue/drain`）

### 2.4 OpenAI 兼容层

Gateway 在同端口暴露：
- `GET /v1/models` — Agent 优先模型列表
- `POST /v1/embeddings` — 嵌入向量
- `POST /v1/chat/completions` — 聊天补全
- `POST /v1/responses` — Agent-native 客户端

### 2.5 可复用设计模式

| 模式 | 描述 | 级别 |
|------|------|------|
| **单进程多协议复用** | HTTP + WS + Canvas 共用一个端口，减少部署复杂度 | ⭐ 必须保留 |
| **Loopback-first 安全模型** | 默认仅本地访问，远程需要显式认证 | ⭐ 必须保留 |
| **挑战-响应握手** | 服务端发 nonce，客户端签名，防重放攻击 | ⭐ 必须保留 |
| **角色-作用域分离** | Operator vs Node，不同角色不同权限集合 | ⭐ 必须保留 |
| **设备密钥对身份** | 持久化设备 ID = 公钥指纹，跨重连稳定 | ⭐ 必须保留 |
| **两阶段运行确认** | Agent 请求立即返回 `accepted`，流式事件后再返回 `ok/error` | ⭐ 必须保留 |
| **事件不回放** | 连接断开后事件丢失，客户端需主动刷新状态 | 🔧 可简化（新项目可考虑事件回放） |
| **配置热加载** | `hybrid` 模式：安全改动热加载，需重启的改动自动重启 | ⭐ 值得借鉴 |

---

## 3. 插件与 Channel 系统

### 3.1 Channel 概览

OpenClaw 支持 25+ 消息平台，每个 Channel 通过 Gateway 建立连接：

```
Telegram ──grammY──┐
WhatsApp ──Baileys─┤
Discord ──Bot API──┤──→ Gateway ──→ Agent Runtime
Slack ────Bolt SDK─┤
Signal ──signal-cli┤
IRC ────IRC client─┘
... (20+ 更多)
```

### 3.2 消息路由机制

**核心原则**：回复发往消息来源的 Channel，模型不选择 Channel，路由完全由配置决定。

#### 路由决策链

```
入站消息
   │
   ▼
1. Exact peer match (bindings + peer.kind + peer.id)
   │
   ▼
2. Parent peer match (线程继承)
   │
   ▼
3. Guild + roles match (Discord)
   │
   ▼
4. Guild match (Discord)
   │
   ▼
5. Team match (Slack)
   │
   ▼
6. Account match (accountId)
   │
   ▼
7. Channel match (任何 account, accountId: "*")
   │
   ▼
8. Default agent (agents.list[].default → first → main)
```

#### Session Key 生成规则

```
DM:     agent:<agentId>:<mainKey>                    ← 共享主会话
群组:   agent:<agentId>:<channel>:group:<id>
频道:   agent:<agentId>:<channel>:channel:<id>
线程:   ...追加 :thread:<threadId>
话题:   ...嵌入 :topic:<topicId>
```

#### 多 Agent 路由

```json5
{
  agents: {
    list: [
      { id: "support", workspace: "~/.openclaw/workspace-support" },
      { id: "main", default: true, workspace: "~/.openclaw/workspace" }
    ]
  },
  bindings: [
    { match: { channel: "slack", teamId: "T123" }, agentId: "support" },
    { match: { channel: "telegram", peer: { kind: "group", id: "-100123" } }, agentId: "support" }
  ]
}
```

每个 Agent 有独立的 workspace、session store、auth profiles。

### 3.3 插件系统

#### 插件类型

| 插件槽位 | 职责 |
|----------|------|
| `contextEngine` | 控制上下文组装、压缩、子代理生命周期 |
| `memory` | 搜索/检索记忆数据 |
| Channel 插件 | 添加新消息平台 |
| 普通插件 | 工具、Hook、RPC 方法扩展 |

#### 插件 Hook 系统

**生命周期 Hook**（在 Agent Loop 内执行）：

```
before_model_resolve     → 模型解析前（无 messages）
    │
    ▼
before_prompt_build      → 会话加载后（有 messages），注入 prependContext/systemPrompt
    │
    ▼
before_agent_reply       → LLM 调用前，可拦截返回合成回复
    │
    ▼
[LLM 推理 + 工具执行]
    │
    ▼
agent_end                → 运行完成，检查最终消息
```

**工具 Hook**：
- `before_tool_call` / `after_tool_call` — 拦截工具参数/结果，可 `block: true` 终止
- `before_install` — 检查安装请求，可阻止

**消息 Hook**：
- `message_received` / `message_sending` / `message_sent`

**网关 Hook**：
- `gateway_start` / `gateway_stop`
- `session_start` / `session_end`
- `before_compaction` / `after_compaction`

### 3.4 上下文引擎（Context Engine）

上下文引擎是可插拔的核心组件，控制模型每次运行时的上下文构建：

```
消息入站 → ingest()          → 存储/索引消息
             │
             ▼
模型运行前 → assemble()       → 返回 token 预算内的消息集合 + systemPromptAddition
             │
             ▼
上下文满   → compact()        → 摘要化旧消息
             │
             ▼
运行完成   → afterTurn()      → 持久化状态、触发后台压缩
```

**内置 Legacy 引擎**：
- ingest: no-op
- assemble: 透传（使用运行时现有的 sanitize → validate → limit 管线）
- compact: 委托给内置摘要压缩
- afterTurn: no-op

**插件引擎示例**（如 lossless-claw）可实现 DAG 摘要、向量检索等高级策略。

### 3.5 可复用设计模式

| 模式 | 描述 | 级别 |
|------|------|------|
| **确定性路由链** | 8 级优先级路由，从精确匹配到默认，完全配置驱动 | ⭐ 必须保留 |
| **Channel 抽象** | 统一的消息入站/出站接口，模型不感知 Channel | ⭐ 必须保留 |
| **Hook 管线** | 可插拔的预处理/后处理钩子，支持 block/cancel 终止传播 | ⭐ 必须保留 |
| **插件槽位** | 互斥的核心能力插槽（contextEngine、memory），一次一个 | ⭐ 必须保留 |
| **Session Key 结构化** | 层级式 key 设计，自然支持 DM/群组/线程/话题隔离 | ⭐ 必须保留 |
| **Broadcast Groups** | 同一 peer 可触发多个 agent 并行运行 | 🔧 可简化 |
| **ownsCompaction 分权** | 插件可声明 ownsCompaction=true 接管压缩策略 | 🔧 可简化 |

---

## 4. Agent 系统

### 4.1 Agent Loop 完整生命周期

```
┌──────────────────────────────────────────────────────┐
│                    Agent Loop                         │
│                                                       │
│  1. Intake                                            │
│     ├─ 接收入站消息 / RPC 请求                         │
│     ├─ 返回 { runId, acceptedAt }                     │
│     └─ 进入会话序列化队列                              │
│                                                       │
│  2. Session + Workspace Preparation                   │
│     ├─ 解析并创建 workspace                            │
│     ├─ 加载 skills 快照                               │
│     ├─ 注入 bootstrap/context 文件                     │
│     └─ 获取 session 写锁                              │
│                                                       │
│  3. Prompt Assembly                                   │
│     ├─ 构建 System Prompt（基础 + 技能 + bootstrap）   │
│     ├─ 调用 before_model_resolve Hook                 │
│     ├─ 调用 Context Engine assemble()                 │
│     ├─ 调用 before_prompt_build Hook                  │
│     └─ 执行 token 限制与 compaction 预留              │
│                                                       │
│  4. Model Inference                                   │
│     ├─ 调用 before_agent_reply Hook                   │
│     ├─ [LLM 推理] → 流式 assistant deltas             │
│     └─ 可选 reasoning stream                          │
│                                                       │
│  5. Tool Execution Loop                               │
│     ├─ 解析工具调用请求                                │
│     ├─ before_tool_call Hook → 可 block               │
│     ├─ 执行工具                                       │
│     ├─ after_tool_call Hook                           │
│     └─ 工具结果返回模型 → 继续推理或结束                │
│                                                       │
│  6. Reply Shaping + Delivery                          │
│     ├─ 组装最终回复（文本 + 工具摘要 + 错误）          │
│     ├─ 过滤 NO_REPLY / no_reply                       │
│     ├─ 去重 messaging tool 发送                       │
│     ├─ Block streaming 分块发送                       │
│     └─ Chat channel 投递                              │
│                                                       │
│  7. Compaction + Persistence                          │
│     ├─ 检查上下文窗口使用率                            │
│     ├─ 自动压缩（如果接近上限）                        │
│     ├─ Context Engine afterTurn()                     │
│     └─ agent_end Hook                                 │
│                                                       │
│  8. Final Response                                    │
│     └─ { status: ok|error, summary, usage }           │
└──────────────────────────────────────────────────────┘
```

### 4.2 队列与并发控制

- **Session Lane**：每个 sessionKey 串行化运行（同一会话不并发）
- **Global Lane**：可选的全局串行队列
- **Queue Mode**（消息渠道可选）：
  - `steer`：当前 turn 完成后注入排队的消息
  - `followup`：当前 turn 结束后以排队消息开始新 turn
  - `collect`：积累消息直到当前 turn 结束

### 4.3 上下文管理

#### 压缩（Compaction）

```
会话接近 token 上限
       │
       ▼
提醒 Agent 保存重要笔记到 memory 文件
       │
       ▼
旧消息 → LLM 摘要 → 写入 session transcript
       │
       ▼
保留近期消息不变
```

- **自动压缩**：默认开启，接近上限或模型返回 overflow 错误时触发
- **手动压缩**：`/compact [指令]` 可指导摘要方向
- **压缩模型**：可单独配置更强大的模型用于摘要
- **工具结果配对**：压缩分块时保持 assistant tool call 与 toolResult 配对

#### 修剪（Pruning）

| 维度 | 压缩 | 修剪 |
|------|------|------|
| 操作 | 摘要旧对话 | 截断旧工具输出 |
| 持久化 | 写入 transcript | 仅内存中，不持久化 |
| 范围 | 整个对话 | 仅工具结果 |

### 4.4 记忆系统

#### 内置记忆引擎

```
MEMORY.md + memory/*.md
       │
       ▼ 分块 (~400 tokens, 80 token overlap)
       │
       ▼
SQLite FTS5 (BM25) + Embedding Vectors
       │
       ▼
混合搜索（关键词 + 向量）
       │
       ▼
memory_search 工具调用
```

- **索引位置**：`~/.openclaw/memory/<agentId>.sqlite`
- **文件监控**：memory 文件变化触发 1.5s 去抖重新索引
- **嵌入提供商**：OpenAI / Gemini / Voyage / Mistral（自动检测）/ Ollama / Local
- **CJK 支持**：trigram 分词

#### 记忆读写流程

```
Agent Loop 启动
    │
    ├─→ 读取 MEMORY.md + memory/YYYY-MM-DD.md（当日 + 前一日）
    │
    ├─→ memory_search 语义搜索相关记忆
    │
    ├─→ [Agent 运行中] → 更新 SESSION-STATE.md / memory/YYYY-MM-DD.md
    │
    └─→ [运行结束] → Context Engine afterTurn() → 可触发后台压缩
```

### 4.5 子代理委派（Subagent）

#### 委派模型

主 Agent 可通过 `sessions_spawn` 创建子代理，每个子代理：
- 独立的 session 和 context
- 明确的任务描述和边界
- 完成后自动报告结果给主 Agent（push-based）
- 深度限制（如 depth 1/1）

#### Delegate 架构（组织级代理）

| 个人模式 | 代理模式 |
|---------|---------|
| 使用用户凭据 | 拥有自己的凭据 |
| 回复来自用户 | 回复来自代理，代表用户 |
| 一个委托人 | 一个或多个委托人 |
| 信任边界 = 你 | 信任边界 = 组织策略 |

**能力层级**：
1. **只读 + 草稿**：读取数据，起草消息供人工审核
2. **代表发送**：以代理身份发送消息、创建日历事件
3. **主动执行**：基于 standing orders 自主运行，人工异步审核

**安全强化**：
- 工具白名单/黑名单（`tools.allow` / `tools.deny`）
- 沙箱隔离（sandbox mode）
- 审计日志（session transcripts + cron runs）
- 硬性阻断规则（SOUL.md / AGENTS.md）

### 4.6 Session 管理

```
入站消息路由
    │
    ├─ DM → 共享 main session (默认)
    │       或 per-channel-peer 隔离
    │
    ├─ 群组 → 按 group 隔离
    │
    ├─ 线程 → 追加 thread 标识
    │
    ├─ Cron → 每次运行新建 session
    │
    └─ Webhook → 按 hook 隔离
```

**Session 生命周期**：
- **每日重置**（默认凌晨 4:00）
- **空闲重置**（可选，按 `idleMinutes`）
- **手动重置**：`/new` 或 `/reset`

**存储**：
- 索引：`~/.openclaw/agents/<agentId>/sessions/sessions.json`
- 转录：`~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`

### 4.7 可复用设计模式

| 模式 | 描述 | 级别 |
|------|------|------|
| **两阶段确认** | 请求立即 accepted，异步流式完成后 final ok/error | ⭐ 必须保留 |
| **Session Lane 串行化** | 同一会话不并发，保证状态一致性 | ⭐ 必须保留 |
| **自动压缩 + 提醒保存** | 压缩前自动提醒保存重要信息到 memory | ⭐ 必须保留 |
| **可插拔上下文引擎** | 核心能力（上下文组装/压缩）可替换 | ⭐ 必须保留 |
| **Push-based 子代理** | 子代理完成自动通知，无需轮询 | ⭐ 必须保留 |
| **工具 Hook 拦截** | before/after hook 可 block 工具调用 | ⭐ 必须保留 |
| **DM 隔离选项** | main/per-peer/per-channel-peer 可配置 | ⭐ 必须保留 |
| **Delegate 三层能力** | 只读 → 代表发送 → 自主执行，渐进信任 | ⭐ 值得借鉴 |
| **Block Streaming** | 长回复分块发送，避免长时间无响应 | 🔧 可简化 |

---

## 5. 跨领域设计模式总结

### 5.1 架构级模式

| # | 模式 | 体现位置 | 核心思想 |
|---|------|---------|---------|
| 1 | **单进程 Gateway** | Gateway | 一个进程拥有所有 channel 连接和状态，避免分布式一致性 |
| 2 | **Loopback-first** | 网络模型 | 默认仅本地可达，远程需显式认证+隧道 |
| 3 | **设备密钥身份** | 配对系统 | 持久化设备 ID = 公钥指纹，跨重连稳定 |
| 4 | **角色-作用域** | 协议 | Operator vs Node，不同角色不同权限集合 |
| 5 | **约定文件结构** | Workspace | 固定文件名映射固定语义 |
| 6 | **分层优先级** | 技能/配置 | 最高优先级覆盖最低 |
| 7 | **可插拔引擎** | 上下文/记忆 | 核心能力通过接口替换 |
| 8 | **Hook 管线** | Agent/Plugin | 可插拔的前后处理，支持终止传播 |

### 5.2 数据流模式

| 流 | 路径 |
|----|------|
| **入站消息** | Channel → Gateway → Route → Session → Agent Loop → LLM |
| **出站回复** | LLM → Reply Shaping → Channel → 用户 |
| **工具调用** | LLM → Hook → 执行（本地/Node）→ Hook → LLM |
| **记忆写入** | Agent → Workspace 文件 → File Watcher → SQLite 重新索引 |
| **记忆检索** | memory_search → SQLite (FTS5 + Vector) → 混合排序 → 注入上下文 |
| **配置变更** | 编辑文件 → Gateway hot reload → 原子交换内存快照 |

### 5.3 安全模式

| 模式 | 实现 |
|------|------|
| **挑战-响应** | connect.challenge nonce 签名 |
| **Token 作用域绑定** | deviceToken 不可扩展到超出审批范围 |
| **审批门控** | exec approval + node pairing + plugin approval |
| **工具策略** | allow/deny 白名单 + sandbox |
| **身份分离** | Delegate 模式从不冒充用户 |

---

## 6. 可简化项与必保留项

### ⭐ 必须保留（核心价值）

1. **单进程 Gateway + 多协议复用** — 简化部署，减少一致性复杂性
2. **Workspace 约定文件结构** — 零配置的上下文注入
3. **挑战-响应握手 + 设备身份** — 安全基础
4. **角色-作用域权限模型** — 灵活的访问控制
5. **Session Lane 串行化** — 状态一致性保证
6. **确定性路由链** — 可预测的消息路由
7. **自动压缩 + 记忆提醒** — 长对话的核心能力
8. **Hook 管线** — 可扩展性基础
9. **可插拔上下文引擎接口** — 允许高级策略替换
10. **Push-based 子代理** — 高效的任务委派

### 🔧 可简化（按需取舍）

1. **25+ Channel 支持** — 新项目可从 2-3 个核心 Channel 开始
2. **Delegate 三层架构** — 个人项目不需要组织级代理
3. **Block Streaming 分块** — 简单场景可用完整回复替代
4. **Broadcast Groups** — 多 Agent 并行回复是高级需求
5. **OpenAI 兼容层** — 如果不需要接入第三方客户端可省略
6. **Cron/Heartbeat 自动化** — 初期可用外部调度器替代
7. **事件不回放策略** — 新项目可考虑实现事件回放以降低客户端复杂度
8. **多 Gateway 实例** — 单实例足够大多数场景
9. **TLS + 证书固定** — 初期可用 Tailscale/SSH 隧道替代
10. **Session maintenance 模式** — 简单场景可手动管理

### 📐 新项目架构建议

基于以上分析，新项目可采用以下最小可行架构：

```
┌─────────────────────────────────────────┐
│            Gateway (单进程)               │
│  ┌─────────────────────────────────────┐│
│  │  WS Server (认证 + 路由)            ││
│  │  + HTTP API                         ││
│  └──────┬──────────┬──────────────────┘│
│         │          │                    │
│    ┌────┘    ┌─────┘                    │
│    ▼         ▼                          │
│  Channel   Agent Loop                   │
│  (1-2个)   ├─ Workspace 注入           │
│            ├─ LLM 调用                  │
│            ├─ 工具执行                  │
│            ├─ 自动压缩                  │
│            └─ 记忆检索                  │
└─────────────────────────────────────────┘
         │
    ┌────┘
    ▼
  Node (可选)
  - 摄像头/屏幕等外设能力
```

**核心接口定义**：
- `ContextEngine`：`ingest / assemble / compact / afterTurn`
- `Channel`：统一的消息入站/出站抽象
- `Tool`：标准化的工具注册与执行
- `MemoryStore`：索引 + 搜索的抽象

---

> **结论**：OpenClaw 的架构设计体现了几个核心思想——**单进程简化一致性**、**约定优于配置**、**安全默认值（loopback-first）**、**可插拔核心能力**。这些设计模式可以直接复用到任何需要 LLM Agent + 多平台消息路由 + 设备控制的新项目中。
