---
title: Vue + Electron 搭建跨端桌面应用
date: 2022-07-02 17:34:37
tags: [ 'Vue', 'Electron', '跨端' ] 
---

# 一、前言
**跨端应用开发**一直都是**热门需求**，目前已经有很多**方案**可以实现**跨端开发**，然而 **[Flutter](https://flutter.dev/)** 对于目前热门的**高刷屏适配摆烂**， **[JetPack Compose](https://developer.android.com/jetpack/compose)** 仍在**发育**之中， **[Electron](https://www.electronjs.org/)** 优势在于**前端技术栈**却亦**受限于此**，但如果我们熟悉**前端开发**，**Electron**仍是一个不错的选择。

本文以 **[Vue2](https://cn.vuejs.org/)** 热门框架 **[Quasar](https://quasar.dev/)** 为例，记录其**Windows桌面程序搭建流程**。

# 二、流程
## 1. 安装NodeJS
**[NVM](https://github.com/nvm-sh/nvm)** 对于**Node版本管理**非常方便，请参考其**安装文档**安装 **[NodeJS](https://nodejs.org/en/)** 。

## 2. 安装Yarn
相比于**NPM**，我更青睐 **[Yarn](https://github.com/yarnpkg/yarn)** 。
安装
```
npm install --global yarn
```
验证
```
yarn --version
```

## 3. 安装Quasar CLI
**Quasar脚手架**可以方便地为我们**创建Quasar工程**。
```
yarn global add @quasar/cli
```

## 4. 创建工程
```
yarn create quasar
```
根据**提示**选择**相关配置**来**创建Quasar工程**。

## 5. 设置Yarn Electron淘宝源
```
yarn config set electron_mirror https://cdn.npm.taobao.org/dist/electron/
```

## 5. 添加Quasar Electron模式
先进入**工程目录**，再添加**Electron模式**。
```
quasar mode add electron
```

## 6. 调试
```
quasar dev -m electron
```
{% asset_img 调试.png 调试 %}