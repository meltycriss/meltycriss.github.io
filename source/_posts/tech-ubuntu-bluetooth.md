---
title: 技术总结《在Ubuntu中使用USB蓝牙适配器》
date: 2021-12-29 22:08:35
tags:
	- Ubuntu USB蓝牙适配器
categories:
	- 技术总结
description:
	- 不再担心客服说USB蓝牙适配器不支持Linux系统
---

最近买了个蓝牙键盘，但是PC不带蓝牙，于是就想着外接个USB蓝牙适配器。在京东上找了一圈，发现就没有一个USB蓝牙适配器是支持Linux的。

抱着死马当活马医的想法，还是买了一款USB蓝牙适配器回来试，发现确实没法像在Windows一样即插即用。所幸的是捣鼓一翻之后，最终还是能够正常使用。

接下来就介绍一下我碰到的问题以及对应的解决方案，主要包含以下内容：

1. 识别碰到的问题
2. 处理`rtl_bt/rtl8761b_fw.bin not found`的问题
3. 处理把`rtl8761b`识别成`rtl8761a`的问题

这里补充一句，**建议购买支持蓝牙5.0的USB蓝牙适配器**，一个原因是这篇文章用的是蓝牙5.0的适配器，另一个原因是有些设备确实需要蓝牙5.0。

## 识别碰到的问题

将USB蓝牙适配器插入PC，然后看一下你的设备能不能连上PC，我相信肯定是不行的，不然你也不会搜到这篇文章。

然后通过`dmesg | grep -i bluetooth`查看报错信息，我这边在不同版本的Ubuntu里面碰到了不同的问题，分别是：

- Ubuntu 20: `rtl_bt/rtl8761b_fw.bin not found`
- Ubuntu 18: 把`rtl8761b`识别成了`rtl8761a`

假如你碰到的也是这两个问题，那么恭喜你，很有可能这篇文章就能帮你解决问题。假如不是这两个问题的话，那good luck。

接下来就分别介绍这两个问题的解决方案。

## 处理`rtl_bt/rtl8761b_fw.bin not found`的问题

这个问题相对好处理，只是没有安装固件，把固件安装上即可：

1. 到[这个网站](https://github.com/Realtek-OpenSource/android_hardware_realtek/tree/rtk1395/bt/rtkbt/Firmware/BT)下载`rtl8761b_fw`和`rtl8761b_config`这两个文件
2. 将上述两个文件复制粘贴到`/usr/lib/firmware/rtl_bt/`这个目录下
3. 重新拔插USB蓝牙适配器

## 处理把`rtl8761b`识别成`rtl8761a`的问题

**这里有个前提是要知道真实的固件是什么，你可以通过查官网来获得这个信息。我这里是因为我在Ubuntu 20下用`rtl8761b`是work的，所以确定该USB蓝牙适配器的固件是`rtl8761b`。**

这个问题相对麻烦，是由于系统蓝牙模块的驱动有问题，因此需要重新编译、安装系统蓝牙模块的驱动：

1. 查看当前系统的内核版本：`uname -r`
2. 下载对应版本的内核源码：到[这个网站](https://github.com/torvalds/linux/tree/master)通过切换`tag`来下载对应版本的内核源码
3. 按照[这个网站](https://discourse.coreelec.org/t/rtl8761b-bluetooth-driver-fix/15286)去修改蓝牙模块的驱动代码
4. 按照[这个网站](https://gist.github.com/rometsch/dfd24fb09c85c1ad2f25223dc1481aaa#patch-the-bluetooth-kernel-module)去编译、安装新的蓝牙模块驱动
5. 重启电脑

## References

- https://linuxreviews.org/Realtek_RTL8761B
- https://blog.csdn.net/avatar1912/article/details/120851889