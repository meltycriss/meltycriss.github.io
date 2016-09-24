---
title: Caffe学习：自定义数据读取
date: 2016-07-04 19:23:42
tags: 
  - caffe学习
description:
  - Caffe基本框架，阅读和修改image_data_layer源码，使用Caffe的基本流程
categories:
  - caffe
---

## Caffe架构

### 简介
Caffe将层的实现（输入层，卷积层，全连接层），整体网络的定义（AlexNet，LeNet），以及超参数的定义（步长，迭代次数）解耦。用c++实现各层，并使用protocol buffer来定义网络和传入参数

### 基础构件

.cpp文件实现一个个层，作为基础构件供上层调用

- **CAFFE_ROOT/include/caffe/layers**: .hpp为层的头文件
- **CAFFE_ROOT/src/caffe/layers**: .cpp为层的实现文件
- **CAFFE_ROOT/src/caffe/proto**: .proto为传参数用的protocol buffer文件

### 自定义网络

- **MODEL_ROOT/train_val.prototxt**: 用作定义整体网络
- **MODEL_ROOT/solver.prototxt**: 为网络传递超参数
- **MODEL_ROOT/train.sh**: 开始训练的脚本
- **MODEL_ROOT/test.sh**: 开始测试的脚本

## 自定义数据读取层

### 数据描述
[MSRA 10K Salient Object Database](http://mmcheng.net/msra10k/)。含有1w个数据，.jpg结尾的是原图，.png结尾的是标注了的图。用9k张作为训练集，用1k张作为测试集

### image_data_layer总览

image_data_layer实现了三个函数和调用了两个宏

- **DataLayerSetUp**: 把容器的规格设置好（设好N, K, H, W)
- **ShuffleImages**: 打乱顺序
- **load_batch**: 真正把图片读入内存
- **INSTANTIATE_CLASS**和**REGISTER_LAYER_CLASS**: 注册创建的层


### DataLayerSetUp

- **读取图片规格**: 从protocol buffer中读取目标图片的k,h,w（有可能原图800\*600，但想要256\*256，这时就需要压缩）
```cpp
  const int new_height = this->layer_param_.image_data_param().new_height();
  const int new_width  = this->layer_param_.image_data_param().new_width();
  const bool is_color  = this->layer_param_.image_data_param().is_color();
  string root_folder = this->layer_param_.image_data_param().root_folder();

```
- **读取图片列表**: 从一个txt文件中读取图片名及其对应label，写进lines_
```cpp
  while (std::getline(infile, line)) {
    pos = line.find_last_of(' ');
    label = atoi(line.substr(pos + 1).c_str());
    lines_.push_back(std::make_pair(line.substr(0, pos), label));
  }
```
- **shuffle & skip**
- **设置容器规格**: 读一张图进来，并用它的规格来设置容器的规格
```cpp
  // Read an image, and use it to initialize the top blob.
  cv::Mat cv_img = ReadImageToCVMat(root_folder + lines_[lines_id_].first,
                                    new_height, new_width, is_color);
  CHECK(cv_img.data) << "Could not load " << lines_[lines_id_].first;
  // Use data_transformer to infer the expected blob shape from a cv_image.
  vector<int> top_shape = this->data_transformer_->InferBlobShape(cv_img);
  this->transformed_data_.Reshape(top_shape);
  // Reshape prefetch_data and top[0] according to the batch_size.
  const int batch_size = this->layer_param_.image_data_param().batch_size();
  CHECK_GT(batch_size, 0) << "Positive batch size required";
  top_shape[0] = batch_size;
  for (int i = 0; i < this->PREFETCH_COUNT; ++i) {
    this->prefetch_[i].data_.Reshape(top_shape);
  }
  top[0]->Reshape(top_shape);

```

### load_batch

- **读取图片规格**
- **设置容器规格**
- **读入图片**: prefetch\_data为起始地址，offset根据容器的规格来求，然后在prefetch\_data+offset的位置设置一个容器transformed\_data\_，并将图片写到容器里面
```cpp
    cv::Mat cv_img = ReadImageToCVMat(root_folder + lines_[lines_id_].first,
        new_height, new_width, is_color);
    CHECK(cv_img.data) << "Could not load " << lines_[lines_id_].first;
    read_time += timer.MicroSeconds();
    timer.Start();
    // Apply transformations (mirror, crop...) to the image
    int offset = batch->data_.offset(item_id);
    this->transformed_data_.set_cpu_data(prefetch_data + offset);
    this->data_transformer_->Transform(cv_img, &(this->transformed_data_));
    trans_time += timer.MicroSeconds();
```

### 自定义image_data_layer_disc

复现的模型为[DISC](http://ieeexplore.ieee.org/xpls/abs_all.jsp?arnumber=7372470&tag=1)，label不是一个简单的int，而是一张图片，所以要对数据输入层进行修改，来产生一个图片label

- **image_data_layer_disc.hpp**: 修改lines_的定义，改成两个string的pair
- **image_data_layer_disc.cpp**: 参考读入data的方式处理label
- **caffe.proto**: 添加一个disc_image_data_para来添加关于label的参数

## 训练和测试

- **train_val.prototxt**: 赋值models里面的AlexNet，将data层改成我们的image_data_layer_disc，将loss层改成欧式距离，删除accuracy层
- **solver.prototxt**: 网络指定为train_val.prototxt，学习率指定为1e-9
- **train.sh**: 
```shell
#!/usr/bin/env sh

./build/tools/caffe train --solver=models/disc/solver.prototxt

```
- **test.sh**:
```shell
#!/usr/bin/env sh

./build/tools/caffe.bin test -model=models/disc/train_val.prototxt -weights=models/disc/caffe_alexnet_train_iter_370000.caffemodel -gpu=1
```
## Protocol Buffer
给出一些介绍Protocol Buffer的网站
- [IMB](https://www.ibm.com/developerworks/cn/linux/l-cn-gpb/)
- [Google](https://developers.google.com/protocol-buffers/docs/encoding#optional)
- [pb1](http://www.iloveandroid.net/2015/10/08/studyPtorobuf/)
- [pb2](http://www.cnblogs.com/royenhome/archive/2010/10/29/1864860.html)