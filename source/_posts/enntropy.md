---
title: enntropy
date: 2021-01-11 17:27:56
tags:
---

## 几种熵的介绍

### 1.信息熵

给定随机变量x,在系统中可能的输出为x1,x2....xn,这些输出对应发生的概率为p(x1),p(x2)...p(xn)

<!-- more -->

这时x的熵的定义为：
$$
H(x) = -\sum_{i=1}^n p(x_i)log(p(x_i))
$$
这个值是非负的，意义是对于不确定性的度量。当等概率出现时，不确定性达到最大，此时系统最公平

### 2.相对熵

给定随机变量x，有两个离散概率分布$\vec p$和$\vec q$，$\vec p$相对于$\vec q$的相对熵定义为
$$
D_{kl}(\vec p||\vec q)=-\sum_{i=1}^n p(x_i)log(\frac{p(x_i)}{q(x_i)})
$$
这种定义的意义是对两个概率分布p和q的分布差异的度量

具有非负性$D_{kl}(\vec p||\vec q)$>=0,并且$D_{kl}(\vec p||\vec q)\not = D_{kl}(\vec q||\vec p)$

### 3.交叉熵

交叉熵也要给定x和离散概率分布$\vec p$和$\vec q$，则交叉熵可定义为：
$$
H(\vec p,\vec q) = H(\vec p)+D_{kl}(\vec p||\vec q)
$$

$$
=-\sum_{i=1}^n p(x_i)log(q(x_i))
$$

是用于估计概率分布q相对于真实概率分布p的度量，具有非负性，所以可以作为loss使用