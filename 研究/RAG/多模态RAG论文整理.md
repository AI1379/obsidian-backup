# 多模态 RAG & Embedding 论文整理

## 问题背景

传统 RAG 召回基于文本，即使能召回图像也需要手工打标。是否存在原生多模态方案？

## 核心思路

统一的多模态 embedding 空间：文本和图像投影到同一向量空间，query（文本/图像）都能直接召回相关图像或文本。

---

## 相关论文

### 1. 多模态 RAG 架构

| 论文 | arXiv ID | GitHub | 核心贡献 |
|------|----------|--------|----------|
| **VimRAG** | 待确认 | - | Multimodal Memory Graph 处理大规模视觉上下文 |
| **Fix Before Search** | 待确认 | - | Agentic Query Visual Pre-processing |
| **Concept-Enhanced Multimodal RAG** | 待确认 | - | 放射学报告生成，增强可解释性 |
| **CrisiSense-RAG** | 待确认 | - | 灾害评估的多模态 RAG |
| **MMed-RAG** | - | [richard-peng-xia/MMed-RAG](https://github.com/richard-peng-xia/MMed-RAG) | ICLR'25 医学多模态 RAG |
| **RULE** | - | [richard-peng-xia/RULE](https://github.com/richard-peng-xia/RULE) | EMNLP'24 医学视觉语言模型的可靠多模态 RAG |
| **URaG** | - | [shi-yx/URaG](https://github.com/shi-yx/URaG) | AAAI 2026 Oral 统一检索与生成 |

> VimRAG 摘要：传统 RAG 依赖线性交互历史，难以处理长上下文任务，尤其是迭代推理中信息稀疏但 token 密集的视觉数据。

### 2. 统一多模态 Embedding

| 论文 | arXiv ID | GitHub | 核心贡献 |
|------|----------|--------|----------|
| **Unified Multimodal Retrieval** (2026.1) | 待确认 | - | 多任务学习 + NLU 统一多模态/多语言检索 |
| **jina-clip-v2** | - | - | 多语言多模态 embedding，CLIP 增强版 |
| **VLM2GeoVec** (2025) | 待确认 | - | 遥感领域通用多模态 embedding |
| **VLM2Vec** | - | [TIGER-AI-Lab/VLM2Vec](https://github.com/TIGER-AI-Lab/VLM2Vec) | ICLR 2025 大规模多模态 embedding |
| **CREM** (2026.2) | 待确认 | - | 压缩驱动的多模态检索表示增强 |

### 3. 文档视觉理解

| 论文 | arXiv ID | GitHub | 核心贡献 |
|------|----------|--------|----------|
| **Unlocking Multimodal Document Intelligence** (2026.2) | 待确认 | - | 视觉文档检索综述 |
| **Unified Vision-Centric Contrastive** (2025) | 待确认 | - | 处理复杂 web 文档的 CLIP 变体 |
| **ReT** | - | [aimagelab/ReT](https://github.com/aimagelab/ReT) | CVPR 2025 多模态文档检索 |

---

## 开源项目

### CLIP 检索
- **[rom1504/clip-retrieval](https://github.com/rom1504/clip-retrieval)** ⭐ - 轻松计算 CLIP embeddings 并构建检索系统

### 多模态 RAG 框架
- **[Mintplex-Labs/anything-llm](https://github.com/Mintplex-Labs/anything-llm)** ⭐ 54k+ - 全合一 Docker AI 应用，内置 RAG、Agent、MCP 兼容
- **[Azure/gpt-rag-ingestion](https://github.com/Azure/gpt-rag-ingestion)** - 多模态文档处理，生成文本和图像 embedding
- **[PromtEngineer/localGPT-Vision](https://github.com/PromtEngineer/localGPT-Vision)** - 使用 VLM 实现文档 RAG

### 视觉语言模型 + RAG
- **[TIGER-AI-Lab/VLM2Vec](https://github.com/TIGER-AI-Lab/VLM2Vec)** - VLM2Vec: 多模态 embedding (ICLR 2025)
- **[dusty-nv/NanoLLM](https://github.com/dusty-nv/NanoLLM)** - 本地 LLM 推理，支持 VLM、RAG

### 多模态检索
- **[dsgiitr/kge-graphRAG](https://github.com/dsgiitr/kge-graphRAG)** - 知识图谱 embeddings 用于 RAG
- **[PrismaticLab/Video-RAC](https://github.com/PrismaticLab/Video-RAC)** - 视频 RAG 自适应分块

---

## 技术路线

1. **CLIP/ViT 系列** - 用对比学习预训练的视觉-语言模型直接做 embedding
2. **多模态 Chunks** - 把图片切分成块，每块生成 caption/embedding，与文本一起索引
3. **原生多模态索引** - 如 VimRAG，构建多模态记忆图，直接对图像内容做检索
4. **VLM-as-Encoder** - 使用 Vision-Language Model 作为 encoder（如 VLM2Vec）

---

## 参考文献

- VimRAG: Navigating Massive Visual Context in Retrieval-Augmented Generation via Multimodal Memory Graph
- Unlocking Multimodal Document Intelligence: From Current Triumphs to Future Frontiers of Visual Document Retrieval
- jina-clip-v2: Multilingual Multimodal Embeddings for Text and Images
- VLM2Vec: Training Vision-Language Models for Massive Multimodal Embedding Tasks (ICLR 2025)

> 生成时间：2026-02-25
> 最后更新：2026-02-25
> 话题来源：与纳西妲的对话
