---
title: GBDT学习总结(一)——决策树
date: 2018-06-09 19:28:43
tags: [GBDT]
toc: true
categories: [Machine learning]
mathjax: true
description: GBDT学习总结，第一部分： 决策树， 内容主要引自《统计学习方法》（李航）
---

### 决策树定义

> **定义（决策树）：** 分类决策树模型是一种对实例进行分类的数形结构，决策树由结点(node)和有向边(directed edge)组成。 结点有两种类型：内部结点（internal node）和叶结点(leaf node)。内部结点表示一个特征或属性，叶结点表示一个类。

决策树学习算法包含特征选择、决策树生成与决策树的剪枝过程。 

### 特征选择

#### 信息增益

**定义 (熵，entropy)：** 是表示随机变量不确定性的度量，设$X$是一个取有限个值的离散随机变量，其概率分布为
$$P(X=x_i)=p_i, i=1,2,...,n$$ 则随机变量X的熵定义为 $$H(X)=-\sum_{i=1}^n p_i log p_i$$ 由定义客可知，熵只依赖于$X$的分布，而与$X$的取值无关，所以也可将$X$的熵记做$H(p)$, 即 $$H(p)=-\sum_{i=1}^n p_i log p_i\tag{1}$$
熵越大，随机变量的不确定性就越大。 从定义可以验证 $$0<=H(p)<=n$$

*条件熵：* 设有随机变量$(X,Y)$, 其联合概率分布为$$P(X=x_i, Y=y_j) = p_{ij} ,   i=1,2,...,n; j=1,2,...,m$$条件熵$H(Y|X)$表示在已知随机变量$X$的条件下随机变量$Y$的不确定性。随机变量$X$给定的条件下随机变量$Y$的条件熵(conditional entropy) $H(Y|X)$定义为$X$给定的条件下$Y$的条件概率分布的熵对$X$的数学期望$$H(Y|X)=\sum_{i=1}^n p_i H(Y|X=x_i)\tag{2}$$这里，$p_i=P(X=x_i), i=1,2,...,n$。

当熵和条件熵中的概率由数据估计（特别是极大似然估计）得到时，所对应的熵与条件熵分别称为经验熵(empirical entropy)和经验条件熵(empirical conditional entropy)。此时如果有0概率， 令 $0log0=0$

信息增益（information gain） 表示得知特征$X$的信息而使得类$Y$的信息不确定性减少的程度。
**定义 (信息增益)：** 特征$A$对训练数据集$D$的信息增益$g(D,A)$，定义为集合$D$的经验熵$H(D)$与特征$A$给定条件下$D$的经验条件熵$H(D|A)$之差，即$$g(D,A) = H(D) - H (D|A)$$  

一般地，熵$H(Y)$与条件熵$H(Y|X)$之差称为互信息(mutual information)。决策树学习中的信息增益等价于训练数据集中类与特征的互信息。

根据信息增益准则的特征选择方法是：对训练数据集（或子集）D， 计算其每个特征的信息增益， 并比较他们的大小， 选择信息增益最大的特征。 

设训练数据集为$D$，$|D|$表示其样本容量，即样本个数。设有$K$个类$C_k, k=1,2,...,K$，$|C_k|$为属于$C_k$类的样本个数，$\sum_{k=1}^K |C_k| = |D|$。 设特征$A$有$n$个不同的取值${a_1, a_2,...,a_n}$，根据特征$A$取值将$D$划分成$n$个子集$D_1,D_2,...,D_n$， $|D_i|$为$D_i$的样本个数， $\sum_i=1^n|D_i| = |D|$。记子集$D_i$中属于类$C_k$的样本集合为$D_{ik}$, 记$D_{ik} = D_i \bigcap C_k$, $|D_{ik}|$为$D_{ik}|$的样本个数，于是信息增益的算法如下： 


**算法 1 （信息增益算法）**

- 输入： 训练数据集$D$和特征$A$
- 输出： 特征$A$对训练数据集$D$的信息增益$g(D,A)$
-  1)计算数据集$D$的经验熵$H(D)$ $$H(D)=-\sum_{k=1}^K \frac{|C_k|}{|D|}\log_2\frac{|C_k|}{|D|} $$
-  2)计算特征$A$对数据集$D$的经验条件熵$H(D|A)$ $$H(D|A)=\sum \frac{|D_i|} {|D|} H(D_i) = -\sum_{i=1}^n \frac{|Di|} {|D|} \sum_{k=1}^K \frac{|D_{ik}|} {|Di|} \log_2 \frac{|D_{ik}|} {D_i}$$
-  3)计算信息增益 $$g(D,A) = H(D) - H(D|A)$$

#### 信息增益比

以信息增益作为划分训练数据集的特征，存在偏向于选择取值较多的特征的问题， 使用信息增益比（information gain ratio）可以对这一问题进行校正。这是特征选择的另一准则。 

