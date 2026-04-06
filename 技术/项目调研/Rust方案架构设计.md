# Rust AI Agent Bot 框架架构设计

> **代号**: IronClaw（暂定）
> **版本**: v0.1 Draft
> **日期**: 2026-04-06
> **作者**: Architecture Review

---

## 目录

1. [项目概述](#1-项目概述)
2. [技术栈选择](#2-技术栈选择)
3. [项目目录结构](#3-项目目录结构)
4. [核心架构](#4-核心架构)
5. [Gateway-Node 通信](#5-gateway-node-通信)
6. [插件系统设计](#6-插件系统设计)
7. [部署方案](#7-部署方案)
8. [开源社区考量](#8-开源社区考量)
9. [与 OpenClaw 的对比](#9-与-openclaw-的对比)
10. [里程碑规划](#10-里程碑规划)

---

## 1. 项目概述

### 1.1 目标

用 Rust 重写一个类似 OpenClaw 的 AI Agent Bot 框架，核心定位：

- **高性能**：单 binary，内存占用 < 50MB（空载），并发处理万级消息
- **可扩展**：声明式插件系统，支持 Python/原生 Rust 插件
- **安全**：权限 manifest，最小权限原则，沙箱执行
- **易部署**：`curl | sh` 一键安装，零外部依赖（除 SQLite）
- **开发者友好**：清晰的 crate 边界、完善的文档、热重载插件

### 1.2 核心需求映射

| 需求 | 对应设计 |
|:---|:---|
| Workspace 设计 | `ironclaw-core::workspace` — 基于 OS 文件系统的隔离工作目录 |
| Gateway-Node 架构 | `ironclaw-gateway` + `ironclaw-node`，QUIC/WebSocket 双协议 |
| 声明式权限 | `plugin.yaml` manifest + `ironclaw-plugins::sandbox` |
| 一键部署 | 静态链接 single binary + install script |
| Agent 系统 | `ironclaw-core::agent` — tool calling loop + context manager |

---

## 2. 技术栈选择

### 2.1 技术栈总览

```
┌─────────────────────────────────────────────────┐
│                  IronClaw Stack                  │
├──────────┬──────────┬──────────┬────────────────┤
│ Language │ Runtime  │ Web      │ Serialization  │
│ Rust     │ Tokio    │ Axum     │ serde + serde_json │
├──────────┼──────────┼──────────┼────────────────┤
│ Database │ Config   │ Plugin   │ LLM Comm       │
│ SQLite   │ TOML     │ PyO3     │ reqwest        │
│ (sqlx)   │ (toml)   │          │                │
├──────────┼──────────┼──────────┼────────────────┤
│ Crypto   │ Logging  │ CLI      │ gRPC/QUIC      │
│ ring     │ tracing  │ clap     │ quinn          │
└──────────┴──────────┴──────────┴────────────────┘
```

### 2.2 Web 框架：Axum

**选择 Axum，而非 Actix-web。**

| 维度 | Axum | Actix-web |
|:---|:---|:---|
| 异步模型 | Tokio 原生 | 自有 runtime（兼容 tokio） |
| 生态兼容 | Tower 中间件生态 | 自有中间件体系 |
| 学习曲线 | 较平缓 | 中等 |
| 社区活跃度 | 极高，Tokio 团队维护 | 高 |
| 类型安全提取 | `FromRequestParts` | `FromRequest` |
| 长连接支持 | WebSocket via `tokio-tungstenite` | 内置 WebSocket |

**理由**：Axum 与 Tokio 生态无缝集成（Tower middleware、`hyper`、`tonic`），避免了 Actix 自有 runtime 的兼容性问题。对于需要长连接（WebSocket）、gRPC（tonic）的场景，Axum 更自然。

### 2.3 异步运行时：Tokio

唯一选择。Rust 异步生态的事实标准。

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

关键特性使用：
- `tokio::spawn` — 并发任务
- `tokio::sync::mpsc` — 内部消息总线
- `tokio::sync::RwLock` — 共享状态
- `tokio::net` — TCP/UDP 监听
- `tokio::fs` — 异步文件操作
- `tokio::signal` — 优雅关闭

### 2.4 Python 插件桥接：PyO3

允许用户用 Python 编写插件，同时保持核心框架为纯 Rust。

```toml
[dependencies]
pyo3 = { version = "0.23", features = ["extension-module", "auto-initialize"] }
```

设计要点：
- Python 插件运行在独立子进程（`ironclaw-plugin-runner`），通过 Unix socket / named pipe 通信
- 避免在主进程中嵌入 Python 解释器（防止 GIL 阻塞和 crash 传播）
- 使用 JSON-RPC 2.0 作为 IPC 协议

### 2.5 LLM Provider 通信

```toml
[dependencies]
reqwest = { version = "0.12", features = ["json", "stream", "rustls-tls"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
eventsource-stream = "0.2"  # SSE 解析
```

支持：
- OpenAI API 兼容接口（覆盖大多数 provider）
- 流式响应（SSE / WebSocket）
- 重试与超时（`tower::retry` + 自定义 backoff）
- 多 provider 路由（根据 model name 自动选择）

### 2.6 数据库：SQLite via sqlx

```toml
[dependencies]
sqlx = { version = "0.8", features = ["runtime-tokio", "sqlite", "migrate"] }
```

选择 sqlx 而非 rusqlite 的理由：
- 异步原生，与 Tokio 配合无需 spawn blocking
- 编译时 SQL 检查（`sqlx::query!`）
- 内置 migration 支持
- 连接池

存储内容：
- 消息历史
- 会话上下文
- 插件注册信息
- 节点配对凭证
- Agent 状态快照

### 2.7 配置管理：TOML

```toml
[dependencies]
toml = "0.8"
serde = { version = "1", features = ["derive"] }
directories = "5"  # 跨平台路径（XDG / Known Folders）
```

配置文件分层：
1. `/etc/ironclaw/config.toml` — 全局配置（系统级安装）
2. `~/.config/ironclaw/config.toml` — 用户配置
3. `./ironclaw.toml` — 项目级配置
4. 环境变量覆盖（`IRONCLAW_*`）

### 2.8 其他关键依赖

| 用途 | Crate | 说明 |
|:---|:---|:---|
| 日志 | `tracing` + `tracing-subscriber` | 结构化日志，支持 span |
| 错误处理 | `anyhow` (app) + `thiserror` (lib) | 分层错误策略 |
| CLI | `clap` v4 | derive 模式 |
| 加密 | `ring` / `ed25519-dalek` | 节点认证，JWT |
| QUIC | `quinn` | 高性能节点通信 |
| gRPC | `tonic` + `prost` | 可选，服务间通信 |
| 信号处理 | `tokio::signal` | 优雅关闭 |
| 进程管理 | `sysinfo` | 节点健康检查 |

---

## 3. 项目目录结构

### 3.1 Cargo Workspace 布局

```
ironclaw/
├── Cargo.toml                    # workspace 根
├── Cargo.lock
├── rust-toolchain.toml           # 固定 Rust 版本
├── cliff.toml                    # git-cliff 配置（changelog）
│
├── crates/
│   ├── ironclaw-core/            # 核心库：agent loop, workspace, types
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── agent/
│   │       │   ├── mod.rs        # Agent loop 主逻辑
│   │       │   ├── context.rs    # 上下文管理（token 预算）
│   │       │   ├── tool_call.rs  # Tool calling 协议
│   │       │   ├── stream.rs     # 流式响应处理
│   │       │   └── prompt.rs     # System prompt 管理
│   │       ├── workspace/
│   │       │   ├── mod.rs
│   │       │   ├── fs.rs         # 文件操作抽象
│   │       │   └── sandbox.rs    # 沙箱限制
│   │       ├── llm/
│   │       │   ├── mod.rs        # LLM provider trait
│   │       │   ├── openai.rs     # OpenAI 兼容
│   │       │   ├── anthropic.rs  # Anthropic Claude
│   │       │   └── types.rs      # Message, Tool, Response 类型
│   │       ├── types/
│   │       │   ├── mod.rs
│   │       │   ├── message.rs    # 统一消息类型
│   │       │   ├── channel.rs    # Channel 抽象
│   │       │   └── permission.rs # 权限类型
│   │       └── error.rs          # 统一错误类型
│   │
│   ├── ironclaw-gateway/         # Gateway 服务器
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs
│   │       ├── server.rs         # Axum HTTP/WS 服务器
│   │       ├── router.rs         # API 路由定义
│   │       ├── session.rs        # 会话管理
│   │       ├── node_manager.rs   # 节点连接管理
│   │       ├── bus.rs            # 内部消息总线
│   │       └── middleware/
│   │           ├── auth.rs       # 认证中间件
│   │           ├── rate_limit.rs # 限流
│   │           └── logging.rs    # 请求日志
│   │
│   ├── ironclaw-node/            # 远程节点客户端
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs
│   │       ├── connector.rs      # 连接 Gateway
│   │       ├── pair.rs           # 配对流程
│   │       ├── executor.rs       # 远程命令执行
│   │       └── heartbeat.rs      # 心跳上报
│   │
│   ├── ironclaw-plugins/         # 插件系统
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── loader.rs         # 插件发现与加载
│   │       ├── manifest.rs       # plugin.yaml 解析
│   │       ├── registry.rs       # 插件注册表
│   │       ├── sandbox.rs        # 权限检查
│   │       ├── channel.rs        # Channel plugin trait
│   │       ├── tool.rs           # Tool plugin trait
│   │       ├── hook.rs           # Hook plugin trait
│   │       └── python/
│   │           ├── mod.rs        # PyO3 桥接
│   │           ├── runner.rs     # Python 子进程管理
│   │           └── ipc.rs       # JSON-RPC IPC
│   │
│   ├── ironclaw-db/              # 数据库层
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── pool.rs           # 连接池
│   │       ├── migrate.rs        # Migration
│   │       ├── message.rs        # 消息 CRUD
│   │       ├── session.rs        # 会话存储
│   │       └── plugin.rs         # 插件持久化
│   │
│   ├── ironclaw-config/          # 配置管理
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── loader.rs         # 分层加载
│   │       ├── validator.rs      # 配置校验
│   │       └── types.rs          # 配置结构体
│   │
│   └── ironclaw-cli/             # CLI 入口
│       ├── Cargo.toml
│       └── src/
│           ├── main.rs
│           ├── commands/
│           │   ├── start.rs      # ironclaw start
│           │   ├── stop.rs       # ironclaw stop
│           │   ├── status.rs     # ironclaw status
│           │   ├── plugin.rs     # ironclaw plugin install/list
│           │   ├── node.rs       # ironclaw node pair/connect
│           │   └── config.rs     # ironclaw config get/set
│           └── output.rs         # 彩色终端输出
│
├── plugins/                      # 内置插件（示例 + 必需）
│   ├── ironclaw-plugin-telegram/
│   │   ├── Cargo.toml
│   │   ├── plugin.yaml
│   │   └── src/
│   │       └── lib.rs
│   ├── ironclaw-plugin-discord/
│   │   ├── Cargo.toml
│   │   ├── plugin.yaml
│   │   └── src/
│   │       └── lib.rs
│   └── ironclaw-plugin-shell/    # Shell 执行工具
│       ├── Cargo.toml
│       ├── plugin.yaml
│       └── src/
│           └── lib.rs
│
├── python-runner/                # Python 插件运行器
│   ├── ironclaw_runner/          # Python 包
│   │   ├── __init__.py
│   │   ├── plugin_base.py        # 插件基类
│   │   ├── ipc_client.py         # IPC 客户端
│   │   └── decorators.py         # @tool, @hook 装饰器
│   └── setup.py
│
├── migrations/                   # SQLite migrations
│   ├── 001_initial.sql
│   ├── 002_sessions.sql
│   └── 003_plugins.sql
│
├── config/                       # 示例配置
│   ├── ironclaw.toml.example
│   └── plugin.yaml.example
│
├── scripts/                      # 构建/安装脚本
│   ├── install.sh                # curl | sh 安装脚本
│   ├── build-release.sh          # 交叉编译发布
│   └── generate-completions.sh   # Shell 补全
│
├── docs/                         # 文档
│   ├── architecture.md
│   ├── plugin-development.md
│   ├── deployment.md
│   └── api-reference.md
│
├── tests/                        # 集成测试
│   ├── common/
│   │   └── mod.rs
│   ├── agent_loop_test.rs
│   ├── plugin_loading_test.rs
│   └── gateway_node_test.rs
│
├── benches/                      # 性能基准
│   └── agent_bench.rs
│
├── .github/
│   ├── workflows/
│   │   ├── ci.yml
│   │   └── release.yml
│   └── dependabot.yml
│
├── LICENSE
├── README.md
└── CONTRIBUTING.md
```

### 3.2 Workspace Cargo.toml

```toml
[workspace]
resolver = "2"
members = [
    "crates/*",
    "plugins/*",
]

[workspace.package]
version = "0.1.0"
edition = "2024"
license = "MIT OR Apache-2.0"
repository = "https://github.com/example/ironclaw"

[workspace.dependencies]
# 内部 crate
ironclaw-core    = { path = "crates/ironclaw-core" }
ironclaw-gateway = { path = "crates/ironclaw-gateway" }
ironclaw-node    = { path = "crates/ironclaw-node" }
ironclaw-plugins = { path = "crates/ironclaw-plugins" }
ironclaw-db      = { path = "crates/ironclaw-db" }
ironclaw-config  = { path = "crates/ironclaw-config" }

# 异步 & Web
tokio = { version = "1", features = ["full"] }
axum = { version = "0.8", features = ["ws", "multipart"] }
tower = { version = "0.5", features = ["full"] }
tower-http = { version = "0.6", features = ["cors", "trace", "limit"] }

# 序列化
serde = { version = "1", features = ["derive"] }
serde_json = "1"
toml = "0.8"

# HTTP 客户端
reqwest = { version = "0.12", features = ["json", "stream", "rustls-tls"] }

# 数据库
sqlx = { version = "0.8", features = ["runtime-tokio", "sqlite", "migrate"] }

# 日志
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }

# 错误
anyhow = "1"
thiserror = "2"

# CLI
clap = { version = "4", features = ["derive"] }

# Python
pyo3 = { version = "0.23", features = ["auto-initialize"] }

# 加密 & 认证
ring = "0.17"
ed25519-dalek = { version = "2", features = ["serde"] }
jsonwebtoken = "9"

# QUIC
quinn = "0.11"

# gRPC
tonic = "0.12"
prost = "0.13"

# 工具
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
directories = "5"
glob = "0.3"
notify = "7"               # 文件监听（热重载）
```

---

## 4. 核心架构

### 4.1 Gateway 进程架构图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        IronClaw Gateway Process                          │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                         Axum HTTP Server                          │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │  │
│  │  │ REST API │  │  WebSocket│  │  Webhook │  │  Health Check    │  │  │
│  │  │  /api/*  │  │  /ws     │  │  /hook/* │  │  /healthz       │  │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────────────────┘  │  │
│  └───────┼──────────────┼─────────────┼───────────────────────────────┘  │
│          │              │             │                                    │
│  ┌───────▼──────────────▼─────────────▼───────────────────────────────┐  │
│  │                     Internal Message Bus                           │  │
│  │                    (tokio::sync::broadcast)                        │  │
│  └───────┬──────────────┬─────────────┬───────────────────────────────┘  │
│          │              │             │                                    │
│  ┌───────▼───────┐ ┌────▼─────┐ ┌────▼──────────┐ ┌──────────────────┐  │
│  │  Channel      │ │  Agent   │ │  Plugin       │ │  Node Manager    │  │
│  │  Manager      │ │  Engine  │ │  Host         │ │                  │  │
│  │               │ │          │ │               │ │  ┌────────────┐  │  │
│  │ ┌───────────┐ │ │ ┌──────┐ │ │ ┌───────────┐ │ │  │ Node Pool  │  │  │
│  │ │ Telegram  │ │ │ │Agent │ │ │ │ Tool      │ │ │  │ ┌────────┐ │  │  │
│  │ │ Plugin    │ │ │ │Loop 1│ │ │ │ Registry  │ │ │  │ │ Node A │ │  │  │
│  │ ├───────────┤ │ │ ├──────┤ │ │ ├───────────┤ │ │  │ ├────────┤ │  │  │
│  │ │ Discord   │ │ │ │Agent │ │ │ │ Channel   │ │ │  │ │ Node B │ │  │  │
│  │ │ Plugin    │ │ │ │Loop 2│ │ │ │ Registry  │ │ │  │ ├────────┤ │  │  │
│  │ ├───────────┤ │ │ ├──────┤ │ │ ├───────────┤ │ │  │ │ Node C │ │  │  │
│  │ │ QQ        │ │ │ │Agent │ │ │ │ Hook      │ │ │  │ └────────┘ │  │  │
│  │ │ Plugin    │ │ │ │Loop N│ │ │ │ Registry  │ │ │  │  Heartbeat │  │  │
│  │ └───────────┘ │ │ └──────┘ │ │ └───────────┘ │ │  └────────────┘  │  │
│  └───────────────┘ └──────────┘ └───────────────┘ └──────────────────┘  │
│          │              │             │                       │           │
│  ┌───────▼──────────────▼─────────────▼───────────────────────▼───────┐  │
│  │                     Workspace Manager                              │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                           │  │
│  │  │ Agent W1 │ │ Agent W2 │ │ Shared   │                           │  │
│  │  │ /ws/agent│ │ /ws/agent│ │ /ws/share│                           │  │
│  │  │    /1    │ │    /2    │ │          │                           │  │
│  │  └──────────┘ └──────────┘ └──────────┘                           │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌─────────────────────┐  ┌──────────────────┐  ┌───────────────────┐   │
│  │    SQLite DB        │  │  Config Manager  │  │  LLM Router      │   │
│  │  (via sqlx)         │  │  (TOML layered)  │  │  (reqwest)       │   │
│  └─────────────────────┘  └──────────────────┘  └───────────────────┘   │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                     Plugin Runner Pool                             │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │  │
│  │  │ Python Proc 1│  │ Python Proc 2│  │ Rust Plugin  │            │  │
│  │  │ (Unix Socket)│  │ (Unix Socket)│  │ (in-process) │            │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘            │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.2 消息流

用户消息从接收到响应的完整流程：

```
 用户 (Telegram/QQ/Discord)
    │
    │  ① 消息到达
    ▼
 ┌──────────────────┐
 │  Channel Plugin  │  解析平台格式 → 统一 Message
 │  (e.g. Telegram) │  识别: @mention, /command, 普通消息
 └────────┬─────────┘
          │  ② UnifiedMessage
          ▼
 ┌──────────────────┐
 │  Message Bus     │  broadcast channel，所有订阅者收到
 │  (broadcast)     │
 └────────┬─────────┘
          │  ③ 路由到目标 Agent
          ▼
 ┌──────────────────┐
 │  Session Manager │  确定会话 ID（用户+频道）
 │                  │  加载或创建 Agent Loop
 └────────┬─────────┘
          │  ④ 加载上下文
          ▼
 ┌──────────────────────────────────────────────┐
 │               Agent Loop                      │
 │                                               │
 │  ┌─────────────┐    ┌─────────────────────┐  │
 │  │ Context     │◄───│ Message History      │  │
 │  │ Manager     │    │ (from SQLite)        │  │
 │  │             │    │ + System Prompt      │  │
 │  │ Token Budget│    │ + Workspace Files    │  │
 │  │ 128K tokens │    │ + Tool Definitions   │  │
 │  └──────┬──────┘    └─────────────────────┘  │
 │         │                                     │
 │         │  ⑤ 构建 API 请求                    │
 │         ▼                                     │
 │  ┌─────────────┐                              │
 │  │ LLM Router  │──► OpenAI / Anthropic / ...  │
 │  │             │    (streaming SSE)            │
 │  └──────┬──────┘                              │
 │         │  ⑥ 流式响应                          │
 │         ▼                                     │
 │  ┌─────────────┐                              │
 │  │ Response    │  解析 delta，检测 tool_call   │
 │  │ Parser      │                              │
 │  └──────┬──────┘                              │
 │         │                                     │
 │         ├── 文本 ──► ⑧ 直接回复                │
 │         │                                     │
 │         └── tool_call ──► ⑦ 执行工具          │
 │              │                                │
 │              ▼                                │
 │         ┌─────────────┐                       │
 │         │ Tool        │  权限检查 → 执行       │
 │         │ Executor    │  收集结果              │
 │         └──────┬──────┘                       │
 │                │                               │
 │                │  返回结果，重新进入 ⑤          │
 │                └───► (loop back to LLM)        │
 │                                                │
 └────────────────┬───────────────────────────────┘
                  │  ⑧ 最终回复
                  ▼
 ┌──────────────────┐
 │  Channel Plugin  │  统一格式 → 平台格式
 │  (e.g. Telegram) │  Markdown/HTML 转换
 └────────┬─────────┘
          │  ⑨ 发送
          ▼
 用户 (Telegram/QQ/Discord)
```

### 4.3 Workspace 文件系统设计

每个 Agent 拥有独立的 Workspace 目录，模拟"Agent 的工作台"。

```
~/.local/share/ironclaw/
├── config.toml                    # 全局配置
├── data.db                        # SQLite 数据库
├── plugins/                       # 已安装插件
│   ├── telegram/
│   │   ├── plugin.yaml
│   │   └── libtelegram.so
│   ├── shell/
│   │   ├── plugin.yaml
│   │   └── libshell.so
│   └── my-python-tool/
│       ├── plugin.yaml
│       └── main.py
│
├── workspaces/                    # Agent 工作空间
│   └── default/                   # 默认 Agent 的 workspace
│       ├── AGENTS.md              # Agent 行为指令
│       ├── SOUL.md                # Agent 人格
│       ├── USER.md                # 用户信息
│       ├── MEMORY.md              # 长期记忆
│       ├── SESSION-STATE.md       # 当前会话状态
│       ├── memory/                # 记忆归档
│       │   ├── 2026-04-05.md
│       │   └── 2026-04-06.md
│       ├── files/                 # Agent 可读写的工作文件
│       │   ├── notes.md
│       │   └── data.csv
│       └── .sandbox/              # 沙箱临时目录
│           └── tmp_XXXX/
│
├── logs/                          # 运行日志
│   └── ironclaw.log
│
├── sockets/                       # IPC Unix Socket
│   └── ironclaw.sock
│
└── cache/                         # 缓存
    └── llm_responses/
```

Workspace 隔离策略：

```rust
/// Workspace 权限控制
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct WorkspacePolicy {
    /// 可读路径白名单（Glob 模式）
    pub read_paths: Vec<String>,
    /// 可写路径白名单
    pub write_paths: Vec<String>,
    /// 最大文件大小（bytes）
    pub max_file_size: u64,
    /// 最大总使用空间（bytes）
    pub max_total_size: u64,
    /// 是否允许执行命令
    pub allow_exec: bool,
    /// 允许执行命令的路径白名单
    pub exec_whitelist: Vec<String>,
    /// 网络访问策略
    pub network: NetworkPolicy,
}

impl WorkspacePolicy {
    /// 默认安全策略
    pub fn default_secure() -> Self {
        Self {
            read_paths: vec!["./**".into()],           // 只读 workspace 内
            write_paths: vec!["./files/**".into()],    // 只写 files/ 子目录
            max_file_size: 10 * 1024 * 1024,           // 10MB
            max_total_size: 500 * 1024 * 1024,         // 500MB
            allow_exec: false,
            exec_whitelist: vec![],
            network: NetworkPolicy::DenyAll,
        }
    }
}
```

### 4.4 Agent Loop 详细实现

```rust
/// Agent Loop 核心结构
pub struct AgentLoop {
    /// 会话 ID
    session_id: SessionId,
    /// 上下文管理器
    context: ContextManager,
    /// LLM Provider（通过 Router 选择）
    llm_router: LlmRouter,
    /// 工具注册表
    tools: ToolRegistry,
    /// Workspace 访问器
    workspace: WorkspaceHandle,
    /// 最大 tool calling 轮次（防止无限循环）
    max_rounds: u32,
    /// Token 预算
    token_budget: TokenBudget,
}

/// Agent Loop 执行流程
impl AgentLoop {
    /// 处理一条用户消息，返回最终回复
    pub async fn process_message(
        &mut self,
        user_message: UserMessage,
    ) -> Result<AgentResponse> {
        // 1. 追加用户消息到上下文
        self.context.push_user_message(user_message.clone()).await?;

        // 2. 读取 workspace 中的上下文文件（如 MEMORY.md, SESSION-STATE.md）
        self.context.inject_workspace_files(&self.workspace).await?;

        // 3. 构建 tool definitions（根据当前权限过滤）
        let available_tools = self.tools.filter_by_permission(
            self.workspace.policy()
        );

        // 4. Agent Loop（多轮 tool calling）
        let mut rounds = 0;
        loop {
            rounds += 1;
            if rounds > self.max_rounds {
                return Ok(AgentResponse::error("超过最大 tool calling 轮次"));
            }

            // 4a. 构建 LLM 请求
            let request = self.context.build_request(&available_tools);

            // 4b. 调用 LLM（流式）
            let stream = self.llm_router.chat_stream(request).await?;

            // 4c. 解析流式响应
            let response = self.parse_stream(stream).await?;

            // 4d. 追加 assistant 响应到上下文
            self.context.push_assistant_message(response.clone()).await?;

            // 4e. 检查是否需要 tool calling
            match response.finish_reason {
                FinishReason::Stop => {
                    // 普通文本回复，结束 loop
                    return Ok(AgentResponse::text(response.content));
                }
                FinishReason::ToolCalls => {
                    // 执行所有 tool calls
                    for tool_call in response.tool_calls {
                        // 权限检查
                        self.tools.check_permission(
                            &tool_call.function.name,
                            self.workspace.policy(),
                        )?;

                        // 执行工具
                        let result = self.tools.execute(
                            tool_call.clone(),
                            &self.workspace,
                        ).await?;

                        // 将工具结果追加到上下文
                        self.context.push_tool_result(
                            tool_call.id,
                            result,
                        ).await?;
                    }
                    // 继续下一轮（回到 4a）
                }
            }
        }
    }

    /// 解析流式响应，支持实时中间输出
    async fn parse_stream(
        &self,
        mut stream: Pin<Box<dyn Stream<Item = Result<ChatDelta>> + Send>>,
    ) -> Result<ChatResponse> {
        let mut content = String::new();
        let mut tool_calls: Vec<ToolCallBuilder> = Vec::new();
        let mut finish_reason = FinishReason::Stop;

        while let Some(delta) = stream.next().await {
            let delta = delta?;

            match delta {
                ChatDelta::Content(text) => {
                    content.push_str(&text);
                    // 发射中间事件（可用于流式回复用户）
                    self.emit_event(AgentEvent::StreamingText(text)).await;
                }
                ChatDelta::ToolCall(tc_delta) => {
                    // 累积 tool call delta
                    self.accumulate_tool_call(&mut tool_calls, tc_delta);
                }
                ChatDelta::Finish(reason) => {
                    finish_reason = reason;
                }
            }
        }

        Ok(ChatResponse {
            content,
            tool_calls: tool_calls.into_iter().map(|b| b.build()).collect(),
            finish_reason,
        })
    }
}
```

### 4.5 上下文管理

```rust
/// Token 预算管理
pub struct TokenBudget {
    /// 模型最大 token 数
    max_tokens: usize,
    /// 保留给回复的 token 数
    completion_reserve: usize,
}

impl TokenBudget {
    /// 计算可用于 prompt 的 token 数
    pub fn available_for_prompt(&self) -> usize {
        self.max_tokens - self.completion_reserve
    }

    /// 检查是否超出预算，若超出则压缩上下文
    pub fn check_and_compact(
        &self,
        context: &mut ContextManager,
    ) -> Result<()> {
        let used = context.estimated_tokens();
        let budget = self.available_for_prompt();

        if used > budget {
            // 策略 1: 移除最旧的非 system 消息
            // 策略 2: 摘要压缩（调用 LLM 做摘要）
            // 策略 3: 截断 workspace 注入内容
            context.compact(used - budget)?;
        }
        Ok(())
    }
}

/// 上下文构建
pub struct ContextManager {
    /// System prompt（来自 AGENTS.md, SOUL.md 等）
    system_prompt: String,
    /// 消息历史（从 SQLite 加载）
    messages: Vec<ChatMessage>,
    /// Workspace 文件注入
    injected_files: Vec<InjectedFile>,
    /// Token 计数器
    token_counter: TokenCounter,
}

impl ContextManager {
    /// 构建 LLM API 请求
    pub fn build_request(
        &self,
        tools: &[ToolDefinition],
    ) -> ChatRequest {
        let mut messages = Vec::new();

        // 1. System prompt（始终在最前）
        let mut system_content = self.system_prompt.clone();

        // 2. 注入 workspace 文件内容
        for file in &self.injected_files {
            system_content.push_str(&format!(
                "\n\n## File: {}\n```\n{}\n```",
                file.path, file.content
            ));
        }

        messages.push(ChatMessage::system(&system_content));

        // 3. 对话历史
        messages.extend(self.messages.iter().cloned());

        ChatRequest {
            messages,
            tools: tools.to_vec(),
            stream: true,
            ..Default::default()
        }
    }
}
```

---

## 5. Gateway-Node 通信

### 5.1 协议选择：QUIC（主） + WebSocket（备）

```
                    ┌──────────────────────┐
                    │    通信协议对比       │
                    └──────────────────────┘

┌───────────┬──────────────┬──────────────┬──────────────┐
│ 维度      │ QUIC         │ WebSocket    │ gRPC         │
├───────────┼──────────────┼──────────────┼──────────────┤
│ 延迟      │ 极低 (0-RTT) │ 中等         │ 低           │
│ 连接建立  │ 1-RTT/0-RTT  │ TCP+TLS握手  │ TCP+TLS+HTTP2│
│ 多路复用  │ 原生支持     │ 不支持       │ HTTP/2       │
│ 穿透NAT   │ 较好         │ 需要中转     │ 较差         │
│ 移动端    │ 连接迁移✓    │ 断线重连     │ 断线重连     │
│ 复杂度    │ 中等(quinn)  │ 低           │ 高           │
│ 浏览器    │ 不支持       │ 原生支持     │ 不支持       │
│ 二进制开销 │ 低           │ 中等         │ 低(protobuf) │
└───────────┴──────────────┴──────────────┴──────────────┘

决策：
- Node ↔ Gateway 主通信：QUIC（quinn crate）
- 浏览器/调试 API：WebSocket（兼容场景）
- 不使用 gRPC（增加依赖复杂度，无显著优势）
```

### 5.2 节点发现与配对流程

```
 ┌──────────┐                                    ┌──────────┐
 │   Node   │                                    │ Gateway  │
 │ (新设备)  │                                    │ (服务器)  │
 └────┬─────┘                                    └────┬─────┘
      │                                               │
      │  ① 用户在 Gateway CLI 生成配对码               │
      │     $ ironclaw node pair                      │
      │     输出: Pair Code: ABCD-1234-EFGH           │
      │                                               │
      │  ② 用户在 Node 端输入配对码                     │
      │     $ ironclaw node connect <gateway_url>     │
      │     Enter pair code: ABCD-1234-EFGH           │
      │                                               │
      │  ③ Node → Gateway: PairRequest               │
      │  {pair_code, node_public_key, node_info}      │
      │ ──────────────────────────────────────────────►│
      │                                               │
      │  ④ Gateway 验证 pair_code                     │
      │     生成 node_token (JWT)                      │
      │     存储 node_public_key                       │
      │                                               │
      │  ⑤ Gateway → Node: PairResponse              │
      │  {node_token, gateway_cert, config}            │
      │ ◄──────────────────────────────────────────────│
      │                                               │
      │  ⑥ QUIC 连接建立（双向认证）                     │
      │ ═══════════════════════════════════════════════│
      │                                               │
      │  ⑦ Node → Gateway: RegisterNode              │
      │  {capabilities: ["exec", "screen", "camera"]} │
      │ ──────────────────────────────────────────────►│
      │                                               │
      │  ⑧ Gateway → Node: Heartbeat (every 30s)      │
      │ ◄──────────────────────────────────────────────│
      │  ⑧ Node → Gateway: HeartbeatAck               │
      │ ──────────────────────────────────────────────►│
      │                                               │
```

### 5.3 认证与安全

```rust
/// 节点配对凭证
#[derive(Debug, Serialize, Deserialize)]
pub struct NodeToken {
    /// 节点唯一 ID
    pub node_id: String,
    /// 签发时间
    pub issued_at: i64,
    /// 过期时间
    pub expires_at: i64,
    /// 节点权限
    pub permissions: NodePermissions,
}

/// 节点权限
#[derive(Debug, Serialize, Deserialize)]
pub struct NodePermissions {
    /// 允许执行的命令
    pub allowed_commands: Vec<String>,
    /// 允许访问的路径
    pub allowed_paths: Vec<String>,
    /// 是否允许截屏
    pub allow_screen: bool,
    /// 是否允许摄像头
    pub allow_camera: bool,
    /// 网络访问
    pub allow_network: bool,
}

/// QUIC 连接配置
pub fn configure_quic_server(
    cert: &Certificate,
    key: &PrivateKey,
) -> Result<ServerConfig> {
    let mut server_config = ServerConfig::with_crypto(
        rustls::ServerConfig::builder()
            .with_no_client_auth()  // 或配置 mTLS
            .with_single_cert(
                vec![cert.clone()],
                key.clone(),
            )?
    );

    // 保持连接活跃
    let mut transport = TransportConfig::default();
    transport.keep_alive_interval(Some(Duration::from_secs(5)));
    transport.max_idle_timeout(Some(Duration::from_secs(60).try_into()?));
    server_config.transport_config(Arc::new(transport));

    Ok(server_config)
}
```

### 5.4 节点消息协议（QUIC Stream）

使用 bincode 进行二进制序列化（比 JSON 更高效）：

```rust
/// 节点间消息协议
#[derive(Debug, Serialize, Deserialize)]
pub enum NodeMessage {
    // 心跳
    Heartbeat { timestamp: i64 },
    HeartbeatAck { timestamp: i64, server_time: i64 },

    // 注册
    Register { node_id: String, capabilities: Vec<String> },

    // 命令执行
    ExecRequest {
        id: String,
        command: String,
        args: Vec<String>,
        env: HashMap<String, String>,
        timeout_ms: u64,
    },
    ExecResponse {
        id: String,
        stdout: Vec<u8>,
        stderr: Vec<u8>,
        exit_code: i32,
    },
    ExecStream {
        id: String,
        stream: ExecStreamType,
        data: Vec<u8>,
    },

    // 截屏
    ScreenRequest { id: String },
    ScreenResponse { id: String, image: Vec<u8>, format: ImageFormat },

    // 文件传输
    FileRead { id: String, path: String },
    FileWrite { id: String, path: String, data: Vec<u8> },

    // 事件推送
    Event { event_type: String, payload: serde_json::Value },
}
```

---

## 6. 插件系统设计

### 6.1 插件类型

| 类型 | 职责 | 示例 |
|:---|:---|:---|
| **Channel** | 消息平台接入（收发消息） | Telegram, Discord, QQ, Slack |
| **Tool** | 提供给 Agent 的工具（tool calling） | Shell, Browser, FileOps, WebSearch |
| **Hook** | 生命周期钩子（事件回调） | OnMessage, OnToolResult, OnStartup |

### 6.2 plugin.yaml Manifest 规范

```yaml
# plugin.yaml — 插件清单文件（必需）

# ===== 元数据 =====
api_version: "v1"
name: "ironclaw-shell"
version: "0.1.0"
description: "Shell command execution tool for IronClaw agents"
author: "IronClaw Team"
license: "MIT"
repository: "https://github.com/example/ironclaw-shell"

# ===== 类型声明 =====
type: "tool"  # channel | tool | hook

# ===== 运行时 =====
runtime: "native"  # native (Rust .so) | python | wasm (future)

# 入口点
entry:
  native: "libshell.so"           # Rust 动态库
  python: "main.py"               # Python 入口模块

# ===== 依赖 =====
depends:
  ironclaw_core: ">=0.1.0"        # 框架版本要求
  system:
    - "bash"                       # 系统依赖（可选，用于提示）

# ===== 权限声明 =====
permissions:
  # 文件系统
  fs:
    read:
      - "${workspace}/**"          # 可读 workspace 内所有文件
    write:
      - "${workspace}/files/**"    # 可写 files/ 子目录
      - "${workspace}/.sandbox/**" # 可写沙箱目录

  # 命令执行
  exec:
    allow: true
    commands:
      - "git"
      - "python3"
      - "node"
      - "cargo"
      - "npm"
      - "pip"
      # 允许 glob 模式
      - "/usr/bin/*"
    deny:
      - "rm -rf /"                 # 显式禁止（即使 allow 匹配也拒绝）
      - "sudo"
      - "mkfs"

  # 网络
  network:
    outbound:
      allow:
        - "github.com:443"
        - "api.openai.com:443"
        - "*.crates.io:443"
      deny:
        - "127.0.0.1:*"            # 禁止访问本地服务（可选）
        - "169.254.*"              # 禁止云元数据端点
    inbound: false                 # 不允许监听

  # 环境
  env:
    read:
      - "PATH"
      - "HOME"
      - "LANG"
    write: []                      # 不允许修改环境变量

  # 系统资源限制
  limits:
    max_memory_mb: 256
    max_cpu_seconds: 30
    max_file_size_mb: 10
    max_processes: 5

# ===== Tool 定义（仅 type: tool 需要）=====
tools:
  - name: "shell_exec"
    description: "Execute a shell command and return output"
    parameters:
      type: "object"
      properties:
        command:
          type: "string"
          description: "The shell command to execute"
        workdir:
          type: "string"
          description: "Working directory for the command"
        timeout:
          type: "integer"
          description: "Timeout in seconds (default 30)"
          default: 30
      required: ["command"]

  - name: "shell_which"
    description: "Find the location of an executable"
    parameters:
      type: "object"
      properties:
        name:
          type: "string"
          description: "Executable name to find"
      required: ["name"]

# ===== Hook 定义（仅 type: hook 需要）=====
hooks:
  - event: "on_message"
    priority: 100  # 数值越低越先执行
  - event: "on_startup"
  - event: "on_shutdown"

# ===== 配置 Schema（用户可配置项）=====
config_schema:
  type: "object"
  properties:
    default_shell:
      type: "string"
      default: "/bin/bash"
      description: "Default shell interpreter"
    allowed_users:
      type: "array"
      items:
        type: "string"
      default: []
      description: "List of user IDs allowed to use this tool (empty = all)"
```

### 6.3 插件 API 设计

#### Rust 原生插件（动态库）

```rust
/// 插件 trait（Rust 原生）
pub trait IronClawPlugin: Send + Sync {
    /// 插件元数据
    fn metadata(&self) -> PluginMetadata;

    /// 初始化
    fn init(&mut self, ctx: PluginContext) -> Result<()>;

    /// 关闭
    fn shutdown(&mut self) -> Result<()>;
}

/// Channel 插件 trait
pub trait ChannelPlugin: IronClawPlugin {
    /// 启动消息接收循环
    fn start_listening(
        &self,
        sender: mpsc::Sender<IncomingMessage>,
    ) -> JoinHandle<Result<()>>;

    /// 发送消息
    fn send_message(&self, msg: OutgoingMessage) -> Result<()>;

    /// 编辑消息
    fn edit_message(&self, msg_id: &str, new_text: &str) -> Result<()>;

    /// 删除消息
    fn delete_message(&self, msg_id: &str) -> Result<()>;

    /// 支持的反应/表情
    fn supported_reactions(&self) -> Vec<String>;
}

/// Tool 插件 trait
pub trait ToolPlugin: IronClawPlugin {
    /// 获取工具定义列表
    fn tool_definitions(&self) -> Vec<ToolDefinition>;

    /// 执行工具调用
    fn execute_tool(
        &self,
        call: ToolCall,
        ctx: ToolExecutionContext,
    ) -> Result<ToolResult>;
}

/// Hook 插件 trait
pub trait HookPlugin: IronClawPlugin {
    /// 处理事件
    fn on_event(&self, event: HookEvent) -> Result<HookAction>;
}

/// 插件上下文（注入给插件使用）
pub struct PluginContext {
    /// Workspace 访问
    pub workspace: WorkspaceHandle,
    /// HTTP 客户端（受权限控制）
    pub http: PermissionAwareHttpClient,
    /// 配置
    pub config: serde_json::Value,
    /// 日志
    pub log: tracing::Span,
}

/// 插件入口宏（简化导出）
#[macro_export]
macro_rules! declare_plugin {
    ($plugin_type:ty) => {
        #[no_mangle]
        pub extern "C" fn ironclaw_create_plugin() -> *mut dyn $crate::IronClawPlugin {
            let plugin = Box::new(<$plugin_type>::default());
            Box::into_raw(plugin)
        }
    };
}

// 使用示例
pub struct ShellTool;

impl IronClawPlugin for ShellTool { /* ... */ }
impl ToolPlugin for ShellTool { /* ... */ }

declare_plugin!(ShellTool);
```

#### Python 插件 API

```python
# ironclaw_runner/plugin_base.py
"""IronClaw Python 插件基类"""

from abc import ABC, abstractmethod
from typing import Any, Optional
from dataclasses import dataclass


@dataclass
class ToolResult:
    """工具执行结果"""
    success: bool
    output: str
    error: Optional[str] = None
    metadata: Optional[dict] = None


class IronClawTool(ABC):
    """Tool 插件基类"""

    @abstractmethod
    def name(self) -> str:
        """工具名称"""
        ...

    @abstractmethod
    def description(self) -> str:
        """工具描述"""
        ...

    @abstractmethod
    def parameters(self) -> dict:
        """JSON Schema 格式的参数定义"""
        ...

    @abstractmethod
    def execute(self, **kwargs) -> ToolResult:
        """
        执行工具调用

        Args:
            **kwargs: 工具参数

        Returns:
            ToolResult 执行结果

        Raises:
            PermissionError: 权限不足
            ToolExecutionError: 执行失败
        """
        ...

    def on_error(self, error: Exception) -> ToolResult:
        """错误回调（可选覆盖）"""
        return ToolResult(success=False, output="", error=str(error))


class IronClawChannel(ABC):
    """Channel 插件基类"""

    @abstractmethod
    def connect(self, config: dict) -> None:
        """连接到消息平台"""
        ...

    @abstractmethod
    def send_message(self, chat_id: str, text: str, **kwargs) -> str:
        """发送消息，返回 message_id"""
        ...

    @abstractmethod
    def listen(self) -> Iterator[dict]:
        """监听消息（生成器）"""
        ...


class IronClawHook(ABC):
    """Hook 插件基类"""

    @abstractmethod
    def event_name(self) -> str:
        """监听的事件名"""
        ...

    @abstractmethod
    def handle(self, event: dict) -> Optional[dict]:
        """处理事件，返回修改后的事件或 None"""
        ...
```

#### Python 插件使用示例

```python
# plugins/web_search/main.py
"""网页搜索工具示例"""

from ironclaw_runner.plugin_base import IronClawTool, ToolResult
import requests


class WebSearchTool(IronClawTool):
    def name(self) -> str:
        return "web_search"

    def description(self) -> str:
        return "Search the web using DuckDuckGo"

    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "Search query"
                },
                "max_results": {
                    "type": "integer",
                    "description": "Maximum results",
                    "default": 5
                }
            },
            "required": ["query"]
        }

    def execute(self, **kwargs) -> ToolResult:
        query = kwargs["query"]
        max_results = kwargs.get("max_results", 5)

        # 网络请求受 manifest 中的 network 权限控制
        # ironclaw_runner 会自动检查权限
        resp = requests.get(
            "https://api.duckduckgo.com/",
            params={"q": query, "format": "json"}
        )

        results = resp.json().get("RelatedTopics", [])[:max_results]
        output = "\n".join(
            f"- {r['Text']}" for r in results if "Text" in r
        )

        return ToolResult(success=True, output=output)


# 插件入口（必须）
PLUGIN = WebSearchTool
```

### 6.4 PyO3 插件加载流程

```
 ┌─────────────────────────────────────────────────────────────┐
 │                    Python 插件加载流程                        │
 │                                                              │
 │  ① Gateway 启动，扫描 plugins/ 目录                          │
 │     └── 发现 web_search/plugin.yaml → runtime: "python"     │
 │                                                              │
 │  ② 解析 plugin.yaml                                          │
 │     ├── 验证 permissions 声明                                 │
 │     ├── 注册 tool definitions 到 ToolRegistry                │
 │     └── 记录权限需求                                          │
 │                                                              │
 │  ③ 启动 Python Runner 子进程                                 │
 │     ironclaw-plugin-runner \                                 │
 │       --socket /tmp/ironclaw-py-XXXX.sock \                  │
 │       --plugin /path/to/web_search \                         │
 │       --manifest plugin.yaml                                 │
 │                                                              │
 │  ④ Runner 内部                                               │
 │     ├── 加载 main.py                                         │
 │     ├── 实例化 PLUGIN 类                                     │
 │     ├── 通过 Unix Socket 建立 JSON-RPC 服务                   │
 │     └── 发送 Ready 信号                                      │
 │                                                              │
 │  ⑤ Gateway 通过 JSON-RPC 调用插件                            │
 │     Request:  {"method": "execute", "params": {...}}         │
 │     Response: {"result": {"success": true, "output": "..."}} │
 │                                                              │
 │  ⑥ 健康检查                                                  │
 │     Gateway 定期 ping Runner → 无响应则重启                   │
 │                                                              │
 └─────────────────────────────────────────────────────────────┘
```

```rust
/// JSON-RPC IPC 协议（Gateway ↔ Python Runner）
#[derive(Debug, Serialize, Deserialize)]
pub struct JsonRpcRequest {
    pub jsonrpc: String,          // "2.0"
    pub id: u64,
    pub method: String,
    pub params: serde_json::Value,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct JsonRpcResponse {
    pub jsonrpc: String,
    pub id: u64,
    pub result: Option<serde_json::Value>,
    pub error: Option<JsonRpcError>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct JsonRpcError {
    pub code: i32,
    pub message: String,
    pub data: Option<serde_json::Value>,
}

// IPC 方法列表
// ─────────────────
// plugin.info       → 返回插件元数据
// tool.list         → 返回工具定义列表
// tool.execute      → 执行工具调用
// channel.send      → 发送消息
// channel.listen    → 开始监听（streaming）
// hook.handle       → 处理钩子事件
// ping              → 健康检查
// shutdown          → 优雅关闭
```

### 6.5 权限控制实现

```rust
/// 权限检查器
pub struct PermissionChecker {
    manifest: PluginManifest,
    fs_matcher: GlobSet,
    exec_matcher: CommandMatcher,
    network_matcher: NetworkMatcher,
}

impl PermissionChecker {
    /// 检查文件读取权限
    pub fn check_fs_read(&self, path: &Path) -> Result<()> {
        let resolved = self.resolve_path(path)?;
        let policy = &self.manifest.permissions.fs;

        // 检查是否在 read 白名单
        if !self.fs_matcher.is_match(
            &resolved,
            &policy.read,
        ) {
            return Err(PermissionError::FsReadDenied {
                path: resolved,
                allowed: policy.read.clone(),
            });
        }

        Ok(())
    }

    /// 检查命令执行权限
    pub fn check_exec(&self, command: &str) -> Result<()> {
        let policy = &self.manifest.permissions.exec;

        if !policy.allow {
            return Err(PermissionError::ExecDenied);
        }

        // 先检查 deny list（优先级更高）
        if self.exec_matcher.matches_any(command, &policy.deny) {
            return Err(PermissionError::ExecDeniedCommand {
                command: command.to_string(),
            });
        }

        // 再检查 allow list
        if !self.exec_matcher.matches_any(command, &policy.commands) {
            return Err(PermissionError::ExecNotWhitelisted {
                command: command.to_string(),
                allowed: policy.commands.clone(),
            });
        }

        Ok(())
    }

    /// 检查网络请求权限
    pub fn check_network_outbound(&self, host: &str, port: u16) -> Result<()> {
        let policy = &self.manifest.permissions.network;
        let target = format!("{}:{}", host, port);

        // deny 优先
        if self.network_matcher.matches_any(&target, &policy.outbound.deny) {
            return Err(PermissionError::NetworkDenied { target });
        }

        // 如果有 allow list 且非空，检查是否匹配
        if !policy.outbound.allow.is_empty() {
            if !self.network_matcher.matches_any(&target, &policy.outbound.allow) {
                return Err(PermissionError::NetworkNotWhitelisted {
                    target,
                    allowed: policy.outbound.allow.clone(),
                });
            }
        }

        Ok(())
    }
}

/// 带权限的 HTTP 客户端（注入给插件）
pub struct PermissionAwareHttpClient {
    client: reqwest::Client,
    checker: Arc<PermissionChecker>,
}

impl PermissionAwareHttpClient {
    pub async fn get(&self, url: &str) -> Result<reqwest::Response> {
        let parsed = Url::parse(url)?;
        self.checker.check_network_outbound(
            parsed.host_str().unwrap_or(""),
            parsed.port().unwrap_or(443),
        )?;
        Ok(self.client.get(url).send().await?)
    }
}
```

---

## 7. 部署方案

### 7.1 配置文件格式（TOML）

```toml
# ~/.config/ironclaw/config.toml — IronClaw 主配置

[server]
# Gateway 监听地址
host = "0.0.0.0"
port = 8443
# TLS（可选，用于生产环境）
# cert_path = "/etc/ironclaw/cert.pem"
# key_path = "/etc/ironclaw/key.pem"

[agent]
# 默认 LLM 配置
default_model = "gpt-4o"
max_tool_rounds = 10
token_budget = 128000

[agent.context]
# 自动注入到上下文的 workspace 文件
inject_files = [
    "AGENTS.md",
    "SOUL.md",
    "USER.md",
    "MEMORY.md",
]
# 最大历史消息数
max_history = 100

[llm]
# Provider 配置（支持多个）
[llm.openai]
api_key = "${OPENAI_API_KEY}"     # 支持环境变量引用
base_url = "https://api.openai.com/v1"
models = ["gpt-4o", "gpt-4o-mini", "o1", "o3"]

[llm.anthropic]
api_key = "${ANTHROPIC_API_KEY}"
base_url = "https://api.anthropic.com"
models = ["claude-sonnet-4-20250514"]

[llm.custom]
api_key = "${CUSTOM_API_KEY}"
base_url = "https://my-llm.example.com/v1"
models = ["my-model"]

[workspace]
# Workspace 根目录
root = "~/.local/share/ironclaw/workspaces"
# 默认 workspace
default = "default"
# 沙箱策略
sandbox = "strict"  # strict | permissive | disabled

[node]
# 节点通信配置
protocol = "quic"     # quic | websocket
heartbeat_interval = 30   # 秒
max_idle_timeout = 120    # 秒
# 配对码有效期
pair_code_ttl = 600       # 秒

[plugins]
# 插件搜索路径
search_paths = [
    "~/.local/share/ironclaw/plugins",
    "/usr/lib/ironclaw/plugins",
]
# 是否允许自动加载（未在配置中显式启用的插件）
auto_load = true
# Python Runner 配置
[plugins.python]
runner_path = "/usr/lib/ironclaw/python-runner"
max_processes = 4
max_memory_mb = 512

[logging]
level = "info"         # trace | debug | info | warn | error
format = "json"        # json | pretty | compact
output = "file"        # stdout | file | both
file_path = "~/.local/share/ironclaw/logs/ironclaw.log"
rotation = "daily"     # daily | hourly | size
max_files = 30

[database]
path = "~/.local/share/ironclaw/data.db"
# WAL 模式（推荐用于并发）
journal_mode = "wal"
# 连接池
max_connections = 10
```

### 7.2 一键安装脚本

```bash
#!/bin/bash
# install.sh — IronClaw 一键安装脚本
# 用法: curl -fsSL https://ironclaw.dev/install.sh | sh

set -euo pipefail

VERSION="${IRONCLAW_VERSION:-latest}"
INSTALL_DIR="${IRONCLAW_INSTALL_DIR:-/usr/local/bin}"
PLATFORM="$(uname -s)_$(uname -m)"

# 颜色输出
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

info()  { echo -e "${GREEN}[INFO]${NC} $*"; }
warn()  { echo -e "${YELLOW}[WARN]${NC} $*"; }
error() { echo -e "${RED}[ERROR]${NC} $*"; exit 1; }

# 检查系统
check_system() {
    case "$(uname -s)" in
        Linux)  echo "linux" ;;
        Darwin) echo "macos" ;;
        *)      error "不支持的操作系统: $(uname -s)" ;;
    esac
}

# 检查架构
check_arch() {
    case "$(uname -m)" in
        x86_64|amd64)   echo "x86_64" ;;
        aarch64|arm64)  echo "aarch64" ;;
        *)              error "不支持的架构: $(uname -m)" ;;
    esac
}

# 下载
download() {
    local os="$1" arch="$2"
    local url="https://github.com/example/ironclaw/releases"

    if [ "$VERSION" = "latest" ]; then
        url="${url}/latest/download/ironclaw-${os}-${arch}"
    else
        url="${url}/download/${VERSION}/ironclaw-${os}-${arch}"
    fi

    info "下载 IronClaw ${VERSION} (${os}-${arch})..."
    curl -fsSL "$url" -o "${INSTALL_DIR}/ironclaw" || \
        error "下载失败"

    chmod +x "${INSTALL_DIR}/ironclaw"
    info "已安装到 ${INSTALL_DIR}/ironclaw"
}

# 初始化
init_config() {
    local config_dir="${HOME}/.config/ironclaw"
    local data_dir="${HOME}/.local/share/ironclaw"

    mkdir -p "$config_dir" "$data_dir/workspaces" "$data_dir/plugins" "$data_dir/logs"

    if [ ! -f "$config_dir/config.toml" ]; then
        ironclaw config init --output "$config_dir/config.toml"
        info "配置文件已生成: $config_dir/config.toml"
        warn "请编辑配置文件，添加 LLM API Key"
    fi
}

# 主流程
main() {
    info "IronClaw 安装程序"
    echo "─────────────────────────────"

    local os arch
    os=$(check_system)
    arch=$(check_arch)

    # 检查依赖
    command -v curl >/dev/null || error "需要 curl"
    command -v sqlite3 >/dev/null || warn "建议安装 SQLite3"

    # 下载
    download "$os" "$arch"

    # 验证
    ironclaw --version || error "安装验证失败"

    # 初始化配置
    init_config

    echo ""
    info "✅ IronClaw 安装完成！"
    echo ""
    echo "下一步："
    echo "  1. 编辑配置: ~/.config/ironclaw/config.toml"
    echo "  2. 启动服务: ironclaw start"
    echo "  3. 查看状态: ironclaw status"
    echo ""
    echo "文档: https://ironclaw.dev/docs"
}

main "$@"
```

### 7.3 systemd 服务

```ini
# /etc/systemd/system/ironclaw.service
[Unit]
Description=IronClaw AI Agent Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=ironclaw
Group=ironclaw
ExecStart=/usr/local/bin/ironclaw start --foreground
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5
StartLimitBurst=3
StartLimitIntervalSec=60

# 安全加固
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/home/ironclaw/.local/share/ironclaw
ReadOnlyPaths=/etc/ironclaw
PrivateTmp=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictNamespaces=true
RestrictRealtime=true
MemoryMax=1G
CPUQuota=200%

# 环境变量
Environment=IRONCLAW_LOG_LEVEL=info
Environment=RUST_LOG=ironclaw=info
EnvironmentFile=-/etc/ironclaw/env

# 日志
StandardOutput=journal
StandardError=journal
SyslogIdentifier=ironclaw

[Install]
WantedBy=multi-user.target
```

```bash
# 安装服务
sudo cp ironclaw.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable ironclaw
sudo systemctl start ironclaw

# 查看状态
sudo systemctl status ironclaw

# 查看日志
journalctl -u ironclaw -f
```

### 7.4 可选 Docker 部署

```dockerfile
# Dockerfile — 多阶段构建
FROM rust:1.85-bookworm AS builder

WORKDIR /app
COPY . .

# 静态链接 SQLite
ENV SQLITE3_STATIC=1
RUN cargo build --release --bin ironclaw

# ──────────────────────────────
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    sqlite3 \
    && rm -rf /var/lib/apt/lists/*

# 创建非 root 用户
RUN useradd -m -s /bin/bash ironclaw

COPY --from=builder /app/target/release/ironclaw /usr/local/bin/
COPY --from=builder /app/python-runner /opt/ironclaw/python-runner

USER ironclaw
WORKDIR /home/ironclaw

EXPOSE 8443

VOLUME ["/home/ironclaw/.local/share/ironclaw"]
VOLUME ["/home/ironclaw/.config/ironclaw"]

ENTRYPOINT ["ironclaw"]
CMD ["start", "--foreground"]
```

```yaml
# docker-compose.yml
version: "3.8"

services:
  ironclaw:
    build: .
    container_name: ironclaw
    restart: unless-stopped
    ports:
      - "8443:8443"
    volumes:
      - ./data:/home/ironclaw/.local/share/ironclaw
      - ./config:/home/ironclaw/.config/ironclaw
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - IRONCLAW_LOG_LEVEL=info
    healthcheck:
      test: ["CMD", "ironclaw", "healthcheck"]
      interval: 30s
      timeout: 5s
      retries: 3
```

---

## 8. 开源社区考量

### 8.1 Crate 发布策略

```
crate 发布层级：

crates.io (公开)
├── ironclaw-core          v0.1.0    # 核心 lib（稳定后发布）
├── ironclaw-plugins       v0.1.0    # 插件系统 lib
├── ironclaw-config        v0.1.0    # 配置管理 lib
├── ironclaw-db            v0.1.0    # 数据库层 lib
└── ironclaw               v0.1.0    # CLI binary

暂不发布到 crates.io（开发期）：
├── ironclaw-gateway       # Gateway 服务器（内部）
├── ironclaw-node          # Node 客户端（内部）
└── ironclaw-plugin-*      # 官方插件（独立仓库）
```

发布节奏：
- **v0.1.x**：内部预览，API 可能随时变
- **v0.2.x**：Alpha，API 基本稳定，接受社区反馈
- **v0.3~0.9**：Beta，功能完善，文档齐全
- **v1.0**：正式发布，保证 API 兼容性

### 8.2 插件生态引导

```
插件发现与分发：

ironclaw plugin search <keyword>     # 搜索插件（从 registry）
ironclaw plugin install <name>       # 安装插件
ironclaw plugin install <git-url>    # 从 Git 安装
ironclaw plugin list                 # 列出已安装
ironclaw plugin update [<name>]      # 更新插件
ironclaw plugin remove <name>        # 卸载插件
ironclaw plugin create <name>        # 创建插件脚手架
ironclaw plugin validate             # 验证 plugin.yaml

插件 Registry（未来）：
- 类似 crates.io 的中心化仓库
- plugin.yaml + 代码打包
- 版本管理与签名验证
```

插件开发脚手架：

```bash
$ ironclaw plugin create my-tool --type tool --runtime python

Creating plugin 'my-tool'...
├── plugin.yaml          # 权限 manifest（需编辑）
├── main.py              # Python 入口
├── README.md            # 文档
└── tests/
    └── test_main.py     # 测试

Next steps:
  1. Edit plugin.yaml to declare permissions
  2. Implement your tool in main.py
  3. Test: ironclaw plugin validate .
  4. Install: ironclaw plugin install .
```

### 8.3 文档结构

```
docs/
├── getting-started/
│   ├── installation.md         # 安装指南
│   ├── quick-start.md          # 5 分钟上手
│   └── configuration.md        # 配置详解
│
├── architecture/
│   ├── overview.md             # 架构概览
│   ├── agent-loop.md           # Agent Loop 详解
│   ├── workspace.md            # Workspace 设计
│   ├── gateway-node.md         # 通信架构
│   └── plugin-system.md        # 插件系统
│
├── plugin-development/
│   ├── rust-plugin.md          # Rust 插件开发
│   ├── python-plugin.md        # Python 插件开发
│   ├── manifest-reference.md   # plugin.yaml 参考
│   ├── permissions.md          # 权限系统详解
│   └── examples/               # 示例插件集合
│
├── deployment/
│   ├── standalone.md           # 单机部署
│   ├── systemd.md              # systemd 服务
│   ├── docker.md               # Docker 部署
│   └── security.md             # 安全加固
│
├── api-reference/
│   ├── rest-api.md             # REST API 文档
│   ├── websocket.md            # WebSocket 协议
│   └── cli-reference.md        # CLI 命令参考
│
└── contributing/
    ├── guide.md                # 贡献指南
    ├── code-of-conduct.md      # 行为准则
    └── architecture-decisions/ # ADR 记录
        ├── 001-use-axum.md
        ├── 002-quic-for-nodes.md
        └── 003-manifest-permissions.md
```

文档工具：
- **mdBook**（Rust 社区标准）— 生成在线文档站
- **rustdoc** — API 文档自动生成
- **comrak** — Markdown 渲染（CLI 帮助）

---

## 9. 与 OpenClaw 的对比

### 9.1 直接借鉴的设计

| OpenClaw 设计 | IronClaw 对应 | 说明 |
|:---|:---|:---|
| Workspace 目录 | `WorkspaceManager` | 几乎照搬，包括 AGENTS.md / SOUL.md / MEMORY.md 等约定 |
| Gateway + Node 架构 | `ironclaw-gateway` + `ironclaw-node` | 中心 Gateway 管理连接，Node 可远程配对 |
| Channel Plugin | `ChannelPlugin` trait | 消息平台抽象，统一接口 |
| Tool Calling Loop | `AgentLoop` | 多轮 tool calling，上下文管理 |
| Session 管理 | `SessionManager` | 基于用户+频道的会话隔离 |
| 配置分层 | TOML 分层加载 | 全局 > 用户 > 项目 > 环境变量 |
| 流式响应 | `ChatDelta` stream | SSE 流式，支持中间输出 |

### 9.2 改进之处

| 维度 | OpenClaw (Node.js) | IronClaw (Rust) | 改进幅度 |
|:---|:---|:---|:---|
| **启动速度** | ~500ms（Node.js 冷启动） | ~10ms（原生 binary） | **50x** |
| **内存占用** | ~150MB（空载 V8） | ~15MB（空载） | **10x** |
| **并发性能** | 单线程 event loop | Tokio M:N 调度器 | **显著** |
| **部署** | 需要 Node.js + npm | 单 binary | **大幅简化** |
| **权限** | 运行时检查，隐式 | 编译时 + 运行时 manifest | **更安全** |
| **插件隔离** | 进程内（crash 影响） | 子进程隔离 + 沙箱 | **更稳定** |
| **类型安全** | TypeScript（运行时擦除） | Rust（编译时保证） | **更可靠** |
| **二进制体积** | 需 node_modules | ~20MB（静态链接） | **更小** |
| **冷启动延迟** | 首次请求需 JIT | 无 JIT，即时执行 | **更快** |

### 9.3 风险与挑战

#### 🔴 高风险

| 风险 | 影响 | 缓解措施 |
|:---|:---|:---|
| **开发周期长** | Rust 开发效率低于 TypeScript，预计 3-6 个月 MVP | 先实现核心功能，插件系统分期交付 |
| **PyO3 复杂度** | Python 桥接的内存安全和错误处理 | 子进程隔离，IPC 而非嵌入式解释器 |
| **生态差距** | npm 生态远大于 Rust crates | 提供 FFI 桥接，优先 Python 插件 |

#### 🟡 中风险

| 风险 | 影响 | 缓解措施 |
|:---|:---|:---|
| **QUIC 穿透** | 部分网络环境可能阻止 UDP | 保留 WebSocket fallback |
| **动态库加载** | Rust ABI 不稳定，跨版本兼容困难 | 限定 Rust 版本，或改用 C ABI |
| **热重载** | Rust 不支持运行时重载代码 | 插件独立进程，主进程不重载 |

#### 🟢 低风险

| 风险 | 影响 | 缓解措施 |
|:---|:---|:---|
| **SQLite 并发** | WAL 模式下写入串行 | 写操作批量提交，读操作无锁 |
| **跨平台** | Windows 兼容性 | CI 测试 Windows/macOS/Linux |
| **社区接受度** | Rust 学习曲线 | 提供完善文档和脚手架 |

### 9.4 技术决策记录（ADR）

```
ADR-001: 选择 Axum 而非 Actix-web
  理由: Tokio 生态兼容性 > 理论性能差异

ADR-002: 选择 QUIC + WebSocket 双协议
  理由: QUIC 低延迟 + WebSocket 兼容性

ADR-003: Python 插件使用子进程而非 PyO3 嵌入
  理由: 隔离 crash、避免 GIL、简化依赖

ADR-004: 权限使用声明式 manifest
  理由: 安全审计可在安装时完成，而非运行时

ADR-005: SQLite 而非 PostgreSQL
  理由: 零配置、单机部署、减少依赖

ADR-006: TOML 而非 YAML 作为配置格式
  理由: Rust 生态偏好、类型安全、注释保留
```

---

## 10. 里程碑规划

```
Phase 1: 核心骨架 (Week 1-4)
├── 项目结构搭建 (Cargo workspace)
├── ironclaw-core: Message types, LLM provider trait
├── ironclaw-config: TOML 配置加载
├── ironclaw-cli: 基础 CLI (start/stop/status)
└── ironclaw-db: SQLite 初始化 + migrations

Phase 2: Agent 引擎 (Week 5-8)
├── Agent Loop: 多轮 tool calling
├── Context Manager: token 预算 + 压缩
├── LLM Router: OpenAI/Anthropic 适配
├── Workspace Manager: 文件读写 + 隔离
└── 内置 Tool: Shell, FileOps

Phase 3: Gateway 服务 (Week 9-12)
├── Axum HTTP 服务器 + REST API
├── WebSocket 长连接（消息平台）
├── Session Manager
├── Message Bus (broadcast)
└── Telegram Plugin (MVP)

Phase 4: 插件系统 (Week 13-16)
├── plugin.yaml 解析 + 验证
├── Rust 原生插件加载
├── Python Runner 子进程
├── 权限检查器实现
└── 插件脚手架 CLI

Phase 5: Node 架构 (Week 17-20)
├── QUIC 通信 (quinn)
├── 节点配对流程
├── 命令执行代理
├── 心跳与健康检查
└── WebSocket fallback

Phase 6: 生产化 (Week 21-24)
├── 安装脚本
├── systemd service
├── Docker 镜像
├── 日志系统完善
├── 性能基准测试
├── 安全审计
└── 文档完善

Phase 7: 社区准备 (Week 25-28)
├── crates.io 发布
├── 插件 Registry 原型
├── 贡献指南
├── 示例插件集合
└── 网站 + 文档站上线
```

---

## 附录

### A. 关键依赖版本

```toml
# 推荐依赖版本（2026-04）
rustc = "1.85"          # Edition 2024
tokio = "1.43"
axum = "0.8"
sqlx = "0.8"
serde = "1.0"
pyo3 = "0.23"
quinn = "0.11"
clap = "4.5"
tracing = "0.1"
reqwest = "0.12"
```

### B. 性能目标

| 指标 | 目标 | 测量方法 |
|:---|:---|:---|
| 冷启动时间 | < 50ms | `time ironclaw start` |
| 空载内存 | < 30MB | `ps aux \| grep ironclaw` |
| 消息处理延迟 | < 100ms (不含 LLM) | 端到端 benchmark |
| 并发连接数 | > 10,000 | WebSocket 压测 |
| 插件加载时间 | < 200ms | `ironclaw plugin install` |
| SQLite 写入 QPS | > 1,000 | `sqlx` benchmark |

### C. 安全检查清单

- [ ] 所有文件操作经过路径规范化（防目录穿越）
- [ ] 命令执行经过白名单检查（防命令注入）
- [ ] LLM 输出经过 sanitize（防 prompt injection）
- [ ] 插件 manifest 经过 schema 验证
- [ ] 节点通信使用 TLS 1.3 / QUIC 加密
- [ ] 配对码一次性使用 + 时效性
- [ ] 日志不包含敏感信息（API key, token）
- [ ] SQLite 使用参数化查询（防 SQL 注入）
- [ ] 子进程资源限制（rlimit / cgroup）
- [ ] 优雅关闭（drain connections）

---

> **文档状态**: Draft v0.1
> **下一步**: 团队评审 → Phase 1 开工
> **反馈**: 请在 GitHub Issues 中标记 `architecture` 标签
