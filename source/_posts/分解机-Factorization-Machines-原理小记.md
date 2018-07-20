---
title: 分解机(Factorization Machines)原理小记
date: 2018-07-08 15:44:18
tags: [Machine learning]
toc: true
categories: [Machine learning]
mathjax: true
description: FM算法，全称Factorization Machines, 一般翻译为“因子分解机”。 2010年，它由当时还在日本大阪大学的Steffen Rendle提出。此算法的主要作用是可以把所有特征进行高阶组合，减少人工参与特征组合的工作，工程师可以将经历集中在模型参数调优。FM只需要线性时间复杂度，可以应用于大规模机器学习。经过部分数据集试验，此算法在稀疏数据集上的效果要明显好于SVM。
---

FM算法，全称Factorization Machines, 一般翻译为“因子分解机”。 2010年，它由当时还在日本大阪大学的Steffen Rendle提出。此算法的主要作用是可以把所有特征进行高阶组合，减少人工参与特征组合的工作，工程师可以将经历集中在模型参数调优。FM只需要线性时间复杂度，可以应用于大规模机器学习。经过部分数据集试验，此算法在稀疏数据集上的效果要明显好于SVM。

## 预测任务

在机器学习中，预测(prediction)是一项最基本的任务。所谓的预测就是估计一个函数$$\hat{y}:\Bbb{R} ^n \rightarrow T \tag{1.1}$$该函数将一个$n$维的是指特征向量$x \in \Bbb{R}^n$映射到一个目标域$T$，例如对于回归$T=\Bbb{R}$对于分类问题$T=\{1, -1\}$在监督学习场景中，通常还有一个带标签的训练数据集$$D=\{(x^{(1)}, y^{(1)}), (x^{(2)}, y^{(2)}), ..., (x^{(N)}, y^{(N)}) \} \tag{1.2}$$其中$x^{(i)} \in \Bbb{R} ^n$表示输入数据，对应样本的特征向量，$y^{(i)}$对应标签，$N$为样本的数目。

在现实世界中，许多应用问题(如文本分析、推荐系统等)会产生高度稀疏的（特征）数据，即特征向量$x^{(i)}$中几乎所有的分量都为0.这里以电影评分系统为例，给出一个关于高度稀疏数据的实例。 

* 例1.1 * 考虑电影系统中的数据，它的每一条记录都包含了哪个用户$u \in U$在什么时候 $t \in \Bbb{R}$对哪部电影$i \in I$打了多少分$r \in \{1, 2, 3, 4, 5\}$这样的信息，假定用户集$U$和电影集$I$分别为$$U = \{Alice(A), Bob(B), Charlie(C), ...\}$$ $$I=\{Titanic(TI), Notting Hill(NH), Star Wars(SW), Star Trek(ST), ...\}$$设观测数据集为$$S=\{(A, TI, 2010-1, 5), (A, NH, 2010-2, 3), (C, TI, 2009-9, 1) ...\}$$

利用观测数据集$S$来进行预测任务的一个实例是：估计一个函数$\hat{y}$来预测某个用户在某个时刻对某部电影的打分行为。

有了观测数据$S$，如何构造形如$(1.2)$那样的样本数据呢？图1给出了利用$S$构造特征向量和标签的示例。

