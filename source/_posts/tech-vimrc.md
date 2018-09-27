---
title: 技术总结《配置Vim》
date: 2018-09-27 11:15:27
tags:
	- Vim
categories:
	- 技术总结
description:
	- 关于配置Vim的一些技术总结
---

本文主要介绍我自己的Vim配置（Ubuntu 16.04，主要用于写Python和C++）以及管理方案（如何将Vim配置快速部署到新机子上）。接下来首先介绍一款**强大的Vim配置**，然后介绍如何在该Vim配置的基础上**添加更多插件**，最后给出一个Vim配置**管理方案**。

## The ultimate Vim configuration

### 核心框架

[The ultimate Vim configuration](https://github.com/amix/vimrc)是一份强大的Vim配置，它使用了Pathogen作为插件管理器（关于Vim插件管理器的发展可以看[这里](https://www.zhihu.com/question/24294358)），安装插件只需要**把插件放到对应目录**就好了（一般来说就是`git clone`）。按照[指引](https://github.com/amix/vimrc#install-for-your-own-user-only)安装完后，这份Vim配置的**核心目录结构**如下：

```sh
.vim_runtime
│
# default plugins & configurations
├── sources_forked
├── sources_non_forked
├── vimrcs
│   ├── basic.vim # general configurations
│   └── plugins_config.vim # plugin configurations
│
# custom plugins & configurations
├── my_plugins
├── my_configs.vim # may not exist
│
# installer
├── install_awesome_vimrc.sh
```

从以上目录结构可见，**核心框架**包含以下**三部分**：

- 自带插件
	- `sources_forked`和`sources_non_forked`中包含了自带的插件；
	- `vimrcs`中的
		- `basic.vim`包含插件无关的通用配置（e.g., 一个tab等于4个空格）；
		- `plugins_config.vim`中包含了针对自带插件的配置。
- 自定义插件
	- `my_plugins`是安装自定义插件的地方；
	- `my_configs.vim`则用于编写针对自定义插件的配置和插件无关的通用配置（值得注意的是，因为`my_configs.vim`在最后的`~/.vimrc`中是最后一个被引用的，因此假如它的配置与其他`.vim`有冲突的话，以`my_configs.vim`中的配置为准）。
- 安装脚本
	- `install_awesome_vimrc.sh`包含修改`~/.vimrc`的代码。

### 使用方法

本节主要介绍**自带插件的使用方法**。

- 打开/关闭文件
	- `,nn`：在Vim中打开目录树；
	- `,f`：查看最近使用过的文件；
	- `:W`：以sudo权限保存文件；
	- `gf`：当光标在一个路径上时，打开该文件。
- 编辑
	- `gcc`：注释选中的行；
	- `$123 / $q / $e`：为选中的内容添加不同的括号/引号；
	- `Ctrl+s / Ctrl+x / Ctrl+p`：多光标（同时选中多个相同的内容，重构的时候很有用）；
		- 要实现这个功能，需要进行一些修改，参考[这里](https://github.com/amix/vimrc/issues/340)。简单来说就是要在`~/.bashrc`中加上`stty -ixon`。
	- `+ / _`：在逻辑意义上扩展/收缩选中的内容（e.g., 变量，函数，类）；
		- 要实现这个功能，需要再安装一些插件来告诉Vim怎么在逻辑意义上划分内容。以Python为例（参考[这里](https://github.com/terryma/vim-expand-region/issues/15)），大概分为以下两步：
			1. 将[vim-textobj-user](https://github.com/kana/vim-textobj-user)和[vim-textobj-python](https://github.com/bps/vim-textobj-python)下载到`my_plugins`；
			2. 在`my_configs.vim`中加上以下代码。

```vim
call expand_region#custom_text_objects('python', {
      \ 'af' :1,
      \ 'if' :1,
      \ 'ac' :1,
      \ 'ic' :1,
      \ })
```

## 更多插件

本节介绍如何在[The ultimate Vim configuration](https://github.com/amix/vimrc)的基础上，安装更多的插件，使Vim用起来更顺手。


- [SimpylFold](https://github.com/tmhedberg/SimpylFold)
	- 功能：代码折叠插件，方便以不同层级去阅读代码；
	- 安装：下载到`my_plugins`文件夹中；
	- 使用：`za / zm / zr`。
- [YouCompleteMe](https://github.com/Valloric/YouCompleteMe)
	- 功能：代码补全插件，同时还提供跳转功能（变量/函数的定义），能够大幅提高阅读代码的速度。
	- 安装：跟[指引](https://github.com/Valloric/YouCompleteMe#linux-64-bit)，用`install.py`来装；
	- 使用（下面仅针对Python，C++还要进一步配置来告诉这个插件编译的flag是什么，参考[这里](https://github.com/Valloric/YouCompleteMe#c-family-semantic-completion)）
		- 自动补全装好就能用；
		- 代码跳转需要设置一下快捷键，参考[这里](https://github.com/Valloric/YouCompleteMe#ycmcompleter-subcommands)。简单来说，就是将`nnoremap <leader>jd :YcmCompleter GoTo<CR>`加到`my_configs.vim`中，之后就可以通过`,jd`进行跳转了（跳转后可以用`Ctrl+o`返回）。

## 管理方案

用Vim的一个很大的motivation是通用性，也就是说在不同的机子上都能够用同一套顺手的IDE，这就对快速部署提出了要求。

下面介绍我自己的方案，大体来说分为以下步骤：

1. 将[The ultimate Vim configuration](https://github.com/amix/vimrc)`fork`到自己的Github上；
2. 通过`git submodule add xxx`来安装插件；
	- 之所以不用`git clone`是预防嵌套Git repository无法`git push`的问题。
3. 每次更新完插件/配置后`git push`到自己的Github；
4. 在一台新的机子上，用`git clone --recursive`从自己的Github上下载配置文件
	- `--recursive`表明要把submodule都下载下来
5. 运行`install_awesome_vimrc.sh`安装Vim配置

## References

- [Vim配置文件语法cheatsheet](https://devhints.io/vimscript)
- [可用的Vim快捷键](https://vi.stackexchange.com/questions/8856/mapping-ctrl-with-equal-sign)
- [Vim中编译、运行C++代码](https://www.quora.com/How-can-I-compile-and-execute-a-c++-program-directly-from-Vim-in-Ubuntu)
- [Git submodule](https://blog.github.com/2016-02-01-working-with-submodules/)