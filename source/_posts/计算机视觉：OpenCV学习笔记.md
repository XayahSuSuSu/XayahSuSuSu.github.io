---
title: 计算机视觉：OpenCV学习笔记
date: 2022-03-10 13:54:59
tags: [ 'OpenCV', 'Linux' ]
---

# 一、前言
**OpenCV**是**计算机视觉**领域一款热门的**框架**，**人脸识别**、**二维码处理**、**深度学习**中都能见到它的身影~

# 二、安装
本文采用**源码编译**的方式进行**安装**。
## 1. 下载源码
> 在[官网](https://opencv.org/releases/)中下载**OpenCV**源码，本文以**OpenCV – 4.5.5**为例。

首先**解压**至任意目录
```
unzip opencv-4.5.5.zip
```
进入解压后的文件夹，创建`build`文件夹并进入
```
cd opencv-4.5.5
mkdir build && cd build
```
**CMake**生成**Makefile**
```
cmake -D CMAKE_BUILD_TYPE=Release -D CMAKE_INSTALL_PREFIX=/usr/local/ ..
```
编译**OpenCV**（`-j`为线程数，视**实际情况**更改，太大可能会**爆内存**，实在不行就**单线程**吧hhh）
```
make -j8
```
**安装**
```
sudo make install
```
**验证**（运行源码**根目录下`samples/cpp/example_cmake`**后弹出`Hello OpenCV`即为**成功**）
```
cd ../samples/cpp/example_cmake
cmake .
make
./opencv_example
```
{% asset_img Hello_OpenCV.png Hello_OpenCV %}