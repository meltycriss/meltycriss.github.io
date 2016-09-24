---
title: Caffe学习：s2vt数据处理部分源码阅读
date: 2016-07-22 11:32:32
tags: 
  - s2vt data
description:
  - s2vt数据预处理部分源码，即txt转hdf5格式的python脚本
categories:
  - caffe
---

## 任务简介
s2vt做的是从视频生成文字，输入端的数据量较于传统任务庞大很多，对数据流的输入速率提出了要求。传统的以txt方式保存数据的读取方式不再适用，转而使用了更大但是更快的hdf5格式。于是就需要实现两种功能，分别是将数据从**txt格式转换成hdf5格式的脚本**，以及能够**读取hdf5的数据输入层**。

## 源码框架

### txt转hdf5的python脚本

- 采用了**模板方法**的设计模式
- **hdf5\_npstreamsequence\_generator.py**: 其下定义了两个类，分别是
	* **SequenceGenerator: **txt转hdf5类的父类，定义了获取下一个batch内容的算法框架
	* **HDF5SequenceWriter: **I/O类，给定一个SequenceGenerator，将其内容输出到.h5文件
- **framefc7\_stream\_text\_to\_hdf5\_data.py**: 子类，定义了获取最小粒度数据的方法

### 读取hdf5数据的输入层

- **hdf5\_data\_layer.cpp**

### 核心调用关系(从上往下调用)

- **HDF5SequenceWriter::write\_to\_exhaustion**
	* 调用write\_batch，直到所有数据读完
- **HDF5SequenceWriter::write\_batch**
	* **input:** get\_next\_batch获取的batch
	* **output:** 将batch内容输出为.h5文件
- **SequenceGenerator::get\_next\_batch**
	* **intput:** 流数据
	* **output:** batch\[Name]\[T]\[N]
- **SequenceGenerator::reset\_stream**
	* **input:** 最小粒度流数据
	* **output:** 重设batch中第i条流为下一条输入流
- **fc7FrameSequenceGenerator::get_streams**
	* **input:** txt文件
	* **output:** 由下段line及其对应视频的frames所生成的数据

## hdf5\_npstreamsequence\_generator.py

### HDF5SequenceWriter::write\_to\_exhaustion

不停地调用write\_batch，直到所有输入的流（即line）都读完

```python
def write_to_exhaustion(self):
    while not self.generator.streams_exhausted():
      self.write_batch(stop_at_exhaustion=True)
```

### HDF5SequenceWriter::write\_batch

