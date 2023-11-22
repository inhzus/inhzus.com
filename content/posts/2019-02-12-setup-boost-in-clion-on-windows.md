---
title: Windows MinGW 安装 Boost 与 CLion 的配置
date: 2019-02-12 13:36:06
tags: [C++, Boost, Windows, Configuration]
description: "在 Windows 下使用 MinGW gcc 安装 boost，配置 CMake 依赖"
---



关于 boost 在 Windows 下的使用 gcc 安装与 CLion 的配置, 能够查到的英文资料都比较少, 踩过坑后记录一下.

### MinGW 安装 Boost

[Boost Download](https://www.boost.org/users/download/)

下载并解压 Boost 文件夹到一个**稳定**的文件夹, 此时我的文件名为 boost_1_69_0.

在进行以下几步之前请先把 gcc 添加至环境变量.

在该文件夹目录下打开命令行, 首先执行:

```bash
bootstrap gcc
```

如果你只安装指定的少数几个库, 可以使用:

```bash
b2 --show-libraries
```

得到你可以在这里单独安装的所有库的名称. 然后你可以安装指定的某个库, 如 program_options 和 filesystem:

```bash
b2 toolset=gcc --with-program_options --with-filesystem
```

或者你想一次性编译完毕, 以后就无需再进行编译:

```bash
b2 toolset=gcc
```

*以上的指令复制至 cmd 可直接执行*

全部编译完成的时间视处理器性能而定, 大概在十分钟左右.

此时你可以在 boost 根目录下的 `stage/lib` 文件夹下看到以 a 为文件扩展名的静态库.

### CLion CMakeLists 配置 Boost

网上搜索到的教程容易导致以下几个误区
<!-- more -->

- 认为链接库的文件夹目录在 bootstrap_1_69_0/libs 下
- 无法找到原因为什么无法成功 find_package

在 Boost 的配置中, 需要指明 BOOST_ROOT, BOOST_INCLUDEDIR, BOOST_LIBRARYDIR 这三个变量. 我的 boost 文件夹放在 `C:/Local` :

```cmake
set(Boost_DEBUG on)
set(Boost_DETAILED_FAILURE_MSG ON)
#查看 Boost 配置问题所在
set(BOOST_ROOT C:/Local/Boost_1_69_0)
set(BOOST_INCLUDEDIR ${BOOST_ROOT})
set(BOOST_LIBRARYDIR ${BOOST_ROOT}/stage/lib)
find_package(Boost COMPONENTS REQUIRED program_options)
```

提示 find_package 失败.

查看 debug 信息:

```
-- [ .../FindBoost.cmake:1809 ] Searching for PROGRAM_OPTIONS_LIBRARY_RELEASE: boost_program_options-mgw51-mt-1_69;boost_program_options-mgw51-mt;boost_program_options-mt-1_69;boost_program_options-mt;boost_program_options-mt;boost_program_options
-- [ .../FindBoost.cmake:1862 ] Searching for PROGRAM_OPTIONS_LIBRARY_DEBUG: boost_program_options-mgw51-mt-d-1_69;boost_program_options-mgw51-mt-d;boost_program_options-mt-d-1_69;boost_program_options-mt-d;boost_program_options-mt;boost_program_options
```

此时对照 `stage/lib` 文件夹下的文件名, 发现名称为`libboost_program_options-mgw51-mt-d-x64-1_69.a` 或`libboost_program_options-mgw51-mt-d-x32-1_69.a` 

因此, 只需将 x64 文件中的文件名删掉 "-x64" 即可.

故在 `stage` 文件夹下新建一个 python3 重命名文件脚本:

```python
# -*- coding: utf-8 -*-
# renamer.py

from os import listdir, rename
import re

for filename in listdir('lib'):
    new_filename = re.sub('libboost(.*)-x64(.*).a', r'libboost\1\2.a', filename)
    rename('lib/' + filename, 'lib/' + new_filename)

```

执行后再次进行 cmake, 就可以成功 find_package

此时按照网上教程, 完整的 CMakeLists.txt 为:

```cmake
cmake_minimum_required(VERSION 3.13)
project(dot)

set(CMAKE_CXX_STANDARD 14)

set(Boost_DETAILED_FAILURE_MSG ON)
set(Boost_DEBUG on)
set(BOOST_ROOT C:/Local/Boost_1_69_0)
set(BOOST_INCLUDEDIR ${BOOST_ROOT})
set(BOOST_LIBRARYDIR ${BOOST_ROOT}/stage/lib)
find_package(Boost COMPONENTS REQUIRED filesystem program_options)

add_executable(dot main.cpp command.h)
include_directories(${BOOST_ROOT})
target_link_libraries(dot ${Boost_LIBRARIES})
#Boost_LIBRARIES 为 find_package 自动生成的变量

```

CLion 配置 Boost 完成.

