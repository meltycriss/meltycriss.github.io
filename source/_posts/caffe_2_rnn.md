---
title: Caffe学习：RNN源码阅读
date: 2016-07-13 15:20:42
tags: 
  - rnn
description:
  - Caffe中的RNN源码阅读
categories:
  - caffe
---

## 简介
RNN(Recurrent Neural Network)是一种能考虑上下文信息的神经网络，在求解的时候不止考虑当前的输入是什么，还考虑上一次的输出是什么，有种马尔可夫链的感觉，比较适用于包含上下文语义的任务。这里选了标准RNN源码入手，学习RNN的实现。

## 源码框架

### 目录结构
- **sequence_layers.hpp:**  抽象类RecurrentLayer，子类RNN和LSTM的头文件
- **recurrent_layer.cpp:** 抽象类RecurrentLayer的定义文件
- **rnn_layer.cpp:** 子类RNN的定义文件

### 逻辑结构
- 使用了**模板方法设计模式**
- **RecurrentLayer:** 定义了网络的通用框架，包括输入，输出，循环部分的入口输入
- **RNNLayer:** 定义了循环部分的网络结构
- **bottom:**
	* bottom[0]: T，帧数
	* bottom[1]: N，视频数
	* ...: 真正的data
- **top:** 同bottom

## sequence_layers.hpp

### 成员变量
- **shared\_ptr< Net< Dtype> \> unrolled\_net\_:** 最终生成的网络
- **int N\_:** 视频流的数量
- **int T\_:** 视频帧数
- **bool static\_input\_:** 输入是否静态
- **vector< Blob< Dtype\>* \> recur\_input\_blobs\_:** 子类循环部分的入口输入($h_0$)
- **vector< Blob< Dtype\>* \> recur\_output\_blobs\_:** 子类循环部分的出口输出($h_T$)
- **vector< Blob< Dtype\>* \> output\_blobs\_:** 总输出(top)
- **Blob< Dtype\>* x\_input\_blob\_:** 总视频输入(bottom[0])
- **Blob< Dtype\>* x\_static\_input\_blob\_:** 总静态输入(bottom[2])
- **Blob< Dtype\>* cont\_input\_blob\_:** 总标识输入(bottom[1])。cont为0表示应该新开一个序列，不用再参照上一次的输出结果

### 成员函数
- **FillUnrolledNet(NetParameter* net\_param):** 	
	* 子类根据自己的内部网络修改net\_param
	* 原理同我们在外面写prototxt生成网络
	* 类比安卓开发中动态生成按钮选项，都是因为数量经常变化，写死在xml文件中不便于改动
- **xxxBlobNames:** 
	* 返回与主控部分交互的blob名字，主控部分根据名字找到对应的blob
	* 类比安卓开发中findViewByID将控件和变量绑定

## recurrent_layer.cpp

### Reshape
适配blob，以及绑定blob，使得操作变量等价于修改bottom与top

1. **Reshape:** 将x\_input\_blob\_, cont\_input\_blob\_和x\_static\_input\_blob\_几个输入blob的shape适配bottom
2. **Reshape:** 将recur\_input\_blobs\_适配子类的入口输出blob(即$h_0$)
3. **Share:** 将x\_input\_blob\_, cont\_input\_blob\_和x\_static\_input\_blob\_与bottom的data及diff进行绑定
4. **Share:** 将output\_blobs\_与top的data及diff进行绑定

### LayerSetUp
等价于将prototxt转为代码，核心为：初始化net\_param来生成一个Net

1. **检查bottom是否规范**
	- bottom[0]: 各帧，各视频的数据 (T \* N \* ...)
	- bottom[1]: 各帧，各视频的标识，0为开始，1为继续 (T \* N \* 1)
	- bottom[2]: 静态输入
2. **为网络接入总输入，并为各输入命名**
	- x: bottom[0]
	- cont: bottom[1]
	- x_static: bottom[2]
3. **进一步构建网络**，调用FillUnrolledNet修改net\_param
4. **用net\_param生成最终的网络**
```cpp
  unrolled_net_.reset(new Net<Dtype>(net_param));
```
5. **将变量与网络对应部分绑定**
	- 在上一步中网络已经生成完毕，但为了操作方便，将要用到的网络部分跟变量绑定，以后直接用变量进行操作
	- 绑定的时候使用net->blob\_by\_name("xxx")，类似findViewById。找blob的key是名字，这就是为什么要有xxxBlobNames这样的函数
6. **设置辅助参数**
	- blobs\_: 这个参数应该要记录本层所有的blob，但由于我们的网络定义不仅是prototxt，还有中途动态生成的部分，所以不能依赖caffe帮我们自动生成的blobs\_，要手动将所有参数添加
	- param\_propagate\_down\_: 记录是否要bp
	- recur\_output\_blobs\_全部置0**（不懂）**

## rnn_layer.cpp

### 跟主控交接部分

- **循环输入入口:** $h_0$，1 \* N \* num\_output（循环输入跟循环输出shape一致，所以这里既说明了输入由说明了输出，即对于每一帧，生成num\_output维的向量）
- **循环输出出口:** $h_t$
- **总输出:** o

### FillUnrolledNet

核心部分，实现网络的定义。实际效果近似于这样的prototxt

``` 
################################
#            input             #
################################
layer{
	#sliced x
	bottom: x
	top: x_1
	top: x_2
	.
	.
	.
	top: x_t
}
layer{
	#sliced cont
	bottom: cont
	top: cont_1
	top: cont_2
	.
	.
	.
	top: cont_t
}
################################
#          recur layer         #
################################
layer{
	#recur_unit_1
	bottom: x_1, cont_1, h_0
	top: o_1, h_1
}
layer{
	#recur_unit_2
	bottom: x_2, cont_2, h_1
	top: o_2, h_2
}
		.
		.
		.

layer{
	#recur_unit_t
	bottom: x_t, cont_t, h_t-1
	top: o_t, h_t
}
################################
#           output             #
################################
layer{
	#concated output
	bottom: o_1
	bottom: o_2
	.
	.
	.
	bottom: o_t
	top: o
}
```

## 收获

- 阅读源码前前思考代码的**输入**是什么，**输出**是什么，给你这样的任务**你会怎么做**
- 然后在阅读的时候合理地进行**假设**，一步步验证并修改
- 先看**数据结构**再看算法，**粒度**从粗到细（类级，函数级，块级，行级）


## Reference
- 论文地址：[s2vt](https://github.com/vsubhashini/caffe/tree/recurrent/examples/s2vt)
- 项目地址：[Sequence to Sequence -- Video to Text](https://arxiv.org/abs/1505.00487)
- 参考资料：[LSTM](http://colah.github.io/posts/2015-08-Understanding-LSTMs/), [Sequences in Caffe](http://tutorial.caffe.berkeleyvision.org/caffe-cvpr15-sequences.pdf)