将batch的内容以hdf5的格式保存为.h5文件，hdf5的基本操作流程如下（基本复用了NumPy的表示方式），详情参考[h5py.doc](http://docs.h5py.org/en/latest/high/dataset.html#creating-datasets)

```python
h5file = h5py.File(filename, 'w')
# return the container
dataset = h5file.create_dataset('cont', shape=cont_indicators.shape, dtype=cont_indicators.dtype)
# write data intot the container
dataset[:] = cont_indicators
h5file.close()
```

### SequenceGenerator::get\_next\_batch

变量解释：

- **batch:** batch\[Name]\[T]\[N]
	* 存放数据内容
	* **Name:** 是指framefc7, intput_sentence之类的
	* **T:** 是时间，注意到batch的T是1000，而每一个输入的T是80，所以说在相通的N下，T=80的流跟T=81的流不是同一个流，而是相隔了N的两个流
- **batch\_indicators:** batch\_indicators\[T]\[N]
	* 指示对应流的开始和结束，0是开始，1是延续
- **self.substream\_names:** batch中Name维的值域，即framefc7, input_sentence
- **self.array\_type\_inputs:** 类型是数组的name，比如framefc7\[num_frames]\[4096]
- **exhausted:** vector<bool>(N)
	* 指示batch中第N个流是否已经结束（80的倍数或者轮到第N个流时输入已经结束）
- **all\_exhausted:** true if all exahusted are true
- **reached\_exhaustion:** 基本同streams_exhausted()
- **self.stream\_indices:** 标识当前第N个流读到T几

过程解释：

初始化batch和batch\_indicator的shape，以及各种辅助变量
```python
if not self.streams_initialized:
	self.init_streams()
# format: len0: [s0, s2, num of streams, s_n]
#         len1: [s0, s2, num of streams, s_n]
#         len2: [s0, s2, num of streams, s_n]
batch_size = self.batch_num_streams * self.batch_stream_length
batch = {}
batch_indicators = np.zeros((self.batch_stream_length, self.batch_num_streams))
# reshape batch[name] like batch_indicators
# and set value to pad value
for name in self.substream_names:
	# if value is high dimension
	if name in self.array_type_inputs.keys():
    	dim = self.array_type_inputs[name]
        batch[name] = self.get_pad_value(name) * np.ones((self.batch_stream_length, self.batch_num_streams, dim))
	# if value is 1d
	else:
        # each batch[name] is a T * N * dim blob
        batch[name] = self.get_pad_value(name) * np.ones_like(batch_indicators)
```
假如第i个流从来没有用过或者上一个位于i位置的流已经读完，就reset\_stream(i)
```python
# never been initialized or come to the end of a stream
if self.streams[i] is None or \
	self.stream_indices[i] == len(self.streams[i][self.substream_names[0]]): 
	self.stream_indices[i] = 0
	# Q: self.streams_exhausted() always return false, so the expression is meaningless?
	# A: derived class will override function streams_exhausted
	# reached_exhaustion is forever True after pass through all lines
	reached_exhaustion = reached_exhaustion or self.streams_exhausted()
	# exhausted[i] indicates the end of ith stream i.e. all lines in ith stream are read
	if reached_exhaustion: exhausted[i] = True
	# Q: why reset stream i? self.streams is the same data for all stream i
	# A: get_streams() in reset_stream() is wrapped around
	if not reached_exhaustion or not truncate_at_exhaustion:
		self.reset_stream(i)
	else:
		continue
```

将各个name的t, i写到对应的batch\[name]\[t][i]
```python
for name in self.substream_names:
    if isinstance(self.streams[i][name], np.ndarray) and self.streams[i][name].ndim > 1:
        batch[name].resize((batch_size, self.streams[i][name].shape[1],1))
        batch[name][(t*self.batch_num_streams + i), :,0] = self.streams[i][name][self.stream_indices[i],:]
    elif name in self.array_type_inputs.keys():
        batch[name][t, i] = self.streams[i][name][self.stream_indices[i]][0,:]
    else:
        batch[name][t, i] = self.streams[i][name][self.stream_indices[i]]
```

### SequenceGenerator::reset\_stream

1. 通过get_streams()得到下一个数据流（即下一个line对应的input, framefc7 ...)
2. 修改实例变量streams[stream_index]为下一个数据流

## framefc7\_stream\_text\_to\_hdf5\_data.py

### fc7FrameSequenceGenerator::\__init__

从txt中读取数据并将数据存入以下变量

- self.vid\_framefeats[video\_id]: 存放video_id对应的frames(frame1, frame2)的feats(4096)
- self.lines: pair< vid, line >

### fc7FrameSequenceGenerator::get_streams

将下一条line对应的frames feats及其他数据规范化为MAX_WORD长度的out，out的示意如下

```
		     MAX_WORD
cont_sentence	x x x x ... x x x x
input_sentence  x x x x ... x x x x 
frame_fc7	x x x x ... x x x x
		| | | |     | | | |  
		| | | |     | | | |  4096

```

## 收获

- 在看源码前大概**交流**一下各个函数是干嘛用的，把握整体思路
- 在纸上画出**核心函数调用链**
- 像python这样的弱类型语言，可以看**被调函数返回数据**的数据结构

## resource
- [含注释的hdf5\_npstreamsequence\_generator.py](https://github.com/meltycriss/commented_src/blob/master/s2vt_data/hdf5_npstreamsequence_generator.py)

## reference

- 项目地址：[Sequence to Sequence -- Video to Text](https://arxiv.org/abs/1505.00487)

