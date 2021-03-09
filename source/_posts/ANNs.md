---
title: ANNs
date: 2021-01-11 17:29:06
tags:
---

## 人工神经网络

神经网络解决的问题是线性不可分的问题，但线性不可分问题可以拆分为线性运算的组合

<!-- more -->

### 1.神经网络结构

可以将感知机看作是一种单层神经网络，可以解决线性问题

那么多层感知机就被称为神经网络，用来解决线性不可分问题

### 2.模型表示（全链接）

假设$z_i^{(l)}$是第l层第i个神经单元的的净输入，$a_i^{(l)}$是第l层第i个神经单元的的激活值，$g_l(.)$是第l层的激活函数

最终模型的表达无法表示，但每一层的均可知：
$$
\vec {z^{(l)}} = (\vec{w^{(l)}})^T\vec{a^{(l-1)}}+\vec{b^{(l)}} 
$$
前向传播：$\vec x =\vec {a^{(0)}}\rightarrow (\vec {z^{(1)}}\rightarrow\vec {a^{(1)}})\rightarrow......\rightarrow(\vec {z^{(l)}}\rightarrow\vec {a^{(l)}}=h_{\vec w,\vec b}(x))$

### 3.损失函数（多分类为例）

设最后一层的激活函数是softmax，即$\vec {a^{(l)}} = softmax(\vec {z^{(l)}})$
$$
J_{\vec w,\vec b}(h_{\vec w,\vec b}(\vec x),\vec y) = -\frac {1}{m} \sum_{i=1}^m \sum_{c=1}^Cy^{(i)}log((h_{\vec w,\vec b}(\vec x^{(i)})_c
$$

### 5.优化方法

1.梯度下降

2.反向传播

### 7.自动梯度计算

数值微分，符号微分（对公式求导）、自动微分$\Rightarrow$计算图

### 8.激活函数

1.Sigmoid
$$
Sigmoid： h_{\vec \theta}(\vec x) = \frac {1} {1+e^{-{\vec \theta}^T \vec x}}
$$
2.Softmax
$$
Softmax:h_{\vec \theta}(\vec x) = \frac {e^{-{\vec \theta}_c^T \vec x}} {\sum_{i=1}^Ne^{-{\vec \theta}^T \vec x}}
$$
3.Relu
$$
Relu(x) = \begin{cases} x& x\geq0\\ 0& x=0 \end{cases}
$$
4.LeakyRelu
$$
LeakyRelu(x) = \begin{cases} x& x\geq0\\ \delta x& x=0 \end{cases}
$$
5.swish函数
$$
swish(x) = x\times Sigmoid(\beta x)
$$
$\beta$是可学习固定参数或者超参数

6.maxout
$$
maxout(\vec z)=max_{k\in (l,k)}z_k \
$$

### 9.卷积神经网络

全连接NN无法表示局部不变特征（无法胜任图像分类）

所以需要卷积神经网络来进行扩展，卷积神经网络学习的参数是卷积核的参数

这里的卷积表示的是直接对位相乘$\vec r = \vec w *\vec x$ 

基本结构：一般由卷积层,polling层（降采样层）和全连接层构成

### 10.循环神经网络RNN

用来处理具有时序的数据
$$
\vec h_t=g(\vec w \vec h_{t-1}+\vec w \vec x_t +\vec b)
$$
