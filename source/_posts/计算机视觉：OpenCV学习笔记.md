---
title: 计算机视觉：OpenCV学习笔记
date: 2022-03-10 13:54:59
tags: [ 'OpenCV', 'Linux' ]
mathjax: true
---

# 一、前言
**OpenCV**是**计算机视觉**领域一款热门的**框架**，**人脸识别**、**二维码处理**、**深度学习**中都能见到它的身影~

# 二、安装
## 1. C++版安装
> 在[官网](https://opencv.org/releases/)中下载**OpenCV**源码，本文以**OpenCV – 4.5.5**为例。

**安装**相关**依赖**
```
sudo apt-get install libgtk2.0-dev
```
**解压**至任意目录
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

## 2. Python版安装
通过`pip`安装
```
pip3 install opencv-python -i https://pypi.tuna.tsinghua.edu.cn/simple
```

# 三、基础用法
> 本文主要记录基于**Python**的**OpenCV**笔记

## 1. 导入OpenCV
```
import cv2
```

## 2. 读取图片 - imread()
```
img = cv2.imread("ari.jpg")
```
第一个参数是**图片的路径**，**绝对路径**和**相对路径**均可。
第二个参数是**可选参数**，可填以下内容：
- **cv2.IMREAD_COLOR**（默认）： 加载**彩色图像**。任何图像的**透明度**都将被忽略。
- **cv2.IMREAD_GRAYSCALE**：以**灰度模式**加载图像。
- **cv2.IMREAD_UNCHANGED**：保留读取图片原有**颜色通道**（包括**透明通道**）。

## 3. 显示图片 - imshow()
```
cv2.imshow("ari", img)
```
第一个参数是窗口的**标题**，第二个参数传入读取的**img对象**。
{% asset_img ari.png ari %}

## 4. 调整窗口 - namedWindow()
```
cv2.namedWindow("ari")
```
第一个参数是窗口的**标题**。
第二个参数是**可选参数**，可填以下内容：
- **cv2.WINDOW_NORMAL**：可调整窗口**大小**。
- **cv2.WINDOW_AUTOSIZE**（默认）：窗口内容图片为**真实大小**，窗口大小**不可调整**。
- **cv2.WINDOW_OPENGL**：支持**OpenGL**。

## 5. 等待按键 - waitKey()
```
cv2.waitKey(0)
```
当调用`imshow()`后，需要再调用waitKey()控制窗口显示时间。
当参数**≤0**时，则窗口一直等待，返回 **-1** 或**按键值**，当为`$time`时，则窗口显示`$time`ms，在此期间**按键**则返回**按键值**，否则返回 **-1** 。

## 6. 保存图片 - imwrite()
```
cv2.imwrite("ari_out.jpg", img)
```
第一个参数是保存的**路径及名称**，第二个参数传入读取的**img对象**。