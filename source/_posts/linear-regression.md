---
title: linear-regression
date: 2021-01-10 15:39:59
tags:
---

## 机器学习之线性回归

### 什么是线性回归

线性回归问题作为机器学习的经典，是一种有监督学习

<!-- more -->

所谓的线性回归问题就是希望用线性模型去拟合数据，利用回归方法从而得到回归方程，从而预测结果

线性回归的学习可以用如下公式表示
$$
\hat y = h_\theta(x)
$$
其中`x`是输入，`y`是输出/目标，$\theta$是要学习的参数。$h_\theta()$是指和$\theta$相关的模型，h()由模型给定。

当$\hat y$是连续的，且$h_\theta()$是线性的，那么这个模型就是线性回归模型。线性模型表示为
$$
h(x) = \theta_0+\theta_1x
$$
其中$\theta_0,\theta_1$可以表示为向量$\theta = (\theta_0,\theta_1)^T$

则有$h_{\vec \theta}(x)=\theta_0+\theta_1x$

### 监督学习框架

#### 训练： 

$$
训练集(x,y) \rightarrow 学习算法 \rightarrow h:假设函数 \hat y = h_\theta(x)
$$

#### 测试：

$$
x_{new} \rightarrow \hat y=h_\theta(x_{New})
$$

#### 启发式评估：

Evaluation

### 损失函数

由于训练的最终目标是让$h_\theta$与y越接近越好，那么损失函数可以定义为：
$$
J(\theta)=\frac {1}{2m} \sum_{i=1}^N (h_\theta(x^{(i)})-y^{(i)})^2
$$
平方误差最小准则：优化问题就是找到使$J(\theta)$最小的解

### 求闭式解

$$
J(\theta)=\frac {1}{2m} \sum_{i=1}^N (h_\theta(x^{(i)})-y^{(i)})^2
$$

$$
=\frac {1}{2m} \sum_{i=1}^N (\theta_0+\theta_1(x^{(i)})-y^{(i)})^2
$$

$$
\frac {\delta J}{\delta \theta_0}=\frac{1}{m}\sum_{i=1}^N (\theta_0+\theta_1(x^{(i)})-y^{(i)})
$$

$$
\frac {\delta J}{\delta \theta_1}=\frac{1}{m}\sum_{i=1}^N (\theta_0+\theta_1(x^{(i)})-y^{(i)})x^{(i)}
$$

令(8)与(9)为0，解得：
$$
\theta_0=\frac{1}{m}\sum_{i=1}^N (y^{(i)}-\theta_1(x^{(i)}))
$$

$$
\theta_1=\frac{\sum_{i=1}^N (y^{(i)}-\theta_0) x^{(i)}}{\sum_{i=1}^N {x^{(i)}}^2}
$$

### 梯度下降

沿着梯度反方向改变（函数值下降最快的地方）即
$$
\theta_j = \theta_j-\alpha \frac {\delta J(\theta_0,\theta_1)}{\theta_j}
$$
这种方式让每次都算出$\theta_j$再进行下一轮更新，但可能出现局部最优解

### 线性回归中的梯度下降

$$
\frac {\delta J(\theta_0,\theta_1)}{\theta} =\frac{1}{m}\sum_{i=1}^N (\theta_0+\theta_1(x^{(i)})-y^{(i)})(x^{(i)})
$$

### 多元线性回归

假设函数为$h_\theta(x)=\theta_0+\theta_1x+......\theta_nx$

即$h_{\vec \theta}(x)=\vec \theta^T \vec x $

### 多元梯度：

$$
\frac {\delta J_{\vec \theta}}{\vec{\theta_j}} =\frac {1}{m} \sum_{i=1}^N (h_{\vec \theta}({\vec x}^{(i)})-y^{(i)}){x_j}^{(i)}
$$

### 多元闭式解

$$
{ J_{\vec \theta}} =\frac {1}{2m} \sum_{i=1}^N (h_{\vec \theta}({\vec x}^{(i)})-y^{(i)})^2 = \frac{1}{2}(\vec x \vec \theta -\vec y)^T(\vec x \vec \theta -\vec y)
$$

$$
\bigtriangledown_\theta J(\vec \theta) = \bigtriangledown_\theta\frac{1}{2}(\vec x \vec \theta -\vec y)^T(\vec x \vec \theta -\vec y)
$$

$$
{\vec x}^T\vec x\vec \theta-{\vec x}^T\vec y = 0 \Rightarrow \vec \theta =  { {\vec x}^T\vec x}^{-1}{\vec x}^T\vec y
$$
