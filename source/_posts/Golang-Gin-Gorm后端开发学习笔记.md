---
title: Golang+Gin+Gorm后端开发学习笔记
date: 2021-08-18 14:13:47
tags: [ 'Golang', 'Gin', 'Gorm' ] 
---

# 前言
早已久仰Golang大名，以其优异的高并发支持著称。那么让我们来一探究竟吧！

# 安装及配置

## Golang的安装及环境配置
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

## GoLand的安装
工欲善其事，必先利其器。优秀的IDE能让我们开发效率直线上升。JetBrains系列的IDE深受大众喜爱。这里我们选择GoLand作为开发工具

下载[GoLand](https://www.jetbrains.com/go/)并安装。

## 运行第一个项目
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

# Gorm(Sqlite3)笔记
定义数据库结构体
```
type Doc struct {
	gorm.Model
	Type  string
	Title string
	Date  string
}
```
创建数据库并添加一条记录
```
func main() {
	db, err := gorm.Open(sqlite.Open("doc.db"), &gorm.Config{})
	if err != nil {
		panic("failed to connect database")
	}
	// 迁移 schema
	db.AutoMigrate(&Doc{})
	// 添加一条记录
	db.Create(&Doc{
		Type:  "文件学习",
		Title: "打响疫情第二波湖南必胜",
		Date:  "2021.08.18",
	})
}
```
运行后，会在源码根目录创建doc.db，并且写入一条记录

{% asset_img Gorm-create.png Gorm-create%}



# Gin笔记
### 定义标记常量
```
const (
	JSON_SUCCESS int = 1
	JSON_ERROR   int = 0
)
```
### 定义数据库结构体和返回结构体
```
type Doc struct {
	gorm.Model
	Type  string
	Title string
	Date  string
}

type rDoc struct {
	Type  string
	Title string
	Date  string
}
```
### 路由分组
```
v1 := r.Group("/api/v1/docs")
{
	v1.POST("/add", add)
	v1.GET("/get", get)
}
```
### 添加记录函数
```
func add(c *gin.Context) {
	db, err := gorm.Open(sqlite.Open("doc.db"), &gorm.Config{})
	if err != nil {
		panic("failed to connect database")
	}
	db.AutoMigrate(&Doc{})
	db.Create(&Doc{
		Type:  "文件学习",
		Title: "打响疫情第二波湖南必胜",
		Date:  "2021.08.18",
	})
	c.JSON(http.StatusOK, gin.H{
		"status":  JSON_SUCCESS,
		"message": "创建成功",
	})
}
```
### 获取记录函数
```
func get(c *gin.Context) {
	db, err := gorm.Open(sqlite.Open("doc.db"), &gorm.Config{})
	if err != nil {
		panic("failed to connect database")
	}
	var docs []Doc
	var rdocs []rDoc
	db.Find(&docs)
	for _, i := range docs {
		rdocs = append(rdocs, rDoc{
			Type:  i.Type,
			Title: i.Title,
			Date:  i.Date,
		})
	}
	c.JSON(http.StatusOK, gin.H{
		"status":  JSON_SUCCESS,
		"message": &rdocs,
	})
}
```
### 实现实例：
```
package main

import (
	"github.com/gin-gonic/gin"
	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
	"net/http"
)

const (
	JSON_SUCCESS int = 1
	JSON_ERROR   int = 0
)

type Doc struct {
	gorm.Model
	Type  string
	Title string
	Date  string
}

type rDoc struct {
	Type  string
	Title string
	Date  string
}

func main() {
	r := gin.Default()
	r.GET("/")
	v1 := r.Group("/api/v1/docs")
	{
		v1.POST("/add", add)
		v1.GET("/get", get)
	}
	r.Run()
}

func add(c *gin.Context) {
	db, err := gorm.Open(sqlite.Open("doc.db"), &gorm.Config{})
	if err != nil {
		panic("failed to connect database")
	}
	db.AutoMigrate(&Doc{})
	db.Create(&Doc{
		Type:  "文件学习",
		Title: "打响疫情第二波湖南必胜",
		Date:  "2021.08.18",
	})
	c.JSON(http.StatusOK, gin.H{
		"status":  JSON_SUCCESS,
		"message": "创建成功",
	})
}

func get(c *gin.Context) {
	db, err := gorm.Open(sqlite.Open("doc.db"), &gorm.Config{})
	if err != nil {
		panic("failed to connect database")
	}
	var docs []Doc
	var rdocs []rDoc
	db.Find(&docs)
	for _, i := range docs {
		rdocs = append(rdocs, rDoc{
			Type:  i.Type,
			Title: i.Title,
			Date:  i.Date,
		})
	}
	c.JSON(http.StatusOK, gin.H{
		"status":  JSON_SUCCESS,
		"message": &rdocs,
	})
}
```
### 测试API
{% asset_img Gin-Test.png Gin-Test%}