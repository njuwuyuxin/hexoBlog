---
title: libiconv 未定义的引用解决
date: 2020-06-25 12:33:44
categories: 踩坑整理
tags:
- Makefile
- 常见问题
- c++
---
----
最近在安装libconfig库时，编译期间出现找不到libiconv库的问题
`/usr/local/lib/../lib64/libstdc++.so: undefined reference to "libiconv" `
在仔细检查重新安装了libiconv库之后，问题依然无法解决。因此根据make时的记录进行追溯，发现链接错误出现在/example/c++样例编译期间
```
make[3]: 离开目录“/home/downloads/libconfig-1.7.2/examples/c”
Making all in c++
make[3]: 进入目录“/home/downloads/libconfig-1.7.2/examples/c++”
  CXX      example1.o
  CXXLD    example1
/usr/local/lib/../lib64/libstdc++.so: undefined reference to `libiconv'
/usr/local/lib/../lib64/libstdc++.so: undefined reference to `libiconv_close'
/usr/local/lib/../lib64/libstdc++.so: undefined reference to `libiconv_open'
collect2: error: ld returned 1 exit status
```
手动去查看/example/c++/下的Makefile文件时发现
```
LIBOBJS =
LIBS = 
LIBTOOL = $(SHELL) $(top_builddir)/libtool
LIPO =
```
其中LIBS变量存放编译时所需引用的外部库，而这里默认置空，我的环境使用的是CentOS7，而这里需要手动添加libiconv库的引用，因此将其改为
```
LIBOBJS =
LIBS = -liconv
LIBTOOL = $(SHELL) $(top_builddir)/libtool
LIPO =
```
手动指定链接libiconv库。之后重新执行make，成功编译并链接。

### 后记
类似库找不到定义的问题原因可能有很多，但大致都可以按照一下思路进行解决
- 首先判断库是否确实未安装（最简单的情况，安装该库即可）
- 如果该库已安装，查看其所在位置是否加入到系统搜索范围内
- 如果以上均无问题，可能需要检查Makefile或其他编译选项，尝试手动链接该动态链接库