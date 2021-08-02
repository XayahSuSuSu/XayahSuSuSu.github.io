---
title: Python3.9交叉编译到Android平台
date: 2021-08-01 15:56:29
tags: [ 'Python', '交叉编译', 'Android' ]
---

# 前言

手机上也能运行**Python**？当然，我们只需要将其**交叉编译**到**Android平台**就可以了！

本次笔记将使用NDK的交叉编译链，为**AArch64**平台交叉编译**Python3.9**。

> Android NDK 版本：r21e
>
> 编译环境：Ubuntu 21.04

# 步骤

### 一、下载交叉编译所需源码

[Python3.9](https://www.python.org/downloads/release/python-396/)

### 二、交叉编译Python3.9

解压**Python3.9的源码**

然后在**根目录**输入以下指令：

```
export NDK=/home/xayah/NDK                              # NDK根目录绝对路径

export TOOLCHAIN=$NDK/toolchains/llvm/prebuilt/linux-x86_64     # 交叉编译链路径

export TARGET=aarch64-linux-android                             # 交叉编译目标

export API=24                                                   # 最小目标SDK版本配置(24 即为 Android 7.0)

export AR=$TOOLCHAIN/bin/llvm-ar

export CC=$TOOLCHAIN/bin/$TARGET$API-clang

export AS=$CC

export CXX=$TOOLCHAIN/bin/$TARGET$API-clang++

export LD=$TOOLCHAIN/bin/ld

export RANLIB=$TOOLCHAIN/bin/llvm-ranlib

export STRIP=$TOOLCHAIN/bin/llvm-strip

./configure --host=$TARGET --build=aarch64 --disable-ipv6 ac_cv_file__dev_ptmx=no ac_cv_file__dev_ptc=no --prefix=/home/xayah/py/install
```

然而，我们遇到了一个**错误**：

```
configure: error: readelf for the host is required for cross builds
```

看来是**工具链**没有配置完整，还差一个**readelf**，NDK的**交叉编译链**里已经**集成**了它，我们只需要加上去即可。

```
export READELF=$TOOLCHAIN/bin/readelf
```

再配置一次

```
./configure --host=$TARGET --build=aarch64 --disable-ipv6 ac_cv_file__dev_ptmx=no ac_cv_file__dev_ptc=no --prefix=/home/xayah/py/install
```

这次配置**成功**了！

让我们进行接下来的**编译**吧：

```
make -j128 && make install -j128
```

稍加等待，编译**成功**！

```
xayah@xayah-virtual-machine:~/py/install$ pwd
/home/xayah/py/install
xayah@xayah-virtual-machine:~/py/install$ ls
bin  include  lib  share
xayah@xayah-virtual-machine:~/py/install$ cd bin
xayah@xayah-virtual-machine:~/py/install/bin$ ls
2to3      idle3    pydoc3    python3    python3.9-config
2to3-3.9  idle3.9  pydoc3.9  python3.9  python3-config
```

### 在Android平台上测试
首先将**编译产物打包**再将其**推送**到**Android平台**中

```
adb push py.zip /data/local/tmp/
```

由于**Android**对**权限**的**限制**，我们不能推送到任意目录，但可以将其推送到`/data/local/tmp/`目录，因为它具有**完整的文件操作权限**。

```
xayah@xayah-virtual-machine:~/py/install$ adb push py.zip /data/local/tmp/
py.zip: 1 file pushed, 0 skipped. 37.2 MB/s (56531714 bytes in 1.450s)
```

**推送**成功，接下来我们进入`adb shell`操作

```
xayah@xayah-virtual-machine:~/py/install$ adb shell
cas:/ $ cd /data/local/tmp
cas:/data/local/tmp $ ls
py.zip
cas:/data/local/tmp $ unzip py.zip > /dev/null
cas:/data/local/tmp $ ls
bin  include  lib  py.zip  share
cas:/data/local/tmp $ cd bin
cas:/data/local/tmp/bin $ ls
2to3      idle3    pydoc3    python3         python3.9
2to3-3.9  idle3.9  pydoc3.9  python3-config  python3.9-config
cas:/data/local/tmp/bin $ ./python3.9
Python 3.9.6 (default, Aug  1 2021, 16:27:01) 
[Clang 9.0.9 (https://android.googlesource.com/toolchain/llvm-project a2a1e703c on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> print("Hello World!")
Hello World!
>>> 
```

可以看到**Python3.9**成功的**运行**起来，并且输出**Hello World!**