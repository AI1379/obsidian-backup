---
tags: [paper, paper/rag]
---

# From Local to Global: A Graph RAG Approach to Query-Focused Summarization

## 📒 引用

```plaintext
Edge, Darren, Ha Trinh, Newman Cheng, et al. “From Local to Global: A Graph RAG Approach to Query-Focused Summarization.” arXiv:2404.16130. Preprint, arXiv, February 19, 2025. [https://doi.org/10.48550/arXiv.2404.16130](https://doi.org/10.48550/arXiv.2404.16130).
```

Cite Key:

```plaintext
edgeLocalGlobalGraph2025
```

URL：[http://arxiv.org/abs/2404.16130](http://arxiv.org/abs/2404.16130)

## 📌 摘要

The use of retrieval-augmented generation (RAG) to retrieve relevant information from an external knowledge source enables large language models (LLMs) to answer questions over private and/or previously unseen document collections. However, RAG fails on global questions directed at an entire text corpus, such as "What are the main themes in the dataset?", since this is inherently a query-focused summarization (QFS) task, rather than an explicit retrieval task. Prior QFS methods, meanwhile, do not scale to the quantities of text indexed by typical RAG systems. To combine the strengths of these contrasting methods, we propose GraphRAG, a graph-based approach to question answering over private text corpora that scales with both the generality of user questions and the quantity of source text. Our approach uses an LLM to build a graph index in two stages: first, to derive an entity knowledge graph from the source documents, then to pregenerate community summaries for all groups of closely related entities. Given a question, each community summary is used to generate a partial response, before all partial responses are again summarized in a final response to the user. For a class of global sensemaking questions over datasets in the 1 million token range, we show that GraphRAG leads to substantial improvements over a conventional RAG baseline for both the comprehensiveness and diversity of generated answers.

## 💡 我的思考

### 2026-01-31

GraphRAG 的开山鼻祖，似乎也是最重要的一份工作。

### 2026-02-02

成功在当前的数据上跑通了 GraphRAG，做了一些小修改。总的来说效果还不错