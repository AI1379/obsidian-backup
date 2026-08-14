---
tags:
  - paper
citekey: "miaoFisherR1TrainingLLM2026a"
---

# Fisher-R1: Training LLM Agents for Reliable Hypothesis Testing

## 📒 引用

```plaintext
Miao, Jiacheng, Jin Mu, Guanhua Chen, and James Zou. “Fisher-R1: Training LLM Agents for Reliable Hypothesis Testing.” arXiv:2608.07437. Preprint, arXiv, August 7, 2026. [https://doi.org/10.48550/arXiv.2608.07437](https://doi.org/10.48550/arXiv.2608.07437).
```

Cite Key:

```plaintext
miaoFisherR1TrainingLLM2026a
```

URL：[http://arxiv.org/abs/2608.07437](http://arxiv.org/abs/2608.07437)

## 📌 摘要

Reliable hypothesis testing is the foundation of many empirical scientific claims. Large language model (LLM) agents are increasingly used to automate this process, as they can inspect datasets, generate code, and produce analyses end-to-end. However, we show that they frequently make subtle inferential errors that lead to incorrect conclusions despite correctly executed analyses. Existing benchmarks fail to capture this failure mode, as they rarely assess whether a reported p-value is statistically valid given the assumptions underlying the data. We address this gap by building P-Bench, a benchmark comprising 425 open-ended, realistic hypothesis-testing tasks spanning economics, biology, and medicine. Each task requires an agent to select a statistical method, compute a p-value, and draw a conclusion given only a scientific hypothesis and a dataset. We further introduce Fisher-R1, an open-weight LLM agent trained for rigorous hypothesis testing using synthetic tasks and reinforcement learning. On P-Bench, Fisher-R1-14B substantially improves over its backbone and outperforms strong proprietary and open-source baselines, including GPT-5.4 and DeepSeekV4-Pro, achieving a 21% average relative improvement in single-trial success over DeepSeek-V4-Pro, with gains up to 26% on the most challenging tasks. Our results demonstrate that current LLM agents lack reliable statistical reasoning for hypothesis testing and that reinforcement learning on tasks with verified statistical reward substantially improves reliability.

## 💡 我的思考

- [ ] 本文与我的课题关联点：_________
- [ ] 可复现的实验：_________
