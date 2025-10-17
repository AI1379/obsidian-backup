---
tags:
  - 日志
  - RAG
  - LLM
---

### 2025-10-14

近期尝试进行RAG系统的设计，目前有几个思路：

- 关键词匹配+向量数据库
- 分层数据库
- Rerank 模型

## 2025-10-17

大致完成了对 [[huangRetrievalAugmentedGenerationHierarchical2025|HiRAG]] 的阅读，计划参考其官方实现重新实现一遍其算法。目前有一个 10.22 的 ddl，因此考虑使用简单的RAG先写一版处理脚本。