---
title: RESTful API笔记
date: 2021-09-27 10:17:49
tags: [ '后端', 'API', '笔记' ] 
---

> 作者：覃超
> 链接：https://www.zhihu.com/question/28557115/answer/48094438
> 来源：知乎

# 概念

**REST** -- **Resource Representational State Transfer** **资源**在**网络**中以某种**表现形式**进行**状态转移**

**Resource **：**资源**，即**数据**。例如 **friends**，**profile**等；

**Representational **：某种**表现形式**，例如**JSON**，**XML**，**JPEG**等；

**State Transfer **：**状态变化**。通过**HTTP动词**实现。

一言以蔽之：**URL定位资源，用HTTP动词（GET,POST,DELETE,DETC）描述操作。**

# 规则

1. **URL**中只使用**名词**来指定**资源**，原则上**不使用动词**。

2. **Server**和**Client**之间传递某**资源**的一个**表现形式**，比如用**JSON**，**XML传输文本**，或者用**JPG**，**WebP传输图片**等。当然还可以**压缩HTTP传输时的数据（On-wire data compression）**。
3. 用 **HTTP Status Code**传递**Server**的**状态信息**。例如最常用的 **200** 表示**成功**，**500** 表示**Server内部错误**等。
{% asset_img REST_API_Design.png REST_API_Design %}

# 示例

```
# 获取某人的好友列表
/v1/friends
```

```
# 获取某人的详细信息
/v1/profile
```

用**HTTP协议**里的**动词**来实现**资源**的**添加**，**修改**，**删除**等**操作**。即通过**HTTP动词**来实现**资源**的**状态变化**：

```
# 删除某人的好友
DELETE /v1/friends
```

```
# 添加好友
POST /v1/friends
```

```
# 更新个人资料
UPDATE /v1/profile
```

如下使用不符合规范：

```
GET /v1/deleteFriend
```

