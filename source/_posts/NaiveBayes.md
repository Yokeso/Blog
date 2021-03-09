---
title: NaiveBayes
date: 2021-01-11 17:30:04
tags:
---

## 朴素贝叶斯

logistic回归：本质是对$p(y|\vec x)$进行建模

<!-- more -->

贝叶斯公式：求$\vec x$和y的联合概率（联合分布）$p(y|\vec x) = \frac {p(\vec x,y)}{p(\vec x)}$

判别模型：对$p(y|\vec x)$进行建模，且$\sum_{y}p(y|\vec x)=1$（分类问题）

生成模型：对$p(\vec x,y)$进行建模，且$\sum_{y}p(\vec x,y)=1$（联合分布）

决策函数：$\hat y =argmax_cp(\vec x,c)=argmax_cp(c)\prod p(\vec x_i,c)$

样本修正（拉普拉斯修正）
$$
\breve p(x_i|c)=\frac{|D_{c_i}x_i|+1}{|D_c|+N_i}
$$
分子加1是为了避免0概率，分母则是为了将第i个特征的可能取值数添加进去