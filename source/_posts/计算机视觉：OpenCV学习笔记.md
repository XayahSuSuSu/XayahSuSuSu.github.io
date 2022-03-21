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

# 三、进阶操作
## 1. 读取网络摄像机rtsp流
```
import cv2

cam = cv2.VideoCapture("rtsp://admin:123456@192.168.56.56:554/ch01.264")  # rtsp流地址
while cam.isOpened():
    flag, img = cam.read()
    cv2.imshow('Cam', img)
    cv2.waitKey(1)  # 1ms刷新一次
```

## 2. 单目相机标定与坐标系转换
最近有一个项目需要将**单目摄像机**拍摄到的照片中的**像素坐标**转换为**世界坐标**，于是研究了一下相关的**原理**和**代码**。
### 1) 原理
在**相机模型**中，我们需要理解**四个坐标系**以及它们之间的**关系**：
- **像素坐标系（u, v）**
- **图像坐标系（x, y）**
- **相机坐标系（X<sub>c</sub>, Y<sub>c</sub>, Z<sub>c</sub>）**
- **世界坐标系（X<sub>w</sub>, Y<sub>w</sub>, Z<sub>w</sub>）**
如图：
{% asset_img pinhole_camera_model.png 来源于opencv-4.5.5/modules/calib3d/doc/pics/pinhole_camera_model.png %}

#### a. 像素坐标系（u, v）与图像坐标系（x, y）转换
{% asset_img xyuv.png xyuv %}
假设**像素**在**u轴**和**v轴**方向上的**物理尺寸**为**dx**和**dy**。
根据上图可以推导出该**公式**：
$$
\begin{cases}
u = \frac{x}{dx} + u_0\\\\
v = \frac{y}{dy} + v_0
\end{cases}
$$
转换为**矩阵形式**：
$$
\begin{bmatrix}
u \\\\
v \\\\
1
\end{bmatrix}
=
\begin{bmatrix}
\frac{1}{dx} & 0 & u_0\\\\
0 & \frac{1}{dy} & v_0\\\\
0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
x \\\\
y \\\\
1
\end{bmatrix}
$$

#### b. 相机坐标系（X<sub>c</sub>, Y<sub>c</sub>, Z<sub>c</sub>）与世界坐标系（X<sub>w</sub>, Y<sub>w</sub>, Z<sub>w</sub>）转换
{% asset_img WC.png WC %}
从**世界坐标系**变换到**相机坐标系**属于**刚体变换**，物体**不会发生形变**，只需进行**旋转**和**平移**。
如上图，**R**表示**旋转矩阵**，**T**表示**旋转向量**。
用**矩阵**表示其**关系**：
$$
\begin{bmatrix}
X_c \\\\
Y_c \\\\
Z_c \\\\
1
\end{bmatrix}
=
\begin{bmatrix}
R_{11} & R_{12} & R_{13} & T_1\\\\
R_{21} & R_{22} & R_{23} & T_2\\\\
R_{31} & R_{32} & R_{33} & T_3\\\\
0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
X_w \\\\
Y_w \\\\
Z_w \\\\
1
\end{bmatrix}
$$

令：
$$
R=
\begin{bmatrix}
R_{11} & R_{12} & R_{13}\\\\
R_{21} & R_{22} & R_{23}\\\\
R_{31} & R_{32} & R_{33}
\end{bmatrix}
,
T=
\begin{bmatrix}
T_1\\\\
T_2\\\\
T_3
\end{bmatrix}
,
0=
\begin{bmatrix}
0 & 0 & 0
\end{bmatrix}
,
1=
\begin{bmatrix}
1
\end{bmatrix}
$$
因此可表示为
$$
\begin{bmatrix}
X_c \\\\
Y_c \\\\
Z_c \\\\
1
\end{bmatrix}
=
\begin{bmatrix}
R & T\\\\
0 & 1
\end{bmatrix}
\begin{bmatrix}
X_w \\\\
Y_w \\\\
Z_w \\\\
1
\end{bmatrix}
$$

#### c. 相机坐标系（X<sub>c</sub>, Y<sub>c</sub>, Z<sub>c</sub>）与图像坐标系（x, y）转换
{% asset_img Xx.png Xx %}
如图，根据**相似三角形原理**，可得：
$$
\begin{cases}
x = f \frac{X_c}{Z_c}\\\\
y = f \frac{Y_c}{Z_c}
\end{cases}
$$
可**变换**为：
$$
\begin{cases}
Z_c · x = f · X_c\\\\
Z_c · y = f · Y_c\\\\
\end{cases}
$$
因此可用**矩阵**表示：
$$
Z_c
\begin{bmatrix}
x \\\\
y \\\\
1
\end{bmatrix}
=
\begin{bmatrix}
f & 0 & 0 & 0\\\\
0 & f & 0 & 0\\\\
0 & 0 & 1 & 0
\end{bmatrix}
\begin{bmatrix}
X_c \\\\
Y_c \\\\
Z_c \\\\
1
\end{bmatrix}
$$

