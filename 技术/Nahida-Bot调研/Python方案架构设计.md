# Python AI Agent Bot 框架 — 架构设计文档

> **项目代号**: NahidaBot（暂定）
> **版本**: v0.1-draft
> **日期**: 2026-04-06
> **作者**: Renatus Madrigal + Nahida

---

## 目录

1. [项目愿景与目标](#1-项目愿景与目标)
2. [技术栈选择](#2-技术栈选择)
3. [项目目录结构](#3-项目目录结构)
4. [核心架构](#4-核心架构)
5. [Gateway-Node 通信](#5-gateway-node-通信)
6. [插件系统设计](#6-插件系统设计)
7. [部署方案](#7-部署方案)
8. [开源社区考量](#8-开源社区考量)
9. [竞品对比分析](#9-竞品对比分析)
10. [类型安全策略](#10-类型安全策略)
11. [路线图](#11-路线图)

---

## 1. 项目愿景与目标

### 1.1 为什么用 Python 重写？

OpenClaw（Node.js/TypeScript）证明了 AI Agent Bot 的可行架构，但存在以下痛点：

| 痛点 | 说明 |
|------|------|
| 生态壁垒 | Python 在 AI/ML 领域拥有绝对优势，LLM SDK、向量数据库、数据处理库远比 JS 丰富 |
| 部署复杂度 | Node.js 运行时 + pnpm + 系统依赖导致部署链路长 |
| 类型安全 | TypeScript 的运行时类型验证依赖第三方（zod），Python 有 pydantic 原生支持 |
| 学习曲线 | 目标用户（个人开发者、学生）更熟悉 Python |

### 1.2 核心目标

```
┌─────────────────────────────────────────────────┐
│                 核心设计目标                       │
├─────────────────────────────────────────────────┤
│  1. Workspace 隔离 — Agent 有独立工作目录          │
│  2. Gateway-Node 架构 — 中心调度 + 远程节点        │
│  3. 声明式插件系统 — 权限 manifest，类 Android     │
│  4. 一键部署 — pipx/uvx 安装即用                   │
│  5. 强大 Agent — tool calling、多轮、上下文管理     │
│  6. Pythonic — 充分利用 Python 生态和语言特性       │
└─────────────────────────────────────────────────┘
```

### 1.3 设计原则

- **Convention over Configuration**: 合理默认值，开箱即用
- **Progressive Complexity**: 简单场景简单用，复杂场景有能力支撑
- **Fail-Safe**: 永远不要因为插件崩溃导致核心宕机
- **Observability**: 结构化日志、健康检查、指标暴露

---

## 2. 技术栈选择

### 2.1 总览

| 层次 | 技术选型 | 理由 |
|------|----------|------|
| **语言** | Python 3.12+ | 性能足够、生态最强、match statement、type parameter syntax |
| **异步运行时** | asyncio (原生) | 无需额外框架，Python 标准库 |
| **Web 框架** | FastAPI | 自动 OpenAPI 文档、依赖注入、原生 async、Pydantic 深度集成 |
| **HTTP 客户端** | httpx (async) | 统一 sync/async API、HTTP/2 支持、超时控制 |
| **数据模型** | pydantic v2 | Rust 核心性能、JSON Schema 生成、严格模式 |
| **数据库** | SQLite + aiosqlite | 零配置、单文件、嵌入式、够用 |
| **配置管理** | pydantic-settings + TOML | 类型安全的配置、环境变量覆盖 |
| **包管理** | uv | 极速依赖解析、lockfile、virtualenv 管理 |
| **CLI 框架** | typer + rich | 声明式 CLI、美观输出 |
| **日志** | structlog | 结构化 JSON 日志、可配置输出格式 |
| **测试** | pytest + pytest-asyncio | 社区标准、异步支持好 |
| **类型检查** | pyright (strict) + mypy | CI 双重检查 |

### 2.2 关键决策说明

#### 为什么选 FastAPI 而不是 aiohttp？

```python
# FastAPI: 声明式、类型安全、自动文档
@app.post("/api/v1/messages")
async def handle_message(msg: InboundMessage) -> Response:
    ...

# aiohttp: 手动路由、无类型安全
routes = web.RouteTableDef()

@routes.post("/api/v1/messages")
async def handle_message(request: web.Request) -> web.Response:
    data = await request.json()  # 无类型
    ...
```

FastAPI 的优势：
1. 请求/响应自动校验（pydantic）
2. 自动生成 OpenAPI 文档（内置 API 测试界面）
3. 依赖注入系统（适合管理 DB session、配置等）
4. WebSocket 原生支持（Gateway-Node 通信需要）

#### 为什么选 SQLite 而不是 PostgreSQL？

- **目标用户是个人开发者**：不需要分布式数据库
- **零运维**：无需安装、配置、备份外部数据库
- **性能足够**：单机场景下 SQLite 的读性能优于网络数据库
- **迁移路径清晰**：通过抽象层，未来可切换到 PostgreSQL

```python
# 数据库抽象层设计
from abc import ABC, abstractmethod

class MessageStore(ABC):
    @abstractmethod
    async def save(self, message: Message) -> str: ...
    
    @abstractmethod
    async def get_history(self, session_id: str, limit: int) -> list[Message]: ...

class SQLiteMessageStore(MessageStore):
    """默认实现 — SQLite"""
    ...

class PostgresMessageStore(MessageStore):
    """未来扩展 — PostgreSQL"""
    ...
```

#### 为什么选 uv 而不是 poetry？

| 维度 | uv | poetry |
|------|-----|--------|
| 安装速度 | 10-100x 更快 | 较慢 |
| lockfile | uv.lock (通用) | poetry.lock |
| Python 管理 | 内置 python install | 需要额外工具 |
| 项目管理 | pyproject.toml | pyproject.toml |
| 成熟度 | 2024 起快速成熟 | 成熟稳定 |

**结论**: uv 速度快、功能覆盖全面，且由 Astral（ruff 团队）维护，发展势头强劲。对于新项目，优先 uv。

---

## 3. 项目目录结构

### 3.1 src layout

```
nadabot/
├── pyproject.toml                  # 项目元数据、依赖、构建配置
├── uv.lock                         # 锁定依赖版本
├── README.md
├── LICENSE
├── Makefile                        # 常用命令快捷方式
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                  # 测试 + 类型检查 + lint
│   │   ├── release.yml             # PyPI 发布
│   │   └── docs.yml                # 文档构建
│   └── dependabot.yml
│
├── docs/                           # MkDocs 文档
│   ├── mkdocs.yml
│   └── docs/
│       ├── index.md
│       ├── getting-started.md
│       ├── architecture.md
│       ├── plugin-development.md
│       └── api-reference.md
│
├── src/
│   └── nadabot/                    # 主包
│       ├── __init__.py             # 版本号 + 公开 API
│       ├── __main__.py             # python -m nadabot 入口
│       ├── py.typed                # PEP 561 标记
│       │
│       ├── core/                   # 核心模块
│       │   ├── __init__.py
│       │   ├── app.py              # Application 主类
│       │   ├── config.py           # pydantic-settings 配置
│       │   ├── events.py           # 事件系统定义
│       │   ├── exceptions.py       # 自定义异常
│       │   └── logging.py          # structlog 配置
│       │
│       ├── agent/                  # Agent 核心
│       │   ├── __init__.py
│       │   ├── loop.py             # Agent Loop 主循环
│       │   ├── context.py          # 上下文管理
│       │   ├── tools.py            # Tool 注册与调度
│       │   ├── memory.py           # 对话记忆管理
│       │   ├── prompt.py           # 系统提示词管理
│       │   └── providers/          # LLM Provider 抽象
│       │       ├── __init__.py
│       │       ├── base.py         # Provider 基类 / Protocol
│       │       ├── openai.py       # OpenAI 兼容 API
│       │       ├── anthropic.py    # Anthropic Claude
│       │       ├── gemini.py       # Google Gemini
│       │       └── registry.py     # Provider 注册表
│       │
│       ├── gateway/                # Gateway 服务
│       │   ├── __init__.py
│       │   ├── server.py           # FastAPI 应用
│       │   ├── router.py           # API 路由定义
│       │   ├── middleware.py       # 认证、限流、日志中间件
│       │   ├── node_manager.py     # 节点管理与心跳
│       │   └── session.py          # 会话管理
│       │
│       ├── workspace/              # 工作空间
│       │   ├── __init__.py
│       │   ├── manager.py          # 工作空间生命周期
│       │   ├── sandbox.py          # 文件操作沙盒
│       │   └── templates/          # 工作空间模板
│       │       ├── default/
│       │       │   ├── AGENTS.md
│       │       │   ├── SOUL.md
│       │       │   └── MEMORY.md
│       │       └── minimal/
│       │           └── AGENTS.md
│       │
│       ├── plugins/                # 插件系统
│       │   ├── __init__.py
│       │   ├── loader.py           # 插件加载器
│       │   ├── manager.py          # 插件生命周期管理
│       │   ├── manifest.py         # plugin.yaml 解析模型
│       │   ├── permissions.py      # 权限检查引擎
│       │   ├── registry.py         # 插件注册表
│       │   └── builtin/            # 内置插件
│       │       ├── __init__.py
│       │       ├── file_ops.py     # 文件读写工具
│       │       ├── shell.py        # 命令执行工具
│       │       ├── web.py          # 网页获取工具
│       │       └── memory.py       # 记忆搜索工具
│       │
│       ├── channels/               # Channel 插件基类
│       │   ├── __init__.py
│       │   ├── base.py             # Channel 抽象基类
│       │   └── types.py            # 消息类型定义
│       │
│       ├── node/                   # Node 客户端
│       │   ├── __init__.py
│       │   ├── client.py           # Node 客户端主类
│       │   ├── connector.py        # 连接管理
│       │   └── executor.py         # 远程命令执行
│       │
│       ├── db/                     # 数据层
│       │   ├── __init__.py
│       │   ├── engine.py           # 数据库引擎抽象
│       │   ├── models.py           # 数据模型（SQLModel / Pydantic）
│       │   ├── migrations.py       # 简易迁移（或用 alembic）
│       │   └── repositories/       # 仓储模式
│       │       ├── __init__.py
│       │       ├── messages.py
│       │       ├── sessions.py
│       │       └── plugins.py
│       │
│       └── cli/                    # 命令行界面
│           ├── __init__.py
│           └── main.py             # typer CLI 定义
│
├── tests/                          # 测试
│   ├── conftest.py                 # 共享 fixtures
│   ├── unit/
│   │   ├── test_agent.py
│   │   ├── test_plugins.py
│   │   └── test_workspace.py
│   ├── integration/
│   │   ├── test_gateway.py
│   │   └── test_node_connection.py
│   └── e2e/
│       └── test_full_flow.py
│
├── examples/                       # 示例插件
│   ├── telegram-channel/
│   │   ├── plugin.yaml
│   │   └── telegram_plugin.py
│   └── weather-tool/
│       ├── plugin.yaml
│       └── weather.py
│
└── scripts/
    ├── dev-setup.sh                # 开发环境初始化
    └── build.sh                    # 构建脚本
```

### 3.2 模块依赖关系

```
cli  ──→  core.app  ──→  gateway.server
               │              │
               ├──→  agent.loop  ├──→  node_manager
               │         │              │
               │         ├──→  providers     ├──→  node.client
               │         ├──→  tools         │
               │         └──→  context       │
               │                            │
               ├──→  plugins.manager  ←─────┘
               │         │
               │         ├──→  loader
               │         ├──→  permissions
               │         └──→  registry
               │
               ├──→  workspace.manager
               │
               └──→  db.engine
```

**依赖规则**:
- 外层模块可以依赖内层，内层不能依赖外层
- `core` 不依赖任何业务模块
- `plugins` 不依赖具体 `channels`
- `agent` 不依赖 `gateway`

---

## 4. 核心架构

### 4.1 Gateway 进程架构图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        NadaBot Gateway 进程                          │
│                                                                      │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐                │
│  │  Telegram    │   │  Discord    │   │  QQ/NapCat  │   ...         │
│  │  Channel     │   │  Channel    │   │  Channel    │                │
│  │  Plugin      │   │  Plugin     │   │  Plugin     │                │
│  └──────┬───────┘   └──────┬──────┘   └──────┬──────┘                │
│         │                  │                  │                       │
│         └──────────────────┼──────────────────┘                       │
│                            ▼                                         │
│  ┌──────────────────────────────────────────────┐                    │
│  │              Message Dispatcher               │                    │
│  │     (消息分发、预处理、权限检查)                  │                    │
│  └──────────────────────┬───────────────────────┘                    │
│                         │                                            │
│                         ▼                                            │
│  ┌──────────────────────────────────────────────┐                    │
│  │               Agent Loop                      │                    │
│  │                                                │                    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐    │                    │
│  │  │ Context  │  │  Tool    │  │  Memory  │    │                    │
│  │  │ Manager  │  │ Executor │  │ Manager  │    │                    │
│  │  └──────────┘  └──────────┘  └──────────┘    │                    │
│  │                                                │                    │
│  │  ┌──────────────────────────────────────────┐ │                    │
│  │  │          LLM Provider (httpx)            │ │                    │
│  │  │   OpenAI / Anthropic / Gemini / Local    │ │                    │
│  │  └──────────────────────────────────────────┘ │                    │
│  └──────────────────────┬───────────────────────┘                    │
│                         │                                            │
│              ┌──────────┼──────────┐                                 │
│              ▼          ▼          ▼                                  │
│  ┌──────────────┐ ┌──────────┐ ┌──────────────┐                     │
│  │   Workspace  │ │  Node    │ │   Plugin     │                     │
│  │   (文件系统)  │ │  Manager │ │   Registry   │                     │
│  └──────────────┘ └────┬─────┘ └──────────────┘                     │
│                        │                                             │
│  ┌─────────────────────┼─────────────────────────┐                  │
│  │         WebSocket / HTTP Server (FastAPI)       │                  │
│  └─────────────────────┼─────────────────────────┘                  │
│                        │                                             │
└────────────────────────┼─────────────────────────────────────────────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
     ┌────────────┐ ┌────────┐ ┌────────┐
     │  Node A    │ │ Node B │ │ Node C │
     │ (Windows)  │ │ (Mac)  │ │ (VPS)  │
     └────────────┘ └────────┘ └────────┘
```

### 4.2 消息流（Message Flow）

```
用户消息 (Telegram/QQ/Discord/...)
    │
    ▼
┌─────────────────── Channel Plugin ───────────────────┐
│  1. 接收原始消息（webhook / long polling）             │
│  2. 标准化为 InboundMessage                           │
│  3. 投递到 Message Dispatcher                         │
└────────────────────────┬──────────────────────────────┘
                         │
                         ▼
┌─────────────────── Dispatcher ───────────────────────┐
│  1. 查找会话 (session_id)                             │
│  2. 加载上下文 (context manager)                      │
│  3. 检查权限 (plugin permissions)                     │
│  4. 注入 workspace 文件 (AGENTS.md 等)               │
│  5. 调用 Agent Loop                                   │
└────────────────────────┬──────────────────────────────┘
                         │
                         ▼
┌─────────────────── Agent Loop ────────────────────────┐
│                                                        │
│  while not finished:                                   │
│    1. 构建 messages = system + history + user_msg      │
│    2. 调用 LLM Provider (stream=True)                  │
│    3. 解析响应：                                       │
│       ├── 纯文本 → 收集，继续                          │
│       ├── tool_call → 执行 tool，追加结果到 history     │
│       └── finish → 退出循环                            │
│    4. 保存对话到记忆                                   │
│                                                        │
└────────────────────────┬──────────────────────────────┘
                         │
                         ▼
┌─────────────────── Response ──────────────────────────┐
│  1. 格式化响应文本                                     │
│  2. 通过 Channel Plugin 发送                           │
│  3. （可选）流式发送 chunks                             │
└───────────────────────────────────────────────────────┘
```

### 4.3 核心代码示例

#### Application 主类

```python
# src/nadabot/core/app.py
from __future__ import annotations

import asyncio
from pathlib import Path
from typing import Optional

from nadabot.core.config import AppConfig
from nadabot.core.events import EventBus
from nadabot.agent.loop import AgentLoop
from nadabot.plugins.manager import PluginManager
from nadabot.workspace.manager import WorkspaceManager
from nadabot.db.engine import DatabaseEngine
from nadabot.gateway.server import GatewayServer
from nadabot.gateway.node_manager import NodeManager


class NadaBot:
    """NadaBot Application 主类 — 整个系统的入口和容器。"""

    def __init__(self, config: AppConfig) -> None:
        self.config = config
        self.event_bus = EventBus()
        self.db = DatabaseEngine(config.database)
        self.workspace = WorkspaceManager(
            root=Path(config.workspace.path),
            templates_dir=Path(__file__).parent.parent / "workspace" / "templates",
        )
        self.plugins = PluginManager(
            config=config.plugins,
            workspace=self.workspace,
            event_bus=self.event_bus,
        )
        self.agent = AgentLoop(
            config=config.agent,
            providers=config.agent.providers,
            plugin_manager=self.plugins,
            workspace=self.workspace,
        )
        self.node_manager = NodeManager(config=config.nodes)
        self.gateway = GatewayServer(
            config=config.gateway,
            agent=self.agent,
            plugins=self.plugins,
            node_manager=self.node_manager,
        )

    async def start(self) -> None:
        """启动所有子系统。"""
        await self.db.connect()
        await self.db.run_migrations()
        await self.plugins.load_all()
        await self.node_manager.start()
        await self.gateway.start()

    async def stop(self) -> None:
        """优雅关闭。"""
        await self.gateway.stop()
        await self.node_manager.stop()
        await self.plugins.unload_all()
        await self.db.disconnect()

    async def run(self) -> None:
        """主运行入口。"""
        try:
            await self.start()
            # 保持运行直到被中断
            await asyncio.Event().wait()
        finally:
            await self.stop()
```

#### 消息类型定义

```python
# src/nadabot/channels/types.py
from __future__ import annotations

from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Optional


class MessageType(str, Enum):
    TEXT = "text"
    IMAGE = "image"
    FILE = "file"
    VOICE = "voice"
    VIDEO = "video"
    STICKER = "sticker"
    INTERACTIVE = "interactive"


@dataclass(frozen=True)
class MediaAttachment:
    """媒体附件。"""
    url: str
    mime_type: str
    size_bytes: Optional[int] = None
    caption: Optional[str] = None


@dataclass(frozen=True)
class InboundMessage:
    """从消息平台接收的标准化消息。"""
    message_id: str
    channel_id: str          # 来源 channel 标识（如 "telegram"）
    chat_id: str             # 对话 ID
    sender_id: str           # 发送者 ID
    sender_name: str         # 发送者显示名
    text: str                # 消息文本
    message_type: MessageType = MessageType.TEXT
    media: list[MediaAttachment] = field(default_factory=list)
    reply_to: Optional[str] = None  # 引用的消息 ID
    metadata: dict[str, Any] = field(default_factory=dict)
    timestamp: float = 0.0


@dataclass(frozen=True)
class OutboundMessage:
    """发送到消息平台的标准化消息。"""
    chat_id: str
    text: str
    media: list[MediaAttachment] = field(default_factory=list)
    reply_to: Optional[str] = None
    interactive: Optional[dict[str, Any]] = None  # 按钮/选择器
    silent: bool = False
```

### 4.4 Agent Loop 详细设计

```python
# src/nadabot/agent/loop.py
from __future__ import annotations

import asyncio
import json
from typing import AsyncIterator

from nadabot.agent.context import ContextManager
from nadabot.agent.memory import MemoryManager
from nadabot.agent.providers.base import LLMProvider, Message, ToolCall
from nadabot.agent.tools import ToolExecutor
from nadabot.channels.types import InboundMessage, OutboundMessage
from nadabot.plugins.manager import PluginManager


class AgentLoop:
    """
    Agent 主循环 — 核心推理引擎。
    
    工作流:
    1. 构建上下文 (system prompt + workspace files + history + user message)
    2. 调用 LLM
    3. 如果返回 tool_call → 执行工具 → 追加结果 → 回到步骤 2
    4. 如果返回文本 → 发送响应 → 结束
    5. 如果超过 max_turns → 截断 → 发送摘要
    """

    def __init__(
        self,
        *,
        provider: LLMProvider,
        context: ContextManager,
        memory: MemoryManager,
        tools: ToolExecutor,
        plugin_manager: PluginManager,
        max_turns: int = 20,
        max_context_tokens: int = 128_000,
    ) -> None:
        self.provider = provider
        self.context = context
        self.memory = memory
        self.tools = tools
        self.plugins = plugin_manager
        self.max_turns = max_turns
        self.max_context_tokens = max_context_tokens

    async def run(
        self,
        message: InboundMessage,
        *,
        stream: bool = True,
    ) -> AsyncIterator[str]:
        """
        处理一条用户消息，yield 响应文本片段（流式）。
        """
        session_id = self._resolve_session(message)

        # 1. 加载上下文
        system_prompt = await self.context.build_system_prompt(
            workspace_files=["AGENTS.md", "SOUL.md", "USER.md"],
        )
        history = await self.memory.get_history(
            session_id=session_id,
            limit=50,
        )

        messages: list[Message] = [
            Message(role="system", content=system_prompt),
            *history,
            Message(role="user", content=self._format_user_message(message)),
        ]

        # 2. Agent Loop
        for turn in range(self.max_turns):
            # 截断上下文到 token 限制
            messages = self.context.truncate(messages, self.max_context_tokens)

            # 调用 LLM（流式）
            response_chunks: list[str] = []
            tool_calls: list[ToolCall] = []

            async for chunk in self.provider.chat(
                messages=messages,
                tools=self.tools.get_definitions(),
                stream=stream,
            ):
                if chunk.text:
                    response_chunks.append(chunk.text)
                    yield chunk.text
                if chunk.tool_calls:
                    tool_calls.extend(chunk.tool_calls)

            # 无 tool_call → 对话结束
            if not tool_calls:
                break

            # 3. 执行 tool calls
            assistant_content = "".join(response_chunks)
            messages.append(Message(
                role="assistant",
                content=assistant_content or None,
                tool_calls=tool_calls,
            ))

            for tc in tool_calls:
                result = await self.tools.execute(tc)
                messages.append(Message(
                    role="tool",
                    content=result.output,
                    tool_call_id=tc.id,
                ))

        # 4. 保存到记忆
        full_response = "".join(response_chunks)
        await self.memory.save_exchange(
            session_id=session_id,
            user_message=self._format_user_message(message),
            assistant_response=full_response,
        )

    def _resolve_session(self, message: InboundMessage) -> str:
        """从消息中解析 session ID。"""
        return f"{message.channel_id}:{message.chat_id}"

    def _format_user_message(self, message: InboundMessage) -> str:
        """将 InboundMessage 格式化为 LLM 可读的文本。"""
        parts = [message.text]
        for media in message.media:
            parts.append(f"\n[附件: {media.mime_type}]")
        return "\n".join(parts)
```

### 4.5 Context Manager — 上下文管理

```python
# src/nadabot/agent/context.py
from __future__ import annotations

from pathlib import Path
from typing import Optional

from nadabot.agent.providers.base import Message


class ContextManager:
    """
    管理发送给 LLM 的完整上下文。
    
    职责:
    - 组装 system prompt（从 workspace 文件读取）
    - Token 计数与截断
    - 上下文压缩策略
    """

    def __init__(
        self,
        workspace_root: Path,
        tokenizer_model: str = "cl100k_base",
    ) -> None:
        self.workspace_root = workspace_root
        self._tokenizer = None  # tiktoken 延迟加载

    async def build_system_prompt(
        self,
        workspace_files: list[str] | None = None,
    ) -> str:
        """从 workspace 文件构建 system prompt。"""
        parts: list[str] = []
        
        default_files = workspace_files or ["AGENTS.md", "SOUL.md", "USER.md"]
        
        for filename in default_files:
            filepath = self.workspace_root / filename
            if filepath.exists():
                content = filepath.read_text(encoding="utf-8")
                parts.append(f"## {filename}\n\n{content}")
        
        return "\n\n---\n\n".join(parts) if parts else "You are a helpful assistant."

    def truncate(
        self,
        messages: list[Message],
        max_tokens: int,
        strategy: str = "sliding_window",
    ) -> list[Message]:
        """
        截断消息列表以适应 token 限制。
        
        策略:
        - sliding_window: 保留 system prompt + 最近 N 条消息
        - summary: 用 LLM 摘要历史消息（未来实现）
        """
        if strategy == "sliding_window":
            # 保留第一条（system）和最后 N 条
            total = self._count_tokens(messages)
            if total <= max_tokens:
                return messages
            
            system_msg = messages[0] if messages[0].role == "system" else None
            conversation = messages[1:] if system_msg else messages[:]
            
            # 从最旧的消息开始移除
            while conversation and self._count_tokens(
                ([system_msg] if system_msg else []) + conversation
            ) > max_tokens:
                conversation.pop(0)
            
            return ([system_msg] if system_msg else []) + conversation
        
        raise ValueError(f"Unknown truncation strategy: {strategy}")

    def _count_tokens(self, messages: list[Message]) -> int:
        """粗略 token 计数。"""
        total = 0
        for msg in messages:
            total += len(msg.content or "") // 4  # 粗略估计
            total += 4  # 消息格式化开销
        return total
```

### 4.6 Workspace 文件系统设计

```
~/.nadabot/                          # 全局根目录
├── config.toml                      # 全局配置
├── data/
│   ├── nadabot.db                   # SQLite 数据库
│   └── logs/                        # 日志文件
│
├── workspaces/
│   └── default/                     # 默认工作空间
│       ├── AGENTS.md                # Agent 行为指令
│       ├── SOUL.md                  # 人格定义
│       ├── USER.md                  # 用户信息
│       ├── MEMORY.md                # 长期记忆
│       ├── TOOLS.md                 # 工具使用笔记
│       ├── SESSION-STATE.md         # 当前会话状态
│       ├── TODO.md                  # 待办事项
│       ├── memory/                  # 日志目录
│       │   ├── 2026-04-06.md
│       │   └── 2026-04-05.md
│       ├── skills/                  # 自定义技能
│       │   ├── weather/
│       │   │   └── SKILL.md
│       │   └── memory-setup/
│       │       └── SKILL.md
│       └── media/                   # 媒体文件
│
├── plugins/                         # 第三方插件
│   ├── telegram-channel/
│   │   ├── plugin.yaml
│   │   └── ...
│   └── weather-tool/
│       ├── plugin.yaml
│       └── ...
│
└── nodes/                           # 节点配置
    ├── work-laptop.json
    └── home-pc.json
```

**关键设计决策**:
- Workspace 与插件分离：workspace 是 agent 的"大脑"，plugins 是"四肢"
- 文件即配置：Markdown 文件既人类可读又 agent 可读
- 支持多 workspace：不同 persona/用途可以有不同的 workspace

```python
# src/nadabot/workspace/manager.py
from __future__ import annotations

import shutil
from pathlib import Path
from typing import Optional

from nadabot.core.exceptions import WorkspaceError


class WorkspaceManager:
    """管理 Agent 的工作空间。"""

    def __init__(self, root: Path, templates_dir: Path) -> None:
        self.root = root.resolve()
        self.templates_dir = templates_dir
        self.root.mkdir(parents=True, exist_ok=True)

    def init_workspace(self, name: str, template: str = "default") -> Path:
        """从模板初始化新工作空间。"""
        template_path = self.templates_dir / template
        workspace_path = self.root / name
        
        if workspace_path.exists():
            raise WorkspaceError(f"Workspace '{name}' already exists")
        
        shutil.copytree(template_path, workspace_path)
        return workspace_path

    def get_workspace(self, name: str = "default") -> Path:
        """获取工作空间路径。"""
        path = self.root / name
        if not path.exists():
            return self.init_workspace(name)
        return path

    def read_file(self, workspace: str, relative_path: str) -> str:
        """安全读取工作空间内的文件。"""
        ws = self.get_workspace(workspace)
        target = (ws / relative_path).resolve()
        
        # 安全检查：防止路径遍历
        if not str(target).startswith(str(ws.resolve())):
            raise WorkspaceError(f"Path traversal detected: {relative_path}")
        
        if not target.exists():
            raise WorkspaceError(f"File not found: {relative_path}")
        
        return target.read_text(encoding="utf-8")

    def write_file(self, workspace: str, relative_path: str, content: str) -> None:
        """安全写入文件到工作空间。"""
        ws = self.get_workspace(workspace)
        target = (ws / relative_path).resolve()
        
        if not str(target).startswith(str(ws.resolve())):
            raise WorkspaceError(f"Path traversal detected: {relative_path}")
        
        target.parent.mkdir(parents=True, exist_ok=True)
        target.write_text(content, encoding="utf-8")

    def list_files(self, workspace: str, pattern: str = "**/*") -> list[Path]:
        """列出工作空间内的文件。"""
        ws = self.get_workspace(workspace)
        return [p.relative_to(ws) for p in ws.glob(pattern) if p.is_file()]
```

---

## 5. Gateway-Node 通信

### 5.1 通信协议选择

| 协议 | 优势 | 劣势 | 适用场景 |
|------|------|------|----------|
| **WebSocket** | 原生支持、双向实时、HTTP 兼容 | 无消息边界、无内置重连 | ✅ 首选 |
| ZeroMQ | 高性能、多模式 | 额外依赖、NAT 穿透难 | 高吞吐场景 |
| gRPC | 强类型、代码生成 | 复杂、HTTP/2 必须 | 微服务间 |
| SSE | 简单、HTTP 原生 | 单向 | 仅推送 |

**选择: WebSocket**

理由：
1. FastAPI 原生支持（`@app.websocket`）
2. 双向通信满足 Gateway→Node 指令下发和 Node→Gateway 状态上报
3. 断线重连机制成熟
4. 跨防火墙/NAT 兼容性好（可走 HTTPS/WSS）

### 5.2 通信协议设计

```
┌─────────── Gateway ───────────┐          ┌──────────── Node ───────────┐
│                               │          │                              │
│  ws://gateway:8080/node/ws    │◄────────►│  WebSocket Client            │
│                               │          │                              │
│  消息类型:                     │          │  消息类型:                     │
│  - auth.request               │          │  - auth.response             │
│  - exec.command               │          │  - exec.result               │
│  - heartbeat.ping             │          │  - heartbeat.pong            │
│  - config.update              │          │  - status.report             │
│  - file.read                  │          │  - file.content              │
│  - screen.capture             │          │  - screen.image              │
│                               │          │                              │
└───────────────────────────────┘          └──────────────────────────────┘
```

#### 消息格式（JSON-RPC 2.0 风格）

```python
# 通信消息基类
from pydantic import BaseModel, Field
from typing import Any, Optional
from enum import Enum


class MessageType(str, Enum):
    # 认证
    AUTH_REQUEST = "auth.request"
    AUTH_RESPONSE = "auth.response"
    # 执行
    EXEC_COMMAND = "exec.command"
    EXEC_RESULT = "exec.result"
    # 心跳
    HEARTBEAT_PING = "heartbeat.ping"
    HEARTBEAT_PONG = "heartbeat.pong"
    # 状态
    STATUS_REPORT = "status.report"
    # 文件操作
    FILE_READ = "file.read"
    FILE_CONTENT = "file.content"
    FILE_WRITE = "file.write"
    FILE_WRITTEN = "file.written"
    # 截屏
    SCREEN_CAPTURE = "screen.capture"
    SCREEN_IMAGE = "screen.image"


class NodeMessage(BaseModel):
    """Node 通信消息。"""
    type: MessageType
    id: str = Field(default_factory=lambda: uuid4().hex[:16])
    payload: dict[str, Any] = Field(default_factory=dict)
    timestamp: float = Field(default_factory=time.time)
    error: Optional[str] = None
```

#### WebSocket 连接流程

```
Node                                    Gateway
  │                                        │
  │  ─── WebSocket Connect ──────────────► │
  │                                        │
  │  ◄── auth.request {challenge} ─────── │  1. Gateway 发送认证挑战
  │                                        │
  │  ─── auth.response {signature} ──────► │  2. Node 用 token 签名回应
  │                                        │
  │  ◄── auth.response {ok=true} ──────── │  3. 认证成功
  │                                        │
  │  ─── status.report {info} ───────────► │  4. Node 上报系统信息
  │                                        │
  │  ◄── heartbeat.ping ──────────────── │  5. 定期心跳
  │  ─── heartbeat.pong ────────────────► │
  │                                        │
  │  ◄── exec.command {cmd, id} ──────── │  6. Gateway 下发命令
  │  ─── exec.result {output, id} ──────► │  7. Node 返回结果
  │                                        │
```

#### Node 管理器实现

```python
# src/nadabot/gateway/node_manager.py
from __future__ import annotations

import asyncio
import time
from typing import Optional
from dataclasses import dataclass, field

from nadabot.core.events import EventBus


@dataclass
class NodeInfo:
    """已连接节点的信息。"""
    node_id: str
    name: str
    websocket: Any  # WebSocket 连接
    os_info: str = ""
    connected_at: float = field(default_factory=time.time)
    last_heartbeat: float = field(default_factory=time.time)
    capabilities: list[str] = field(default_factory=list)

    @property
    def is_alive(self) -> bool:
        """30 秒内有心跳视为存活。"""
        return time.time() - self.last_heartbeat < 30


class NodeManager:
    """管理所有已连接的 Node。"""

    def __init__(self, *, heartbeat_interval: float = 10.0) -> None:
        self._nodes: dict[str, NodeInfo] = {}
        self._heartbeat_interval = heartbeat_interval
        self._event_bus: Optional[EventBus] = None
        self._cleanup_task: Optional[asyncio.Task] = None

    async def start(self) -> None:
        """启动心跳和清理任务。"""
        self._cleanup_task = asyncio.create_task(self._cleanup_loop())

    async def stop(self) -> None:
        """停止管理器。"""
        if self._cleanup_task:
            self._cleanup_task.cancel()
        for node in self._nodes.values():
            await node.websocket.close()
        self._nodes.clear()

    def register(self, node: NodeInfo) -> None:
        """注册新节点。"""
        self._nodes[node.node_id] = node

    def unregister(self, node_id: str) -> None:
        """注销节点。"""
        self._nodes.pop(node_id, None)

    def get(self, node_id: str) -> Optional[NodeInfo]:
        """获取节点信息。"""
        return self._nodes.get(node_id)

    def get_alive_nodes(self) -> list[NodeInfo]:
        """获取所有存活节点。"""
        return [n for n in self._nodes.values() if n.is_alive]

    async def execute_command(
        self,
        node_id: str,
        command: str,
        timeout: float = 30.0,
    ) -> str:
        """在指定节点上执行命令。"""
        node = self._nodes.get(node_id)
        if not node or not node.is_alive:
            raise RuntimeError(f"Node {node_id} not available")

        request_id = uuid4().hex[:16]
        await node.websocket.send_json({
            "type": "exec.command",
            "id": request_id,
            "payload": {"command": command},
        })

        # 等待结果（简化，实际用 Future/Event）
        result = await self._wait_for_result(request_id, timeout)
        return result

    async def _cleanup_loop(self) -> None:
        """定期清理失联节点。"""
        while True:
            await asyncio.sleep(self._heartbeat_interval)
            dead = [nid for nid, n in self._nodes.items() if not n.is_alive]
            for nid in dead:
                self._nodes.pop(nid, None)
```

### 5.3 认证与安全

```python
# 认证流程使用 HMAC-SHA256 签名
import hmac
import hashlib

def verify_node_token(challenge: str, token: str, signature: str) -> bool:
    """验证 Node 的签名。"""
    expected = hmac.new(
        token.encode(),
        challenge.encode(),
        hashlib.sha256,
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

**安全措施**:
1. **Token 认证**: Node 连接时需提供预共享 token 的 HMAC 签名
2. **TLS 加密**: 生产环境强制 WSS（WebSocket over TLS）
3. **命令白名单**: 可配置 Node 允许执行的命令范围
4. **速率限制**: 每个连接的消息频率限制
5. **沙盒执行**: 可选在容器/沙盒中执行远程命令

---

## 6. 插件系统设计

### 6.1 插件类型

```
┌─────────────────────────────────────────────┐
│                插件类型                       │
├──────────────┬──────────────┬───────────────┤
│   Channel    │    Tool      │    Hook       │
│   消息通道    │   工具扩展    │  生命周期钩子  │
├──────────────┼──────────────┼───────────────┤
│  - Telegram  │  - shell     │  - on_startup │
│  - Discord   │  - web_fetch │  - on_message │
│  - QQ        │  - memory    │  - on_error   │
│  - WeChat    │  - image     │  - on_tool    │
│  - HTTP API  │  - browser   │  - on_shutdown│
│  - CLI       │  - custom... │  - cron       │
└──────────────┴──────────────┴───────────────┘
```

### 6.2 plugin.yaml Manifest 规范

```yaml
# plugin.yaml — 插件清单文件（类比 AndroidManifest.xml）

# === 元数据 ===
name: telegram-channel
version: "1.0.0"
description: "Telegram Bot Channel — 接收和发送 Telegram 消息"
author: "NadaBot Contributors"
license: MIT
homepage: "https://github.com/nadabot/plugins/telegram-channel"
min_nadabot_version: "0.1.0"

# === 类型 ===
type: channel  # channel | tool | hook

# === 权限声明 ===
permissions:
  # 网络权限
  - network.outbound:
      hosts:
        - "api.telegram.org"
        - "*.telegram.org"
      reason: "发送和接收 Telegram API 请求"
  
  # 文件系统权限
  - filesystem.read:
      paths:
        - "${workspace}/media/*"
      reason: "读取发送的媒体文件"
  
  - filesystem.write:
      paths:
        - "${workspace}/media/downloads/*"
      reason: "保存下载的媒体文件"
  
  # 系统权限
  - env.read:
      keys:
        - TELEGRAM_BOT_TOKEN
      reason: "读取 Bot API Token"

# === 依赖 ===
dependencies:
  python:
    - "python-telegram-bot>=20.0"
  plugins:
    - "memory-tool>=0.1.0"  # 可选依赖其他插件

# === 配置 schema ===
config:
  $schema: "https://nadabot.dev/schema/plugin-config/v1"
  properties:
    bot_token:
      type: string
      required: true
      secret: true
      description: "Telegram Bot API Token"
    
    allowed_chats:
      type: array
      items: { type: string }
      default: []
      description: "允许的 chat ID 列表（空=全部允许）"
    
    parse_mode:
      type: string
      enum: ["HTML", "MarkdownV2", "None"]
      default: "MarkdownV2"
    
    rate_limit:
      type: object
      properties:
        messages_per_minute:
          type: integer
          default: 30
        max_message_length:
          type: integer
          default: 4096

# === 入口 ===
entrypoint: "telegram_plugin:TelegramChannel"

# === 标签 ===
tags:
  - channel
  - messaging
  - telegram
```

### 6.3 插件 API 设计

#### Channel 基类

```python
# src/nadabot/channels/base.py
from __future__ import annotations

from abc import ABC, abstractmethod
from typing import AsyncIterator, Callable, Optional

from nadabot.channels.types import InboundMessage, OutboundMessage


class ChannelPlugin(ABC):
    """
    Channel 插件基类。
    
    所有消息平台适配器必须继承此类。
    """

    def __init__(self, config: dict) -> None:
        self.config = config
        self._message_handler: Optional[Callable] = None

    @abstractmethod
    async def start(self) -> None:
        """启动 channel（开始监听消息）。"""
        ...

    @abstractmethod
    async def stop(self) -> None:
        """停止 channel。"""
        ...

    @abstractmethod
    async def send(self, message: OutboundMessage) -> str:
        """
        发送消息到平台。
        
        Returns:
            发送成功后的平台消息 ID。
        """
        ...

    @abstractmethod
    async def edit(self, message_id: str, message: OutboundMessage) -> None:
        """编辑已发送的消息。"""
        ...

    @abstractmethod
    async def delete(self, message_id: str) -> None:
        """删除消息。"""
        ...

    def on_message(self, handler: Callable[[InboundMessage], Awaitable[None]]) -> None:
        """注册消息处理器。"""
        self._message_handler = handler

    async def _dispatch(self, message: InboundMessage) -> None:
        """内部方法：将收到的消息分发给 handler。"""
        if self._message_handler:
            await self._message_handler(message)
```

#### Tool 基类

```python
# src/nadabot/agent/tools.py
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Any, Optional


@dataclass(frozen=True)
class ToolDefinition:
    """工具的 OpenAI function calling 定义。"""
    name: str
    description: str
    parameters: dict[str, Any]  # JSON Schema


@dataclass
class ToolResult:
    """工具执行结果。"""
    output: str
    error: bool = False
    metadata: dict[str, Any] | None = None


class ToolPlugin(ABC):
    """
    Tool 插件基类。
    
    所有工具必须继承此类，并实现 definition 和 execute。
    """

    @abstractmethod
    def definition(self) -> ToolDefinition:
        """返回工具的 JSON Schema 定义（用于 LLM function calling）。"""
        ...

    @abstractmethod
    async def execute(self, **kwargs: Any) -> ToolResult:
        """
        执行工具。
        
        Args:
            **kwargs: LLM 传递的参数（已校验）
        
        Returns:
            工具执行结果
        """
        ...

    def validate_permissions(self) -> bool:
        """验证插件是否有执行所需权限。可选覆盖。"""
        return True


# === 内置工具示例 ===

class ReadFileTool(ToolPlugin):
    """读取工作空间中的文件。"""

    def __init__(self, workspace_root: Path) -> None:
        self.workspace_root = workspace_root

    def definition(self) -> ToolDefinition:
        return ToolDefinition(
            name="read_file",
            description="读取工作空间中的文件内容",
            parameters={
                "type": "object",
                "properties": {
                    "path": {
                        "type": "string",
                        "description": "文件路径（相对于工作空间根目录）",
                    },
                    "offset": {
                        "type": "integer",
                        "description": "起始行号（1-indexed）",
                    },
                    "limit": {
                        "type": "integer",
                        "description": "最大读取行数",
                    },
                },
                "required": ["path"],
            },
        )

    async def execute(self, *, path: str, offset: int = 1, limit: int = 2000) -> ToolResult:
        target = (self.workspace_root / path).resolve()
        
        # 安全检查
        if not str(target).startswith(str(self.workspace_root.resolve())):
            return ToolResult(output=f"Error: path traversal denied", error=True)
        
        if not target.exists():
            return ToolResult(output=f"Error: file not found: {path}", error=True)
        
        try:
            lines = target.read_text(encoding="utf-8").splitlines()
            selected = lines[offset - 1 : offset - 1 + limit]
            return ToolResult(output="\n".join(selected))
        except Exception as e:
            return ToolResult(output=f"Error reading file: {e}", error=True)
```

#### Hook 插件

```python
# src/nadabot/plugins/hooks.py
from __future__ import annotations

from enum import Enum
from typing import Any, Callable, Awaitable


class HookEvent(str, Enum):
    STARTUP = "startup"
    SHUTDOWN = "shutdown"
    BEFORE_MESSAGE = "before_message"
    AFTER_MESSAGE = "after_message"
    BEFORE_TOOL = "before_tool"
    AFTER_TOOL = "after_tool"
    ON_ERROR = "on_error"
    CRON = "cron"


class HookPlugin:
    """
    Hook 插件 — 监听生命周期事件。
    
    用装饰器注册钩子：
    
    class MyHook(HookPlugin):
        @hook("before_message")
        async def check_spam(self, message: InboundMessage) -> InboundMessage | None:
            if is_spam(message):
                return None  # 拦截消息
            return message  # 放行
    """

    def __init__(self) -> None:
        self._hooks: dict[HookEvent, list[Callable]] = {}
    
    def register(self, event: HookEvent, handler: Callable) -> None:
        """注册事件处理器。"""
        self._hooks.setdefault(event, []).append(handler)
    
    def hook(self, event: str):
        """装饰器：注册钩子。"""
        def decorator(fn):
            self.register(HookEvent(event), fn)
            return fn
        return decorator
    
    async def fire(self, event: HookEvent, *args: Any, **kwargs: Any) -> list[Any]:
        """触发事件，收集所有处理器的返回值。"""
        results = []
        for handler in self._hooks.get(event, []):
            result = await handler(*args, **kwargs)
            results.append(result)
        return results
```

### 6.4 插件加载器

```python
# src/nadabot/plugins/loader.py
from __future__ import annotations

import importlib
import importlib.util
import sys
from pathlib import Path
from typing import Type, Any

from nadabot.plugins.manifest import PluginManifest
from nadabot.core.exceptions import PluginLoadError


class PluginLoader:
    """
    插件加载器 — 负责从文件系统发现和加载插件。
    
    加载流程:
    1. 扫描 plugin.yaml
    2. 解析 manifest（校验权限、依赖）
    3. 动态导入 entrypoint 模块
    4. 实例化插件类
    5. 注入配置
    """

    def __init__(self, plugin_dirs: list[Path]) -> None:
        self.plugin_dirs = plugin_dirs
        self._loaded_modules: dict[str, Any] = {}

    def discover(self) -> list[tuple[Path, PluginManifest]]:
        """发现所有插件目录。"""
        discovered = []
        for dir_path in self.plugin_dirs:
            if not dir_path.exists():
                continue
            for plugin_dir in dir_path.iterdir():
                if not plugin_dir.is_dir():
                    continue
                manifest_path = plugin_dir / "plugin.yaml"
                if manifest_path.exists():
                    manifest = PluginManifest.from_yaml(manifest_path)
                    discovered.append((plugin_dir, manifest))
        return discovered

    def load(self, plugin_path: Path, manifest: PluginManifest) -> Any:
        """加载单个插件。"""
        # 动态导入
        module_path, class_name = manifest.entrypoint.split(":")
        
        # 将插件目录添加到 sys.path
        if str(plugin_path) not in sys.path:
            sys.path.insert(0, str(plugin_path))
        
        module = importlib.import_module(module_path)
        plugin_class = getattr(module, class_name)
        
        return plugin_class

    def unload(self, module_name: str) -> None:
        """卸载插件模块。"""
        if module_name in sys.modules:
            del sys.modules[module_name]
```

### 6.5 权限检查引擎

```python
# src/nadabot/plugins/permissions.py
from __future__ import annotations

import fnmatch
from dataclasses import dataclass
from typing import Any

from nadabot.plugins.manifest import PluginManifest


@dataclass(frozen=True)
class PermissionRequest:
    """运行时权限请求。"""
    plugin_id: str
    permission: str        # 如 "filesystem.read"
    target: str            # 如 "/workspace/media/image.png"
    reason: str = ""


class PermissionEngine:
    """
    权限检查引擎。
    
    检查插件的运行时行为是否在其 manifest 声明的权限范围内。
    类比 Android 的权限系统：安装时声明，运行时检查。
    """

    def __init__(self) -> None:
        self._manifests: dict[str, PluginManifest] = {}
        self._granted: dict[str, set[str]] = {}  # plugin_id -> granted perms

    def register(self, plugin_id: str, manifest: PluginManifest) -> None:
        """注册插件 manifest。"""
        self._manifests[plugin_id] = manifest

    def check(self, request: PermissionRequest) -> bool:
        """
        检查权限请求是否被允许。
        
        示例:
        - manifest 声明了 filesystem.read: ["${workspace}/media/*"]
        - 运行时请求读取 /workspace/media/image.png
        - → 匹配 * 通配符 → 允许
        """
        manifest = self._manifests.get(request.plugin_id)
        if not manifest:
            return False

        for perm in manifest.permissions:
            if perm.name != request.permission:
                continue
            
            # 检查 target 是否在声明的范围内
            for allowed in perm.targets:
                pattern = self._expand_variables(allowed, request.plugin_id)
                if fnmatch.fnmatch(request.target, pattern):
                    return True
        
        return False

    def _expand_variables(self, pattern: str, plugin_id: str) -> str:
        """展开 manifest 中的变量。"""
        # ${workspace} → 实际工作空间路径
        # ${plugin_dir} → 插件目录
        # ${env:VAR_NAME} → 环境变量
        ...
```

### 6.6 热加载支持

```python
# src/nadabot/plugins/hot_reload.py
from __future__ import annotations

import asyncio
from pathlib import Path
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler, FileSystemEvent

from nadabot.plugins.manager import PluginManager
from nadabot.core.logging import get_logger

logger = get_logger(__name__)


class PluginReloader(FileSystemEventHandler):
    """监视插件目录变化，自动重新加载。"""

    def __init__(self, manager: PluginManager) -> None:
        self.manager = manager
        self._debounce: dict[str, asyncio.Task] = {}

    def on_modified(self, event: FileSystemEvent) -> None:
        if event.is_directory or not event.src_path.endswith(".py"):
            return
        
        plugin_dir = str(Path(event.src_path).parent)
        
        # 防抖：500ms 内同一目录的多次变更只触发一次
        if plugin_dir in self._debounce:
            self._debounce[plugin_dir].cancel()
        
        self._debounce[plugin_dir] = asyncio.create_task(
            self._reload_after_delay(plugin_dir, delay=0.5)
        )

    async def _reload_after_delay(self, plugin_dir: str, delay: float) -> None:
        await asyncio.sleep(delay)
        logger.info("hot_reload", plugin_dir=plugin_dir)
        try:
            await self.manager.reload_plugin(plugin_dir)
        except Exception as e:
            logger.error("hot_reload_failed", plugin_dir=plugin_dir, error=str(e))
        finally:
            self._debounce.pop(plugin_dir, None)
```

---

## 7. 部署方案

### 7.1 一键安装（pipx / uvx）

```bash
# 方式一：pipx（推荐给普通用户）
pipx install nadabot
nadabot init          # 初始化配置和工作空间
nadabot start         # 启动

# 方式二：uvx（零安装，临时运行）
uvx nadabot start

# 方式三：从源码
git clone https://github.com/nadabot/nadabot.git
cd nadabot
uv sync
uv run nadabot start
```

### 7.2 pyproject.toml 入口点配置

```toml
[project]
name = "nadabot"
version = "0.1.0"
description = "AI Agent Bot Framework — Pythonic, Pluginable, Powerful"
readme = "README.md"
license = "MIT"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.32",
    "httpx>=0.27",
    "pydantic>=2.10",
    "pydantic-settings>=2.6",
    "aiosqlite>=0.20",
    "typer>=0.12",
    "rich>=13.9",
    "structlog>=24.4",
    "pyyaml>=6.0",
    "watchdog>=6.0",
]

[project.scripts]
nadabot = "nadabot.cli.main:app"

[project.optional-dependencies]
telegram = ["python-telegram-bot>=20.0"]
discord = ["discord.py>=2.4"]
all = ["nadabot[telegram,discord]"]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.24",
    "pytest-cov>=6.0",
    "pyright>=1.1",
    "mypy>=1.13",
    "ruff>=0.8",
    "pre-commit>=4.0",
]
```

### 7.3 systemd 服务

```ini
# /etc/systemd/system/nadabot.service
[Unit]
Description=NadaBot AI Agent
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=nadabot
Group=nadabot
WorkingDirectory=/home/nadabot/.nadabot
ExecStart=/home/nadabot/.local/bin/nadabot start --daemon
Restart=on-failure
RestartSec=5
StartLimitBurst=5
StartLimitIntervalSec=60

# 安全加固
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/home/nadabot/.nadabot
PrivateTmp=true
Environment="NADABOT_LOG_LEVEL=info"
Environment="NADABOT_CONFIG=/home/nadabot/.nadabot/config.toml"

[Install]
WantedBy=multi-user.target
```

```bash
# 启用并启动
sudo systemctl daemon-reload
sudo systemctl enable nadabot
sudo systemctl start nadabot

# 查看状态
sudo systemctl status nadabot
journalctl -u nadabot -f
```

### 7.4 Docker（可选）

```dockerfile
# Dockerfile — 多阶段构建
# 阶段1：构建
FROM python:3.12-slim AS builder

RUN pip install --no-cache-dir uv
WORKDIR /build

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project

COPY src/ src/
RUN uv build && uv pip install dist/*.whl --python /usr/local/bin/python

# 阶段2：运行
FROM python:3.12-slim AS runtime

RUN groupadd -r nadabot && useradd -r -g nadabot nadabot

COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin/nadabot /usr/local/bin/nadabot

USER nadabot
WORKDIR /home/nadabot

VOLUME ["/home/nadabot/.nadabot"]
EXPOSE 8080

ENTRYPOINT ["nadabot"]
CMD ["start", "--host", "0.0.0.0", "--port", "8080"]
```

```yaml
# docker-compose.yml
version: "3.8"

services:
  nadabot:
    build: .
    container_name: nadabot
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - ./data:/home/nadabot/.nadabot
    environment:
      - NADABOT_LOG_LEVEL=info
    env_file:
      - .env  # API keys 等敏感配置
```

### 7.5 CLI 命令设计

```bash
nadabot                          # 等同于 nadabot --help
nadabot init                     # 初始化工作空间和配置
nadabot start                    # 启动 gateway
nadabot start --daemon           # 后台运行
nadabot stop                     # 停止

nadabot plugin list              # 列出已安装插件
nadabot plugin install <name>    # 安装插件（从 PyPI / Git）
nadabot plugin remove <name>     # 卸载插件
nadabot plugin reload <name>     # 热重载插件

nadabot node list                # 列出已连接节点
nadabot node pair                # 生成配对码
nadabot node exec <id> <cmd>     # 在节点执行命令

nadabot workspace list           # 列出工作空间
nadabot workspace init <name>    # 创建工作空间

nadabot config show              # 显示当前配置
nadabot config set <key> <val>   # 修改配置

nadabot doctor                   # 诊断检查
```

---

## 8. 开源社区考量

### 8.1 PyPI 发布

```yaml
# .github/workflows/release.yml
name: Release to PyPI

on:
  release:
    types: [published]

jobs:
  release:
    runs-on: ubuntu-latest
    environment: release
    permissions:
      id-token: write  # Trusted Publishing
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      
      - name: Install uv
        uses: astral-sh/setup-uv@v4
      
      - name: Build package
        run: uv build
      
      - name: Publish to PyPI
        run: uv publish
```

### 8.2 插件生态引导

**插件注册表（未来）**:

```
nadabot plugin search telegram
nadabot plugin install nadabot-plugin-telegram
```

插件命名规范：
- PyPI: `nadabot-plugin-<name>`
- 入口点: `nadabot.plugins` (entry_points)

```toml
# 在插件的 pyproject.toml 中注册
[project.entry-points."nadabot.plugins"]
telegram = "nadabot_plugin_telegram:TelegramChannel"
```

### 8.3 文档结构（MkDocs）

```
docs/
├── mkdocs.yml
├── docs/
│   ├── index.md                   # 首页
│   ├── getting-started.md         # 5 分钟上手
│   ├── installation.md            # 安装指南
│   ├── configuration.md           # 配置参考
│   ├── architecture.md            # 架构概览
│   ├── plugin-development/
│   │   ├── index.md               # 插件开发指南
│   │   ├── channel-plugin.md      # Channel 插件
│   │   ├── tool-plugin.md         # Tool 插件
│   │   ├── hook-plugin.md         # Hook 插件
│   │   ├── manifest.md            # Manifest 规范
│   │   └── permissions.md         # 权限系统
│   ├── deployment/
│   │   ├── systemd.md
│   │   ├── docker.md
│   │   └── vps.md
│   ├── api-reference/             # 自动生成
│   └── changelog.md
```

```yaml
# mkdocs.yml
site_name: NadaBot Documentation
theme:
  name: material
  language: zh
  features:
    - navigation.instant
    - navigation.tabs
    - content.code.copy
    - content.code.annotate

plugins:
  - mkdocstrings:
      handlers:
        python:
          options:
            show_source: true
            show_root_heading: true

markdown_extensions:
  - admonition
  - codehilite
  - footnotes
  - toc:
      permalink: true
```

### 8.4 CI/CD（GitHub Actions）

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    strategy:
      matrix:
        python-version: ["3.12", "3.13"]
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      
      - name: Install uv
        uses: astral-sh/setup-uv@v4
      
      - name: Install dependencies
        run: uv sync --frozen
      
      - name: Lint (ruff)
        run: uv run ruff check .
      
      - name: Format check (ruff)
        run: uv run ruff format --check .
      
      - name: Type check (pyright)
        run: uv run pyright
      
      - name: Type check (mypy)
        run: uv run mypy src/
      
      - name: Test
        run: uv run pytest --cov=nadabot --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
```

---

## 9. 竞品对比分析

### 9.1 与 OpenClaw 的关系

| 方面 | OpenClaw (Node.js) | NadaBot (Python) | 说明 |
|------|-------------------|-------------------|------|
| **Workspace 设计** | ✅ 已有 | ✅ 直接借鉴 | 文件即配置的理念非常好，原样保留 |
| **Gateway-Node** | ✅ 已有 | ✅ 直接借鉴 | 架构验证过，Python 实现 |
| **插件系统** | 简单 JS require | ✅ 改进 | 添加 manifest、权限声明、类型安全 |
| **LLM 集成** | OpenAI SDK | ✅ 改进 | 多 Provider 抽象、统一接口 |
| **配置管理** | YAML | ✅ 改进 | TOML + pydantic-settings，类型安全 |
| **部署** | pnpm + manual | ✅ 改进 | pipx 一键安装、systemd 原生 |
| **类型安全** | TypeScript | ✅ 改进 | pydantic v2 运行时 + pyright 编译时 |

**直接借鉴的设计**:
- Workspace 文件结构（AGENTS.md, SOUL.md, USER.md）
- Gateway-Node 配对连接模型
- Skill/Plugin 概念
- 多 Channel 同时运行

**主要改进**:
- 声明式权限系统（Android 式 manifest）
- 多 LLM Provider 统一抽象
- 更好的插件隔离和错误恢复
- Python 生态的 AI/ML 工具链
- 零配置 SQLite 替代外部数据库

### 9.2 Python 生态竞品

| 框架 | 定位 | 优势 | 劣势 | 与 NadaBot 的关系 |
|------|------|------|------|------------------|
| **nonebot2** | 聊天机器人框架 | 成熟、插件生态丰富、多适配器 | 无 Agent 概念、无 LLM 深度集成 | 上层竞品（更偏消息平台适配） |
| **aiogram** | Telegram Bot 框架 | Telegram 专项优化、FSM | 单平台、无 Agent | 互补（可作为 Channel 插件底层） |
| **AstrBot** | AI 聊天机器人 | 多平台、开箱即用 | 架构不够灵活、插件系统弱 | 直接竞品 |
| **LangChain** | LLM 应用框架 | 工具链丰富、RAG | 过度抽象、性能差、非 Bot 框架 | 可作为底层库（不推荐） |
| **AutoGen** | 多 Agent 系统 | Agent 编排 | 研究导向、非 Bot 框架 | 互补（可参考 Agent 间通信） |
| **Semantic Kernel** | AI 编排框架 | 微软支持、多语言 | 企业导向、复杂 | 非竞品 |

**NadaBot 的差异化**:
1. **Agent-First**: 从设计之初就以 AI Agent 为核心，不是事后添加
2. **Workspace-Native**: Agent 有持久化工作空间，像真正的助手
3. **Gateway-Node**: 唯一支持远程节点控制的 Python Bot 框架
4. **Permission-Manifest**: 唯一有声明式权限系统的 Bot 框架
5. **Zero-Config**: SQLite + 文件配置，无需数据库和复杂配置

---

## 10. 类型安全策略

### 10.1 pydantic v2 模型

```python
# 所有数据模型使用 pydantic v2
from pydantic import BaseModel, Field, ConfigDict
from typing import Literal


class LLMConfig(BaseModel):
    """LLM Provider 配置。"""
    model_config = ConfigDict(frozen=True)
    
    provider: Literal["openai", "anthropic", "gemini", "ollama"]
    model: str
    api_key: str | None = None
    base_url: str | None = None
    max_tokens: int = 4096
    temperature: float = Field(default=0.7, ge=0.0, le=2.0)
    timeout: float = Field(default=60.0, gt=0.0)


class AppConfig(BaseModel):
    """全局应用配置。"""
    model_config = ConfigDict(frozen=True)
    
    app_name: str = "NadaBot"
    debug: bool = False
    log_level: Literal["DEBUG", "INFO", "WARNING", "ERROR"] = "INFO"
    
    agent: AgentConfig
    gateway: GatewayConfig
    database: DatabaseConfig
    workspace: WorkspaceConfig
    plugins: PluginsConfig
    nodes: NodesConfig


# 从 TOML 配置文件加载
from pydantic_settings import BaseSettings, SettingsConfigDict


class AppSettings(BaseSettings):
    """从环境变量和配置文件加载的设置。"""
    model_config = SettingsConfigDict(
        toml_file="config.toml",
        env_prefix="NADABOT_",
        env_nested_delimiter="__",
    )
    
    config: AppConfig
```

### 10.2 pyright strict 配置

```json5
// pyrightconfig.json
{
  "include": ["src"],
  "exclude": ["**/__pycache__"],
  "strict": ["src"],
  "typeCheckingMode": "strict",
  "reportMissingTypeStubs": "error",
  "reportUnknownMemberType": "warning",
  "reportUnknownArgumentType": "warning",
  "reportUnknownVariableType": "warning",
  "reportUntypedFunctionDecorator": "error",
  "reportMissingTypeArgument": "error",
  "pythonVersion": "3.12",
  "pythonPlatform": "All",
  "executionEnvironments": [
    { "os": "Linux" },
    { "os": "Windows" },
    { "os": "Darwin" }
  ]
}
```

### 10.3 mypy 配置

```toml
# pyproject.toml [tool.mypy]
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_any_generics = true
check_untyped_defs = true

[[tool.mypy.overrides]]
module = "telegram.*"
ignore_missing_imports = true

[[tool.mypy.overrides]]
module = "watchdog.*"
ignore_missing_imports = true
```

### 10.4 类型标注覆盖要求

```python
# CI 中强制检查类型覆盖率
# pyproject.toml
[tool.coverage.run]
source = ["nadabot"]

[tool.coverage.report]
fail_under = 80  # 80% 覆盖率要求
```

**类型标注规则**:
1. 所有公开 API 必须有完整类型标注
2. 所有 pydantic model 必须使用 `ConfigDict(frozen=True)` 或明确可变
3. 优先使用 `Protocol` 而非 `ABC` 进行接口定义
4. 使用 Python 3.12+ 新语法（`X | Y` 替代 `Union[X, Y]`）

```python
# Protocol 示例（推荐替代 ABC）
from typing import Protocol, runtime_checkable


@runtime_checkable
class LLMProvider(Protocol):
    """LLM Provider 接口。"""
    
    async def chat(
        self,
        messages: list[Message],
        *,
        tools: list[ToolDefinition] | None = None,
        stream: bool = False,
        **kwargs: Any,
    ) -> AsyncIterator[ResponseChunk]:
        ...

    async def count_tokens(self, text: str) -> int:
        ...
```

---

## 11. 路线图

### Phase 1: MVP（4 周）

- [ ] 核心框架骨架（app、config、cli）
- [ ] Agent Loop（单 Provider、tool calling）
- [ ] Workspace 文件系统
- [ ] 插件系统基础（加载、生命周期）
- [ ] 2 个内置工具（read_file、shell）
- [ ] 1 个 Channel 插件（Telegram）
- [ ] 基础测试覆盖

### Phase 2: Gateway-Node（3 周）

- [ ] Gateway FastAPI 服务
- [ ] WebSocket 通信协议
- [ ] Node 客户端（Python）
- [ ] 认证与配对
- [ ] 心跳与自动重连
- [ ] 远程命令执行

### Phase 3: 插件生态（3 周）

- [ ] plugin.yaml manifest 规范
- [ ] 权限检查引擎
- [ ] Hook 事件系统
- [ ] 热加载支持
- [ ] 插件 CLI（install/remove/reload）
- [ ] 更多内置工具

### Phase 4: 完善（2 周）

- [ ] 多 LLM Provider（Anthropic、Gemini、Ollama）
- [ ] 上下文压缩策略
- [ ] 流式响应优化
- [ ] 文档（MkDocs）
- [ ] PyPI 发布
- [ ] CI/CD 流水线

### Phase 5: 社区（持续）

- [ ] 更多 Channel 插件（Discord、QQ、微信）
- [ ] 插件注册表
- [ ] 示例和教程
- [ ] 社区反馈收集

---

## 附录 A: 配置文件示例

```toml
# ~/.nadabot/config.toml

[app]
name = "NadaBot"
debug = false
log_level = "INFO"

[agent]
max_turns = 20
max_context_tokens = 128000

[agent.provider]
provider = "openai"
model = "gpt-4o"
api_key = "${OPENAI_API_KEY}"  # 从环境变量读取
temperature = 0.7

[gateway]
host = "0.0.0.0"
port = 8080
cors_origins = ["*"]

[database]
path = "${app.data_dir}/nadabot.db"

[workspace]
path = "${app.data_dir}/workspaces/default"

[plugins]
dirs = [
    "${app.data_dir}/plugins",
    "./plugins",
]
auto_reload = false  # 开发模式设为 true

[nodes]
heartbeat_interval = 10
connection_timeout = 30
max_nodes = 10
```

## 附录 B: 快速上手

```bash
# 1. 安装
pipx install nadabot

# 2. 初始化
nadabot init
# → 创建 ~/.nadabot/ 目录结构
# → 生成 config.toml

# 3. 配置 API Key
export OPENAI_API_KEY="sk-..."
# 或编辑 config.toml

# 4. 安装插件
nadabot plugin install nadabot-plugin-telegram

# 5. 配置插件
# 编辑 ~/.nadabot/plugins/telegram-channel/config.toml

# 6. 启动
nadabot start

# 🎉 你的 AI Agent Bot 已经在运行了！
```

---

> 🌳 *"最初的种子很小，但只要根扎得够深，终会长成参天大树。"*
> 
> — 纳西妲
