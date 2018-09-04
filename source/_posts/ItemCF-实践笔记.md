---
title: ItemCF 实践笔记
date: 2018-09-04 16:29:26
tags: [Item CF, 推荐系统]
toc: true
categories: [推荐系统]
mathjax: true
description: 基于物品的协同过滤(item-based collaboritive filtering)算法是目前业界应用最多的算法。无论是亚马逊，还是Netflix、Hulu、YouTube，其推荐算法的基础都是该算法。本文结合ItemCF算法原理， 给出了Spark版本的算发实现，并基于MovieLens数据集对算法进行评测。 
---

## 基于物品的协同过滤

基于物品的协同过滤(item-based collaboritive filtering)算法是目前业界应用最多的算法。无论是亚马逊，还是Netflix、Hulu、YouTube，其推荐算法的基础都是该算法。本文结合ItemCF算法原理， 给出了Spark版本的算发实现，并基于MovieLens数据集对算法进行评测。 完整代码实现请参见[github](https://github.com/jluchuang)

算法计算过程主要包含以下两步： 

1. 计算物品之间的相似度； 
2. 根据物品之间的相似度和用户历史行为给用户生成推荐列表。 

## 物品相似度计算

### 对数似然比相似度

与《推荐系统实践》里面给出的物品基本相似度计算公式不同， 这里使用对数似然相似度来计算两个物品之间的相似度。 详细了解对数似然相似度请参见：[对数似然比相似度](https://blog.csdn.net/u014374284/article/details/49823557)和[推荐系统算法——最大似然比](http://uohzoaix.github.io/studies/2013/03/25/logLikelihoodRatio/)，这里直接给出计算公式如下
$$
ItemSimilarty = 2 * (matrixEntropy - rowEntropy - columnEntropy)
$$

代码如下： 

```scala
object LogLikelihood {
    def xLogX(x: Long): Double = if (x == 0) 0.0 else x * Math.log(x)

    def entropy(elements: Long*): Double = {
        var sum: Long = 0
        var result: Double = 0.0
        for (element <- elements) {
            result += xLogX(element)
            sum += element
        }
        xLogX(sum) - result
    }

    def logLike(item1Count: Long, item2Count: Long, common: Long, all: Long): Double = {
        val k11 = common // 同时喜欢item1和item2的人数
        val k12 = item1Count - common // 喜欢item2不喜欢item1的人数
        val k21 = item2Count - common // 喜欢item1不喜欢item2的人数
        val k22 = all - item1Count - item2Count + common // 不喜欢item1也不喜欢item2的人数
        val rowEntropy = entropy(k11 + k12, k21 + k22)
        val columnEntropy = entropy(k11 + k21, k12 + k22)
        val matrixEntropy = entropy(k11, k12, k21, k22)
        val sim = Math.max(0.0, 2 * (rowEntropy + columnEntropy - matrixEntropy))

        sim
    }
}
```

### 相似度矩阵计算

ItemCF算法计算物品之间相似度与的过程： 

- 算法输入： *用户->物品* 打分矩阵$R_{ui}$
- 算法输出： *物品->物品* 相似度矩阵$Sim_{ij}$

- 1 基于$R_{ui}$建立 *用户->物品* 倒排表$U$
- 2 基于倒排表$U$计算物品共现矩阵$common_{ij}$，物品$i$和$j$同时被一个用户打过一次分，或有有效行为， 则$common_{ij} += 1$ 
- 3 基于共现矩阵$common_{ij}$按照上一小节的对数似然比公式计算相似度$Sim_{ij}$

具体的Spark实现代码片段如下： 

```scala
    /**
      * Item CF ：基于item协同
      *
      * @param sc
      * @param data        : User -> item, rating
      * @param minFreq     : 共现次数阈值
      * @param topSimItems : 相似商品截断阈值
      * @return
      */
    def calculateItemSim(sc: SparkContext,
                         data: RDD[(String, (Int, Double))],
                         minFreq: Int = 1,
                         topSimItems: Int = 200): RDD[(Int, Array[(Int, Double)])] = {

        val userRatings = data.groupByKey(8000)
        val itemCounts = data.map(l => (l._2._1.toInt, 1)).reduceByKey(_ + _)
        val itemCountsMap = sc.broadcast(itemCounts.collectAsMap())

        val all_event_num = sc.broadcast(userRatings.count())

        val itemSim: RDD[(Int, Array[(Int, Double)])] = userRatings
                .flatMap { l =>
                    val items = l._2.toList
                    val pairs = ListBuffer[((Int, Int), Long)]()
                    for (i <- items.indices; j <- i + 1 until items.length) {
                        var (itemi, itemj) = (items(i), items(j))
                        if (itemi._1 > itemj._1) {
                            itemi = items(j)
                            itemj = items(i)
                        }
                        pairs.append(((itemi._1, itemj._1), 1))
                    }
                    pairs
                }
                .reduceByKey(_ + _)
                // 同时听过 itemi 和 itemj 的人数（c[i][j]）共现矩阵
                .flatMap {
            case ((item1, item2), freq) =>
                val pairs = ListBuffer[(Int, (Int, Double))]()
                if (freq >= minFreq) {
                    // loglikelihood ratio
                    val a: Long = itemCountsMap.value(item1) // 喜欢item1的人数
                    val b: Long = itemCountsMap.value(item2) // 喜欢item2的人数

                    val sim = logLike(a, b, freq, all_event_num.value)
                    pairs.append((item1, (item2, sim)))
                    pairs.append((item2, (item1, sim)))
                }
                pairs
        }
                .topByKey(topSimItems)(Ordering[Double].on(_._2)) // 这里是不是可以卡一个相似度的阈值?
                .mapValues(
            arr => {
                // 归一化
                val maxSim = arr.map(_._2).max
                arr.map(x => (x._1, x._2 / maxSim))
            })

        itemSim
    }
```

## 计算推荐列表

得到物品之间的相似度之后， ItemCF通过如下公式计算用户$u$对于一个物品$j$的兴趣：
$$
p_{uj} = \sum_{i \in N(u) \bigcap S(j,K)} sim_{ji}r_{ui} \tag{1}
$$

这里$N(u)$是用户喜欢的物品的集合，$S(j.K)$是和物品j最相似的$K$个物品的集合， $sim_{ji}$是物品$j$和$i$的相似度，$r_{ui}$是用户$u$对$i$的兴趣。 

公式1没有考虑到用户打分的偏置和商品被打分的偏置， 如果某个用户喜欢对物品做出一个高于平均水平的反馈， 那么我们的预测值也应该是高于平均水平的，[这里](http://courses.ischool.berkeley.edu/i290-dm/s11/SECURE/a1-koren.pdf)给出了一种考虑到用户偏置和商品偏置的预测打分$p_{uj}$计算方式

$$
p_{uj} = b_{uj} + \frac {\sum _{j \in N^k(u)} sim_{ji}(r_{ui} - b_{ui})} {\sum_{j \in N^k(u)} sim_{ji}} \tag{2}
$$

其中

$$
b_{ui} = \mu + b_u + b_i
$$

$\mu$是全体用户打分的平均数， $b_u$和$b_i$可以通过极小化一下公式得到： 
$$
arg \min_{b_*} \sum (r_{ui} - \mu - b_u - b_i)^2 + \lambda(\sum _u b_u^2 + \sum _i b_i^2) 
$$

对于以上公式中使用到的${b_i}和${b_u}$可以通过SGD或者ALS的方式进行计算， 具体公式请参看参考引用中的论文， 这里给出ALS代码实现： 

```scala
    /**
      * Calculate Bi and Bu for item and user
      * reference: http://courses.ischool.berkeley.edu/i290-dm/s11/SECURE/a1-koren.pdf
      *
      * @param sc
      * @param userRating
      * @param lambda2
      * @param lambda3
      * @param epochs
      * @return
      */
    def calculateBiBu(sc: SparkContext,
                      userRating: RDD[(String, (Int, Double))],
                      lambda2: Int = 25,
                      lambda3: Int = 10,
                      epochs: Int = 20): (Map[Int, Double], Map[String, Double]) = {
        val globalMean = userRating.map {
            case (userId, (itemId, rate)) => rate
        }.mean

        val u = sc.broadcast(globalMean)

        var bi = mutable.Map[Int, Double]()
        var bu = mutable.Map[String, Double]()

        for (dummy <- 0 until epochs) {
            bi ++= userRating
                    .map {
                        case (userId, (itemId, rate)) => (itemId, (userId, rate))
                    }
                    .groupByKey()
                    .map {
                        case (itemId, ratings) => {
                            val sumI = ratings.map {
                                case (userId, rate) =>
                                    rate - u.value - bu.getOrElse(userId, 0D)
                            }.sum
                            val biValue = sumI / (lambda2 + ratings.size)
                            (itemId, biValue)
                        }
                    }.collectAsMap()

            bu ++= userRating.groupByKey()
                    .map {
                        case (userId, ratings) => {

                            val sumU = ratings.map {
                                case (itemId, rate) =>
                                    rate - u.value - bi.getOrElse(itemId, 0D)
                            }.sum

                            val buValue = sumU / (lambda3 + ratings.size)
                            (userId, buValue)
                        }
                    }.collectAsMap()
        }

        (bi.toMap, bu.toMap)
    }
```

进一步基于公式2，我们给出ItemCF算法的Predict代码： 

```scala
    /**
      * 根据用户打分和item协同矩阵进行item推荐的分值预估
      *
      * @param sc
      * @param ratingData     : userId -> itemId, rating
      * @param itemSimRDD     : 协同item相似度矩阵
      * @param ratingThresold : 推荐召回截断阈值
      * @return
      */
    def predict(sc: SparkContext,
                ratingData: RDD[(String, (Int, Double))],
                itemSimRDD: RDD[(Int, Array[(Int, Double)])],
                ratingThreshold: Int): RDD[(String, Array[(Int, Double, String)])] = {

        val globalMean = ratingData.map(l => l._2._2).mean()

        val (bi, bu) = calculateBiBu(sc, ratingData)
        val biMap = sc.broadcast(bi)
        val buMap = sc.broadcast(bu)

        ratingData
                .map {
                    case (userId, (itemId, rating)) => (itemId, (userId, rating))
                }
                .join(itemSimRDD)
                .flatMap {
                    case (itemId, ((userId, rating), simItems)) => {
                        simItems.map {

                            case (simId, simValue) => {
                                val buj = globalMean + biMap.value.getOrElse(itemId, 0D) + buMap.value.getOrElse(userId, 0D)
                                ((userId, simId), (simValue * (rating - buj), simValue, itemId.toString))
                            }
                        }
                    }
                }
                .reduceByKey((a, b) => (a._1 + b._1, a._2 + b._2, s"${a._3},${b._3}"))
                .map {
                    case ((userId, itemId), (sumRate, simSum, reason)) => {
                        val bui = globalMean + biMap.value.getOrElse(itemId, 0D) + buMap.value.getOrElse(userId, 0D)
                        val s = if (simSum == 0) 0 else sumRate / simSum
                        val preRate = math.max(math.min(s + bui, 5), 0)
                        (userId, (itemId, preRate, reason))
                    }
                }
                .topByKey(ratingThreshold)(Ordering[Double].on(_._2))

    }
```

## 指标评价

With datasets on [MovieLens](https://grouplens.org/datasets/movielens/) 100K and 1M.

**MovieLens 100K**

- RMSE = 0.9049792521615789
- MAE = 0.7078706510784817

**MovieLens 1M**

- RMSE = 0.8624739281707584
- MAE = 0.6485303967964491

## 参考引用

- [1] [KNNBaseline](https://surprise.readthedocs.io/en/stable/knn_inspired.html#surprise.prediction_algorithms.knns.KNNBaseline)
- [2] [推荐系统实践](https://book.douban.com/subject/10769749/)
- [3] [Factor in the Neighbors: Scalable and Accurate Collaborative Filtering](http://courses.ischool.berkeley.edu/i290-dm/s11/SECURE/a1-koren.pdf)
- [4] [推荐系统算法——最大似然比](http://uohzoaix.github.io/studies/2013/03/25/logLikelihoodRatio/)
- [5] [对数似然比相似度](https://blog.csdn.net/u014374284/article/details/49823557)
