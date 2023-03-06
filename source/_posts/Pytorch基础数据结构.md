---
title: Pytorch基础数据结构
date: 2023-02-23 10:46:11
categories: AI模块
tags:
    - 人工智能
    - 算法
    - 深度学习
    - pytorch
    - 编程语言
    - 啃书
photos:
    - /2023/02/23/Pytorch基础数据结构/pytorch.png
---

开始学深度学习咯！在正式进入深度神经网络的学习之前，还是需要先选好框架实现。在这里，我选择的还是比较简单的pytorch（基础好的选TensorFlow也可以的！）

这一篇博客主要介绍pytorch的常用数据结构及其常见操作，还有深度学习的基本流程。

<!-- more -->

主要内容包括：
1. tensor张量
2. 数据加载工具：Dataset、Sampler、DataLoader等
3. 深度学习基本流程

## **tensor张量**

tensor张量是pytorch中最基础的数据结构，相当于numpy中的ndarray，下面介绍一些基础的tensor操作（基本上，numpy有的tensor都有）

<iframe src="https://nbviewer.org/github/WA-automat/deep_learning/blob/main/%E7%AC%AC%E4%B8%80%E9%83%A8%E5%88%86%EF%BC%9Apytorch%E5%9F%BA%E7%A1%80%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/tensor.ipynb" width="100%" height="2200"></iframe>

## **Dataset与DataLoader**

这两个存储结构是为了便利于pytorch访问大数据的数据结构，具体创建与使用方法如下：

<iframe src="https://nbviewer.org/github/WA-automat/deep_learning/blob/main/%E7%AC%AC%E4%B8%80%E9%83%A8%E5%88%86%EF%BC%9Apytorch%E5%9F%BA%E7%A1%80%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/dataloader.ipynb" width="100%" height="1800"></iframe>

## **深度学习开发的基本过程**

1. 数据准备：加载——变换——批处理
2. 模型开发：设计——训练——测试
3. 模型部署
