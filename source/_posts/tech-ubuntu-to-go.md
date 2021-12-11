---
title: 技术总结《Ubuntu to Go》
date: 2021-12-11 10:11:00
tags:
	- Ubuntu to Go
categories:
	- 技术总结
description:
	- 将Ubuntu安装到U盘中，到哪都可以使用自己的电脑
---

带自己的笔记本电脑太麻烦，用别人的电脑不顺手。如果你也有这种困扰的话，可以尝试一下*Ubuntu to Go*。

*Ubuntu to Go*是仿照*Windows to Go*起的名字，干的事情就是将操作系统安装到U盘中。这样一来，你只需要携带一个U盘，就可以在不同的电脑上运行你自己的操作系统了。

本文不会详细罗列所有的步骤，只会介绍区别于常规安装*Ubuntu*的地方，换句话说就是假设你已经知道常规安装*Ubuntu*的方法。

接下来会：

1. 首先介绍安装*Ubuntu to Go*和安装普通的*Ubuntu*区别在哪
2. 然后介绍具体操作

## 区别在哪

直觉上来讲，将*Ubuntu*安装到U盘中是很简单的事情，只需要在安装的时候将Boot Loader的安装路径设置为U盘就可以了。

问题就出在了*Ubuntu*的安装程序有Bug（也不知道是不是个Feature），不管你怎么设置安装路径，它一定会将Boot Loader安装在第一个EFI分区里面。往往第一个EFI分区都在我们原来的硬盘，而不在我们的U盘（而且一般来说，U盘中也不包含EFI分区）。Boot Loader不在U盘就意味着没办法通过U盘启动*Ubuntu*，就达不成我们的目标。

因此，相比于一般的安装流程，我们还需要额外做两件事情：

1. 安装时为U盘创建EFI分区
2. 安装后将Boot Loader安装到所创建的EFI分区中

## 为U盘创建EFI分区

在设置*Ubuntu*分区的时候，多加一个格式为FAT32、大小为100MB的分区，并且将这个分区设置为EFI。

## 将Boot Loader安装到所创建的EFI分区中

按照Installer提示，完成常规的安装之后，打开一个终端，去完成以下操作：

1. 查看第一个EFI分区以及U盘中的EFI分区在哪，在这里是`/dev/sda2`和`/dev/sdb1`
```sh
sudo fdisk -l

Device         Start       End   Sectors   Size Type
/dev/sda1       2048    923647    921600   450M Windows recovery environment
/dev/sda2     923648   1126399    202752    99M EFI System
/dev/sda3    1126400   1159167     32768    16M Microsoft reserved
/dev/sda4    1159168 248347899 247188732 117.9G Microsoft basic data
/dev/sda5  248348672 250066943   1718272   839M Windows recovery environment

...

Device       Start        End    Sectors   Size Type
/dev/sdb1     2048     194559     192512    94M EFI System
/dev/sdb2   194560    8194047    7999488   3.8G Linux swap
/dev/sdb3  8194048 1953457719 1945263672 927.6G Linux filesystem

```
2. 将Boot Loader安装到U盘中的EFI
```sh
# 挂载EFI
mkdir windows_epi ubuntu_epi
sudo mount /dev/sda2 windows_epi
sudo mount /dev/sdb1 ubuntu_epi
# 安装
mkdir ubuntu_epi/EFI
cp -r windows_epi/EFI/Boot ubuntu_epi/EFI
mv windows_epi/EFI/ubuntu ubuntu_epi/EFI
sudo sync; umount windows_epi; umount ubuntu_epi; sync
```

## 收尾

至此应该已经将*Ubuntu*安装到U盘中了，接下来只需要重启，然后在UEFI中选择一下Boot的优先级就可以从U盘中启动*Ubuntu*了。

## References

- [1] [Legacy / UEFI / MBR / GPT的关系](https://zhuanlan.zhihu.com/p/262069479)
- [2] [EFI / Boot Loader / GRUB的关系](https://blog.csdn.net/u011433762/article/details/49934253)
- [3] [迁移Boot Loader到U盘](https://meaningofstuff.blogspot.com/2019/09/linux-ubuntu-1904-full-install-on-usb.html)
- [4] [删除Windows中的Grub](https://askubuntu.com/questions/429610/uninstall-grub-and-use-windows-bootloader/869888#869888)
