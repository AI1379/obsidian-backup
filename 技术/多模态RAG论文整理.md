# 多模态 RAG & Embedding 论文整理

## 问题背景

传统 RAG 召回基于文本，即使能召回图像也需要手工打标。是否存在原生多模态方案？

## 核心思路

统一的多模态 embedding 空间：文本和图像投影到同一向量空间，query（文本/图像）都能直接召回相关图像或文本。

---

## 相关论文

### 1. 多模态 RAG 架构

| 论文 | arXiv ID | 核心贡献 |
|------|----------|----------|
| **VimRAG** | - | Multimodal Memory Graph 处理大规模视觉上下文 |
| **Fix Before Search** | - | Agentic Query Visual Pre-processing |
| **Concept-Enhanced Multimodal RAG** | - | 放射学报告生成，增强可解释性 |
| **CrisiSense-RAG** | - | 灾害评估的多模态 RAG |

> VimRAG 摘要：传统 RAG 依赖线性交互历史，难以处理长上下文任务，尤其是迭代推理中信息稀疏但 token 密集的视觉数据。

### 2. 统一多模态 Embedding

| 论文 | arXiv ID | 核心贡献 |
|------|----------|----------|
| **Unified Multimodal Retrieval** (2026.1) | - | 多任务学习 + NLU 统一多模态/多语言检索 |
| **jina-clip-v2** (2025) | - | 多语言多模态 embedding，CLIP 增强版 |
| **VLM2GeoVec** (2025) | - | 遥感领域通用多模态 embedding |
| **CREM** (2026.2) | - | 压缩驱动的多模态检索表示增强 |

### 3. 文档视觉理解

| 论文 | arXiv ID | 核心贡献 |
|------|----------|----------|
| **Unlocking Multimodal Document Intelligence** (2026.2) | - | 视觉文档检索综述 |
| **Unified Vision-Centric Contrastive** (2025) | - | 处理复杂 web 文档的 CLIP 变体 |

---

## 技术路线

1. **CLIP/ViT 系列** - 用对比学习预训练的视觉-语言模型直接做 embedding
2. **多模态 Chunks** - 把图片切分成块，每块生成 caption/embedding，与文本一起索引
3. **原生多模态索引** - 如 VimRAG，构建多模态记忆图，直接对图像内容做检索

---

## 参考文献

- VimRAG: Navigating Massive Visual Context in Retrieval-Augmented Generation via Multimodal Memory Graph
- Unlocking Multimodal Document Intelligence: From Current Triumphs to Future Frontiers of Visual Document Retrieval
- jina-clip-v2: Multilingual Multimodal Embeddings for Text and Images

> 生成时间：2026-02-25
> 话题来源：与纳西妲的对话
