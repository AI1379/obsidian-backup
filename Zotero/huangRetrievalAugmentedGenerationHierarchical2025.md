---
tags: [paper, paper/rag]
---

# Retrieval-Augmented Generation with Hierarchical Knowledge

## 📒 引用

```plaintext
Huang, Haoyu, Yongfeng Huang, Junjie Yang, et al. “Retrieval-Augmented Generation with Hierarchical Knowledge.” arXiv:2503.10150. Version 3. Preprint, arXiv, September 26, 2025. [https://doi.org/10.48550/arXiv.2503.10150](https://doi.org/10.48550/arXiv.2503.10150).
```

URL：[http://arxiv.org/abs/2503.10150](http://arxiv.org/abs/2503.10150)
GitHub: https://github.com/hhy-huang/HiRAG

## 📌 摘要

Graph-based Retrieval-Augmented Generation (RAG) methods have significantly enhanced the performance of large language models (LLMs) in domain-specific tasks. However, existing RAG methods do not adequately utilize the naturally inherent hierarchical knowledge in human cognition, which limits the capabilities of RAG systems. In this paper, we introduce a new RAG approach, called HiRAG, which utilizes hierarchical knowledge to enhance the semantic understanding and structure capturing capabilities of RAG systems in the indexing and retrieval processes. Our extensive experiments demonstrate that HiRAG achieves significant performance improvements over the state-of-the-art baseline methods.

## 💡 我的思考

- [ ] 本文与我的课题关联点：这篇文章提出的 Hierarchical RAG 或许与我设想的 YOLO 类似的 RAG 比较相近。
- [ ] 可复现的实验：_________

### 2025-11-28

目前回顾这篇文献，可以看到 HiRAG 的核心思想似乎是继承自 GraphRAG 的。从这个角度进一步延伸，我们需要的似乎不是一个 YOLO 形式的召回系统，而是一个图神经网络形式的一个找回系统，因此也许可以查询 GNN 方向的文献进一步研究。

