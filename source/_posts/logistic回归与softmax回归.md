---
title: logistic回归与softmax回归
date: 2021-01-11 09:59:06
tags: [神经网络, DeepLearning]
---

## Logistic回归

逻辑回归问题不是回归问题，而是一种分类问题，默认为二分类

### 1.模型表示

<!-- more -->
$$
Sigmoid： h_{\vec \theta}(\vec x) = \frac {1} {1+e^{-{\vec \theta}^T \vec x}}
$$

物理意义：这是$h_{\vec \theta}(\vec x)$是$\vec x$为正样本的概率（输出为1的概率）是人为规定的，用指数形式表示是为了加大两个概率之间的区别。

### 2.分类的决策

在分类问题中，函数只会输出$h_{\vec \theta}(\vec x)$中$\vec x$属于某一种类的概率，但没有给出样本究竟属于哪一类，所以就需要利用决策函数进行分类。

决策函数：$\hat y = f(x)= \begin{cases} 0& \text{g(z)<0.5}\\ 1& \text{g(z) >= 0.5} \end{cases}$

### 3.损失函数


$$
loss：Cost(h_{\vec \theta}(\vec x^{(i)}),y^{(i)})\begin{cases} -log(h_{\vec \theta}(\vec x^{(i)})& \text{y^{(i)}=1}\\ -log(1-h_{\vec \theta}(\vec x^{(i)})& \text{y =0} \end{cases}
$$
类似于$-p_ilogq_i$这种形式，我们称为交叉熵，它表示了预测值与真实值之间的差异,其中$-p_i$表示真实值，$logq_i$表示预测值

上式可以同一形式变化为
$$
J(\vec \theta) = \frac {1}{m} \sum_{i=1}^N [-y^{(i)}log(h_{\vec \theta}(\vec x^{(i)})-(1-y^{(i)})log(1-h_{\vec \theta}(\vec x^{(i)})]
$$

### 4.梯度下降优化

$$
\frac {\delta J_{\vec \theta} }{\vec{\theta_j} } =\frac {1}{m} \sum_{i=1}^N\frac {\delta J_{\vec \theta} }{\delta h_{\vec \theta}(\vec x^{(i)})}\frac {\delta h_{\vec \theta}(\vec x^{(i)})}{ {\delta \vec \theta}^T \vec x^{(i)}}\frac { {\delta \vec \theta}^T \vec x^{(i)} }{\delta \vec{\theta_j} }
$$


$$
=\frac {1}{m}\sum_{i=1}^N(h_{\vec \theta}({\vec x}^{(i)})-y^{(i)}){x_j}^{(i)}
$$
那么梯度下降的跟新规则即为
$$
\vec \theta=\vec \theta-\alpha\frac {1}{m}\sum_{i=1}^N(h_{\vec \theta}({\vec x}^{(i)})-y^{(i)}){x_j}^{(i)}
$$

### 5.随机梯度下降优化

随机梯度下降优化是随机选择一个样本进行梯度计算，然后更新权重，并重复下一过程

### 6.小批量梯度下降法

这个方法最常用，是GD与SGD的这种方法，取一个小批量的数据对参数进行更改。

### 7.牛顿法

$$θ:=θ−f(θ)f′(θ)$$

### 8.二分类评价指标

|                | $y_{gt}=1$         | $y_{gt}\not= 1$     |
| -------------- | ------------------ | ------------------- |
| $\hat y=1$     | True Positive (TP) | Flase Positive (FP) |
| $\hat y\not=1$ | Flase Negative(FN) | True Negative (TN)  |

$$
precision(精确率，查找率)= \frac{TP}{TP+FP} 【\hat y=1的所有项】 高代表找的更对
$$

$$
Recall(召回率)=\frac{TP}{TP+FN} 【y_{gt}=1的所有项】 高代表保证不漏
$$

这两个参数不可兼得，需要根据实际情况来。

## SoftMax分类

Softmax是一种多分类问题的解决方法，是logistics的一种多维形式的推广，多维分类通常采用独热码表示。

### 1.模型表示

$$
Softmax:h_{\vec \theta}(\vec x) = \frac {e^{-{\vec \theta}_c^T \vec x} } {\sum_{i=1}^Ne^{-{\vec \theta}^T \vec x} }
$$

### 2.损失函数

$$
loss: J(\vec \theta) = -\frac {1}{m} \sum_{i=1}^m \sum_{c=1}^Cy^{(i)}log(h_{\vec \theta}(\vec x^{(i)})
$$

$$
=-\frac {1}{m} \sum_{i=1}^m(y^{(i)})^Tlog(h_{\vec \theta}(\vec x^{(i)})
$$

实际上只算了$y_c^{(i)}$为1的那个loss

### 3.优化

$$
\vec \theta=\vec \theta-\alpha\frac {1}{m}\sum_{i=1}^N{x_j}^{(i)}(h_{\vec \theta}({\vec x}^{(i)})-y^{(i)})
$$

## 正则化

### 1.过拟合：

可能是因为模型过于复杂，模型可以很好的拟合训练数据，但不能拟合预测数据。可能会导致预测失败。

### 2.解决方法：

1.减少特征数量 

2.正则化（通过降低$J_{\vec \theta}$的值来改变一些高次特征）

### 3.正则项

通式：
$$
J(\theta)=\frac {1}{2m} [\sum_{i=1}^N (h_\theta(x^{(i)})-y^{(i)})^2+\lambda\sum_{j=1}^N\theta_j^2]
$$
