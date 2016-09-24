---
title: 论文笔记《ImageNet Classification with Deep Convolutional Neural Networks》
date: 2016-07-01 19:10:02
tags: 
  - AlexNet
description:
  - AlexNet介绍，包括其架构，如何加快收敛，怎样加强泛化能力以及如何评估模型
categories:
  - 论文笔记
---

第一次阅读论文，没有什么阅读技巧，再加上对CNN[(CNN解析)](http://www.moonshile.com/post/juan-ji-shen-jing-wang-luo-quan-mian-jie-xi)没有什么概念，读了几次才大概明白讲的是什么。

## The Architecture

1. **ReLU Nonlinearity**
	- **目的：** 加快收敛速度
	- **手段：** 使用Rectified Linear Units(ReLUs)作为神经元。即采用$$f(x) = max(0, x)$$作为激活函数
	- **原理：** 一般使用的激活函数$$f(x) = tanh(x) \\\\ f(x) = \frac{1}{1 + e^{-x} }$$具有饱和(saturating)的特性，即导数在越接近目标的时候越小[(原理参考)](http://zhangliliang.com/2014/07/01/paper-note-alexnet-nips2012/)。当使用梯度下降的方法时，步长由学习率以及导数决定，导数太小会导致步长太小，收敛会慢。ReLU对于大于0部分导数恒为1，所以能加快收敛。


2. **Training on Multiple GPUs**
	- **目的：** 加快训练速度

3. **Local Response Normalization**
	- **目的：** 加强泛化能力
	- **手段：** 神经元受附近神经元抑制，各神经元的输出为$$b_i = \frac{a_i}{ {(k+\alpha\sum{a_j^2})}^\beta}$$其中$b_i$为神经元i的输出，$a_i$为神经元i的输入
	- **原理：** 模拟神经元间的抑制作用

4. **Overlapping Pooling**
	- **目的：** 加强泛化能力
	- **手段：** pooling的时候stride < filter size，重叠地pooling
	- **原理：** 实验性结果

## Reduce Overfitting

1. **Data Augmentation**
	- **目的：** 加强泛化能力
	- **手段1：** 随机裁剪，将原来256×256的图片随机裁剪为224×224,并且允许水平翻转，则增加了$(256-224)^2*2$倍的样本量。（裁剪的时候一定是要连片的，不能是随机取的）
	- **手段2：** PCA，这个部分不太懂原理。[(原理参考)](http://zhangliliang.com/2014/07/01/paper-note-alexnet-nips2012/)
	- **原理：** 增大样本数增强泛化能力。裁剪，旋转，滤镜并不影响图片内容，但是对于机器来说就是完全不同的样本

2. **Dropout**
	- **目的：** 加强泛化能力
	- **手段：** 随机disable神经元
	- **原理：** 隐式多模型，迫使模型要学习更robust的feature，抓住核心特征

## Qulitative Evaluations

1. **准确性：** 对于每张图片，给出CNN分类结果前5的类别及概率
2. **一致性：** 对于每张照片，给出CNN提取特征相似的照片 e.g. 图片$i_1$提取的特征为$f_1$，找特征跟$f_1$相似的$i_k$，看下是不是同一个物体
