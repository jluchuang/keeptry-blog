---
title: 矩阵分解在推荐系统中的应用(3)_SVD在推荐系统中的应用
date: 2018-07-14 19:59:41
tags: [Machine learning]
toc: true
categories: [Machine learning]
mathjax: true
description: 矩阵分解在推荐系统中的应用翻译系列第三篇， SVD在推荐系统中的应用。 
---

**前言** 这是“矩阵分解在推荐系统中的应用”系列的第3篇翻译文章，英文原文请参考[nicolas-hug.com](http://nicolas-hug.com/blog/matrix_facto_2)。 

## SVD在推荐系统中的应用

现在我们对SVD以及怎么按照SVD计算用户打分已经有一个较好的认知，我们进一步来看核心问题： 使用SVD来解决推荐问题。或者说，使用SVD来预测输入的系数打分矩阵的位置分值。回到最初的稀疏矩阵$R$ 
![](https://wx2.sinaimg.cn/mw690/7c35df9bly1ft9ot143z7j20gy09uaai.jpg)

我们的最终目标是预测未知的 $\color {red} {?}$

### Ooooops

现在的问题是： 矩阵$R$的SVD是没有被定义的。它并不存在，也就没有办法被计算。 

如果$R$是稠密矩阵，我们可以很简单的计算$M$和$U$:矩阵$M$的每一列都是$RR^{\top}$的特征向量，矩阵$U$的每一列都是$R^{\top}R$的特征向量，对应的特征值构成了对角矩阵$\Sigma$。很多高效的算法可以用来进行稠密矩阵SVD分解。

但是，当$R$是稀疏矩阵的时候，$RR^{\top}$和$R^{\top}R$是不存在的，因此我们无法得到他们的特征向量，也就无法对$R$进行矩阵分解成$M \Sigma U ^{\top}$。这种情况下，我们可以选择用启发式算法来近似的估计输入矩阵$R$中的缺失分值，我们将传统的SVD分解计算过程，转化成一个最小化问题。

### 一种可选的计算方案

如果我们可以计算出所有的特征向量$p_u$和$q_i$($p_u$构成了$M$的所有行，$q_i$构成了矩阵$U^{\top}$的所有列)：

- $r_{ui} = p_u \cdot q_i$对于所有的 $u$ 和$i$
- 所有的特征向量$p_u$之间都是正交的，$q_i$之间也是正交的

计算所有用户的$p_u$和item的$q_i$的过程可以被转化为极小化下面这个方程的问题（在满足正交。约束的前提下）：
$$\min _{p_u, q_i; p_u \bot p_v; q_i \bot q_j} \sum _{r_{ui} \in R} (r_{ui} - p_u \cdot q_i) ^2$$

直观的看上面的方程是计算$p_u$和$q_i$使得整体的$r_{ui} = p_u \cdot q_i$的平方误差最小，也就是说我们在近似的估计$p_u$和$q_i$

现在的问题的是，输入矩阵$R$是有缺失的，注意到，到目前为止我们并没有把缺失的$r_{ui}$设置成0， 我们是单纯的忽略这些缺失值。同时我们也不会考虑正交约束，即使正交约束能够更有好的对问题进行解释，但它对我们更准确的预测$r_{ui}$目标是没有帮助的。 这样我们最终的优化问题被转化为：$$\min _{p_u, q_i} \sum _{r_{ui} \in R} (r_{ui} - p_u \cdot q_i) ^2$$

### 算法 

感谢Simon Funk提出的算法。从上面的方程式可以看出，我们要优化的目标函数，并不是凸函数。也就是说要找到最有的$p_u$和$q_i$是很难的（同时计算的次优结果也并不唯一）。有很多中方式可以计算出次优的临近解，这里我们详细讨论SGD(Stochastic Gradiant Desent)。 

梯度下降算法是一个经典的算法，用来获取方程的最小值（有时是局部最小）。如果你有了解过神经网络中使用反向传播算法（back-propagation）来计算梯度,我们接下来的算法中也将使用这种方式进行梯度计算。 这里我们并不详细的讨论SGD， 整个SGD的整体过程如下： 

对于具有以下形式的最优化问题$$f(\Theta) = \sum _k f_k(\Theta)$$其中$f$是优化目标函数，$\Theta$是参数。我们的最终目标是找到能使得$f(\Theta)$值最小的参数$\Theta$，过程如下： 

- 1 随机初始化$\Theta$
- 2 在有限次迭代次数内， 重复以下计算过程：
  - 对于所有的$k$重复以下过程：
    - 计算$\frac {\partial f_k} {\partial \Theta}$
    - 更新$\Theta$: $\Theta \leftarrow \Theta - \alpha \frac {\partial f_k} {\partial \Theta}$,其中$\alpha$是学习率(learning rate)。

映射到我们的场景里， 我们要获取的目标参数$\Theta$对应于所有的特征向量$p_u$和$q_i$（下面的公式中用$p_*$,$q_*$表示），我们要最小化的目标函数如下：
$$
f(p_*, q_*) = \sum_{r_{ui} \in R} (r_{ui} - p_u \cdot q_i) ^2 = \sum _{r_{ui} \in R} f_{ui}(p_u, q_i)
$$
其中$f_{ui}$被定义为$f_{ui}(p_u,q_i) = (r_{ui} - p_u \cdot q_i) ^2$

到目前为止， 我们要使用SGD进行极小化计算， 我们需要计算函数$f_{ui}$对于每个$p_u$和$q_i$的偏导数。 

- $f_{ui}$对于向量$p_u$的偏导数：
$$
\frac {\partial f_{ui}} {\partial p_u} = 
\frac {\partial} {\partial p_u} (r_{ui} - p_u \cdot q_i) ^2 = 
-2 q_i(r_{ui} - p_u \cdot q_i) 
$$

- $f_{ui}$对于向量$q_i$的偏导数：
$$
\frac {\partial f_{ui}} {\partial q_i} = 
\frac {\partial} {\partial q_i} (r_{ui} - p_u \cdot q_i) ^2 = 
-2 p_u(r_{ui} - p_u \cdot q_i) 
$$

### SGD算法计算过程

- 1. 随机初始化向量$p_u$和$q_i$
- 2. 再有限次迭代次数内，重复以下计算过程： 
  - 针对所有已知的$r_{ui}$重复以下过程： 
    - 计算$\frac {\partial f_{ui}} {\partial q_i}$ 和 $\frac {\partial f_{ui}} {\partial p_u}$
    - 更新$p_u$和$q_i$:
      - $p_u \leftarrow p_u + \alpha \cdot q_i(r_{ui} - p_u \cdot q_i)$
      - $q_i \leftarrow q_i + \alpha \cdot p_u(r_{ui} - p_u \cdot q_i)$

*注意上面的计算过程中我们将导数公式中的系数2，归一到学习率$\alpha$中*

在上面的算法中， 所有的特征向量$p_u$和$q_i$都同时更新。Funk给出的最初算法和上面的算法略有不同：原始的算法中，每次只训练一个向量，这也使得原始的算法更有SVDesque风格。 

当所有的特征向量$p_u$和$q_i$被计算出来后，我们可以按照下面公式估计所有的分值：
$$\hat{r}_{ui} = p_u \cdot q_i$$
$\hat{r}_{ui}$代表的是一个估计值，并非真实用户对对应item的打分。

### 降维

在我们对算法进行Python实现之前。 我们需要对算法的超参进行处理：具体的，算法重点的特征向量$p_u$$q_i$的长度应该是多少？ 有一点是显而易见的，$p_u$和$q_i$的长度是相同的， 不然我们没有办法根据特征向量计算内积。 

回顾系列文章中的第一篇:[PCA](http://nicolas-hug.com/blog/matrix_facto_1)， 我们不用对用户或者电影的所有特征维度都进行很好的近似估计，我们可以只是用很小的特征维度对用户和电影进行估计。 

读者很可能对这里特征维度的选取有质疑， SVD和PCA最神奇的地方就在于我们可以只用很小的$k$维特征向量，来对原始矩阵中的未知值进行低阶近似。具体的细节已经超出了这篇文章要讨论的范围，这里推荐阅读[ this Stanford course notes ](http://theory.stanford.edu/~tim/s15/l/l9.pdf) 中的第5节，文章的作者提出了一种恢复SVD缺失值的方法。

