---
title: Android 6.0+ 动态权限获取(包含对Android 10+Android11+的支持)
date: 2021-07-08 22:11:07
tags: [ 'Android', '权限' ]
---
# [原CSDN博客地址](https://blog.csdn.net/weixin_43267515/article/details/111239279?spm=1001.2014.3001.5501)
从 **Android6.0** 开始，权限获取不再是简单地在 **AndroidManifest.xml** 添加几行代码了， **Google** 引入了 **动态权限** 的概念，需要在代码中添加。
# 添加步骤（以读取和写入权限为例）：
1. **AndroidManifest.xml** 中添加权限
    ```xml
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    ```
2. 从 **Android10** 开始，还需要添加：
    ```xml
    <application
        android:requestLegacyExternalStorage="true"
        ...
    </application>
    ```

3. 在 **Activity** 相应位置调用：
    ```java
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.WRITE_EXTERNAL_STORAGE) == -1) {
            // 没有Write权限，动态获取
            ActivityCompat.requestPermissions(MainActivity.this, new String[]{
                    Manifest.permission.WRITE_EXTERNAL_STORAGE}, 1);
            Log.d("TAG", "onCreate: 申请获得Write权限！");
        } else {
            Log.d("TAG", "onCreate: 已获得Write权限！");
        }
    }
    ```
    然而在 **Android11** 开始，** WRITE_EXTERNAL_STORAGE** 等特殊权限的获取又发生了变化...

    {% asset_img WRITE_EXTERNAL_STORAGE.png Android11+的特殊权限变化 %}

    当仍沿用 **Android10-** 的权限获取方式时，会在调用权限时 **抛出异常** 

    因为在 **Andorid11+** 中， **Google** 添加了一个新的权限： **MANAGE_EXTERNAL_STORAGE**

    当 **APP** 动态请求权限时，会引导用户进入一个 **权限设置界面**

    {% asset_img BuildPropHelper.png 文件所有权设置界面 %}

    所以，在Android11+中：
    
    AndroidManifest.xml中添加权限：

    ```xml
    <uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE" />
    ```

    {% asset_img MANAGE_EXTERNAL_STORAGE.png 该权限仅支持Android11+ %}

    该权限仅支持 **Android11+** 

4. 在Activity相应位置调用：

    ```java
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        // new一个intent转到系统设置界面
        Intent intent = new Intent(Settings.ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION);
        intent.setData(Uri.parse("package:" + mContext.getPackageName()));
        // 1024为REQUEST_CODE
        startActivityForResult(intent, 1024);
    }
    ```

5. 所以合并起来的代码是：

    ```java
    public void getWriteRight() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            // new一个intent转到系统设置界面
            Intent intent = new Intent(Settings.ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION);
            intent.setData(Uri.parse("package:" + mContext.getPackageName()));
            // 1024为REQUEST_CODE
            startActivityForResult(intent, 1024);
        } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            if (ActivityCompat.checkSelfPermission(this, Manifest.permission.WRITE_EXTERNAL_STORAGE) == -1) {
                // 没有Write权限，动态获取
                ActivityCompat.requestPermissions(MainActivity.this, new String[]{
                        Manifest.permission.WRITE_EXTERNAL_STORAGE}, 1);
                Log.d("TAG", "onCreate: 申请获得Write权限！");
            } else {
                Log.d("TAG", "onCreate: 已获得Write权限！");
            }
        }
    }
    ```