![图1 利用观测数据 S 构造特征向量和标签的示例](https://wx2.sinaimg.cn/mw690/7c35df9bly1ft2j3mmnefj20z60g2n64.jpg)

图1中的标签部分比较简单，直接将当前用户对当前电影的评分作为标签，如第一条观测记录中，Alice对TI的评分是5，则有$y ^{(1)} = 5$, 而特征向量$x$由以下5部分构成：

1. 第一部分（对应蓝色方框）表征*当前评分用户信息*，其维度为$|U|$，该部分的分量中，当前电影评分用户所在的位置为1，其它为0，例如在第一条观测记录中就有$x_A^{(1)} = 1$。
2. 第二部分（对应红色方框）表征*当前被评分电影的信息*，其维度为$|I|$，该部分的分量中，当前被评分的电影所在的位置为1，其他为0.例如第一条观测记录中就有$x_{TI} ^{(1)} = 1$。
3. 第三部分（对应黄色方框）表征*当前评分用户评分过的所有电影信息*，其维度为|I|，该部分的分量中，被当前用户评论过的所有电影（设个数为$n_I$）的位置为$\frac{1}{n_I}$,其他为0.例如Alice评价过散步电影TI,NH,和SW，因此就有$x^{(1)}_{TI} = x^{(1)}_{NH} = x^{(1)}_{SW} = 1/3$。 
4. 第四部分（对应绿色方框）表征*评分日期信息*，其维度为1，用来表示用户评价电影的时间，表示方法是将（记录中最早的日期）2009年1月作为基数1每增加一个月就加1，例如2009年5月就为5. 
5. 第五部分（对应棕色方框）表征*当前评分用户最近评分过的一部电影的信息*，其维度为$|I|$，该部分的分量中，若当前用户评价当前电影之前还评价过其他电影，则将当前用户评价的上一步电影的位置取为1其他为0；若当前用户评价当前电影之前没有评价过其他电影，则所有分量都取为0.例如，对于第二条观测记录，Alice评价NH之前评价的是TI,因此有$x_{TI} ^{(2)} = 1$。

*注1.1 上述描述中，x的上标表示观测记录的编号，下标表示分量的位置，为简单起见，这里通过颜色和符号来进行区分，而实际应用中，会将$x$的所有分量进行连续编号。*

在本例中，特征向量$x$的总维度为$|U| + |I| + |I| + 1 + |I| = |U| + 3|I| + 1$。在一个真是的电影评分系统中，用户数目|U|和电影的数目|I|是很大的，而每个用户参与评论的电影数目则相对很小，于是，可想而知，每条记录对应的额特征向量将会多么的稀疏。为定量刻画这种稀疏性，记$N_z(x)$表示$x$中非零向量的个数，并记$$\overline{N_z}(X) = \frac {1} {N} \sum _{i=1} ^N N_z(x^{(i)})$$其中$X$表示所有样本按行拼成的矩阵(Design Matrix)，即
$$ 
   \begin{pmatrix} 
   x_1^{(1)} & x_2^{(1)} & \cdots & x_n^{(1)} \\  
   x_1^{(2)} & x_2^{(2)} & \cdots & x_n^{(2)} \\  
   \vdots & \vdots  & \ddots & \vdots \\
   x_1^{(N)} & x_2^{(N)} & \cdots & x_n^{(N)} \\  
   \end{pmatrix}
$$
则$\overline{N_z}(X)$表征了训练集$D$中所有特征向量中非零分量的数的平均值。显然对于稀疏数据，成立$\overline{N_z}(X) << n$

## 模型方程

### 二阶表达式

在介绍FM的模型方程之前，先回顾一下线性回归。对于一个给定的特征向量$x = (x_1, x_2, ..., x_n)^{\top}$，线性回归建模时采用的函数是$$\begin{align} \hat{y}(x) &= w_0 + w_1 x_1 + w_2 x_2 + ... + w_n x_n \\ &= w_0 + \sum _{i = 1} ^n w_i x_i \end{align} \tag{2.3}$$其中，$w_0$和$\bf{w} = \{w_1, w_2, ..., w_n \} ^{\top}$为模型参数。从方程易见：各特征分量$x_i$和$x_j(i \neq j)$之间是相互独立的，即$\hat{y}(x)$中仅考虑单个的特征分量，而没有考虑特征分量之间的相互关系(interaction)。 

接下来我们在$(2.3)$的基础上，将函数$\hat{y}$改写为$$\hat{y}(x) = w_0 + \sum _{i = 1} ^n w_i x_i + \sum _{i=1} ^{n-1} \sum _{j=i+1} ^n w_{ij} x_i x_j \tag{2.4}$$这样，便将任意两个（互异）特征分量之间的关系也考虑进来了。

不过遗憾的是，这种直接在$x_i x_j$前面配上一个系数$w_{ij}$的方式在稀疏矩阵中有一个很大的缺陷，我们通过举例来进行说明。 

仍然考虑前面的例$(1.1)$，观测数据集$S$中没有Alice评价电影Star Trek的记录，如果要直接估计Alice(A)和Star Trek（ST）之间，或者说特征分量$x_A$和$x_{ST}$之间的相互关系，显然会得到系数$w_{A,ST} = 0$，即**对于观察样本中未出现过交互的特征分量，不能对相应的参数进行估计**。注意，在高度系数数据场景中，由于数量的不足，样本中出现未交互的特征分量是很普遍的。 

为了克服这个缺陷，我们在$(2.4)$中的系数$w_{ij}$上做文章，将其表成另外的形式。为此，针对每个维度的特征分量$x_i$引入辅助向量$$v_i = (v_{i1}, v_{i2}, ..., v_{ik}) ^{\top} \in \Bbb{R} ^k, i = 1,2, ..., n$$其中$k \in \Bbb {N} ^+$为超参数，并将$w_{ij}$改写为$$\hat{w}_ij = v_i ^{\top} v_j := \sum _{l=1} ^k v_{il}v_{jl} \tag{2.5}$$

这样就能克服上面提到的缺陷了么？我们粗略的分析一下。在观测数据中，Bob(B)对Start Wars(SW)和Star Trek(ST)的评分差不多。由此可认为他们对应的辅助向量$v_{ST}$和$v_{SW}$也比较相似。从而，利用$\hat{w}_{A,SW} = v_A ^{\top} v_{SW}$就可以对$\hat{w}_{A,ST} = v_A ^{\top} v_{ST}$进行估计了。（当然这并不是说，真的要先计算出$\hat{w}_{A,SW}$然后用它作为$\hat{w}_{A,ST}$的近似。因为这些性质都是观测数据中内在的，我们要做的只是建好模型，让模型在训练阶段自动去学习。）

于是，函数$\hat{y}$可以进一步写成$$\hat{y}(x) = w_0 + \sum _{i = 1} ^n w_i x_i + \sum _{i=1} ^{n-1} \sum _{j=i+1} ^n (v_i^{\top} v_j) x_i x_j \tag{2.6}$$

从数学的角度来看，$(2.4)$和$(2.6)$的主要区别在哪呢？对于交叉项$x_i x_j$的系数，前者用的是$w_{ij}$，后者用的是$\hat{w}_{ij}$为更好的看清楚他们之间的关系，引入几个矩阵：

- 将$\{v_i\}_{i=1}^{n}$按行拼接成长方$V$ 
$$ V = 
   \begin{pmatrix} 
   v_{11} & v_{12} & \cdots & v_{1k} \\  
   v_{21} & v_{22} & \cdots & v_{2k} \\  
   \vdots & \vdots  & \ddots & \vdots \\
   v_{n1} & v_{n2} & \cdots & v_{nk} \\  
   \end{pmatrix}
   = 
   \begin{pmatrix}
   v_1^{\top} \\ 
   v_2^{\top} \\ 
   \vdots\\
   v_n^{\top} \\ 
   \end{pmatrix}
   \tag{2.7}
$$

- 交互矩阵（interaction matrix）W
$$ W = 
   \begin{pmatrix} 
   w_{11} & w_{12} & \cdots & w_{1n} \\  
   w_{21} & w_{22} & \cdots & w_{2n} \\  
   \vdots & \vdots  & \ddots & \vdots \\
   w_{n1} & w_{n2} & \cdots & w_{nn} \\  
   \end{pmatrix}
   \tag{2.8}
$$

- 交互矩阵$\hat{W}$
$$ 
\begin{align}
\hat{W} & = VV^{\top} = 
   \begin{pmatrix}
     v_1^{\top} \\ 
     v_2^{\top} \\ 
     \vdots\\
     v_n^{\top} \\ 
   \end{pmatrix}
   (v_1, v_2, ..., v_n) \\
   & = 
   \begin{pmatrix}
   v^{\top}_1 v_1 & v^{\top}_1 v_2 & \cdots & v^{\top}_1 v_n \\  
   v^{\top}_2 v_1 & v^{\top}_2 v_2 & \cdots & v^{\top}_2 v_n \\  
   \vdots & \vdots  & \ddots & \vdots \\
   v^{\top}_n v_1 & v^{\top}_n v_2 & \cdots & v^{\top}_n v_n \\  
   \end{pmatrix} \\
   & = 
   \begin{pmatrix}
   v^{\top}_1 v_1 & \hat{w}_{12} & \cdots & \hat{w}_{1n} \\  
   \hat{w}_{21} & v^{\top}_2 v_2 & \cdots & \hat{w}_{2n} \\  
   \vdots & \vdots  & \ddots & \vdots \\
   \hat{w}_{n1} & \hat{w}_{n2} & \cdots & v^{\top}_n v_n \\  
   \end{pmatrix} \\
\end{align}
\tag{2.9}
$$

由此可见，$(2.4)$和$(2.6)$分别采用了交互矩阵$W$和$\hat{W}$的非对角线元素作为$x_i x_j$的系数。由于$\hat{W} = V V^{\top}$对应一种矩阵分解，因此我们将这种以$(2.6)$作为模型方程的方法称为Factorization Machines方法。

读者可能要问了，$VV^{\top}$的表达能力（expressiveness）怎么样呢？即任意给定一个交互矩阵$\hat{W}$，能否找到相应的分解矩阵$V$呢？答案是肯定的，这里需要用到一个结论“**当$k$足够大时，对任意对称正定的实矩阵$\hat{W} \in \Bbb{R}^{n*n}$均存在实矩阵$V \in \Bbb{R}^{n*k}$，使得成立$\hat{W} = VV^{\top}$。**”

由于$(2.6)$中只涉及满足$i < j$的$x_i, x_j$，因此， $\hat{W}$的对称性没有问题， 那如何保证$\hat{W}$的正定性呢？注意，我们只关心互异特征矩阵分量之间的相互关系
，因此$\hat{W}$的对角线元素是可以任意取值的，只需将它们取的足够大（例如）保证行元素满足严格对角占优，就可以保证$\hat{W}$的正定性。 

理论分析中，我们要求参数$k$取得足够大，但是，在高度系数数据场景中，由于没有足够的样本来估计复杂的交互矩阵，因此通常应该取得很小。事实上，对参数$k$(亦即FM的表达能力)的限制，在一定程度上可以提高模型的泛化能力。这种能够利用$\hat{W}$的低秩近似的性质，也是FM的一个优势。

### 复杂度分析

FM模型方程（2.6）中需要估计的参数包括
$$
w_0 \in \Bbb{R}, \bf w \in \Bbb R^n, V \in \Bbb R^{n \times k} \tag{2.10}
$$
共有$1 + n + nk$(关于$n$和$k$都是线性的)个，其中$w_0$为整体的偏置量，$\bf w$对特征向量的各个分量的强度进行建模， $V$对特征向量中任意两个分量之间的关系进行了建模。 

那么模型方程$(2.6)$的计算复杂度如何呢？直接从公式上来看，其时间复杂度为
$$
\{n + (n - 1)\} + \left \{ \frac {n(n-1)} {2} [k + (k-1) + 2] + 
 \frac {n(n-1)} {2} - 1 \right\} + 2 = \mathcal{O}(k n^2 )
$$
其中第一个花括号对应$\sum_{i=1}^n w_i x_i$的加法和乘法擦作数，第二个花括号对应$\sum _{i = 1} ^{n-1} \sum _{j = i+1} ^n (v_i ^{\top} v_j)x_i x_j$
的加法和乘法操作数。

不过有意思的是，通过对$(2.6)$进行改写，可以将其计算复杂度降成线性的$\mathcal{O}(kn)$，具体推到过程如下：
$$
\begin{align}
\sum_{i=1} ^{n-1} \sum_{j=i+1} ^n (v_i^{\top} v_j) x_i x_j 
& = \frac {1} {2} \left ( \sum_{i=1} ^{n} \sum_{j=1} ^n (v_i^{\top} v_j) x_i x_j - \sum _{i=1}^n (v_i^{\top} v_i) x_i x_i \right ) \\
& = \frac {1} {2} \left ( \sum_{i=1} ^{n} \sum_{j=1} ^n \sum_{l=1} ^k v_{il}v_{jl} x_i x_j - \sum _{i=1}^n \sum _{l=1} ^k v_{il} ^2 x_i ^2 \right) \\
& = \frac {1} {2} \sum_{i=1}^k \left ( \sum_{i=1} ^n (v_{il})(x_i) \sum_{j=1}^n(v_{jl}x_j) - \sum_{i=1}^n v_{il}^2 x_i^2 \right) \\ 
& = \frac {1} {2} \sum_{l=1}^k \left [ \left( \sum_{i=1} ^n (v_{il} x_i) \right)^2 - \sum_{i=1}^n v_{il}^2 x_i^2  \right]
\tag{2.11}
\end{align}
$$

易见，$2.11$的计算时间复杂度为
$$
k\{[n+(n-1)+1] + [3n + (n - 1)] + 1\} + (k - 1) + 1 = \mathcal{O}(k n)
$$
注意在高度稀疏的数据场景中，特征向量$x$中绝大部分分量为0，即$N_z(x)$很小，而$(2.11)$中关于$i$的求和只需对非零元素进行，于是，计算复杂度变成了$\mathcal{O}(k N_z(x))$，对整个数据集而言，平均复杂度就是$\mathcal{O}(\overline{N_z(X)})$

## 回归和分类

利用FM的模型方程，可以进行各种预测任务，如回归（Regression）、分类（Classification）和排名（Ranking）等，下面我们以回归和二分类为例，给出这两种预测任务中优化目标函数的构造，优化目标函数通常取为形如：
$$L = \sum _{i = 1} ^N loss(\hat {y}(\bf{x} \rm ^{(i)}), y^{(i)})$$
的整体损失函数，优化的目标是将其最小化，其中$loss(\hat {y}(\bf{x} \rm ^{(i)}), y^{(i)})$ 成为关于第$i$个样本$(\bf{x} \rm ^{(i)}, y^{(i)})$的损失函数。

### 回归问题

对于回归问题， 损失函数可取为最小平方误差（least square error）即$$loss(\hat {y}, y) = (\hat{y} - y) ^2 \tag{3.14}$$

### 二分类问题

对于二分类（Binary Classification）问题（其中标签$y \in \{ +1, -1\}$）,损失函数可取为hinge loss函数活logit loss函数

1. hinge loss函数

$$
loss(\hat{y}, y) = \max(0, 1 - y \hat{y}) \tag{3.15}
$$

当$y = +1$时 
$$
loss(\hat {y}, y) = max\{0, 1 - \hat{y}\} = 
\begin{cases} 
0, \quad \hat{y} \geq 1; \\ 
1 - \hat{y}, \quad \hat{y} < 1
\end{cases}
$$

当$y = -1$时
$$
loss(\hat {y}, y) = max\{0, 1 + \hat{y}\} = 
\begin{cases} 
0, \quad \hat{y} \leq -1; \\ 
1 + \hat{y}, \quad \hat{y} > -1
\end{cases}
$$

模型训练好之后，就可以利用$\hat{y}(\bf{x} \rm)$的的正负号来预测$\bf{x}$的分类了。

2. logit loss函数

$$loss(\hat{y}, y) = -\ln \sigma (\hat{y}y) \tag{3.16}$$ 

其中， $\sigma(x) = \frac {1} {1 + \exp(-x)}$为sigmod函数。由定义$(3.16)$可见$\hat{y}$和$y$越接近，损失函数$loss(\hat{y}, y)$就越小。 

此外，为了防止过拟合，我们通常会在优化目标函数中加入正则项（如L2正则）。 

## 学习算法

目前FM的学习算法主要包括以下三种

- 随机梯度下降法（Stochastic Gradiant Descent, SGD）; 
- 交替最小二乘法（Alternating Least-Squares, ALS）; 
- 马尔科夫链蒙特卡罗法（Markov Chain Monte Carlo， MCMC）， 

本节将对前面两种学习算法做简要介绍。 

### Multilinearity

在讨论具体的学习算法之前，我们先引入FM的一个重要性质——multilinearity：若记$\Theta = (w_0, w_1, w_2, \cdots , w_n, v_{11}, v_{12}, \cdots, v_{nk}) ^{\top}$表示FM的所有模型参数， 则对任意的$\theta \in \Theta$存在两个与$\theta$取值无关的函数$g_{\theta}(\bf{x} \rm)$和$h_{\theta}(\bf{x} \rm)$，使得成立
$$
\hat{y}(\bf{x} \rm) = g_{\theta}(\bf{x} \rm) + \theta h_{\theta}(\bf{x} \rm) \tag{4.17}
$$

具体有

1. 当$\theta = w_0$时 
$$
\hat{y}(\mathrm{x}) = \color{red}{\sum _{i = 1} ^n w_i x_i + \sum _{i=1}^{n-1}\sum_{j=i+1}^n(v_i^{\top}v_j)x_ix_j} +  w_0 \cdot \color{blue}{1} \tag{4.18}
$$

其中红色部分为$g_{\theta}(\bf{x} \rm)$，蓝色部分为$h_{\theta}(\bf{x} \rm)$。下同。 

2. 当$\theta = w_l (l=1,2,\cdots,n)$时
$$
\hat{y}(\mathrm{x}) = \color{red}{w_0 + \sum _{i=1;i \neq l} ^n w_i x_i + \sum _{i=1}^{n-1} \sum _{j=i+1} ^n (v_i^{\top}v_j)x_ix_j} + w_l \cdot \color{blue} {x_l} \tag{4.19}
$$

3. 当$\theta = v_{lm}$时
$$
\hat{y}(\mathrm{x}) = \color{red}{w_0 + \sum _{i=1} ^n w_i x_i + \sum_{i=1} ^{n=1} \sum _{j=i+1} ^n (\sum _{s=1;is \neq lm; js \neq lm} v_{is}v_{js})x_ix_j} + v_{lm} \cdot \color{blue}{x_l \sum_{i \neq l} v_{im}x_i} \tag{4.20}
$$

从$(4.18)$,$(4.19)$和$(4.20)$可见，$g_{\theta}(\mathrm{x})$的表达式比较复杂，而$h_{\theta}(\mathrm{x})$的表达式比较简单，因此在实际推导时，通常只计算$h_{\theta}(\mathrm{x})$，而$h_{\theta}(\mathrm{x})$则利用$\hat{y}(\mathrm{x}) - \theta h_{\theta}(\mathrm{x})$来计算。

利用$(2.6)$和$(2.11)$，不难算得$\hat{y}(mathrm{x})$关于$\theta$的偏导数
$$
\frac {\partial \hat{y}(x)} {\partial \theta} = 
\begin{cases}
1, & \quad \theta = w_0; \\  
x_l, & \quad \theta = w_l, \quad l = 1, 2, \cdots, n; \\
x_l \sum _{s=1;s \neq l} ^{n} v_{sm} x_s, & \quad \theta = v_{lm}, \quad l=1,2,\cdots,n; \quad m=1,2,\cdots,l \\[2ex]
\end{cases}
\tag{4.21}
$$

将其与$h_{\theta}(x)$即($(4.18)$,$(4.19)$和$(4.20)$)中的蓝色部分做对比，容易发现两者是一致的，这是巧合么？当然不是，事实上，在$(4.17)$两边同时对$\theta$求偏导数，便有
$$h_{\theta} = \frac {\partial \hat{y}(x)} {\partial \theta} \tag{4.22}$$ 

### 最优化问题

前面我们提到，FM的优化目标函数是整体损失函数
$$L = \sum_{i=1} ^N loss(\hat{y}(x^{(i)}), y^{(i)})$$

其中对于回归问题，loss去最小平方误差$$loss^R(\hat{y},y) = (\hat{y}-y)^2$$

对于二分类问题，loss取logit函数
$$loss^C(\hat{y}, y) = -\ln \sigma (\hat{y}y) $$ 

于是FM的最优化问题就变成了
$$\Theta ^*= \arg_{\theta} \min \sum_{i=1}^N loss(\hat{y}(x^{(i)}),y^{(i)})$$

通常我们还考虑L2正则，因此，最优化问题就变成
$$\Theta ^*= \arg_{\theta} \min \sum_{i=1}^N \left( loss(\hat{y}(x^{(i)}),y^{(i)}) + 
\sum _{\theta \in \Theta} \lambda _{\theta} \theta ^2 \right)$$
其中，$\lambda_{\theta}$表示参数$\theta$的正则化系数。 

FM的参数$\Theta$通常较大，如果每一个参数$\theta \in \Theta$都对应一个正则化系数，那么这些正则化系数的确定将变得很繁琐。因此我们考虑分组策略，即先对参数进行分组，然后分在同一个组中的参数适用同一个正则化系数，从而减少正则化系数的个数。具体可以这样来对参数进行分组：$w_0$单独一组，$w_1,w_2,\cdots,w_n$按特征分量的意义组成$\Pi$组，从而相应的正则化系数即$\lambda$就是
$$
\lambda^0, \lambda_{\pi(i)}^w, \lambda_{\pi(i),j}^v, \quad i \in \{1, 2, \cdots, n\}, \quad j \in \{1, 2, ..., k\} 
$$
其中$\pi : \{1,2,\cdots, n\} \rightarrow \{1,2,...,\Pi\}$, $\pi(i)$表示参数$w_i$被分在第$\pi(i)$组。 

### 随机梯度下降法（SGD）

首先给出利用SGD训练FM模型的算法流程。 

![](https://wx3.sinaimg.cn/mw690/7c35df9bly1ftdvklbg1wj20xu0xyqas.jpg)

算法$4.1$中涉及损失函数的偏导数，具体的计算公式如下

- 当$loss = loss^R$时 
$$
\frac {\partial loss ^R (\hat{y}(\mathrm{x}), y)} {\partial \theta} = 2(\hat{y}(\mathrm{x}) - y) \frac {\partial \hat{y}(\mathrm {x})} {\partial \theta}
$$

- 当$loss = loss^C$时 
$$
\begin{align}
\frac {\partial loss ^C (\hat{y}(\mathrm{x}), y)} {\partial \theta} & = - \frac {1} {\sigma(\hat{y}(\mathrm{x})y)}\sigma(\hat{y}(\mathrm{x})y)[1- \sigma(\hat{y}(\mathrm{x})y)] \frac {\partial \hat{y}(\mathrm {x})} {\partial \theta} y \\
& = [\sigma(\hat{y}(\mathrm{x})y) - 1]y \frac {\partial \hat{y}(\mathrm {x})} {\partial \theta}
\end{align}
$$

最后都可以化归到$\frac {\partial \hat{y}(\mathrm {x})} {\partial \theta}$的计算。

从算法$4.1$可见:对于一个给定的样本$(\mathrm{x}, y)$，计算复杂度为$\mathcal{O}(kN_z(\mathrm{x}))$，从而，对于一轮迭代，计算复杂度就是$\mathcal{O}(kN_z(\mathrm{X}))$。

接下来讨论算法$4.1$中的几个超参数： 

1. 学习率$\eta$: SGD的收敛依赖于$\eta$的取值，$\eta$太大，则可能不收敛，$\eta$太小，则收敛会很慢，因此这个参数要取得合适。 
2. 这则化系数$\lambda$: FM模型的泛化能力（相应的预测质量）很大程度上依赖于这些正则化系数的选取。他们通常在单独的holdout set上通过grid search方法获取，由于他们个数多取值空间大，因此，通过grid search方法选取会非常费时。一种提高效率的做法是减少他们的个数，例如可以考虑让矩阵$V$汇总行号经$pi$映射后为同一个值的哪些行使用同一个正则化系数。此时正则化系数集$\lambda$变为
$$
\lambda ^0, \lambda ^w _{pi(i)}, \lambda ^v _{pi(i)}, \quad i \in \{1,2,\cdots,n\}
$$
文$[5]$提出了一种自适应选取这些正则化系数的方法。 
3. 正态分布方差参数$\sigma$： 矩阵$V$的初始化采用复合正太分布$\mathcal{N}(0, \sigma)$的随机初始化，方差参数$\sigma$通常取的很小。

### 交替最小二乘法（ALS）

对于带L2正则的最小二乘回归，除了上述的SGD法外，还可以采用交替最小二乘（Alternating Least-Squares）法（也称为坐标下降（Coordinate Descent）)来学习。其基本思路是：每次对一个参数进行优化，并逐轮进行迭代。 

学习的过程中，需要解决的只含单个参数的最优化问题是
$$\theta ^* = arg _{\theta} \min \left \{ \sum _{(x,y) \in D} [\hat{y}(\mathrm{x}) - y]^2 
+ \sum _{\theta \in \Theta} \lambda_{\theta} \theta ^2 \right \}$$

对于这个最优化问题，我们可以尝试计算其解析解，记 
$$T = \sum _{(x,y) \in D} [\hat {y} (\mathrm{x}) - y]^2 + \sum _{\theta \in \Theta} \lambda _{\theta} \theta ^2$$ 则最小值点$\theta ^*$ 应该满足 
$$
\frac {\partial T} {\partial \theta} \mid _{\theta ^*} = 0
$$

利用$(4.17)$，有 
$$
T = \sum _{(x,y) \in D} [g_{\theta} (x) + \theta h_{\theta} (\mathrm{x}) - y]^2 + \sum _{\theta \in \Theta} \lambda _{\theta} \theta ^2
$$

于是 
$$
\frac {\partial T } {\partial \theta} = \sum _{(x,y) \in D} 2[g_{\theta} (\mathrm{x}) + \theta h_{\theta}(\mathrm{x}) - y]h_{\theta}(\mathrm{x}) + 2 \lambda _{\theta} \theta
$$
令$\frac {\partial T } {\partial \theta} = 0$可解得
$$
\theta ^* = \frac {\sum _{(x,y) \in D} [y - g_{\theta}(\mathrm{x})] h_{\theta}(\mathrm{x})} {\sum _{(x,y) \in D} h_{\theta} ^2 (\mathrm{x}) + \lambda_{\theta}}
= \frac {\sum _{i=1} ^N [y ^{(i)} - g_{\theta}(\mathrm{x}^{(i)})] h_{\theta}(\mathrm{x} ^{(i)})} {\sum _{i = 1}^N h_{\theta} ^2 (\mathrm{x} ^{(i)}) + \lambda_{\theta}}
\tag{4.23}
$$

注意，$(4.23)$中还包含了$g_{\theta}(\mathrm{x})$，直接计算比较麻烦，因此，我们希望将其划归到$h_{\theta}(\mathrm{x})$为此，引入误差
$$
e_i = \hat{y} (\mathrm{x} ^{(i)}) - y^{(i)} 
$$
则有
$$
y^{(i)} - g_{\theta}(\mathrm{x}^{(i)}) = [\hat{y}(\mathrm{x}^{(i)}) - g_{\theta}(\mathrm{x}^{(i)})] - [\hat{y}(\mathrm{x}^{(i)}) - y^{(i)}] = \theta h_{\theta}(\mathrm{x} ^{(i)}) - e_i \tag{4.24}
$$

将$(4.24)$代入$(4.23)$有
$$
\theta ^* = \frac {\sum _{i=1} ^N [\theta h_{\theta}(\mathrm{x} ^{(i)} - e_i)]h_{\theta}(\mathrm{x}^{(i)})} {\sum _{i=1} ^N h_{\theta} ^2 (\mathrm{x} ^{(i)}) + \lambda _{\theta}}
 = \frac {\theta \sum_{i=1} ^N h_{\theta} ^2(\mathrm{x}^{(i)}) \sum _{i=1} ^N e_i h_{\theta}(\mathrm{x}^{(i)})} {\sum _{i=1} ^N h_{\theta} ^2 (\mathrm{x} ^{(i)}) + \lambda _{\theta}} 
\tag{4.25}
$$

这样每个$\theta ^*$的计算都归结于$$\sum _{i=1} ^N h_{\theta} ^2, \sum _{i=1}^N [\hat{y}(\mathrm{x}^{(i)}) - y^{(i)}]h_{\theta}(\mathrm{x}^{(i)})$$的计算

公式$(4.25)$是一个统一的计算公式，接下来，我们结合$h_{\theta}$的表达式$(4.21)$，考虑当$\theta$分别取为$w_0$,$W$和$V$时的具体公式。

1. 当$\theta = w_0$时，$h_{\theta}(\mathrm{x}) = 1$因此
$$w_0 ^* = \frac {w_0 \sum _{i=1}^N 1^2 - \sum_{i=1} ^N \cdot 1} {\sum _{i=1}^N 1^2 + \lambda_{w_0}} = \frac {Nw_0 - \sum _{i=1}^N e_i} {N + \lambda_{w_0}} \tag{4.26}$$

2. 当$\theta = w_l(l=1,2,\cdots,n)$时，$h_{\theta}(\mathrm{x}) = x_l$，因此 
$$w_l ^* = \frac {w_l \sum _{i=1}^N (x_l^{(i)})^2 - \sum _{i=1}^N e_i x_l ^{(i)}} {\sum _{i=1} ^N (x_l ^{(i)})^2 + \lambda _{w_l}} \tag{4.27}$$

3. 当$\theta = v_{lm}(l=1,2,\cdots,n, m = 1,2,\cdots,k)$时，$h_{\theta} (\mathrm{x})= x_l \sum _{s=1; s \neq l} ^n v_{sm}x_s$，因此
$$
\begin{align}
v_{lm} ^* & = \frac {v_{lm} \sum _{i=1} ^N(x_l^{(i)} \sum _{s=1;s \neq l} ^n v_{sm} x_s^{(i)})^2 - \sum _{i=1}^N e_i (x_l{(i)} \sum _{s=1;s \neq l} ^n v_{sm} x_s^{(i)}) } 
              {\sum _{i=1}^N (x_l^{(i)} \sum _{s=1;s \neq l} ^n v_{sm} x_s^{(i)}) ^2 + \lambda_{v_{lm}}} \\
          & = \frac {v_{lm} \sum _{i=1} ^N [x_l^{(i)}(q_m^{(i)} - v_{lm}x_l{(i)})]^2 - \sum _{i=1}^N e_i [x_l^{(i)}(q_m^{(i)}-v_{lm}x_l^{(i)})]} 
              {\sum _{i=1} ^N [x_l^{(i)}(q_m^{(i)} - v_{lm}x_l^{(i)})]^2 + \lambda _{v_{lm}}}
\end{align}
\tag{4.28}
$$
其中$$q_m^{(i)} = \sum _{s=1}^n v_{sm} x_s^{(i)} \tag{4.29}$$

有了上面的计算公式， 下面给出利用ALS训练FM模型的算法流程。

![](https://wx1.sinaimg.cn/mw690/7c35df9bly1ftd68eapncj20gq0nu42n.jpg)

由于向量$\mathrm{e} = (e_1, e_2,..., e_N)^{\top}$的计算依赖于整个参数集$\Theta$，因此每当$\Theta$中有任何元素发生变化时，$\mathrm{e}$就需要重新进行计算刷新，这就是为什么算法$4.2$
中有三处Update e的原因。 

类似地，由于$q_m^{(i)}(i=1,2,\cdots,N)$计算依赖于矩阵$V$的第$m$列，因此当其中任意元素发生变化时，$q_m^{(i)}(i=1,2,\cdots,N)$的值也要及时刷新。 

与SGD一样， ALS的计算复杂度也是$\mathcal{O}(kN_z(X))$，但它不需要用到学习率参数$\eta$。

算法$4.2$中更新参数$V$的时候逐列更新，我为什么要这样做呢？为了更好的理解这个问题。我们不妨假设算法以逐行更新的方式来更新参数$V$,此时相应的更新流程就成了

![](https://wx3.sinaimg.cn/mw690/7c35df9bly1ftdu46htmej20rc0hkn0q.jpg)

它需要保存所有的辅助参数$q_m^{(i)}, i=1,2,\cdots,N; m=1,2,\cdots,k$,而算法$4.2$则只需保存 $q_m^{(i)}, i=1,2,\cdots,N$,存储开销就由原来的$\mathcal{O}(kN)$变成$\mathcal{O}(N)$了。

## 参考引用

- [1] [Factorization Machines](http://www.algo.uni-konstanz.de/members/rendle/pdf/Rendle2010FM.pdf)
- [2] [一数一世界——“因子分解机FM-高效的组合高阶特征模型”](http://bourneli.github.io/ml/fm/2017/07/02/fm-remove-combine-features-by-yourself.html)
- [3] [分解机（Factorization Machines）推荐算法原理](https://www.cnblogs.com/pinard/p/6370127.html)
- [4] [Factorization Machines with libFM](https://www.csie.ntu.edu.tw/~b97053/paper/Factorization%20Machines%20with%20libFM.pdf)
- [5] [Learning recommender systems with adaptive regularization](https://dl.acm.org/citation.cfm?id=2124313)

