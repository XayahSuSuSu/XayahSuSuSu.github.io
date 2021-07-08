---
title: 利用清华和科大镜像源全程国内同步Android源码
date: 2021-07-08 22:36:46
tags:
---
# [原CSDN博客地址](https://blog.csdn.net/weixin_43267515/article/details/112528652?spm=1001.2014.3001.5501)

> 环境： 2021-1-12
> 
> 系统： Ubuntu 20.04.1 LTS

网上类似的资料已经很多了，但由于 **时效性** 或多或少会有一些问题，我在这里记录下 **当前环境** 下成功同步的方法
## 具体步骤
1. 安装必要工具
    ```
    sudo apt install curl
    ```
    ```
    sudo apt install vim
    ```
    ```
    sudo apt install git
    ```

2. 在 **~/bin** 目录下载 **repo** ：
    ```
    mkdir ~/bin
    ```
    ```
    cd ~/bin
    ```
    ```
    curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo -o repo
    ```
    ```
    chmod +x repo
    ```

3. 配置 **repo** 环境变量:

   ```
   sudo vim ~/.bashrc
   ```

   1) 输入 `i` 进入编辑模式

   2) 在 **末尾** 添加：
        ```
        export PATH=~/bin:$PATH
        ```
        ```
        export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'
        ```

    1) 按 `Esc` 退出编辑模式，输入 `:wq` 保存并退出

    2) 使环境变量生效： `source ~/.bashrc`

4. 在 **清华/科大镜像源** 下载 **初始包** 到 **工作目录** （90GB左右）：

    [清华源初始包](https://mirrors.tuna.tsinghua.edu.cn/aosp-monthly/aosp-latest.tar)

    [科大源初始包](https://mirrors.ustc.edu.cn/aosp-monthly/)

5. 在工作目录下解压： 
   
   ```
   tar xf aosp-latest.tar
   ```
   
    1) 经测试清华源下载的初始包直接 `repo sync` 会出现奇奇怪怪的问题，解决办法是：
    
        显示隐藏文件 → 打开 `aosp/.repo/manifests.git/config`

    2) 修改其中的 **url** 为：

        `url = git://mirrors.ustc.edu.cn/aosp/platform/manifest`

6. 修改 **符号链接** 将 **python3** 默认指向 **python** 命令：
    ```
    sudo rm /usr/bin/python
    ```
    ```
    sudo ln -s /usr/bin/python3 /usr/bin/python
    ```

7. 设置账号的 **缺省身份标识** ：
    ```
    git config --global user.email "you@example.com"
    ```
    ```
    git config --global user.name "Your Name"
    ```

8. 回到 **aosp** 目录下同步源码： `repo sync` 
   
9.  成功 **同步源码** !