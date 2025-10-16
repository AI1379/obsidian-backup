# RAPTOR: Recursive Abstractive Processing for Tree-Organized Retrieval

## 📒 引用

```plaintext
Sarthi, Parth, Salman Abdullah, Aditi Tuli, Shubh Khanna, Anna Goldie, and Christopher D. Manning. “RAPTOR: Recursive Abstractive Processing for Tree-Organized Retrieval.” arXiv:2401.18059. Preprint, arXiv, January 31, 2024. [https://doi.org/10.48550/arXiv.2401.18059](https://doi.org/10.48550/arXiv.2401.18059).
```

URL：[http://arxiv.org/abs/2401.18059](http://arxiv.org/abs/2401.18059)

## 📌 摘要

Retrieval-augmented language models can better adapt to changes in world state and incorporate long-tail knowledge. However, most existing methods retrieve only short contiguous chunks from a retrieval corpus, limiting holistic understanding of the overall document context. We introduce the novel approach of recursively embedding, clustering, and summarizing chunks of text, constructing a tree with differing levels of summarization from the bottom up. At inference time, our RAPTOR model retrieves from this tree, integrating information across lengthy documents at different levels of abstraction. Controlled experiments show that retrieval with recursive summaries offers significant improvements over traditional retrieval-augmented LMs on several tasks. On question-answering tasks that involve complex, multi-step reasoning, we show state-of-the-art results; for example, by coupling RAPTOR retrieval with the use of GPT-4, we can improve the best performance on the QuALITY benchmark by 20% in absolute accuracy.

## 💡 我的思考

- [ ] 本文与我的课题关联点：RAPTOR 中给出了一种关键的方式进行知识库的重构，尤其是 GMM 的部分
- [ ] 可复现的实验：

### 核心算法

#### GMM Gaussian Mixture Models

分层高斯分布。 TODO。