---
title: Windows Subsystem for Linux 2 的艺术
date: 2022-02-12 19:13:48
tags: [ 'WSL2', 'Linux' ]
---

# 一、前言
**微软赛高**！得益于其强大的技术支持，我们可以在 **Windows** 上无缝体验 **Linux (Windows Subsystem for Linux)** 甚至 **Android (Windows Subsystem for Android)** 。

将 **WSL2** 与 **Jetbrains系列IDE** 或 **VS Code** 结合，我们可以在 **Windows** 下无缝开发并调试 **Linux** 环境程序。

# 二、安装WSL2
## 1. 系统要求
笔者使用的是 **Windows 11 专业版**。
{% asset_img Win11.png Win11 %}

## 2. 启用 适用于Linux的Windows子系统
打开 **启用或关闭Windows功能**
{% asset_img GN.png GN %}

启用 **适用于Linux的Windows子系统**
{% asset_img GN2.png GN2 %}

## 3. 更新 WSL 2 Linux 内核
下载并安装[适用于 x64 计算机的 WSL2 Linux 内核更新包](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)

## 4. 将 WSL 2 设置为默认版本
```
wsl --set-default-version 2
```

## 5. 安装所选的 Linux 分发
这里以[Ubuntu 20.04 LTS](https://www.microsoft.com/store/apps/9n6svws3rx71)为例
你还可以选择

- [Ubuntu 18.04 LTS](https://www.microsoft.com/store/apps/9N9TNGVNDL3Q)
- [openSUSE Leap 15.1](https://www.microsoft.com/store/apps/9NJFZK00FGKV)
- [SUSE Linux Enterprise Server 12 SP5](https://www.microsoft.com/store/apps/9MZ3D1TRP8T1)
- [SUSE Linux Enterprise Server 15 SP1](https://www.microsoft.com/store/apps/9PN498VPMF3Z)
- [Kali Linux](https://www.microsoft.com/store/apps/9PKR34TNCV07)
- [Debian GNU/Linux](https://www.microsoft.com/store/apps/9MSVKQC78PK6)
- [Fedora Remix for WSL](https://www.microsoft.com/store/apps/9n6gdm4k2hnc)
- [Pengwin](https://www.microsoft.com/store/apps/9NV1GV1PXZ6P)
- [Pengwin Enterprise](https://www.microsoft.com/store/apps/9N8LP0X93VCP)
- [Alpine WSL](https://www.microsoft.com/store/apps/9p804crf0395)
- [Raft（免费试用版）](https://www.microsoft.com/store/apps/9msmjqd017x7)

打开**Microsoft Store**，搜索**Ubuntu 20.04**并安装
{% asset_img Ubuntu.png Ubuntu %}

## 6. 配置账号
打开**Ubuntu 20.04**（可从**Microsoft Store**打开），配置**用户名密码**。
{% asset_img Ubuntu2.png Ubuntu2 %}

## 7. 修改安装目录
默认**WSL**位于**C盘**，如果使用频率较高的话，你会发现C盘占用空间蹭蹭往上涨~
这里笔者选择修改安装目录到**D盘**

### 1) 查看已安装的Linux发行版本
```
wsl -l --all -v
```
{% asset_img List.png List %}

### 2) 导出Ubuntu到D盘
```
wsl --export Ubuntu-20.04 D:\WSL2\WSL-Ubuntu20.04.tar
```

### 3) 注销当前Ubuntu
```
wsl --unregister Ubuntu-20.04
```

### 4) 将导出的文件重新导入并安装
```
wsl --import Ubuntu-20.04 D:\WSL2\WSL-Ubuntu20.04 D:\WSL2\WSL-Ubuntu20.04.tar --version 2
```

### 5) 重新设置用户名（将$USERNAME修改为安装时的用户名，密码不变）
```
ubuntu2004 config --default-user $USERNAME
```

### 6) 登录到WSL2
{% asset_img Ubuntu3.png Ubuntu3 %}

# 三、环境配置
## 1. 软件源修改为[阿里源](https://developer.aliyun.com/mirror/ubuntu)
使用**Vim**打开`sources.list`
```
sudo vim /etc/apt/sources.list
```
键入`:%d`清空
粘贴**阿里源**代码

```
deb http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
```
按`Esc`，键入`:wq`保存并退出
更新**源**并升级

```
sudo apt-get update && sudo apt-get upgrade
```

## 2. 开启Systemd
某些程序需要**Systemd**的支持，因此我们使用[ubuntu-wsl2-systemd-script](https://github.com/DamionGans/ubuntu-wsl2-systemd-script)来开启。
安装**git**

```
sudo apt-get install git
```
克隆脚本仓库
```
git clone https://github.com/DamionGans/ubuntu-wsl2-systemd-script.git
```
跳转到`ubuntu-wsl2-systemd-script`目录
```
cd ubuntu-wsl2-systemd-script
```
运行安装脚本
```
bash ubuntu-wsl2-systemd-script.sh
```
重启**Ubuntu**（**宿主机终端执行**）
```
wsl --shutdown Ubuntu-20.04
```
再次登录到**Ubuntu**，**验证Systemd**
```
systemctl
```
{% asset_img Systemd.png Systemd %}
如上图则**配置成功**

# 四、配置并使用宿主机代理
创建并编辑`.proxyrc`文件
```
vim .proxyrc
```
输入以下代码（`$port`为宿主机代理端口，请自行修改）
```
#!/bin/bash
host_ip=$(cat /etc/resolv.conf |grep "nameserver" |cut -f 2 -d " ")
export ALL_PROXY="http://$host_ip:$port"
```
按`Esc`，键入`:wq`保存并退出
以后每次连接宿主机代理只需`source .proxyrc`即可生效

# 五、参考文章
- [Win10使用WSL2正确姿势](https://zhuanlan.zhihu.com/p/165508059)
- [win10 wsl2修改默认安装目录到其他盘](https://blog.csdn.net/w851685279/article/details/108904106)
- [为 WSL2 一键设置代理](https://zhuanlan.zhihu.com/p/153124468)