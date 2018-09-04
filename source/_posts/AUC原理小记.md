---
title: AUC原理小记
date: 2018-07-20 14:14:19
tags: [Machine learning]
toc: true
categories: [Machine learning]
mathjax: true
description: AUC （Area under curve）是一个模型评价指标，只能用于二分类模型的评价。从字面理解，就是一条曲线下区域的面积，所有我们要先来弄清楚这条曲线是什么。这个曲线有个名字叫ROC曲线。ROC曲线是统计里面的概率，最早由电子工程师在二战中提出来。这篇文章主要描述AUC原理。
---

AUC 是Area under curve的首字母缩写。Area under curve是什么呢，从字面理解，就是一条曲线下区域的面积，所有我们要先来弄清楚这条曲线是什么。这个曲线有个名字叫ROC曲线。ROC曲线是统计里面的概率，最早由电子工程师在二战中提出来。这篇文章主要描述AUC原理。

## ROC曲线

ROC曲线是基于样本的真是类别和预测概率算出来的，具体来说，ROC曲线的x轴是伪阳性率(false positive rate)， y轴是真阳性率(true positive rate)。因此首先来明确真、伪阳性的概念。 

对于二分类问题，一个样本的类别只有两种，我们用0，1分别表示两种类别，0和1也可以分别叫做阴性和阳性。当我们使用一个分类器进行概率的预测的时候，对于真实为0的样本，我们可能预测其为0或1，同样对于真实为1的样本我们也可能预测为0或1，这样就存在4种可能性。

![](https://pic1.zhimg.com/917393d21ebd1edbdbee3d5c6fe6e434_r.jpg)

于是有： 
$$
真阳性率（True Positive Rate, TPR） = \frac {真阳性的数量} {真阳性的数量 + 伪阴性的数量}
$$

$$
伪阳性率（True Positive Rate, TPR） = \frac {伪阳性的数量} {伪阳性的数量 + 真阴性的数量}
$$

*例$1.1$：* 比如有5个样本：
真实的类别标签分别是 $y = c(1, 1, 0, 0, 1)$
一个分类器预测样本为1的概率是$p = c(0.5, 0.6, 0.55, 0.4, 0.7)$

我们需要选定阈值才能把概率转化为类别，选定不同的阈值会得到不同的结果。

- 若我们选定的阈值为0.1,那5个样本被分进1的类别；
- 如果选定阈值为0.3， 结果仍然一样； 
- 如果选取了0.45作为阈值，那么只有样本4被分进0，其余都进入1类
......

一旦得到了类别，我们就可以计算相应的真、伪阳性的概率，当我们把所有计算得到的不同真、伪阳性率连起来，就画出了ROC曲线，根据例$1.1$来绘制ROC曲线和计算AUC的Python代码： 

{% codeblock [python] [compute_auc.py] %}
from sklearn.metrics import roc_curve, auc  ###计算roc和auc
import matplotlib.pyplot as plt

y_score = [0.5, 0.6, 0.55, 0.4, 0.7]
y = [1 , 1, 0, 0, 1]

# 计算真阳性率和假阳性率
fpr,tpr,threshold = roc_curve(y, y_score) 
print("force positive rate: ", fpr)
print("true positive rate: ", tpr)
print("threshold: ", threshold)

# 计算auc的值
roc_auc = auc(fpr,tpr) 

lw = 2
plt.figure(figsize=(10,10))

#伪阳性率为横坐标，真阳性率为纵坐标做曲线
plt.plot(fpr, tpr, color='darkorange',
         lw=lw, label='ROC curve (area = %0.2f)' % roc_auc) 
plt.plot([0, 1], [0, 1], color='navy', lw=lw, linestyle='--')
plt.xlim([0.0, 1.0])
plt.ylim([0.0, 1.05])
plt.xlabel('False Positive Rate')
plt.ylabel('True Positive Rate')
plt.title('Receiver operating characteristic example')
plt.legend(loc="lower right")
plt.show()
{% endcodeblock %}

代码输出如下：

![](https://wx1.sinaimg.cn/mw690/7c35df9bly1ftghjlxkpzj20iq02st93.jpg)

![](https://wx1.sinaimg.cn/mw690/7c35df9bly1ftghjolihzj21bi120gq0.jpg)

## AUC 的含义

通过AUC的定义我们知道了AUC是什么，怎么算，但是它的含义是什么呢？如果从定义来理解AUC的含义，比较困难，实际上AUC和Mann-Whitney U test有密切的联系。从Mann–Whitney U statistic的角度来解释，AUC就是从所有真实标签为1的样本中随机选取一个样本，从所有真是标签为0的样本中随机选取一个样本，然后根据你的分类器对两个随机样本进行预测，把1样本预测为1的概率$p_1$，把0样本预测为1的概率为$p_0$，$p_1 > p_0$的概率就是AUC。所以AUC反应的是分类器对样本的排序能力。根据这个解释，如果我们完全随机的对样本分类，那么AUC应该接近0.5。

另外值得注意的是，AUC对样本类别是否均衡并不敏感,这也是不均衡样本通常用AUC评价分类器性能的一个原因。

假设分类器的输出是样本属于正类的score（置信度），则AUC的物理意义为，任取一对（正、负）样本，正样本的score大禹负样本score的概率。从而我们能够理解对于AUC而言，并不关心具体预测的结果是标签或者概率，也不需要卡什么阈值，只要在预测结果之间有排序即可。

## Mann-Whiteney U test and statistic

### 关系 

从定义上看，AUC衡量的额是ROC曲线下与横轴为成的面积值。但从统计角度来理解AUC的意义，还要结合Mann-Whitney U 统计量。

首先，AUC与Mann-Whitney U统计量基本上是等价的：
$$
AUC = \frac {U} {n_1, n_0} 
$$

其中$n_1$ 和$n_0$分别代表样本中，样本1的总个数和样本0的总个数。则上例中$n_0 = 3$， $n_1 =2$。


## 参考引用

- [1] [【杂纪】从ROC曲线到AUC值，再到Mann–Whitney U统计量](https://blog.csdn.net/Joyliness/article/details/79156879)
- [2] [用Python画ROC曲线](https://blog.csdn.net/lz_peter/article/details/78054914)
- [3] [Classification: ROC and AUC](https://developers.google.com/machine-learning/crash-course/classification/roc-and-auc)
- [4] [AUC Optimization vs. Error Rate Minimization](https://papers.nips.cc/paper/2518-auc-optimization-vs-error-rate-minimization.pdf)
- [5] [机器学习和统计里面的auc怎么理解？](https://www.zhihu.com/question/39840928)
- [6] [Optimizing Classifier Performance via an Approximation to the Wilcoxon-Mann-Whitney Statistic](https://pdfs.semanticscholar.org/df27/dde10589455d290eeee6d0ae6ceeb83d0c6b.pdf)
