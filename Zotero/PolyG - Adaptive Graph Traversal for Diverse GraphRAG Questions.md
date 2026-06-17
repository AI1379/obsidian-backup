---
tags: [paper, paper/rag]
---

# PolyG: Adaptive Graph Traversal for Diverse GraphRAG Questions

## 📒 引用

```plaintext
Liu, Renjie, Haitian Jiang, Xiao Yan, Bo Tang, and Jinyang Li. “PolyG: Adaptive Graph Traversal for Diverse GraphRAG Questions.” arXiv:2504.02112. Preprint, arXiv, November 1, 2025. [https://doi.org/10.48550/arXiv.2504.02112](https://doi.org/10.48550/arXiv.2504.02112).
```

Cite Key:

```plaintext
liuPolyGAdaptiveGraph2025
```

URL：[http://arxiv.org/abs/2504.02112](http://arxiv.org/abs/2504.02112)

## 📌 摘要

GraphRAG enhances large language models (LLMs) to generate quality answers for user questions by retrieving related facts from external knowledge graphs. However, current GraphRAG methods are primarily evaluated on and overly tailored for knowledge graph question answering (KGQA) benchmarks, which are biased towards a few specific question patterns and do not reflect the diversity of real-world questions. To better evaluate GraphRAG methods, we propose a complete four-class taxonomy to categorize the basic patterns of knowledge graph questions and use it to create PolyBench, a new GraphRAG benchmark encompassing a comprehensive set of graph questions. With the new benchmark, we find that existing GraphRAG methods fall short in effectiveness (i.e., quality of the generated answers) and/or efficiency (i.e., response time or token usage) because they adopt either a fixed graph traversal strategy or free-form exploration by LLMs for fact retrieval. However, different question patterns require distinct graph traversal strategies and context formation. To facilitate better retrieval, we propose PolyG, an adaptive GraphRAG approach by decomposing and categorizing the questions according to our proposed question taxonomy. Built on top of a unified interface and execution engine, PolyG dynamically prompts an LLM to generate a graph database query to retrieve the context for each decomposed basic question. Compared with SOTA GraphRAG methods, PolyG achieves a higher win rate in generation quality and has a low response latency and token cost. Our code and benchmark are open-source at https://github.com/Liu-rj/PolyG.

## 💡 我的思考

### 2026-01-31

这篇论文给出了一个看上去很不错的手法做路由，区分一个查询是否适合使用 GraphRAG 。暂时不是很确定用的是传统算法还是另外的一个模型。似乎训练一个 BERT 是可以完成这个工作的。