**定义 （信息增益比）：** 特征$A$对训练数据集$D$的信息增益比$g_R (D,A)$定义为其信息增益$g(D,A)$与训练数据集$D$关于特征$A$的值的熵$H_A(D)$之比，即$$g_R(D,A)=
\frac{g(D,A)} {H_A(D)}\tag{3}$$
其中$H_A(D) = -\sum_{i=1}^n \frac {|D_i|} {|D|}log_2 \frac {|D_i|} {|D|}$， $n$是特征$A$取值的个数。

### 决策树的生成

**算法2 （ID3 算法)**

- 输入： 训练数据集$D$, 特征集$A$， 阈值$\varepsilon$：
- 输出： 决策树$T$
-  1) 若$D$中所有实例属于同一类$C_k$, 则T为单结点树，并将类$C_k$作为该节点的类标记，返回$T$
-  2) 若$A=\emptyset$，则$T$为单结点树，并将$D$中实例数最大的类${C_k}$作为该节点的标记， 返回$T$
-  3) 否则， 按算法1计算$A$中各特征对$D$的信息增益，选择信息增益最大的特征$A_g$
-  4) 如果$A_g$的信息增益小于阈值$\varepsilon$，则置$T$为单结点树，并将$D$中实例数最大的类$C_k$作为该节点的类标记， 返回$T$
-  5) 否则，对$A_g$的每一个可能值$a_i$, 依$A_g=a_i$将$D$分割为若干非空子集$D_i$,将$D_i$中实例数最大的类作为标记作为标记， 构建子结点，由结点及其子结点构成树$T$，返回$T$
-  6) 对第$i$个子节点，以$D_i$为训练集，以$A-{A_g}$为特征集，递归地调用算法1）~5）步，得到子树$T_i$，返回$T_i$

**算法 3 (C4.5的生成算法)**
将算法2中的信息增益替换为信息增益比。（略）

### 决策树的剪枝

决策树生成算法递归地产生决策树， 直到不能继续下去为止， 这样产生的树往往对训练数据分类很准确， 但对未知的册数数据分类却没有那么准确， 即出现过拟合的状况。过拟合的原因在于学习时过多的考虑如何提高对训练数据的正确分类，从而构建出过于复杂的决策树。解决这个问题的办法是考虑决策树的复杂度，对已生成的决策树进行简化.

在决策树学习的过程中将已生成的树进行简化的过程称之为剪枝(pruning)。具体地，剪枝从已生成的树上裁掉一些子树或叶结点，并将其根结点或父结点作为新的叶结点，从而简化分类树模型。 

决策树的剪枝往往通过极小化决策树整体的损失函数(loss function)或代价函数(cost function)来实现。设$T$的叶节点个数为$|T|$，$t$是树$T$的叶结点，该叶结点有$N_t$个样本点， 其中$k$类的样本点有$N_{tk}$个。 $k=1,2,...,K$， $H_t(T)$为叶结点$t$上的经验熵, $\alpha >= 0$为参数， 则决策树学习的损失函数可以定义为$$C_{\alpha}(T) = \sum_{t=1}^{|T|}N_tH_t(T)+\alpha|T| \tag{4}$$
其中经验熵为$$H(T)=-\sum_t \frac {N_{tk}} {N_t} \log \frac {N_{tk}} {Nt}$$

在损失函数中，将式4右端第一项记做$$C(T)=\sum_{t=1}^{|T|}N_tH_t(T)=-\sum_{t=1}^{|T|}\sum_{k=1}^KN_{tk}\log \frac {N_{tk}} {N_t}$$这时有$$C_{\alpha}(T)=C(T) + \alpha (T) \tag{5}$$式5中$C(T)$表示模型对训练数据的预测误差，即模型与训练数据的拟合程度， $|T|$表示模型复杂度，参数$\alpha>=0$控制两者之间的影响。较大的$\alpha$促使选择较简单的模型（树）。$\alpha=0$意味着只考虑模型与训练数据的拟合程度，不考虑模型的复杂度。 

剪枝就是当$\alpha$确定时选择损失函数较小的模型，即损失函数最小的子树。 决策树生成学习局部的模型，而决策树剪枝学习整体的模型。 

式4式5定义的损失函数的极小化等价于正则化的极大似然估计。所以利用损失函数最小的原则进行剪枝就是用正则化极大似然估计进行模型选择。 

**算法4 (树的剪枝算法)**

- 输入： 生成算法产生的整棵树$|T|$， 参数$\alpha$
- 输出： 修剪后的子树$T_{\alpha}$

- 1 计算每个结点的经验熵
- 2 递归的从叶结点向上回缩，设一组节点回缩到其父节点之前与之后的整棵树分别为$T_B$与$T_A$, 其对应的损失函数值分别为$C_{\alpha}(T_B)$与$C_{\alpha}(T_A)$， 如果$$C_{\alpha}(T_A) <= C_{\alpha}(T_B)$$则进行剪枝，即将父节点作为新的叶结点。 
- 3 返回2) 直至不能继续为止，得到算是函数最小的子树$T_{\alpha}$