#### d. 综合公式
将上面的**矩阵公式**综合，即可得出以下**公式**：
$$
Z_c
\begin{bmatrix}
u \\\\
v \\\\
1
\end{bmatrix}
=
\begin{bmatrix}
f_x & 0 & u_0 & 0\\\\
0 & f_y & v_0 & 0\\\\
0 & 0 & 1 & 0
\end{bmatrix}
\begin{bmatrix}
R & T\\\\
0 & 1
\end{bmatrix}
\begin{bmatrix}
X_w \\\\
Y_w \\\\
Z_w \\\\
1
\end{bmatrix}
$$
其中，
$$
\begin{bmatrix}
f_x & 0 & u_0 & 0\\\\
0 & f_y & v_0 & 0\\\\
0 & 0 & 1 & 0
\end{bmatrix}
=
\begin{bmatrix}
\frac{1}{dx} & 0 & u_0\\\\
0 & \frac{1}{dy} & v_0\\\\
0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
f & 0 & 0 & 0\\\\
0 & f & 0 & 0\\\\
0 & 0 & 1 & 0
\end{bmatrix}
$$
称为**相机内参**。
$$
\begin{bmatrix}
R & T\\\\
0 & 1
\end{bmatrix}
$$
称为**相机外参**。

#### e. 求解
我们的需求是**根据像素坐标**求**世界坐标**，而**世界坐标系**建立在**地面**，因此可令**Z<sub>w</sub>=0**。
即：
$$
Z_c
\begin{bmatrix}
u \\\\
v \\\\
1
\end{bmatrix}
=
\begin{bmatrix}
f_x & 0 & u_0 & 0\\\\
0 & f_y & v_0 & 0\\\\
0 & 0 & 1 & 0
\end{bmatrix}
\begin{bmatrix}
R & T\\\\
0 & 1
\end{bmatrix}
\begin{bmatrix}
X_w \\\\
Y_w \\\\
0 \\\\
1
\end{bmatrix}
$$
将**相机外参**展开：
$$
Z_c
\begin{bmatrix}
u \\\\
v \\\\
1
\end{bmatrix}
=
\begin{bmatrix}
f_x & 0 & u_0 & 0\\\\
0 & f_y & v_0 & 0\\\\
0 & 0 & 1 & 0
\end{bmatrix}
\begin{bmatrix}
R_{11} & R_{12} & R_{13} & T_1\\\\
R_{21} & R_{22} & R_{23} & T_2\\\\
R_{31} & R_{32} & R_{33} & T_3\\\\
0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
X_w \\\\
Y_w \\\\
0 \\\\
1
\end{bmatrix}
$$
**化简**可得：
$$
Z_c
\begin{bmatrix}
u \\\\
v \\\\
1
\end{bmatrix}
=
\begin{bmatrix}
f_x & 0 & u_0 & 0\\\\
0 & f_y & v_0 & 0\\\\
0 & 0 & 1 & 0
\end{bmatrix}
\begin{bmatrix}
R_{11} & R_{12} & T_1\\\\
R_{21} & R_{22} & T_2\\\\
R_{31} & R_{32} & T_3\\\\
0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
X_w \\\\
Y_w \\\\
1
\end{bmatrix}
$$
将上式改写为**AX=B**的形式：
$$
\begin{bmatrix}
f_x & 0 & u_0 & 0\\\\
0 & f_y & v_0 & 0\\\\
0 & 0 & 1 & 0
\end{bmatrix}
\begin{bmatrix}
R_{11} & R_{12} & T_1\\\\
R_{21} & R_{22} & T_2\\\\
R_{31} & R_{32} & T_3\\\\
0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
\frac{X_w}{Z_c} \\\\
\frac{Y_w}{Z_c} \\\\
\frac{1}{Z_c}
\end{bmatrix}
=
\begin{bmatrix}
u \\\\
v \\\\
1
\end{bmatrix}
$$
**解出**这个**矩阵方程**即可求解 **世界坐标（X<sub>w</sub>, Y<sub>w</sub>, 0）** 以及 **相机坐标系原点到所求点的直线距离Z<sub>c</sub>** ！

### 2) 实现
未完待续...

### 3) 参考
- [世界坐标系、相机坐标系、图像平面坐标系](https://blog.csdn.net/weizhangyjs/article/details/81020177?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~aggregatepage~first_rank_ecpm_v1~rank_v31_ecpm-1-81020177.pc_agg_new_rank&utm_term=opencv%E7%9B%B8%E6%9C%BA%E5%9D%90%E6%A0%87%E7%B3%BB%E7%A4%BA%E6%84%8F%E5%9B%BE&spm=1000.2123.3001.4430)