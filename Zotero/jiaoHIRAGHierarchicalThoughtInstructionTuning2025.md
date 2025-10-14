# HIRAG: Hierarchical-Thought Instruction-Tuning Retrieval-Augmented Generation

## 📒 引用

```plaintext
Jiao, YiHan, ZheHao Tan, Dan Yang, et al. “HIRAG: Hierarchical-Thought Instruction-Tuning Retrieval-Augmented Generation.” arXiv:2507.05714. Preprint, arXiv, September 10, 2025. [https://doi.org/10.48550/arXiv.2507.05714](https://doi.org/10.48550/arXiv.2507.05714).
```

URL：[http://arxiv.org/abs/2507.05714](http://arxiv.org/abs/2507.05714)

## 📌 摘要

Retrieval-augmented generation (RAG) has become a fundamental paradigm for addressing the challenges faced by large language models in handling real-time information and domain-specific problems. Traditional RAG systems primarily rely on the in-context learning (ICL) capabilities of the large language model itself. Still, in-depth research on the specific capabilities needed by the RAG generation model is lacking, leading to challenges with inconsistent document quality and retrieval system imperfections. Even the limited studies that fine-tune RAG generative models often *lack a granular focus on RAG task* or *a deeper utilization of chain-of-thought processes*. To address this, we propose that RAG models should possess three progressively hierarchical abilities (1) Filtering: the ability to select relevant information; (2) Combination: the ability to combine semantic information across paragraphs; and (3) RAG-specific reasoning: the ability to further process external knowledge using internal knowledge. Thus, we introduce our new RAG instruction fine-tuning method, Hierarchical-Thought Instruction-Tuning Retrieval-Augmented Generation (HIRAG) incorporates a "think before answering" strategy. This method enhances the model's open-book examination capability by utilizing multi-level progressive chain-of-thought. Experiments show that the HIRAG training strategy significantly improves the model's performance on datasets such as RGB, PopQA, MuSiQue, HotpotQA, and PubmedQA.
## 💡 我的思考

- [ ] 本文与我的课题关联点：_________
- [ ] 可复现的实验：_________
