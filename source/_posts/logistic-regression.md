---
title: 逻辑回归小记
date: 2018-06-19 19:54:57
tags: [逻辑回归, Machine learning]
toc: true
categories: [Machine learning]
mathjax: true
description: 逻辑斯蒂回归(logistic regression)是统计学习中的经典分类方法。
---

### 预备知识

#### 分布函数

设$X$是一个随机变量， $x$是任意实数，函数$F(x) = p(X \leqslant x)$成为X的分布函数，有时也记为$X \sim F(x)$。 

对任意实数$x_1, x_2 (x_1 < x_2)$ 有 $$P(x_1 < X \leqslant x_2) = P(x \leqslant x_2) - P(x \leqslant x_1) = F(x_2) - F(x_1)$$ 因此，若已知$X$的分布函数，就可以知道$X$落在任意空间上的概率，在这个意义上说，分布函数完整地描述了随机变量的统计规律性。

### 逻辑斯蒂分布

**定义1 （逻辑斯蒂分布）** 设$X$是连续随机变量， $X$服从逻辑斯蒂分布是指$X$具有下列分布函数和密度函数：$$F(x) = P(X \leqslant x) = \frac {1} {1 + e ^{-(x - \mu) / \gamma }}$$ $$f(x) = F'(x) = \frac {e ^{-(x-\mu) / \gamma}} {\gamma (1 + e ^{-(x-\mu) / \gamma}) ^2}$$

式中，$\mu$为位置参数， $\gamma > 0$为形状参数。

逻辑斯蒂分布的密度函数$f(x)$和分布函数$F(x)$的图形如图(略)。分布函数属于逻辑斯蒂函数，其图形是一条S型曲线(sigmoid curve)。该曲线以点$\left( \mu, \frac {1} {2} \right)$为中心对称，即满足$$F(-x + \mu) - \frac {1} {2} = -F(x + \mu) + \frac {1} {2}$$ 曲线在中心附近增长较快，在两端增长速度较慢。形状参数$\gamma$的值越小，曲线在中心附近增长的越快。

#### 二项逻辑斯蒂回归模型

