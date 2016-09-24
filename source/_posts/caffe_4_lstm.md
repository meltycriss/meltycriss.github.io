---
title: Caffe学习：LSTM源码阅读
date: 2016-08-05 10:00:46
tags: 
  - lstm
description:
  - LSTM源码阅读，主要针对几个比较tricky的点展开
categories:
  - caffe
---

## 简介
由于之前已经有一篇RNN的源码阅读文章，这里就不再从超类讲起了，而且整体思路上LSTM跟RNN比较相似，所以本文主要是将某些比较tricky的点提出来，而不再像往常的源码阅读一样对整份源码解析。

## 源码框架

### 目录结构
- **sequence_layers.hpp:**  抽象类RecurrentLayer，子类RNN和LSTM的头文件
- **recurrent_layer.cpp:** 抽象类RecurrentLayer的定义文件
- **lstm_layer.cpp:** 子类LSTM的定义文件
- **lstm\_unit\_layer.cpp:** 子类LSTM的辅助层定义文件

### 逻辑结构
首先要说明的是这里的LSTM不是[标准LSTM](http://www.jianshu.com/p/9dc9f41f0b29/)，而是一种变种，具体参考论文[Sequence to Sequence - Video to Text](https://github.com/vsubhashini/caffe/tree/recurrent/examples/s2vt)，最关键的区别在于x和h的处理，不再是拼接，而是求和。

每个for循环生成了一个像这样的net（省略cont）

```
     |---|         |---|
 c_0 |   |   c_0   |   |  c_1
-----|   |---------|   |-----
 h_0 |   | Wh+Wx+b |   |  h_1
-----|   |---------|   |-----
     |---|         |---|
   lstm_layer  lstm_unit_layer

```

## num_output * 4
可以看到在lstm\_layer.cpp中的内积层都是将num\_output设置成num_output * 4

```cpp
  LayerParameter hidden_param;
  hidden_param.set_type("InnerProduct");
  hidden_param.mutable_inner_product_param()->set_num_output(num_output * 4);
  hidden_param.mutable_inner_product_param()->set_bias_term(false);
  hidden_param.mutable_inner_product_param()->set_axis(2);
  hidden_param.mutable_inner_product_param()->
      mutable_weight_filler()->CopyFrom(weight_filler);
```

而这个内积层后面用作将h全连接成一个num_output * 4维的向量

```cpp
  LayerParameter* w_param = net_param->add_layer();
  w_param->CopyFrom(hidden_param);
  w_param->set_name("transform_" + ts);
  w_param->add_param()->set_name("W_hc");
  w_param->add_bottom("h_conted_" + tm1s);
  w_param->add_top("W_hc_h_" + tm1s);
  w_param->mutable_inner_product_param()->set_axis(2);
```

最后又可以看到，在lstm\_unit\_layer.cpp中，当使用到这个num_output * 4维的向量时，是把它当作4个不同意义的向量来用的


```cpp
  const Dtype i = sigmoid(X[d]);
  const Dtype f = (*flush == 0) ? 0 :
      (*flush * sigmoid(X[1 * hidden_dim_ + d]));
  const Dtype o = sigmoid(X[2 * hidden_dim_ + d]);
  const Dtype g = tanh(X[3 * hidden_dim_ + d]);
```

那为什么要用一种这么不直观的方式来写呢？一般来说，有违直观理解的编码方式都是**出于提高效率**考虑的。下面讨论具体原因

- 创建一个layer有**开销**
- 一个大矩阵的优化会比拆开的几个小矩阵的优化效果好，因为
	* 拆开的几个小矩阵相当于用**循环**
	* 在有**优化矩阵计算**的情况下，直接用矩阵计算会比用循环来计算快很多[（为什么矩阵计算比循环快）](https://www.zhihu.com/question/19706331)

## gate_input

源码里面给到lstm\_unit\_layer.cpp的输入是通过h和x求完内积的gate_input，而不是比较直观地将h和x输入，然后在lstm\_unit\_layer.cpp里面求内积。

```cpp
 // Add LSTMUnit layer to compute the cell & hidden vectors c_t and h_t.
 // Inputs: c_{t-1}, gate_input_t = (i_t, f_t, o_t, g_t), cont_t
 // Outputs: c_t, h_t
 //     [ i_t' ]
 //     [ f_t' ] := gate_input_t
 //     [ o_t' ]
 //     [ g_t' ]
 //         i_t := \sigmoid[i_t']
 //         f_t := \sigmoid[f_t']
 //         o_t := \sigmoid[o_t']
 //         g_t := \tanh[g_t']
 //         c_t := cont_t * (f_t .* c_{t-1}) + (i_t .* g_t)
 //         h_t := o_t .* \tanh[c_t]
 {
   LayerParameter* lstm_unit_param = net_param->add_layer();
   lstm_unit_param->set_type("LSTMUnit");
   lstm_unit_param->add_bottom("c_" + tm1s);
   lstm_unit_param->add_bottom("gate_input_" + ts);
   lstm_unit_param->add_bottom("cont_" + ts);
   lstm_unit_param->add_top("c_" + ts);
   lstm_unit_param->add_top("h_" + ts);
   lstm_unit_param->set_name("unit_" + ts);
 }

```

X先在外面统一求，再通过切片的方式传给unit可以理解成为了提高效率。但是为什么要将h也在外面求完内积，和x求和后以一个统一的gate_input传给unit就是出于**实现方便**的角度考虑。

- 试想将求内积的操作放在unit里面，那么在**求回传梯度**的时候，由于有一个内积层，就变得非常不好求
- 而现在的实现方式，将所有内积操作放在unit外面，使得unit求梯度回传与内积层无关，变得简单

## tanh与sigmoid

之前都没发现原来tanh和sigmoid之间的关系是这样的

```cpp
  template <typename Dtype>
  inline Dtype sigmoid(Dtype x) {
    return 1. / (1. + exp(-x));
  }

  template <typename Dtype>
  inline Dtype tanh(Dtype x) {
    return 2. * sigmoid(2. * x) - 1.;
  }

```

进而他们的导数为

$$
sigmoid' = sigmoid * (1 - sigmoid)\\\\
tanh' = 1 - tanh^{2}
$$

## 对cont的处理

cont为0的时候需要截断操作，具体的表现为

- 作为输入的h为0

```cpp
  // Add layers to flush the hidden state when beginning a new
  // sequence, as indicated by cont_t.
  //     h_conted_{t-1} := cont_t * h_{t-1}
  //
  // Normally, cont_t is binary (i.e., 0 or 1), so:
  //     h_conted_{t-1} := h_{t-1} if cont_t == 1
  //                       0   otherwise
  {
    LayerParameter* cont_h_param = net_param->add_layer();
    cont_h_param->CopyFrom(scalar_param);
    cont_h_param->set_name("h_conted_" + tm1s);
    cont_h_param->add_bottom("h_" + tm1s);
    cont_h_param->add_bottom("cont_" + ts);
    cont_h_param->add_top("h_conted_" + tm1s);
  }
```

- 对c不进行遗忘操作

```cpp
    //         c_t := cont_t * (f_t .* c_{t-1}) + (i_t .* g_t)
    const Dtype f = (*flush == 0) ? 0 :
        (*flush * sigmoid(X[1 * hidden_dim_ + d]));

```

同时也可以看出来，当$h_0$和$c_0$的初始值设为多少不会有影响，因为

- 在时刻0的时候cont为0，$h_0$会被归0
- 且由于f\_t=0，c\_{t-1}对c_{t}没影响

## 收获

- 看源码的时候先确定模型是**标准模型**还是**变种**
- 实现有违直观逻辑时，考虑
	* **效率**
	* **实现便利度**