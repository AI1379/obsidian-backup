---
tags:
  - stats
---
## Abstract

Integrative analysis of data from multiple sources is critical to making generalizable discoveries. Associations that are consistently observed across multiple source populations are more likely to be generalized to target populations with possible distributional shifts. In this paper, we model the heterogeneous multi-source data with multiple high-dimensional regressions and make inferences for the maximin effect (Meinshausen, B{ü}hlmann, AoS, 43(4), 1801--1830). The maximin effect provides a measure of stable associations across multi-source data. A significant maximin effect indicates that a variable has commonly shared effects across multiple source populations, and these shared effects may be generalized to a broader set of target populations. There are challenges associated with inferring maximin effects because its point estimator can have a non-standard limiting distribution. We devise a novel sampling method to construct valid confidence intervals for maximin effects. The proposed confidence interval attains a parametric length. This sampling procedure and the related theoretical analysis are of independent interest for solving other non-standard inference problems. Using genetic data on yeast growth in multiple environments, we demonstrate that the genetic variants with significant maximin effects have generalizable effects under new environments.

## 背景

首先考虑一组线性回归模型：

$$
Y^{(l)} = X^{(l)T}b^{(l)} + \epsilon^{(l)}
$$

并且给出目标分布 $Q$ 的 $X^Q$，需要估计 $Y^Q$。我们总有随机变量 $X^{(l)} \sim P_{X}^{(l)}$ 等，但是 $Q_{X} \ne P_{X}^{(l)}$ 。