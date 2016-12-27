---
title: Machine Learning (西瓜书 周志华)
date: 2016-12-27
tags: [Machine Learning,读书笔记]
---

本文记录西瓜书的读书笔记。

<!--more-->
## 自助法 ##
- 有放回的多次抽样生成训练集，剩余的作为测试集。
- 测试集大小(1-1/m)^m，取极限得1/e，约36.8%。
- 自助法改变了数据集的分布，会引入估计偏差。


## 调参与最终模型 ##
- 训练数据分为训练集和验证集，验证集可用来调参
- 模型优化完后，需在全量数据集上重新模型，再交付出去


## 偏差与反差 ##
-- Error = bias + variance + nosie
-- noise是数据本身带来的，算法不可解，泛化误差的下界，刻画问题本身的难度