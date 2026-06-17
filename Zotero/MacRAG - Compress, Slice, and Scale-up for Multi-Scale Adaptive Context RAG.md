---
tags: [paper, paper/rag]
---

# MacRAG: Compress, Slice, and Scale-up for Multi-Scale Adaptive Context RAG

## 📒 引用

```plaintext
Lim, Woosang, Zekun Li, Gyuwan Kim, et al. “MacRAG: Compress, Slice, and Scale-up for Multi-Scale Adaptive Context RAG.” arXiv:2505.06569. Version 2. Preprint, arXiv, May 20, 2025. [https://doi.org/10.48550/arXiv.2505.06569](https://doi.org/10.48550/arXiv.2505.06569).
```

Cite Key:

```plaintext
limMacRAGCompressSlice2025
```

URL：[http://arxiv.org/abs/2505.06569](http://arxiv.org/abs/2505.06569)

## 📌 摘要

Long-context large language models (LC LLMs) combined with retrieval-augmented generation (RAG) hold strong potential for complex multi-hop and large-document tasks. However, existing RAG systems often suffer from imprecise retrieval, incomplete context coverage under constrained windows, and fragmented information from suboptimal context construction. We introduce Multi-scale Adaptive Context RAG (MacRAG), a hierarchical RAG framework that compresses and partitions documents into coarse-to-fine granularities, then adaptively merges relevant contexts through real-time chunk- and document-level expansions. By initiating with finest-level retrieval and progressively incorporating broader, higher-level context, MacRAG constructs effective query-specific long contexts, optimizing both precision and coverage. Evaluations on challenging LongBench expansions of HotpotQA, 2WikiMultihopQA, and Musique confirm MacRAG consistently surpasses baseline RAG pipelines in single- and multi-step generation using Llama-3.1-8B, Gemini-1.5-pro, and GPT-4o. Our results establish MacRAG as an efficient, scalable solution for real-world long-context, multi-hop reasoning. Our code is available at https://github.com/Leezekun/MacRAG.

## 💡 我的思考

- [ ] 本文与我的课题关联点：这篇文章给出了一个自适应尺度的 RAG 模型，也许对我们目前的工作有帮助。
- [ ] 可复现的实验：_________
