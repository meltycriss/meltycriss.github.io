---
title: 论文笔记《Rich feature hierarchies for accurate object detection and semantic segmentation》
date: 2016-07-08 15:45:08
tags:
	- rcnn
categories:
	- 论文笔记
description:
	- 用于完成object detection的RCNN
---

## 任务简介
- **目标：**object detection
- **输入：**图片
- **输出：**框住前景物体**（有可能多种且每种有多个）**，并且给物体标上类别

## 思路
1. **分解成两个子任务**
	* localization 
	* classification
2. **localization：**枚举
3. **classification：**CNN
4. 最后再**筛掉**多余的

## 实现流程
1. **localization：**用selective search提取region proposals（即一个个候选框）
2. **warped region：**为了适配CNN输入，将region proposals格式归一化
3. **feature extraction：**CNN
4. **classification：**对于每个类别训练一个二分类SVM
5. **综合策略：**NMS(non-maximum suppression)，重叠的选最有信心的

## 实现细节
- **pre-training：**直接用ILSVRC 2012训练CNN
- **fine-tuning：**将pre-training训练出来的CNN换上匹配当前任务的输入层（warped region）和分类层（1000-way to 21-way）来训练模型
	* **训练集：**warped region的label根据其与GT(Ground Truth)的IoU(Intersection-over-Union，交集/并集)的大小来定。这里将threshold设为0.5
- **classifier：**将fine-tuning训练出来的模型换上SVM作为分类层
	* **训练集：**方法同fine-tuning的训练集，不过这里的threshold设为0.3

## Visualization
- **学习：**定义某个unit学习的内容为**让这个unit输出最高的input**，即给定一堆input，unit对某个input\_0的输出最高，则认为这个unit识别的就是这个input\_0。通过这个定义可以将各个unit学习什么输出出来

## Ablation studies（切除）（用于验证有效性）
- **ablation：**比较要不要某一个层，对于结果的影响
- **结果：**
	* 在不进行fine-tuning的情况下，CNN中的全连接层并没有什么卵用
	* 在进行fine-tuning的情况下，提升很大

## Detection error analysis
- **量化误差：**给出了percentage of each type - total false positives图，说明了在各个FP时的误差类型分布。错误类型有以下
	* 框错位置
	* 分成相似的类
	* 分成不相似的类
	* 分成背景
- **结果：**RCNN的错误基本都是框错了位置，作者认为这时因为**bottom-up region proposals**还有**positional invariance learned from pre-training CNN**导致的
- **解决方案：**对于生成的框，要使用**BB(Bounding box regression)**进行进一步调整

## 收获
- object detection的任务
- RCNN
- 评价模型的指标
	* 定性：visualization
	* 定量：Ablation, detection error
	