## Hierarchical Index

论文中在这一部分实现了一个分层的知识图 Knowledge Graph ，从基础的通过 `SentenceSplitter` 得到的文本片段对应的嵌入开始，一层层使用 `GMM` 聚类，并利用一个 LLM 总结得到上一层的节点，这部分可见[[sarthiRAPTORRecursiveAbstractive2024| RAPTOR 的论文]] 。此外还使用 Leiden 算法计算知识图谱中的 Community ，并使用 LLM 给出一个总结。