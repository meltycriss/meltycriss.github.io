---
title: Ubuntu 16.04配置Caffe环境
date: 2016-06-28 19:21:42
tags: 
  - caffe配置
description:
  - Ubuntu 16.04配置Caffe环境
categories:
  - 环境配置
---

## 基本配置流程
1. 从官网克隆到本地 [caffe官网](https://github.com/BVLC/caffe)
> git clone git@github.com:BVLC/caffe.git

2. 按照官方教程安装 [官方安装教程](http://caffe.berkeleyvision.org/installation.html) 
	1. 配置依赖文件 [依赖文件](http://caffe.berkeleyvision.org/install_apt.html)
	> sudo apt-get install libprotobuf-dev libleveldb-dev libsnappy-dev libopencv-dev libhdf5-serial-dev protobuf-compiler
	sudo apt-get install --no-install-recommends libboost-all-dev  
	sudo apt-get install libatlas-base-dev  
	sudo apt-get install libgflags-dev libgoogle-glog-dev liblmdb-dev

	2. 编译(**默认含GPU，可以用-j4多核加速编译**) [编译](http://caffe.berkeleyvision.org/installation.html#compilation)
	> cp Makefile.config.example Makefile.config  
	make all  
	make test  
	make runtest

## 踩过的坑
- **can't find hdf5.h when build caffe**: 用以下sh改变include路径 [source](https://github.com/NVIDIA/DIGITS/issues/156)
```
#!/bin/bash
# manipulate header path, before building caffe on debian jessie
# usage:
# 1. cd root of caffe
# 2. bash <this_script>
# 3. build

# transformations :
#  #include "hdf5/serial/hdf5.h" -> #include "hdf5/serial/hdf5.h"
#  #include "hdf5_hl.h" -> #include "hdf5/serial/hdf5_hl.h"

find . -type f -exec sed -i -e 's^"hdf5.h"^"hdf5/serial/hdf5.h"^g' -e 's^"hdf5_hl.h"^"hdf5/serial/hdf5_hl.h"^g' '{}' \;
```

- **/usr/bin/ld: cannot find -lhdf5_hl**: 修改文件名字 [source](https://github.com/NVIDIA/DIGITS/issues/156)  
> cd /usr/lib/x86_64-linux-gnu  
sudo ln -s libhdf5_serial.so.10.1.0 libhdf5.so  
sudo ln -s libhdf5_serial_hl.so.10.0.2 libhdf5_hl.so

- **‘memcpy’ was not declared in this scope**: 修改Makefile.config [source](https://github.com/BVLC/caffe/issues/4046)
> 将Makefile.config中的  
NVCCFLAGS += -ccbin=$(CXX) -Xcompiler -fPIC $(COMMON_FLAGS)  
修改为  
NVCCFLAGS += -D_FORCE_INLINES -ccbin=$(CXX) -Xcompiler -fPIC $(COMMON_FLAGS)  

- **安装Cuda**:  Cuda官网只有给出14.04和15.04的Cuda7.5版本，但实际上可以直接通过apt-get获取16.04的Cuda7.5
	1. 安装cuda
	> sudo apt-get install nvidia-cuda-toolkit
	2. 修改caffe/Makefile.config中的CUDA_DIR


- **安装显卡驱动**: 系统设置-软件和更新-附加驱动，选择最新的显卡驱动，应用。
	* 安装驱动后屏幕无法调节亮度的解决办法 [调节亮度](http://askubuntu.com/questions/76081/brightness-not-working-after-installing-nvidia-driver)
	* 使用`nvidia-setttings`查看gpu使用情况




