---
title: 技术总结《在Win11上构建UE5版本的Carla》
date: 2024-08-04 23:47:02
tags:
	- CarlaUE5
categories:
	- 技术总结
description:
	- 记录在Win11上构建CarlaUE5所踩的坑
---

Carla是一个开源的自动驾驶仿真平台，其[最新版本](https://github.com/carla-simulator/carla/tree/ue5-dev)基于UE5开发，画面十分的逼真。

本文主要记录在Win11上通过源码构建CarlaUE5时所碰到的问题以及解决方案，整理流程参考[官方教程](https://carla.readthedocs.io/en/latest/build_windows_ue5/)。

在构建的过程中，主要碰到了以下问题：
  
- 本机已有的VS2022专业版，如何跳过默认的VS2022社区版安装流程，并把依赖安装到原有的VS2022专业版上；
- 构建过程中报了一个跟编码格式相关的错；
- 构建过程中报了一个跟路径长度相关的错。

接下来分别介绍各个问题以及对应的解决方案。

---

# 使用已有的VS2022专业版

## 修改`C:\Carla\CarlaUE5\Setup.bat`

- 注释掉安装VS2022社区版相关的代码
- 将类似`C:\Program Files\Microsoft Visual Studio\2022\Community`的字段换成已有的VS2022专业版路径

## 安装依赖

如下所示，`C:\Carla\CarlaUE5\Setup.bat`在安装VS2022社区版的时候，会顺带安装上一些依赖。

```bat
vs_Community.exe --add Microsoft.VisualStudio.Workload.NativeDesktop Microsoft.VisualStudio.Workload.NativeGame Microsoft.VisualStudio.Workload.ManagedDesktop Microsoft.VisualStudio.Component.Windows10SDK.18362  Microsoft.VisualStudio.Component.VC.CMake.Project Microsoft.Net.ComponentGroup.4.8.1.DeveloperTools Microsoft.VisualStudio.Component.VC.Llvm.Clang Microsoft.VisualStudio.Component.VC.Llvm.ClangToolset Microsoft.VisualStudio.ComponentGroup.NativeDesktop.Llvm.Clang --removeProductLang Es-es --addProductLang En-us --installWhileDownloading --passive --wait
```

没有这些依赖的话，在构建CarlaUE5的时候会报错`MSB3073`，因此我们需要手动把这些依赖装到已有的VS2022专业版上：

- 手动安装依赖的方法，可以参考[这个网站](https://blog.csdn.net/weixin_42205218/article/details/107829229)
- 依赖名字到组件名字的映射，可以参考[这个网站](https://learn.microsoft.com/zh-cn/visualstudio/install/workload-component-id-vs-professional?view=vs-2022)

---

# 编码格式报错

## 问题

构建CarlaUE5时，会报以下错误：

```
[33/413] Building C object CMakeFiles\sqlite3.dir\_deps\sqlite3-src\shell.c.obj
FAILED: CMakeFiles/sqlite3.dir/_deps/sqlite3-src/shell.c.obj
C:\PROGRA~1\MICROS~4\2022\PROFES~1\VC\Tools\MSVC\1436~1.325\bin\Hostx64\x64\cl.exe  /nologo -D_CRT_SECURE_NO_WARNINGS -IC:\Carla\CarlaUE5\Build\_deps\zlib-src -IC:\Carla\CarlaUE5\Build\_deps\zlib-build -IC:\Carla\CarlaUE5\Build\_deps\libpng-src -IC:\Carla\CarlaUE5\Build\_deps\libpng-build /DWIN32 /D_WINDOWS /O2 /Ob2 /DNDEBUG -std:c11 -MD /wd4005 /showIncludes /FoCMakeFiles\sqlite3.dir\_deps\sqlite3-src\shell.c.obj /FdCMakeFiles\sqlite3.dir\ /FS -c C:\Carla\CarlaUE5\Build\_deps\sqlite3-src\shell.c
C:\Carla\CarlaUE5\Build\_deps\sqlite3-src\shell.c(1): warning C4819: 该文件包含不能在当前代码页(936)中表示的字符。请将该文件保存为 Unicode 格式以防止数据丢失
C:\Carla\CarlaUE5\Build\_deps\sqlite3-src\shell.c(26303): warning C4819: 该文件包含不能在当前代码页(936)中表示的字符。请将该文件保存为 Unicode 格式以防止数据丢失
C:\Carla\CarlaUE5\Build\_deps\sqlite3-src\shell.c(26338): error C2001: 常量中有换行符
C:\Carla\CarlaUE5\Build\_deps\sqlite3-src\shell.c(26339): error C2143: 语法错误: 缺少“;”(在“const”的前面)
C:\Carla\CarlaUE5\Build\_deps\sqlite3-src\shell.c(26353): error C2065: “zBom”: 未声明的标识符
C:\Carla\CarlaUE5\Build\_deps\sqlite3-src\shell.c(26353): warning C4047: “=”:“int”与“const char *”的间接级别不同
C:\Carla\CarlaUE5\Build\_deps\sqlite3-src\shell.c(26420): error C2065: “zBom”: 未声明的标识符
C:\Carla\CarlaUE5\Build\_deps\sqlite3-src\shell.c(26420): warning C4047: “函数”:“const char *”与“int”的间接级别不同
C:\Carla\CarlaUE5\Build\_deps\sqlite3-src\shell.c(26420): warning C4024: “oPutsUtf8”: 形参和实参 1 的类型不同
C:\Carla\CarlaUE5\Build\_deps\sqlite3-src\shell.c(26433): error C2065: “zBom”: 未声明的标识符
C:\Carla\CarlaUE5\Build\_deps\sqlite3-src\shell.c(26433): warning C4047: “函数”:“const char *”与“int”的间接级别不同
C:\Carla\CarlaUE5\Build\_deps\sqlite3-src\shell.c(26433): warning C4024: “oPutsUtf8”: 形参和实参 1 的类型不同
```

## 解决

按[这个PR](https://github.com/carla-simulator/carla/pull/8033)的方法，在`CMake/Common.cmake`中增加一行`add_compile_options (/utf-8)`即可解决问题。

修改之后重新构建，可以在`C:\Carla\CarlaUE5\Build\build.ninja`中看到编译选项中已经加上了`/utf-8`

```
build CMakeFiles\sqlite3.dir\_deps\sqlite3-src\shell.c.obj: C_COMPILER__sqlite3_unscanned_Release C$:\Carla\CarlaUE5\Build\_deps\sqlite3-src\shell.c || cmake_object_order_depends_target_sqlite3
  DEFINES = -D_CRT_SECURE_NO_WARNINGS
  FLAGS = /DWIN32 /D_WINDOWS /O2 /Ob2 /DNDEBUG -std:c11 -MD /utf-8 /wd4005
  INCLUDES = -IC:\Carla\CarlaUE5\Build\_deps\zlib-src -IC:\Carla\CarlaUE5\Build\_deps\zlib-build -IC:\Carla\CarlaUE5\Build\_deps\libpng-src -IC:\Carla\CarlaUE5\Build\_deps\libpng-build
  OBJECT_DIR = CMakeFiles\sqlite3.dir
  OBJECT_FILE_DIR = CMakeFiles\sqlite3.dir\_deps\sqlite3-src
  TARGET_COMPILE_PDB = CMakeFiles\sqlite3.dir\
  TARGET_PDB = sqlite3.pdb
```

---

# 路径过长报错

## 问题

构建的时候会陷入死循环，一直卡在其中一步，报错的提示指向文件路径名太长。

## 解决

参考[这个网站](https://blog.csdn.net/weixin_46356818/article/details/121029550)，将Windows对长路径的支持打开即可。


<!-- https://github.com/carla-simulator/carla/pull/8033

C:\Carla\CarlaUE5\Build\build.ninja

build CMakeFiles\sqlite3.dir\_deps\sqlite3-src\shell.c.obj: C_COMPILER__sqlite3_unscanned_Release C$:\Carla\CarlaUE5\Build\_deps\sqlite3-src\shell.c || cmake_object_order_depends_target_sqlite3
  DEFINES = -D_CRT_SECURE_NO_WARNINGS
  FLAGS = /DWIN32 /D_WINDOWS /O2 /Ob2 /DNDEBUG -std:c11 -MD /utf-8 /wd4005
  INCLUDES = -IC:\Carla\CarlaUE5\Build\_deps\zlib-src -IC:\Carla\CarlaUE5\Build\_deps\zlib-build -IC:\Carla\CarlaUE5\Build\_deps\libpng-src -IC:\Carla\CarlaUE5\Build\_deps\libpng-build
  OBJECT_DIR = CMakeFiles\sqlite3.dir
  OBJECT_FILE_DIR = CMakeFiles\sqlite3.dir\_deps\sqlite3-src
  TARGET_COMPILE_PDB = CMakeFiles\sqlite3.dir\
  TARGET_PDB = sqlite3.pdb


build CMakeFiles\sqlite3.dir\_deps\sqlite3-src\shell.c.obj: C_COMPILER__sqlite3_unscanned_Release C$:\Carla\CarlaUE5\Build\_deps\sqlite3-src\shell.c || cmake_object_order_depends_target_sqlite3
  DEFINES = -D_CRT_SECURE_NO_WARNINGS
  FLAGS = /DWIN32 /D_WINDOWS /O2 /Ob2 /DNDEBUG -std:c11 -MD /wd4005
  INCLUDES = -IC:\Carla\CarlaUE5\Build\_deps\zlib-src -IC:\Carla\CarlaUE5\Build\_deps\zlib-build -IC:\Carla\CarlaUE5\Build\_deps\libpng-src -IC:\Carla\CarlaUE5\Build\_deps\libpng-build
  OBJECT_DIR = CMakeFiles\sqlite3.dir
  OBJECT_FILE_DIR = CMakeFiles\sqlite3.dir\_deps\sqlite3-src
  TARGET_COMPILE_PDB = CMakeFiles\sqlite3.dir\
  TARGET_PDB = sqlite3.pdb

https://www.cnblogs.com/mechanicoder/p/16894144.html

MSB3073

https://learn.microsoft.com/zh-cn/visualstudio/install/workload-component-id-vs-professional?view=vs-2022

@REM echo Installing Visual Studio 2022...
@REM curl -L -O https://aka.ms/vs/17/release/vs_community.exe || exit /b
@REM vs_Community.exe --add Microsoft.VisualStudio.Workload.NativeDesktop Microsoft.VisualStudio.Workload.NativeGame Microsoft.VisualStudio.Workload.ManagedDesktop Microsoft.VisualStudio.Component.Windows10SDK.18362  Microsoft.VisualStudio.Component.VC.CMake.Project Microsoft.Net.ComponentGroup.4.8.1.DeveloperTools Microsoft.VisualStudio.Component.VC.Llvm.Clang Microsoft.VisualStudio.Component.VC.Llvm.ClangToolset Microsoft.VisualStudio.ComponentGroup.NativeDesktop.Llvm.Clang --removeProductLang Es-es --addProductLang En-us --installWhileDownloading --passive --wait
@REM del vs_community.exe
@REM curl -L -O https://aka.ms/vs/17/release/vs_professional.exe || exit /b
@REM vs_Professional.exe --add Microsoft.VisualStudio.Workload.NativeDesktop Microsoft.VisualStudio.Workload.NativeGame Microsoft.VisualStudio.Workload.ManagedDesktop Microsoft.VisualStudio.Component.Windows10SDK.18362  Microsoft.VisualStudio.Component.VC.CMake.Project Microsoft.Net.ComponentGroup.4.8.1.DeveloperTools Microsoft.VisualStudio.Component.VC.Llvm.Clang Microsoft.VisualStudio.Component.VC.Llvm.ClangToolset Microsoft.VisualStudio.ComponentGroup.NativeDesktop.Llvm.Clang --removeProductLang Es-es --addProductLang En-us --installWhileDownloading --passive --wait
@REM del vs_professional.exe
@REM echo Visual Studion 2022 Installed!!!

C:\Carla\CarlaUE5\Setup.bat

https://blog.csdn.net/weixin_46356818/article/details/121029550 -->