# AI Agent 框架对比调研报告

> 调研时间：2026-04-29 | 调研人：纳西妲

---

## 目录

1. [项目概览](#项目概览)
2. [逐项分析](#逐项分析)
   - [NanoBot](#1-nanobot)
   - [Hermes Agent](#2-hermes-agent)
   - [Khoj](#3-khoj)
   - [AstrBot](#4-astrbot)
   - [OpenClaw（对比基准）](#5-openclaw)
3. [横向对比表](#横向对比表)
4. [结论与建议](#结论与建议)

---

## 项目概览

| 项目 | Stars | 语言 | 许可证 | 创建时间 | 定位 |
|:---|---:|:---|:---|:---|:---|
| **OpenClaw** | 366k | TypeScript | MIT | 2025-11 | 全能个人 AI 助手 |
| **Hermes Agent** | 123.7k | Python | MIT | 2025-07 | 自我学习型 AI Agent |
| **NanoBot** | 41.2k | Python | MIT | 2026-02 | 超轻量个人 AI Agent |
| **Khoj** | 34.3k | Python | AGPL-3.0 | 2021-08 | AI 第二大脑 / 知识管理 |
| **AstrBot** | 30.9k | Python | AGPL-3.0 | 2022-12 | 多平台聊天机器人平台 |

---

## 逐项分析

### 1. NanoBot

**GitHub:** [HKUDS/nanobot](https://github.com/HKUDS/nanobot) | **Stars:** 41,238 | **组织:** 香港大学数据科学实验室 (HKUDS)

| 维度 | 详情 |
|:---|:---|
| **语言/许可证** | Python 3.11+ / MIT |
| **架构** | 明确受 OpenClaw 启发（"in the spirit of OpenClaw"）。保持核心 agent loop 小而可读，支持 channels、memory、MCP。采用插件式 channel 架构（Telegram、Discord、Feishu、QQ、WeChat、Slack、WhatsApp 等） |
| **技术栈** | Python，原用 litellm 后移除，改为原生 openai + anthropic SDK。PyPI 发布 (`nanobot-ai`)。有 WebUI |
| **渠道** | Telegram, Discord, Feishu, QQ, WeChat/WeCom, Slack, WhatsApp, Matrix, 飞书, 钉钉, Email, WebSocket, OpenAI-compatible API, WebUI |
| **记忆** | "Dream two-stage memory" — 两阶段记忆系统（短期 + 长期），token-based memory，自动 context compact |
| **工具/MCP** | MCP 支持（多 server、resources/prompts 暴露为工具），内置 shell sandbox、文件操作、web search、Kagi 等 |
| **模型** | OpenAI, Anthropic, Google Gemini, Azure OpenAI, Ollama, LM Studio, MiniMax, Xiaomi MiMo, StepFun, VolcEngine, OpenRouter, Kimi K2.6, GitHub Copilot 等 |
| **社区** | 极度活跃 — 几乎每天都有更新（2026-04 月每日 changelog），884 open issues。由 HKUDS 实验室维护 |
| **部署** | pip install, Docker |

**特点：** 发展极为迅速（2 个月内 41k stars），功能密集迭代。明显是 OpenClaw 的 Python 替代品，渠道覆盖广泛，尤其在国内平台（QQ、微信、飞书、钉钉）支持较好。

---

### 2. Hermes Agent

**GitHub:** [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) | **Stars:** 123,704 | **组织:** Nous Research

| 维度 | 详情 |
|:---|:---|
| **语言/许可证** | Python / MIT |
| **架构** | 自我改进型 agent —— 内置学习循环：从经验创建技能、使用中改进技能、主动持久化知识、搜索历史对话、跨 session 构建用户模型。支持 subagent 并行、Python RPC 脚本 |
| **技术栈** | Python，六种终端后端（local/Docker/SSH/Daytona/Singularity/Modal），TUI 界面 |
| **渠道** | Telegram, Discord, Slack, WhatsApp, Signal, Email, CLI TUI |
| **记忆** | Agent 策划的记忆 + 周期性 nudge，FTS5 session 搜索 + LLM 摘要实现跨 session 回忆，[Honcho](https://github.com/plastic-labs/honcho) 方言式用户建模 |
| **工具/MCP** | 40+ 内置工具，toolset 系统，MCP 支持，[agentskills.io](https://agentskills.io) 开放标准，自主技能创建与改进 |
| **模型** | Nous Portal, OpenRouter (200+), NVIDIA NIM, Xiaomi MiMo, z.ai/GLM, Kimi/Moonshot, MiniMax, Hugging Face, OpenAI 及自定义 endpoint |
| **社区** | 7,016 open issues，极为活跃。Nous Research 是知名开源 AI 组织（训练 Hermes 系列模型） |
| **部署** | curl 一键安装，支持 Linux/macOS/WSL2/Termux。serverless 方案（Daytona/Modal）可在空闲时休眠节省成本 |

**特点：** 最强的「自我学习」闭环设计。与其他框架相比，Hermes 更注重 agent 的自主进化能力。还提供 `hermes claw migrate` 从 OpenClaw 迁移的命令，直接对标。

---

### 3. Khoj

**GitHub:** [khoj-ai/khoj](https://github.com/khoj-ai/khoj) | **Stars:** 34,306 | **组织:** khoj-ai

| 维度 | 详情 |
|:---|:---|
| **语言/许可证** | Python / AGPL-3.0 |
| **架构** | 以 RAG（检索增强生成）为核心的知识助手。支持文档索引（PDF/Markdown/Notion/Word/Org-mode），语义搜索 + LLM 对话 |
| **技术栈** | Python，Docker，PyPI (`khoj`)。有桌面客户端和 Web UI |
| **渠道** | WhatsApp, Browser, Obsidian, Emacs, Desktop App, Web App |
| **记忆** | 基于文档的知识库（RAG），语义搜索索引。支持自定义 Agent（persona + 知识 + 工具） |
| **工具/MCP** | 内置 web search, image generation, TTS/STT, 自动化调度（newsletter/通知）。无明确的 MCP 支持 |
| **模型** | OpenAI, Anthropic, Google, 本地 LLM (llama3, qwen, gemma, mistral, deepseek)，Online API |
| **社区** | 99 open issues，相对成熟稳定。有云服务 (app.khoj.dev) 和企业版 |
| **部署** | pip install, Docker, 桌面应用, 云服务 |

**特点：** 定位最不同 —— 不是通用 agent，而是「AI 第二大脑」。RAG 能力最强，文档管理和语义搜索是核心竞争力。适合知识工作者。历史最悠久（2021 年创建）。

---

### 4. AstrBot

**GitHub:** [AstrBotDevs/AstrBot](https://github.com/AstrBotDevs/AstrBot) | **Stars:** 30,936 | **组织:** AstrBotDevs

| 维度 | 详情 |
|:---|:---|
| **语言/许可证** | Python / AGPL-3.0 |
| **架构** | 一站式聊天机器人平台，WebUI 管理。支持 MCP、Agent、知识库、人格设定、自动上下文压缩 |
| **技术栈** | Python，Docker，WebUI + WebChatUI |
| **渠道** | **QQ, 企业微信, 飞书, 钉钉, 微信公众号, Telegram, Slack** 及更多（OneBot 协议） |
| **记忆** | 知识库 (RAG)，自动上下文压缩 |
| **工具/MCP** | MCP 支持，Agent Sandbox（隔离执行代码/shell），1000+ 社区插件可一键安装 |
| **模型** | OpenAI, Gemini, GPT, Llama 等主流 LLM |
| **社区** | 1,060 open issues，活跃。有丰富的中文社区生态 |
| **部署** | Docker, pip, WebUI 引导安装 |

**特点：** **国内生态最强**。QQ/微信/钉钉/飞书等国内平台支持最好，1000+ 社区插件。适合中国用户搭建聊天机器人。自称 "openclaw alternative"。

---

### 5. OpenClaw（对比基准）

**GitHub:** [openclaw/openclaw](https://github.com/openclaw/openclaw) | **Stars:** 366,058 | **组织:** openclaw

| 维度 | 详情 |
|:---|:---|
| **语言/许可证** | TypeScript (Node.js) / MIT |
| **架构** | Local-first Gateway 控制面 + 多 agent 路由。Session/channel/tool/event 统一管理。支持 subagent、sandbox (Docker/SSH/OpenShell) |
| **技术栈** | Node 24+, TypeScript, npm/pnpm/bun。Gateway daemon (systemd/launchd) |
| **渠道** | **最广** — WhatsApp, Telegram, Slack, Discord, Google Chat, Signal, iMessage, BlueBubbles, IRC, Teams, Matrix, Feishu, LINE, Mattermost, Nextcloud Talk, Nostr, Synology Chat, Tlon, Twitch, Zalo, WeChat, QQ, WebChat, macOS, iOS/Android |
| **记忆** | 文件式记忆 (MEMORY.md, daily logs)，向量搜索 (LanceDB)，workspace 文件系统。Context files 系统注入项目上下文 |
| **工具/MCP** | 一等公民工具：browser, canvas, nodes, cron, sessions。Skills 注册表 (ClawHub)，MCP 通过 mcporter 集成。完整沙盒系统 |
| **模型** | OpenAI, Anthropic, Google, Azure, 及通过 OpenAI-compatible API 的任意模型。Model failover + auth profile rotation |
| **社区** | 6,982 open issues，极度活跃。366k stars 是所有项目中最高 |
| **部署** | npm install -g, Docker, Nix。macOS/iOS/Android companion apps |
| **独有特性** | Voice Wake + Talk Mode, Live Canvas (A2UI), macOS menu bar app, 伴侣 app, multi-agent routing |

**特点：** 生态最大、渠道最广、功能最全面。唯一用 TypeScript 的项目。有原生 companion apps。适合追求全能个人助手的高级用户。

---

## 横向对比表

| 维度 | OpenClaw | Hermes Agent | NanoBot | Khoj | AstrBot |
|:---|:---|:---|:---|:---|:---|
| **Stars** | 366k | 124k | 41k | 34k | 31k |
| **语言** | TypeScript | Python | Python | Python | Python |
| **许可证** | MIT | MIT | MIT | AGPL-3.0 | AGPL-3.0 |
| **渠道数** | 24+ | 7 | 12+ | 5 | 7+ |
| **国内平台** | QQ/WeChat/Feishu | ❌ | QQ/WeChat/Feishu/DingTalk | ❌ | QQ/企微/飞书/钉钉/公众号 |
| **MCP** | ✅ (mcporter) | ✅ | ✅ | ❌ | ✅ |
| **自我学习** | ❌ | ✅ (核心卖点) | 部分 (Dream memory) | ❌ | ❌ |
| **RAG/知识库** | 向量搜索 | FTS5+LLM | ✅ | ✅ (最强) | ✅ |
| **沙盒** | ✅ (Docker/SSH) | ✅ (6种后端) | ✅ | ❌ | ✅ |
| **Subagent** | ✅ | ✅ | ✅ | ❌ | ❌ |
| **WebUI** | ❌ (Canvas) | ❌ (TUI) | ✅ | ✅ | ✅ |
| **语音** | ✅ (Wake+Talk) | ✅ (STT) | 部分 | ✅ (TTS/STT) | ❌ |
| **移动端 App** | ✅ (iOS/Android) | ❌ | ❌ | ✅ (Desktop) | ❌ |
| **插件生态** | ClawHub | agentskills.io | 内置 | 有限 | 1000+ 社区插件 |
| **Serverless** | ❌ | ✅ (Daytona/Modal) | ❌ | ❌ | ❌ |
| **迁移工具** | — | `hermes claw migrate` | — | — | — |

---

## 结论与建议

### 各项目定位总结

| 项目 | 一句话定位 | 最适合 |
|:---|:---|:---|
| **OpenClaw** | 全能个人 AI 助手瑞士军刀 | 追求极致自定义、多渠道覆盖的高级用户 |
| **Hermes Agent** | 会自我进化的 AI Agent | 研究/实验型用户，希望 agent 自主学习 |
| **NanoBot** | Python 版轻量 OpenClaw | Python 生态用户，需要国内平台支持 |
| **Khoj** | AI 第二大脑 / 知识助手 | 知识工作者、学术研究者、文档管理需求 |
| **AstrBot** | 国内多平台聊天机器人 | 中国用户、QQ/微信生态、快速搭建 |

### 与 OpenClaw 的关键差异

1. **Hermes Agent** 是唯一在「自主学习」维度超越 OpenClaw 的项目。它的 skill 自我创建/改进闭环、Honcho 用户建模、serverless 终端后端是 OpenClaw 没有的。但有 `hermes claw migrate` 说明它视 OpenClaw 为主要竞品。

2. **NanoBot** 是最接近 OpenClaw 的 Python 复刻。由高校实验室维护，迭代极快。国内平台支持比 OpenClaw 更原生态（钉钉、飞书 first-class）。但生态和成熟度仍有差距。

3. **AstrBot** 在国内 IM 平台（尤其 QQ）生态最完善，1000+ 插件是巨大优势。但架构上更偏向聊天机器人平台而非个人 AI 助手。

4. **Khoj** 定位最独特 —— 专注于知识管理和 RAG，不适合作为通用 agent 但在知识助手场景无出其右。

---

*报告完*
