---
title: Hexo添加自定义静态页面
date: 2021-07-08 00:11:09
tags: [ 'Hexo', 'HTML', 'CSS' ]
---
## 具体步骤

1. 将静态页面源码 （以 **unlock-music文件夹** 为例） 放入 **Hexo项目根目录** 下的 **source** 文件夹
2. 修改 **Hexo项目根目录** 下的 **_config.yml**:
   ```
   skip_render: /unlock-music**
   ```
3. 重新部署：

   本地部署：
   ```
   hexo g
   hexo s 
   ```
   上行部署：
   ```
   hexo g -d
   ```
4. 访问静态页面（以 **AcmeZone** 为例）：
   ```
   http://acmezone.tk/unlock-music/
   ```