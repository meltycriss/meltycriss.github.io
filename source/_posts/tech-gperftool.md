---
title: 技术总结《使用gperftools排查内存泄漏》
date: 2024-09-28 16:47:47
tags:
    - Debug
    - 内存泄漏
categories:
	- 技术总结
description:
	- 一种基于gperftools排查内存泄漏的方法
---

# 前言

内存泄漏可以说是很典型的问题了，有一个好用的工具去排查内存泄漏是比较重要的。本文主要介绍的是gperftools这个工具。

gperftools这个工具比较轻量级，对源码是无侵入的，只需要重新链接即可。

接下来按照安装，以及使用两部分展开，最后再补充一个使用gperftools做性能分析的例子。

---

# 安装

ubuntu使用下面这个脚本就能直接安装了。跑完之后可以通过`ls /usr/local/lib/ | grep tcmalloc`来确认是否安装成功。

```sh
#!/bin/bash

SCRIPT=`realpath $0`
SCRIPTPATH=`dirname $SCRIPT`

sudo apt update
sudo apt install -y autoconf libtool ghostscript graphviz

cd /tmp

git clone https://github.com/gperftools/gperftools.git
cd gperftools
./autogen.sh
./configure CC=/usr/bin/gcc
make -j1
sudo make install
sudo ldconfig
```

---

# 内存泄漏

## demo代码

以下demo代码想表达的意思是gperftools有能力检测出没有被释放的内存（`heal_leak`函数里面有没有`delete`的指针）。从中也可以看到，gperftools查内存泄漏对源码完全是无侵入的，只需要重新链接一下就可以了。

```c++
// test_heap.cpp

#include <iostream>

using namespace std;

void heap_normal() {
    int i;
    for (i=0; i<1024*1024; ++i) {
        int* p = new int;
        delete p;
    }
}

void heap_leak() {
    int i;
    for (i=0; i<1024*1024; ++i) {
        int* p = new int;
        //delete p;
    }
}

void heap_func() {
    heap_normal();
    heap_leak();
}

int main(){
   heap_func();
   return 0;
}
```

## 测试指令

```sh
g++ test_heap.cpp -o test -ltcmalloc
HEAPPROFILE=./heap_profile ./test
/usr/local/bin/pprof --pdf test ./*.heap > heap_profile.pdf
```

## 结果

输出的`heap_profile.pdf`长这个样子，可以很清晰地看到`heap_leak`中没被释放的内存，而不包含`heap_normal`中被正常释放的内存：

<div style="width:600px; margin-left:auto; margin-right:auto;" >
  {% asset_img heap_profile.png heap_profile.pdf %}
</div>

---

# 性能分析

## demo代码

这里要注意的是代码里面加了一段dummy code，目的是使用profiler的代码，以保证正常链接。不加的话，对`libprofiler.so`的链接有可能会被优化掉，导致最后profile失败。

这个跟链接器是强相关的，在有的机子上是不需要加的，这里写出来只是以防万一。

建议是先不加，然后通过`ldd test`看是否有`libprofiler.so`，如果有的话就不需要加dummy code了。

```c++
// test_cpu.cpp

#include <gperftools/profiler.h>
#include <iostream>

using namespace std;

void cpu_func1() {
   int i = 0;
   while (i < 100000) {
       ++i;
   }
}

void cpu_func2() {
   int i = 0;
   while (i < 200000) {
       ++i;
   }
}

void cpu_func() {
   for (int i = 0; i < 1000; ++i) {
       cpu_func1();
       cpu_func2();
   }
}

void dummy_code() {
   ProfilerStart("./dummy.log");
   ProfilerStop();
}

int main(){
   cpu_func();
   return 0;
}
```

## 测试指令

```sh
g++ test_cpu.cpp -o test -lprofiler
CPUPROFILE=./cpu_profile ./test
/usr/local/bin/pprof --pdf test cpu_profile > cpu_profile.pdf
```

## 结果

输出的`cpu_profile.pdf`长这个样子：

<div style="width:600px; margin-left:auto; margin-right:auto;" >
  {% asset_img cpu_profile.png cpu_profile.pdf %}
</div>

---

# 参考

- https://xusenqi.site/2020/12/06/C++Profile%E7%9A%84%E5%A4%A7%E6%9D%80%E5%99%A8_gperftools%E7%9A%84%E4%BD%BF%E7%94%A8/