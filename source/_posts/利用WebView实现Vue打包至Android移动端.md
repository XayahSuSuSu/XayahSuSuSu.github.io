---
title: 利用WebView实现Vue打包至Android移动端
date: 2021-12-06 22:57:19
tags: [ 'Android', 'Vue', 'WebView' ]
---
## 前言
Vue打包至移动端目前已经有许多解决方案，例如**HBuilder**、**Cordova**等等。但个人认为过程实在是太过于繁杂，且有的还需要**联网验证**，于是有了这篇文章。原理就是利用原生**Android WebView**本地加载Vue打包后的**静态HTML**，本文章以**Android**为例，使用**Android Studio**本地打包，**IOS端**同理。

## 准备
工具：[Android Studio](https://developer.android.google.cn/studio/)
源码：`npm run build`之后的**静态资源**，例如`dist`文件夹

## 步骤
#### 1. 打开Android Studio
{% asset_img Open.png Open %}

#### 2. 新建项目 New Project - Empty Activity - Next
填写**Name（应用名称）、Package Name（包名）、Language（以Kotlin为例）、Minimum SDK（应用支持的最低Android版本）**，然后点击**Finish**创建项目。初次安装AS并创建项目需要联网下载Gradle等依赖，请耐心等待。
{% asset_img New.png New %}

#### 3. 主题设置为NoActionBar
默认主题会有一个**ActionBar**，需要去掉以保证加载后的Vue UI统一性。
首先在左侧打开**themes**下的两个主题文件，其中一个是**深色模式**的主题，我们需要将两个都改为**NoActionBar**。这里以**浅色模式**为例。
{% asset_img ActionBar.png ActionBar %}
↓
{% asset_img NoActionBar.png NoActionBar %}

#### 4. 设置状态栏背景颜色
同样，在**浅色模式**主题文件中，修改`android:statusBarColor`为你想要的颜色，并将**状态栏文字图标**等设为**深色**
```
<item name="android:windowLightStatusBar">true</item>
```
{% asset_img statusBarColor.png statusBarColor %}
深色模式同理。
{% asset_img statusBarColorDark.png statusBarColorDark %}

#### 5. 布局添加WebView
打开`activity_main.xml`：
{% asset_img activity_main.png activity_main %}
删除**TextView**并添加**WebView**：
```
    <WebView
        android:id="@+id/web_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
```
{% asset_img WebView.png WebView %}

#### 6. 本地加载打包后的HTML
在项目目录下`app\src\main`创建`assets`文件夹，将打包后的**dist**放入**assets**：
{% asset_img assets.png assets %}
打开`MainActivity.kt`
{% asset_img MainActivity.png MainActivity %}
**绑定组件**：
```
val webView: WebView = findViewById(R.id.web_view)
```
**本地加载HTML**：

```
webView.loadUrl("file:////android_asset/dist/index.html")
```
**允许JS交互**：

```
webView.settings.javaScriptEnabled = true
```
{% asset_img code.png code %}
**打开`AndroidManifest.xml`**：
{% asset_img AndroidManifest.png AndroidManifest %}
**添加INTERNET权限**：

```
    <uses-permission android:name="android.permission.INTERNET" />
```
{% asset_img INTERNET.png INTERNET %}

#### 8. 更换APP图标
左栏右键`res - drawable - New - Image Asset`
{% asset_img icon_1.png icon_1 %}
配置图标**前景**、**背景**：
{% asset_img icon_2.png icon_2 %}
{% asset_img icon_3.png icon_3 %}
Next - Finish
{% asset_img icon_4.png icon_4 %}
删除`ic_launcher.webp`、`ic_launcher_round.webp`（如果有的话）
{% asset_img icon_5.png icon_5 %}
{% asset_img icon_6.png icon_6 %}
{% asset_img icon_7.png icon_7 %}
{% asset_img icon_8.png icon_8 %}

#### 7. 调试

##### 1) Android调试
以**真机调试**为例，当然也可以使用**模拟器**。
真机调试需要在**开发者模式**开启**USB调试**模式。
点击右上方**绿色三角形**运行。
{% asset_img run.png run %}
稍等片刻，会自动启动APP：
{% asset_img app_0.png app_0 %}
{% asset_img app_1.png app_1 %}

##### 2) WebView调试
需要在**开发者模式**开启**USB调试**模式。
在`MainActivity.kt`中添加以下代码：
```
WebView.setWebContentsDebuggingEnabled(true)
```
{% asset_img debug.png debug %}
重新在AS中调试运行App，打开谷歌浏览器，地址栏输入：`chrome://inspect/#devices`进入远程调试工具。
稍等片刻工具会自动载入已运行的WebView程序，点击`inspect`进入调试模式。
{% asset_img devices.png devices %}

#### 8. 编译并签名
点击工具栏 **Build - Generate Signed Bundle / APK ...**
{% asset_img generate.png generate %}
选择**APK - Next**
{% asset_img next.png next %}
新建签名 **Create new...** 填写**签名信息**
{% asset_img newjks.png newjks %}
选择刚刚创建的**签名文件 xxx.jks** 填写对应信息后点击 **Next**
{% asset_img sign.png sign %}
{% asset_img sign_next.png sign %}
选择**release**并点击**Finish**
{% asset_img sign_finish.png sign_finish %}
**项目目录**下`app\release`即是**编译完成**的apk文件。
{% asset_img successfully.png successfully %}
{% asset_img release.png release %}

#### 9. 常见问题

##### 1) 跨域
建议从后端解决跨域问题，以Flask为例：
```
pip3 install flask-cors
```
```
from flask import Flask
from flask_cors import CORS

app = Flask(__name__)
app.config['JSON_AS_ASCII'] = False
# 允许全局跨域
CORS(app, supports_credentials=True)
```
##### 2) WebView调试出现ERR_CLEARTEXT_NOT_PERMITTED
{% asset_img ERR_CLEARTEXT_NOT_PERMITTED.png ERR_CLEARTEXT_NOT_PERMITTED %}
打开`AndroidManifest.xml`，在`application`标签下配置：
```
    <application
        ······
        android:usesCleartextTraffic="true"
        ······
    </application>
```