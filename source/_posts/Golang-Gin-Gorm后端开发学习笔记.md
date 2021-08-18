---
title: Golang+Gin+Gorm后端开发学习笔记
date: 2021-08-18 14:13:47
tags: [ 'Golang', 'Gin', 'Gorm' ] 
---

# 前言
早已久仰Golang大名，以其优异的高并发支持著称。那么让我们来一探究竟吧！

# Golang的安装及环境配置
在[Golang官网](https://golang.google.cn/dl/)下载并安装，目前较新的版本都能够自动配置环境变量。

但我们还需要配置一下Goproxy代理，以解决当前国内网络环境所带来的问题。

这里我推荐[goproxy.io](https://goproxy.io/zh/docs/getting-started.html)，那么我们就跟着它的文档配置吧。

```
1. 右键 我的电脑 -> 属性 -> 高级系统设置 -> 环境变量
2. 在 “[你的用户名]的用户变量” 中点击 ”新建“ 按钮
3. 在 “变量名” 输入框并新增 “GOPROXY”
4. 在对应的 “变量值” 输入框中新增 “https://goproxy.io,direct”
5. 最后点击 “确定” 按钮保存设置
```

# GoLand的安装
工欲善其事，必先利其器。优秀的IDE能让我们开发效率直线上升。JetBrains系列的IDE深受大众喜爱。这里我们选择GoLand作为开发工具

下载[GoLand](https://www.jetbrains.com/go/)并安装。

# 运行第一个项目
打开GoLand，新建一个工程，取名为HelloWorld，然后创建项目。
{% asset_img HelloWorld.png HelloWorld %}

### Gin依赖的安装
既然我们使用Gin框架来开发后端，那么肯定要先安装它
打开GoLand的终端（在项目根目录下打开系统终端也行），安装Gin-Gonic依赖
```
go get -u github.com/gin-gonic/gin
```
稍等片刻即可安装成功。

### 创建HelloWorld
接下来我们创建一个Go文件，取名为HelloWorld.go。
键入以下代码：
```
package HelloWorld

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	r := gin.Default()
	r.GET("/", func(context *gin.Context) {
		context.String(http.StatusOK, "Hello World!")
	})
	r.Run(":8080")
}
```
### 配置编译器
点击GoLand右上方Add Configuration...
{% asset_img AddConfiguration.png AddConfiguration %}
在弹出的窗口中点击左边的+号，选择Go Build，然后点击OK。
{% asset_img GoBuild.png GoBuild %}
接下来我们点击IDE右上角绿色的右三角按钮运行。
{% asset_img RunError.png RunError %}
报错了，查询一通资料后才发现，当我们只编译运行单文件时，包名必须为main，因此，我们修改代码：
```
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	r := gin.Default()
	r.GET("/", func(context *gin.Context) {
		context.String(http.StatusOK, "Hello World!")
	})
	r.Run(":8080")
}
```
这下能够成功运行了。
我们打开浏览器，输入`http://localhost:8080/`，即可看到项目正确运行。
{% asset_img RunSuccessfully.png RunSuccessfully %}
