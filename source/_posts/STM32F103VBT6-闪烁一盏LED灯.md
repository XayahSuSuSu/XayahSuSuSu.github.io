---
title: 'STM32F103VBT6: 闪烁一盏LED灯'
date: 2022-11-23 19:03:23
tags: [ 'ARM', '嵌入式' ]
mathjax: true
---

# 一、前言
前面我们学习了很多**单片机**的知识，接下来我们即将进入到**ARM**的世界~
本篇博文将**从零开始**实现一块基于**STM32F103VBT6**芯片的开发板上的**LED灯**的闪烁。

# 二、环境准备
## MCUISP
如何使用它下载到**STM32板**上呢？
首先将开发板上的**COM口**接入**PC**，在软件中`搜索串口`并且选择`CH 340`串口，波特率选择`115200`即可。
{% asset_img 0.png Image %}

然后选择**HEX文件**并按照如下选项进行勾选
{% asset_img 1.png Image %}

准备就绪后，点击**开始编程**按钮，在开发板上**按顺序快速按下REST和ISP按键**即可下载。
{% asset_img 2.png Image %}

## Keil uVision5
该程序安装完成后记得安装本开发板的**支持包**`Keil.STM32F1xx_DFP.xxx`（双击即可安装）

## J-Link
可在此处**下载**并**安装**最新的[J-Link](https://www.segger.com/downloads/jlink/)
{% asset_img 3.png Image %}

## STM32CubeMX
可在此处**下载**并**安装**最新的[STM32CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html#get-software)（**需要注册并登录**）

**安装完成**后首次打开该应用，首先打开`Embedded Software Packages Manager`
{% asset_img 4.png Image %}
选择**STM32F1**并下载**最新的软件包**
{% asset_img 5.png Image %}

# 三、使用STM32CubeMX生成模板代码
## 1. 创建工程
打开**STM32CubeMX**，选择`ACCESS TO MCU SELECTOR`
{% asset_img 6.png Image %}

搜素`STM32F103VBT6`，双击右侧对应的`STM32F103VBT6`。
{% asset_img 7.png Image %}
{% asset_img 8.png Image %}

## 2. 分配LED灯相关引脚
{% asset_img 9.png Image %}
{% asset_img 10.png Image %}
从**原理图**可以看出，**LED灯**的**A - E**分别对应应引脚**PE8 - PE15**，而**LED_SEL**对应**PB3**。
因此我们将其设置为**推挽**，即**Output Push Pull**，当然还可以为其**添加标签**，更便于**标识**。
{% asset_img 11.png Image %}

## 3. 模式配置
将这**9个引脚**设置完毕后，我们接下来设置其**调试模式**。
进入**System Core - SYS**，在**Debug**处选择**Serial Wire**，防止开发板**被上锁**导致**只能下载一次**的问题。
{% asset_img 12.png Image %}

## 4. 时钟配置
这里我们使用**时间中断**来**闪烁LED灯**，因此我们需要**配置时钟**。
进入**Clock Configuration**，观察到**默认频率**为**8MHz**（实际上可以根据自己的需求进行调整，或使用**外部时钟**等等，这里我们使用**系统内部**的时钟即可）
{% asset_img 13.png Image %}

返回**Pinout & Configuration**界面，进入**Timers - TIM1**
首先设置**时钟源(Clock Source)**为**系统时钟(Internal Clock)**，然后在**Parameter Settings**中设置**分频系数(Prescaler)**、**计数周期(Counter Period)**以及**自动重载(auto-reload preload)**
{% asset_img 14.png Image %}

假设**分频系数(Prescaler)**为**A**，**计数周期(Counter Period)**为**B**，**时钟频率**为**C**，则这里**时间溢出公式**为
$$
T = (A+1)*(B+1)/C
$$
我们将**A**设置为**8000-1**，**B**设置为**1000-1**，**C**为**8Mhz**，因此**T = 1s**，即我们的**时间回调函数调用周期为1s**。

接下来我们开启**时间中断**
{% asset_img 15.png Image %}

## 5. 工程配置
在**Project Manager**中设置**工程名称**、**路径**、**工具链**。
{% asset_img 16.png Image %}

在左侧选择**Code Generator**，勾选`Generate peripheral initialization as a pair of '.c/.h' files per peripheral`，即**针对每个外设生成独立的.c和.h文件**。
{% asset_img 17.png Image %}

## 6. 生成模板代码
**配置完成**后，点击右上角**GENERATE CODE**按钮即可生成**模板代码**。
{% asset_img 18.png Image %}
{% asset_img 19.png Image %}

在**工程目录**下，`LED.ioc`即为**STM32CubeMX**的**工程文件**，而`MDK-ARM`下的`LED.uvprojx`即为**Keil工程文件**。
{% asset_img 20.png Image %}

# 四、使用模板代码实现LED闪烁效果。
使用**Keil**或其它工具打开该工程，我们可以发现**STM32CubeMX**已为我们生成了规范的**目录结构**。

## 1. 开启LED灯。
在`gpio.c`中我们可以找到**IO口初始化**的相关代码
```
void MX_GPIO_Init(void)
{

  GPIO_InitTypeDef GPIO_InitStruct = {0};

  /* GPIO Ports Clock Enable */
  __HAL_RCC_GPIOE_CLK_ENABLE();
  __HAL_RCC_GPIOA_CLK_ENABLE();
  __HAL_RCC_GPIOB_CLK_ENABLE();

  /*Configure GPIO pin Output Level */
  HAL_GPIO_WritePin(GPIOE, GPIO_PIN_8|GPIO_PIN_9|GPIO_PIN_10|GPIO_PIN_11
                          |GPIO_PIN_12|GPIO_PIN_13|GPIO_PIN_14|GPIO_PIN_15, GPIO_PIN_RESET);

  /*Configure GPIO pin Output Level */
  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_3, GPIO_PIN_RESET);

  /*Configure GPIO pins : PE8 PE9 PE10 PE11
                           PE12 PE13 PE14 PE15 */
  GPIO_InitStruct.Pin = GPIO_PIN_8|GPIO_PIN_9|GPIO_PIN_10|GPIO_PIN_11
                          |GPIO_PIN_12|GPIO_PIN_13|GPIO_PIN_14|GPIO_PIN_15;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(GPIOE, &GPIO_InitStruct);

  /*Configure GPIO pin : PB3 */
  GPIO_InitStruct.Pin = GPIO_PIN_3;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

}
```
因此我们可以**依葫芦画瓢**在`main.c`中为**LED_SEL口使能**，让其**LED灯正常工作**。
```
int main(void)
{
  HAL_Init();
  SystemClock_Config();
  MX_GPIO_Init();
  MX_TIM1_Init();

  // 允许LED输出
  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_3, GPIO_PIN_SET);

  while (1) {}
}
```
其中`GPIO_PIN_SET`实为**1**，即**高电平**；同理`GPIO_PIN_RESET`为**0**，即**低电平**。

## 2. 实现时间中断函数
{% asset_img 22.png Image %}
在`tim.c`中我们可以发现它有一个**全局实例**，**时钟通过该实例进行区分**。
欲**开启该时钟**，我们需要将其**初始化**。
```
int main(void)
{
  HAL_Init();
  SystemClock_Config();
  MX_GPIO_Init();
  MX_TIM1_Init();

  // 允许LED输出
  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_3, GPIO_PIN_SET);

  // 使能TIM1
  HAL_TIM_Base_Start_IT(&htim1);
  while (1) {}
}
```
然后编写**时间中断函数**即可。
```
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef* htim)
{
  if (htim->Instance == TIM1) {
    // TIM1中断, 1s
    HAL_GPIO_TogglePin(GPIOE, GPIO_PIN_8);
  }
}
```
这里我选择将**LED灯A每1s进行一次翻转**，从而实现**闪烁效果**。

# 五、下载验证
代码编写好后，如果正确安装了**J-Link驱动**，将开发板上的**SWD口接入PC**，则可在**设备管理器**中找到如下条目
{% asset_img 23.png Image %}
然后打开**Keil**，如下图配置**J-Link调试**。
{% asset_img 24.png Image %}
{% asset_img 25.png Image %}
进入**J-Link设置**，按如下**配置端口和频率**
{% asset_img 25.1.png Image %}
{% asset_img 25.2.png Image %}

然后**编译**
{% asset_img 26.png Image %}
{% asset_img 27.png Image %}

至此，我们将其**下载**到开发板上，按下**REST键**即可**验证**。
{% asset_img 28.png Image %}
{% asset_img 29.png Image %}
{% asset_img 30.jpg Image %}

# 六、调试
**设置断点**后，即可进入**调试模式**。
{% asset_img 31.png Image %}
{% asset_img 32.png Image %}