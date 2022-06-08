---
title: PyTorch + OpenVINO™ 深度学习笔记
date: 2022-03-22 15:36:13
tags: [ 'PyTorch', 'OpenVINO', '深度学习' ]
mathjax: true
---

# 一、前言
**PyTorch**是**开源**的**深度学习框架**，目的是**加速从研究原型到产品开发的过程**。其**SDK**主要基于**Python**。而其**模型训练**支持**CPU**与**GPU**、支持**分布式训练**、**云部署**，针对**深度学习特定领域**有不同的丰富的**扩展库**。

**深度学习项目**的开发大致可以分为**两个阶段**：
- 第一个阶段是**训练阶段**。这个阶段最重要的事情就是**数据采集**、**模型设计**、**训练参数调试**，**找到合适的模型**并努力训练到**满足**或者**超过**项目实际需要的**精度**。
- 第二个阶段是**部署阶段**。这个阶段最重要的事情就是把模型**移植部署**到**各种不同的计算设备**上，尽可能地实现模型规模的**小型化**、**推理预测过程**的**加速**。

相比于**PyTorch**、**TensorFlow**等为开发者所熟知的**训练框架**，**推理部署**的框架却显得有些**默默无闻**，但是它在**深度学习模型落地过程**中发挥着**不可替代**的作用。正是在这样的背景之下，**英特尔**在**2018年**发布了专门针对**CPU**、**iGPU(集成显卡)**、**FPGA**、**ARM**等**硬件单元加速**的**模型部署**与**加速推理**框架**OpenVINO™**。

**OpenVINO™**是**英特尔**发布的一套支持**快速开发视觉**、**语音识别**、**自然语言处理应用**的框架，受益于**人工智能技术**的快速发展，框架采用了最新的**人工智能神经网络**包括**卷积神经网络**、**循环神经网络**、**注意力机制网络**等模型。实现**视觉**与**非视觉**任务的**底层硬件加速**、达到**最佳性能**，支持人工智能应用从**云端**到**边缘**的**部署**与**推理**全链路技术。

