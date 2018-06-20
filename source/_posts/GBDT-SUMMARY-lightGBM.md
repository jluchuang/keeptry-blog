---
title: LightGBM 小记
date: 2018-06-19 14:35:43
tags: [GBDT, Machine learning]
toc: true
categories: [Machine learning]
description: LightGBM 是微软开源的基于决策树的高性能分布式的梯度优化机器学习框架。最近项目刚好用到，这里作简要记录。 
---

### LightGBM

**GitHub地址:** [lightGBM](https://github.com/Microsoft/LightGBM)

> A fast, distributed, high performance gradient boosting (GBDT, GBRT, GBM or MART) framework based on decision tree algorithms, used for ranking, classification and many other machine learning tasks. It is under the umbrella of the DMTK(http://github.com/microsoft/dmtk) project of Microsoft.

### [Mac下安装配置](https://lightgbm.readthedocs.io/en/latest/Installation-Guide.html#macos) 

LightGBM 基于OpenMP进行编译， 不支持Apple Clang. 

我的系统版本： Mac OS 10.13.5， 系统内置的GNU版本的 g++ 和 gcc都是8.1

所以安如下配置进行源码安装： 

```
# install gcc 7
brew install cmake
brew install gcc@7
```

接下来安装lightGBM

```
git clone --recursive https://github.com/Microsoft/LightGBM ; cd LightGBM
# 这里的export环境变量是brew install的默认安装路径
export CXX=/usr/local/Cellar/gcc@7/7.3.0/bin/g++-7 CC=/usr/local/Cellar/gcc@7/7.3.0/bin/gcc-7
mkdir build ; cd build
cmake ..
make -j4
```

配置python环境， 直接用pin install lightgbm有坑

```
cd ../python-package
sudo python setup.py install --precompile
```

至此， python环境下调用lightGBM的API应该没有问题了。 



