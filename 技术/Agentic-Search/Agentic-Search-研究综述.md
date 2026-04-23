---
date: 2026-04-23
tags:
  - Agentic-Search
  - RAG
  - LLM
  - 研究综述
---

# Agentic Search 研究综述

> 大语言模型 Agent 搜索能力增强的核心研究方向

## 1. Agentic RAG

**核心思想**: 把 AI Agent 嵌入 RAG 流程，实现动态检索、规划、多步推理

** vs 传统 RAG**: 传统 RAG 是静态工作流，Agentic RAG 可以自主决策何时检索、检索什么

**论文**: 
- [Agentic Retrieval-Augmented Generation: A Survey on Agentic RAG](https://arxiv.org/html/2501.09136v3) (2025)

## 2. Self-RAG

**核心思想**: 让 LLM 自己决定何时检索、生成、批判

**创新**: 引入特殊的 retrieval / critique tokens，按需检索

**论文**:
- [Self-RAG: Learning to Retrieve, Generate, and Critique](https://arxiv.org/abs/2310.11511) (NeurIPS 2024)
- GitHub: https://github.com/AkariAsai/self-rag

## 3. Search-o1

**核心思想**: 把搜索融入 o1 类推理过程，解决大模型"知识不足"问题

**特点**: Agentic Search Workflow + Reason-in-Documents 模块

**论文**:
- [Search-o1: Agentic Search-Enhanced Large Reasoning Models](https://arxiv.org/abs/2501.05366) (EMNLP 2025)
- GitHub: https://github.com/RUC-NLPIR/Search-o1

## 4. RAG-Reasoning

**核心思想**: 统一的 reasoning-retrieval 视角，RAG + 深度推理

**论文**:
- [Towards Agentic RAG with Deep Reasoning](https://aclanthology.org/2025.findings-emnlp.648.pdf) (EMNLP Findings 2025)

## 当前工具

| MCP Server | 工具数 | 用途 |
|:---|:---:|:---|
| paper-search | 57 | 学术论文（arXiv, PubMed, Semantic Scholar, Crossref） |
| ddg-search | 2 | DuckDuckGo 网页搜索 |
| arxiv | 10 | arXiv 论文 |
| playwright-mcp | 21 | 浏览器自动化 |
| mcp-obsidian | 12 | Obsidian 笔记搜索 |

## 潜在提升方向

1. 补充更多 MCP: Tavily AI, Brave Search, Wikipedia MCP, GitHub MCP
2. 实现 Self-RAG 框架：让模型自己判断何时检索
3. 集成 Search-o1 风格的 agentic search
