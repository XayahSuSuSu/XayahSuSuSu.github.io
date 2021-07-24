---
title: GitLab搭建Hexo博客笔记
date: 2021-07-24 22:23:33
tags: [ 'Hexo', 'GitHub', 'GitLab' ]
---

## 源起
GitHub Page国内访问实在是太慢了！早就有了换地的想法，正好 **[noionion](https://noionion.top/)** 提醒了我， **[GitLab](https://gitlab.com/)** 和 **[Gitee](https://gitee.com/)** 在国内的访问速度都不错，但是我实在不喜欢 **Gitee** 的UI，索性就换 **[GitLab](https://gitlab.com/)** 吧！

### 一、在GitLab创建一个仓库
如果你没有自己的域名，那么可以创建一个名为`${你的GitLab ID}.gitlab.io`的仓库，比如我的ID是 **Xayah** ：
{% asset_img username.png 我的ID %}
那么我可以新建一个名为`xayah.gitlab.io`的仓库
{% asset_img create.png 创建仓库 %}

### 二、上传Hexo项目到仓库
创建好后，我们先把仓库 **Clone** 下来，然后把 **Hexo项目** 放进去。
**public** 文件夹是 **Hexo** 构建 **静态页面** 生成的，我们在服务器上不需要它，所以请检查你的`.gitignore`中是否忽略了`/public`，如果没有，请将其加入 **忽略名单** 。
配置好后，将本地仓库 **Push** 到 **GitLab** 。

&emsp;&emsp;&emsp;&emsp;●此处需要[Git基础](https://www.runoob.com/git/git-tutorial.html)和[Hexo基础](https://hexo.io/zh-cn/docs/)，如果不会可以学习一下

### 三、使用CI构建静态页面
接下来，我们要利用 **GitLab** 的 **CI** 为我们 **构建静态网页** 了。
在 **Hexo项目根目录** 新建一个名为`.gitlab-ci.yml`的文件，
然后编辑该文件，复制并粘贴以下构建代码：
```
image: node:14-alpine # use nodejs v14 LTS
cache:
  paths:
    - node_modules/

before_script:
  - npm install hexo-cli -g
  - npm install

pages:
  script:
    - hexo generate
  artifacts:
    paths:
      - public
  only:
    - master
```
**保存** 后 **Push** 到仓库，正常情况下 **GitLab CI** 已经开始为我们进行 **第一次构建** 了，你可以在 **仓库左侧 - CI/CD - Pipelines** 里看到构建情况
稍等片刻，正常情况下即可构建成功。
{% asset_img success.png 成功构建 %}
那么此时访问``${你的GitLab ID}.gitlab.io``就可以看到 **构建成功** 后的 **博客页面** 。
{% asset_img blog.png 博客页面 %}

以后每次在本地**创建文章**、**修改文章**，**Push**到**远程仓库**后**GitLab CI**将会**自动**为你**构建**，**几分钟后即可生效**。

### 四、绑定域名
如果购买了 **自己的域名** ，那么可以在 **仓库左侧 - Settings - Pages** 配置自己的 **域名** 。
{% asset_img domain.png PagesDomain %}
点击右侧 **New Domain** 添加 **域名** ，再根据提示在 **域名提供商处** 配置 **DNS** ，稍等片刻，访问 **域名** ，即可访问 **博客页面** 。