# 二、PyTorch
## 1. 安装
建议使用 **Conda** 安装 **[PyTorch](https://pytorch.org/get-started/locally/)** ，本篇笔记也将以**Conda安装方式**为例。笔者的环境是**Windows11** + **[Anaconda](https://www.anaconda.com/)** + **CUDA11.6**，而**Linux**或**Mac**平台的**环境配置**大同小异，这里就不过多赘述。
### 1) CUDA版本
以**CUDA11+**为例，首先需要安装**[CUDA驱动](https://developer.nvidia.com/cuda-downloads)**
安装**完成**后，**检测安装版本**
```
nvidia-smi
```
{% asset_img CUDA11.6.png CUDA11.6 %}
* 安装**CUDA版本**
```
conda install pytorch torchvision torchaudio cudatoolkit=11.3 -c pytorch
```

### 2) CPU版本
若没有NVIDIA系显卡或其不支持CUDA加速，则可选择安装仅CPU版本。
* 安装**仅CPU版本**
```
conda install pytorch torchvision torchaudio cpuonly -c pytorch
```

## 2. 检验
在**终端**进入**Python3控制台**（笔者`python`命令**默认链接**到**Python3**）
```
python
```
导入**PyTorch**
```
import torch
```
测试`torch.rand()`函数
```
torch.rand(5, 3)
```
若**安装成功**，则会出现类似于以下的**输出**
{% asset_img 检验安装.png 检验安装 %}
测试是否支持**CUDA**
```
import torch
torch.cuda.is_available()
```
{% asset_img 测试CUDA.png 测试CUDA %}

## 3. 概念
很多人学习**深度学习框架**面临的**第一个问题**就是**其专业术语跟基本的编程概念**与**传统面向对象编程**不同，这是**初学者**面临的**第一个学习障碍**。

在**主流**的**面向对象编程语言**中，**结构化代码**最常见的**关键字**是`if`、`else`、`while`、`for`等**关键字**，而在**深度学习框架**中**编程模式**主要是基于**计算图**、**张量数据**、**自动微分**、**优化器**等组件构成。

**面向对象编程**运行的**结果**是**交互式可视化**的，而**深度学习**通过**训练模型**生成**模型文件**，然后再使用**模型预测**、**本质数据流图**的方式工作。所以学习**深度学习框架**首先必须理清**深度学习编程**中**计算图**、**张量数据**、**自动微分**、**优化器**这些**基本术语概念**，下面分别**解释**如下：

###  1) 张量
**张量(Tensor)**是**深度学习框架**中需要**理解**的**最重要**的一个**概念**，**张量**的本质是**数据**，在**深度学习框架**中一切的**数据**都可以看成**张量**。

**深度学习**中的**计算图**是以**张量数据**为**输入**，通过**算子运算**，实现对整个**计算图参数**的**评估优化**。但是到底什么是**张量**？可以看下面这张图：
{% asset_img 张量.png 张量 %}
上图中**标量**、**向量**、**数组**、**3D**、**4D**、**5D**数据矩阵在**深度学习框架**中都被称为**张量**。可见在**深度学习框架**中所有的**数据**都是**张量形式**存在，**张量**是**深度学习数据**组织与存在一种**数据类型**。

###  2) 算子/操作数
**深度学习**主要是针对**张量**的**数据操作**。这些**数据操作**从**简单**到**复杂**，多数都是以**矩阵计算**的形式存在。最常见的**矩阵操作**就是**加减乘除**，此外**卷积**、**池化**、**激活**也是**模型构建**中非常有用的**算子/操作数**。**Pytorch**支持**自定义算子操作**，可以通过**自定义算子**实现复杂的**网络结构**，构建一些特殊的**网络模型**。**张量**跟**算子/操作数**一起构成了**计算图**，它们是也是**计算图**的**基本组成要素**。

###  3) 计算图
**深度学习**基于**计算图**完成**模型构建**，实现**数据**在各个**计算图节点**之间流动，最终输出。因此**计算图**又被称为**数据流图**。

根据构建**计算图**的方式不同还可以分为**静态图**与**动态图**。**Pytorch**默认是基于**动态图**的方式构建**计算图**。

**动态图**采用类似**Python语法**，可以**随时运行**，**灵活修改调整**。

而**静态图**则是**效率优先**，但是在图**构建完成**之前无法**直接运行**。

可以看出**动态图**更加趋向于开发者平时接触的**面向对象**的编程方式，也更容易被开发者理解与接受。

下图是一个简单的**计算图**示例：
{% asset_img 计算图.png 计算图 %}
图中最底层三个**节点**表示**计算图**的**输入张量数据节点（a、b、c）**，剩下**节点**表示**操作**，带**箭头**的线段表示**数据的流向**。

###  4) 自动微分
使用**Pytorch**构建**神经网络（计算图）模型**之后，一般都是通过**反向传播**进行**训练**，**反向传播算法**使用**损失函数功能**对**神经网络**中每个**参数**根据**梯度**进行**参数值**的**调整**。

为了计算这些**梯度**完成**参数调整**，**深度学习框架**中都会自带一个叫做**自动微分**的**内置模块**，来**自动计算神经网络模型训练**时的各个**参数梯度值**并完成**参数值更新**，这种技术就是**深度学习框架**中的**自动微分**。

## 4. PyTorch基础操作
### 1) 张量的定义与声明
**张量**在**PyTorch深度学习框架**中表示**数据**，有几种不同的方式来**创建**与**声明张量数据**。

#### a. 常量声明
```
import torch

a = torch.tensor([[2., 3.], [4., 5.]])
print(a, a.dtype)
```
输出
```
tensor([[2., 3.],
        [4., 5.]]) torch.float32
```
其中`torch.tensor()`默认的**数据类型**是**flaot32**，这点从`a.dtype`的**打印结果**上也得了**印证**。

#### b. 转换声明
`torch.tensor`函数支持从**NumPy数组**直接转换为**张量数据**。
```
import numpy as np
import torch

a = torch.tensor(np.array([[1, 2], [3, 4], [5, 6], [7, 8]]))
print(a, a.dtype)
```
输出
```
tensor([[1, 2],
        [3, 4],
        [5, 6],
        [7, 8]], dtype=torch.int32) torch.int32
```
函数返回的**数据类型**将会根据**NumPy数组**自动识别。

#### c. 初始化声明
**PyTorch框架**支持类似**MATLAB**的**数组初始化方式**，可以定义数组的**维度**，然后**初始化为零**。
```
import torch

a = torch.zeros([2, 4], dtype=torch.float32)
print(a, a.dtype)
```
输出
```
tensor([[0., 0., 0., 0.],
        [0., 0., 0., 0.]]) torch.float32
```
可使用`torch.ones()`函数**初始化为1**
```
import torch

a = torch.ones([2, 4], dtype=torch.float32)
print(a, a.dtype)
```
输出
```
tensor([[1., 1., 1., 1.],
        [1., 1., 1., 1.]]) torch.float32
```

#### d. 随机初始化声明
在**实际的开发**中，经常需要**随机初始化**一些**张量**，可通过`torch.rand()`等函数实现
```
import torch

v1 = torch.rand((2, 3))  # 数组大小: 2x3
print("v1: ", v1)

torch.initial_seed()  # 随机初始化种子
v2 = torch.rand((2, 3))  # 数组大小: 2x3
print("v2: ", v2)

v3 = torch.randint(0, 255, (4, 4))  # 随机范围: 0~255, 数组大小: 4x4
print("v3: ", v3)
```
输出
```
v1:  tensor([[0.6751, 0.0717, 0.4391],
        [0.8088, 0.1570, 0.1111]])
v2:  tensor([[0.4927, 0.1516, 0.3794],
        [0.9748, 0.6124, 0.1192]])
v3:  tensor([[ 60,  88,  33,  53],
        [193, 155,  76, 148],
        [ 86,  47, 177, 192],
        [ 65, 247, 252, 145]])
```

### 2) 张量的操作
#### a. 计算图操作
{% asset_img 计算图.png 计算图 %}
**以上图为例**，用**代码**将它**实现**出来：
```
import torch

a = torch.tensor([[2., 3.], [4., 5.]])
b = torch.tensor([[10, 20], [30, 40]])
c = torch.tensor([[0.1], [0.2]])
x = a + b
y = torch.matmul(x, c)
print("y: ", y)
```
输出
```
y:  tensor([[ 5.8000],
        [12.4000]])
```

#### b. 数据类型转换
可用如下**代码**进行**常见的数据类型转换**：
```
import torch

m = torch.tensor([1., 2., 3., 4., 5., 6], dtype=torch.float32)
print(m, m.dtype)  # 原类型为float32
print(m.double(), m.double().dtype)  # 转换为float64
print(m.int(), m.int().dtype)  # 转换为int32
print(m.long(), m.long().dtype)  # 转换为int64
```
输出
```
tensor([1., 2., 3., 4., 5., 6.]) torch.float32
tensor([1., 2., 3., 4., 5., 6.], dtype=torch.float64) torch.float64
tensor([1, 2, 3, 4, 5, 6], dtype=torch.int32) torch.int32
tensor([1, 2, 3, 4, 5, 6]) torch.int64
```

#### c. 维度转换
可用如下**代码**进行**常见的维度转换**：
```
import torch

a = torch.arange(12.)  # 创建一个(0)~(12-1)区间顺序增长的一维张量
print("a: ", a)
b = torch.reshape(a, (3, 4))  # 转换为3x4
print("b: ", b)
c = torch.reshape(a, (-1, 6))  # 转换为?x6, ?从列数6推理得到
print("c: ", c)
d = torch.reshape(a, (-1,))  # 转换为1行
print("d: ", d)
e = torch.reshape(a, (1, 1, 3, 4))  # 转换为1x1x3x4
print("e: ", e)
```
输出
```
a:  tensor([ 0.,  1.,  2.,  3.,  4.,  5.,  6.,  7.,  8.,  9., 10., 11.])
b:  tensor([[ 0.,  1.,  2.,  3.],
        [ 4.,  5.,  6.,  7.],
        [ 8.,  9., 10., 11.]])
c:  tensor([[ 0.,  1.,  2.,  3.,  4.,  5.],
        [ 6.,  7.,  8.,  9., 10., 11.]])
d:  tensor([ 0.,  1.,  2.,  3.,  4.,  5.,  6.,  7.,  8.,  9., 10., 11.])
e:  tensor([[[[ 0.,  1.,  2.,  3.],
          [ 4.,  5.,  6.,  7.],
          [ 8.,  9., 10., 11.]]]])
```
除此之外，还可以使用基于`tensor`的**维度转换**函数`tensor.view()`
```
import torch

a = torch.arange(12.)  # 创建一个(0)~(12-1)区间顺序增长的一维张量
print("a: ", a)
b = a.view(3, 4)  # 转换为3x4
print("b: ", b)
c = a.view(-1, 6)  # 转换为?x6, ?从列数6推理得到
print("c: ", c)
d = a.view(-1, )  # 转换为1行
print("d: ", d)
e = a.view(1, 1, 3, 4)  # 转换为1x1x3x4
print("e: ", e)
```
输出
```
a:  tensor([ 0.,  1.,  2.,  3.,  4.,  5.,  6.,  7.,  8.,  9., 10., 11.])
b:  tensor([[ 0.,  1.,  2.,  3.],
        [ 4.,  5.,  6.,  7.],
        [ 8.,  9., 10., 11.]])
c:  tensor([[ 0.,  1.,  2.,  3.,  4.,  5.],
        [ 6.,  7.,  8.,  9., 10., 11.]])
d:  tensor([ 0.,  1.,  2.,  3.,  4.,  5.,  6.,  7.,  8.,  9., 10., 11.])
e:  tensor([[[[ 0.,  1.,  2.,  3.],
          [ 4.,  5.,  6.,  7.],
          [ 8.,  9., 10., 11.]]]])
```

#### d. 通道交换
**通道交换**是**PyTorch**中**处理张量数据**常用操作之一。
```
import torch

x = torch.randn(3, 4, 5)  # 随机生成3x4x5的张量
print("x: ", x)
print("Size of x: ", x.size())
y = x.transpose(0, 1)  # 对x的0维和1维进行交换(类似于转置)
print("y: ", y)
print("Size of y: ", y.size())
```
输出
```
x:  tensor([[[-0.9932, -2.1915, -0.8266, -1.8298,  0.5624],
         [ 0.2394, -0.3014,  0.3355,  0.9116,  1.1046],
         [-0.2292, -0.2218, -0.8415, -1.7220, -0.5911],
         [-0.0927, -0.2388,  0.3639, -2.3732, -0.7759]],

        [[ 0.0079, -0.1990, -0.9282,  1.2543, -0.7065],
         [-0.5910,  0.5737, -1.9680,  0.8392,  0.4500],
         [ 0.8449,  1.4641, -0.4619, -0.3084,  0.0082],
         [ 0.2436,  0.1285, -0.1126,  1.7058,  0.5177]],

        [[-1.4615, -1.5110,  0.3243,  0.5885,  0.2760],
         [-0.4879, -1.6806, -0.3202, -2.1921,  1.8557],
         [-1.3212, -0.0511, -0.8245,  1.1485, -0.6952],
         [-0.0165, -0.2692,  0.3099, -0.1915, -0.3242]]])
Size of x:  torch.Size([3, 4, 5])
y:  tensor([[[-0.9932, -2.1915, -0.8266, -1.8298,  0.5624],
         [ 0.0079, -0.1990, -0.9282,  1.2543, -0.7065],
         [-1.4615, -1.5110,  0.3243,  0.5885,  0.2760]],

        [[ 0.2394, -0.3014,  0.3355,  0.9116,  1.1046],
         [-0.5910,  0.5737, -1.9680,  0.8392,  0.4500],
         [-0.4879, -1.6806, -0.3202, -2.1921,  1.8557]],

        [[-0.2292, -0.2218, -0.8415, -1.7220, -0.5911],
         [ 0.8449,  1.4641, -0.4619, -0.3084,  0.0082],
         [-1.3212, -0.0511, -0.8245,  1.1485, -0.6952]],

        [[-0.0927, -0.2388,  0.3639, -2.3732, -0.7759],
         [ 0.2436,  0.1285, -0.1126,  1.7058,  0.5177],
         [-0.0165, -0.2692,  0.3099, -0.1915, -0.3242]]])
Size of y:  torch.Size([4, 3, 5])
```

#### e. 寻找最大值
**寻找最大值**是**PyTorch**中**处理张量数据**常用操作之一。
```
import torch

x = torch.tensor([2., 3., 4., 12., 3., 5., 8., 1.])
print("x: ", x)
print("Max of x: ", torch.argmax(x))  # 求x的最大值索引

y = x.view(-1, 2)
print("y: ", y)
print("Max of y: ", torch.argmax(y))  # 求x的最大值索引(展开为1维)
print("Max of y: ", torch.argmax(y, 0))  # 求x的0维最大值索引
print("Max of y: ", torch.argmax(y, 1))  # 求x的1维最大值索引
print("Max of y: ", y.argmax())  # 求x的最大值索引(展开为1维)
print("Max of y: ", y.argmax(0))  # 求x的0维最大值索引
print("Max of y: ", y.argmax(1))  # 求x的1维最大值索引
```
输出
```
x:  tensor([ 2.,  3.,  4., 12.,  3.,  5.,  8.,  1.])
Max of x:  tensor(3)
y:  tensor([[ 2.,  3.],
        [ 4., 12.],
        [ 3.,  5.],
        [ 8.,  1.]])
Max of y:  tensor(3)
Max of y:  tensor([3, 1])
Max of y:  tensor([1, 1, 1, 0])
Max of y:  tensor(3)
Max of y:  tensor([3, 1])
Max of y:  tensor([1, 1, 1, 0])
```

#### f. CPU与GPU运算支持
**PyTorch**支持**CPU**与**GPU**计算，默认创建的**tensor**是**CPU**版本的，要想使用**GPU**版本，首先需要检测**GPU**支持，然后转换为**GPU**数据，或者直接创建为**GPU**版本数据
```
import torch

gpu = torch.cuda.is_available()
for i in range(torch.cuda.device_count()):
    print("name: ", torch.cuda.get_device_name(i))
    x = torch.randn(2, 3)
    if gpu:
        print("x: ", x)
        print("x.device: ", x.device)
        print("x.cuda(): ", x.cuda())
        print("x.cuda().device: ", x.cuda().device)
    y = torch.tensor([1, 2, 3, 4], device="cuda:0")
    print("y: ", y)
```
输出
```
name:  NVIDIA GeForce RTX 2060
x:  tensor([[-0.1489, -1.4650,  0.3720],
        [ 0.8687,  0.0330, -0.4909]])
x.device:  cpu
x.cuda():  tensor([[-0.1489, -1.4650,  0.3720],
        [ 0.8687,  0.0330, -0.4909]], device='cuda:0')
x.cuda().device:  cuda:0
y:  tensor([1, 2, 3, 4], device='cuda:0')
```

## 5. 线性回归预测
**线性回归**的本质就是根据给出**二维数据集**来**拟合生成一条直线**，如下图：
{% asset_img 线性回归.png 线性回归 %}
**左图**是一组**圆点**表示的**二维坐标点数据集**，**直线**是根据**线性回归算法**生成的。
**右图**则是根据**坐标点数据集**生成的一个**非线性回归**例子。
现在我们已经可以很**直观地**了解什么是**线性回归**了，但**线性回归**是怎么找到这条**直线**的？
我们可以通过**PyTorch**构建一个简单的**计算图**来**不断学习**，最终得到一个**足够逼近真实直线**的**参数方程**，这个过程被称为**线性回归**的**学习/训练**过程。

### 1) 原理
最常见的**直线方程**如下：
$$
y = kx + b
$$

假设有一组**二维坐标点数据集**：

|  | 一 | 二 | 三 | 四 | 五 | 六 |
| :----: | :----: | :----: | :----: | :----: | :----: | :----: |
| x | 1 | 2 | 0.5 | 2.5 | 2.6 | 3.1 |
| y | 3.7 | 4.6 | 1.65 | 5.68 | 5.98 | 6.95 |

**随机赋值**初始**k**、**b**两个**参数**，根据**直线方程**，通过**x**可以得到对应的$\mathop{y}\limits^{\frown}$，它跟 **真实值y** 之间的**差值**称为**损失**，最常见的损失是**均值平方损失（MSE）**，表示如下：
$$
MSE = \frac{1}{n}\sum(y-\mathop{y}\limits^{\frown})^2
$$
假设**当前参数**为**A(k, b)**，**新参数**为**B(k, b)**，我们可以通过下面的**公式**来**更新k**、**b**两个参数：
$$
A(k, b) = B(k, b) - η * grad(η)
$$
其中**η**称为**学习率**，**grad(η)**是对应的**参数梯度**，可根据**深度学习框架**的**自动微分机制**得到。这样就实现了**线性回归模型**的**构建**与**训练**过程，最终可根据输入的**迭代次数**运行并输出**回归直线**的两个参数，从而完成**线性回归**的求解。

### 2) 实现
**PyTorch**提供了丰富的函数，可以帮助我们快速搭建**线性回归模型**并完成**训练预测**。
#### a. 构建数据集
```
import numpy as np

x = np.array([1, 2, 0.5, 2.5, 2.6, 3.1], dtype=np.float32).reshape((-1, 1))
y = np.array([3.7, 4.6, 1.65, 5.68, 5.98, 6.95], dtype=np.float32).reshape(-1, 1)
```
#### b. 构建线性回归模型
```
# 继承torch.nn.Module
class LinearRegressionModel(torch.nn.Module):
    def __init__(self, input_dim, output_dim):
        super(LinearRegressionModel, self).__init__()
        self.linear = torch.nn.Linear(input_dim, output_dim)  # 对应直线方程y = kx + b

    def forward(self, x):
        return self.linear(x)  # 重载forward(), 根据模型计算并返回预测结果
```

#### c. 创建损失功能与优化器
```
data_input_dim = 1  # 输入维度为1
data_output_dim = 1  # 输出维度为1
model = LinearRegressionModel(data_input_dim, data_output_dim)  # 实例化
criterion = torch.nn.MSELoss()  # 均值平方损失
learning_rate = 0.01  # 学习率
optimizer = torch.optim.SGD(model.parameters(), lr=learning_rate)  # 优化器, 求解参数梯度
```

#### d. 迭代训练
```
# 迭代训练
for index in range(100):
    # 将NumPy数组转为torch变量
    input_x = torch.from_numpy(data_x).requires_grad_()  # 需要为其计算梯度
    input_y = torch.from_numpy(data_y)
    optimizer.zero_grad()  # 梯度置零
    output_y = model(input_x)  # 得到输出
    loss = criterion(output_y, input_y)  # 计算损失
    loss.backward()  # 计算梯度，反向传播
    optimizer.step()  # 更新参数
    print('迭代索引: {}, 损失: {}'.format(index, loss.item()))
```

#### e. 绘制结果
```
predicted_y = model(torch.from_numpy(data_x).requires_grad_()).data.numpy()  # 得到预测结果
plt.plot(data_x, data_y, 'go', label='True data', alpha=0.5)  # 绘制数据集
plt.plot(data_x, predicted_y, '--', label='Predictions', alpha=0.5)  # 绘制回归直线
plt.legend()
plt.show()
```
最终**完整代码**为：
```
import numpy as np
import torch
from matplotlib import pyplot as plt

data_x = np.array([1, 2, 0.5, 2.5, 2.6, 3.1], dtype=np.float32).reshape((-1, 1))
data_y = np.array([3.7, 4.6, 1.65, 5.68, 5.98, 6.95], dtype=np.float32).reshape(-1, 1)


# 继承torch.nn.Module
class LinearRegressionModel(torch.nn.Module):
    def __init__(self, input_dim, output_dim):
        super(LinearRegressionModel, self).__init__()
        self.linear = torch.nn.Linear(input_dim, output_dim)  # 对应直线方程y = kx + b

    def forward(self, x):
        return self.linear(x)  # 重载forward(), 根据模型计算并返回预测结果


data_input_dim = 1  # 输入维度为1
data_output_dim = 1  # 输出维度为1
model = LinearRegressionModel(data_input_dim, data_output_dim)  # 实例化
criterion = torch.nn.MSELoss()  # 均值平方损失
learning_rate = 0.01  # 学习率
optimizer = torch.optim.SGD(model.parameters(), lr=learning_rate)  # 优化器, 求解参数梯度

# 迭代训练
for index in range(100):
    # 将NumPy数组转为torch变量
    input_x = torch.from_numpy(data_x).requires_grad_()  # 需要为其计算梯度
    input_y = torch.from_numpy(data_y)
    optimizer.zero_grad()  # 梯度置零
    output_y = model(input_x)  # 得到输出
    loss = criterion(output_y, input_y)  # 计算损失
    loss.backward()  # 计算梯度，反向传播
    optimizer.step()  # 更新参数
    print('迭代索引: {}, 损失: {}'.format(index, loss.item()))

predicted_y = model(torch.from_numpy(data_x).requires_grad_()).data.numpy()  # 得到预测结果
plt.plot(data_x, data_y, 'go', label='True data', alpha=0.5)  # 绘制数据集
plt.plot(data_x, predicted_y, '--', label='Predictions', alpha=0.5)  # 绘制回归直线
plt.legend()
plt.show()
```

最终**得到**：
{% asset_img 结果.png 结果 %}

# 三、OpenVINO™
未完待续...

# 四、参考
- [PyTorch + OpenVINO™ 开发实战系列教程 第一篇](https://mp.weixin.qq.com/s/qWPYYXl50cYgkCO6nOg_gQ)
- [PyTorch + OpenVINO™ 开发实战系列教程 第二篇](https://mp.weixin.qq.com/s/NaUijeZ0mDho49vXvDzCjQ)