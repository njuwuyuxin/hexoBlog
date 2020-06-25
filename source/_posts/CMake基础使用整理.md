---
title: CMake基础使用整理
date: 2020-06-24 20:55:12
categories: 工具学习
tags:
- C++
- CMake
- 构建工具
---
----
CMake是一个跨平台的编译工具，可以一次编写，在不同平台自动生成对应的Makefile文件，减少了手写Makefile以及适配不同平台时的耗时。
## 前言
之前大部分时候在windows端使用VS开发，因此对Makefile、CMake等工具接触较少。最近尝试从头实现一个简单的HTTP服务器，主要开发环境在Linux，因此借此契机熟悉一下CMake等构建工具的使用。
## 目录结构
目前项目文件较少，使用了较简单的目录结构
```
┣━ src
┃   ┣━ CMakeLists.txt
┃   ┣━ HttpRequest.cpp
┃   ┣━ HttpResponse.cpp
┃   ┣━ HttpServer.cpp
┃   ...
┣━ include
┃   ┣━ HttpRequest.h
┃   ┣━ HttpResponse.h
┃   ┣━ HttpServer.h
┃   ...
┣━ cmake-build-debug
┃   ┣━...
┃   ┗━...
┣━ main.cpp
┣━ CMakeLists.txt
```
可以看到源文件和头文件分别存储在对应目录中，根目录下以main.cpp作为程序入口，最终构建目标及中间文件存放在cmake-build-debug这一独立文件夹中。

## CMakeLists编写
为了使上述目录结构能够正确编译链接，我们需要编写CMakeLists.txt，同时CMake能够一定程度上减少多文件多目录时来回链接顺序等头疼的问题。在这个项目里，根目录下和src目录下各有一个CMakeLists.txt文件，这也是CMake的特点，可以将Makefile拆分，每个目录各自进行编译，最终链接起来。

**根目录下的CMakeLists.txt内容如下**
```
cmake_minimum_required (VERSION 2.6)

add_definitions(-std=c++11)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -g -Wall -Wno-unused-variable -pthread")

project (Minihttpd)

include_directories(include)
add_subdirectory(src)

# 顺序不可修改，先link_directories 再 add_executable 最后 target_link_libraries
link_directories(/usr/local/lib)

add_executable(Minihttpd main.cpp)

target_link_libraries(Minihttpd src)
target_link_libraries(Minihttpd -lconfig++)
```
接下来对每条语句进行简单的解释
- add_definations 指令可以用来手动设置某些宏的开闭，控制编译选项，这里主要是标注使用c++11标准
- set指令能够用前面的变量替代后面的字符串，这里实际上是对预定义的CMAKE_CXX_FLAGS变量进行了一个修改，来设置某些选项，主要链接pthread库使用多线程
- project指令用来设置项目名（包括版本、所用语言等信息，这里缺省）
- include_directories指令用来指定寻找头文件的路径，这里把include目录加入到头文件搜索范围内，使得项目内文件可以找到对应头文件
- add_subdirectory指令用来把子目录加入到构建列表中，最终构建结果存放在src变量中

之后的几行是在项目需要引用其他动态链接库，非常需要注意的地方
- add_executable指令用来指定项目最终构建的目标文件，以及所需要的所有源文件。可以看到这里只有main.cpp，为什么没有包含src目录下其他源文件？这里其实在下面使用 target_link_libraries 指令，以动态链接库的形式引入进来。其顺序是在子目录中首先进行了部分构建，在src目录下生成了相应的libsrc.a文件，最终链接到程序入口文件上，实现了构建。
- link_directories和target_link_libraries指令用来引入外部的动态链接库。其中
  - link_directories用来指定该动态链接库所在目录
  - target_link_libraries用来把所需的动态链接库引入到该项目中

**这里非常需要注意的是几条语句的顺序，一定是**
1. link_directories 把动态链接库所在目录加入寻找列表中
2. add_executable 指定最终构建目标名称
3. target_link_libraries 把需要的动态链接库加入到项目中

**这里的顺序错误将导致链接失败，出现找不到动态链接库等各种问题（踩过的坑，心酸的泪）**

**子目录下的CMakeLists.txt内容如下**
```
aux_source_directory(. srcs)
add_library(src ${srcs})
```
这里的内容就非常简单
- aux_source_directory指令把当前目录下所有源文件加入到srcs变量中存储
- add_library使用srcs变量中所有源文件进行构建，结果输出为src。不同于add_executable，add_library的构建的最终目标为动态链接库文件，而add_executable的构建结果为一个可执行文件。这里构建成动态链接库文件也是为了在根目录下构建时进行链接

## 扩展
在理解了多目录下CMakeLists的编写后，如果需要把源文件存放在多个不同目录中，也可以以动态链接库的形式分别进行构建、链接。而对于多级目录，也可以依次逐级构建并链接。

对于需要引用的外部动态链接库，也可以通过link_directories和target_link_libraries指令的配合进行引入。

同时本项目内使用了ninja作为构建工具，更方便了项目的构建，主要使用方法为
```
cd cmake-build-debug
cmake -G Ninja ..     //cmake支持根据CMakeLists.txt自动化生成ninja构建所需要的ninja.build等文件
ninja                 //在该目录下构建
```
