---
title: AARCH64交叉编译笔记
date: 2021-07-07 13:38:27
tags: [ 'Cross Compile', 'Cmake', 'Python' ]
---
# [AARCH64交叉编译笔记](https://github.com/XayahSuSuSu/Notes-Cross-Compile)

> Android NDK 版本：r22b (22.1.7171670)
>
> 目标平台：AArch64

## 环境准备

1. 下载 [Android NDK](https://developer.android.google.cn/ndk/downloads/)

2. 解压

## **CMake** 交叉编译 (以 [Brotli](https://github.com/google/brotli) 为例)

1. 更新Cmake版本 (最低 **3.19** )

   以 **cmake-3.21.0-rc1** 为例：
   
   ```
   wget https://cmake.org/files/v3.21/cmake-3.21.0-rc1.tar.gz

   tar -xvf cmake-3.21.0-rc1.tar.gz

   cd cmake-3.21.0-rc1

   ./configure

   sudo make

   sudo make install

   cmake --version         # 检查版本
   ```
   

2. 克隆 Brotli
   ```
   git clone https://github.com/google/brotli
   ```

3. 交叉编译
   ```
   cd brotli

   mkdir out && cd out

   export NDK=/home/xayah/Compile/NDK             # NDK根目录绝对路径

   export ABI=arm64-v8a                           # ABI配置(arm64-v8a 即为 AArch64)

   export MINSDKVERSION=24                        # 最小目标SDK版本配置(24 即为 Android 7.0)

   cmake \
      -DCMAKE_TOOLCHAIN_FILE=$NDK/build/cmake/android.toolchain.cmake \
      -DANDROID_ABI=$ABI \
      -DANDROID_NATIVE_API_LEVEL=$MINSDKVERSION \
      -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=./installed ..

   cmake --build . --config Release
   ```

4. 编译产物生成于 **out** 文件夹中

## **Autoconf Makefile** 交叉编译 (以 [Make-4.3](http://ftp.gnu.org/gnu/make/make-4.3.tar.gz) 为例)

1. 下载 **make-4.3.tar.gz** 并解压

2. 交叉编译
   ```
   cd make-4.3

   export NDK=/home/xayah/Compile/NDK                              # NDK根目录绝对路径

   export TOOLCHAIN=$NDK/toolchains/llvm/prebuilt/linux-x86_64     # 交叉编译链路径

   export TARGET=aarch64-linux-android                             # 交叉编译目标

   export API=24                                                   # 最小目标SDK版本配置(24 即为 Android 7.0)

   export AR=$TOOLCHAIN/bin/llvm-ar

   export CC=$TOOLCHAIN/bin/$TARGET$API-clang

   export AS=$CC

   export CXX=$TOOLCHAIN/bin/$TARGET$API-clang++

   export LD=$TOOLCHAIN/bin/ld

   export RANLIB=$TOOLCHAIN/bin/llvm-ranlib

   export STRIP=$TOOLCHAIN/bin/llvm-strip

   ./configure --enable-checking=release --enable-languages=c,c++ --disable-multilib --host $TARGET

   make
   ```

   注：其中的 **AR** 、 **CC** 、 **AS** 、 **CXX** 、 **LD** 、 **RANLIB** 、 **STRIP** 等决定于Makefile，我这里图方便就直接复制了 [NDK文档](https://developer.android.google.cn/ndk/guides/other_build_systems#autoconf) 中的变量，恰好已经覆盖完全，不会报错。当交叉编译报错时，请自行添加相应变量。

3. 编译产物生成于当前文件夹中

## **非 Autoconf Makefile** 交叉编译 (以 [Dtc](https://github.com/dgibson/dtc) 为例)

1. 克隆 **dtc**
   ```
   git clone https://github.com/dgibson/dtc
   ```

2. 交叉编译
   ```
   cd dtc

   export NDK=/home/xayah/Compile/NDK                              # NDK根目录绝对路径

   export TOOLCHAIN=$NDK/toolchains/llvm/prebuilt/linux-x86_64     # 交叉编译链路径

   export TARGET=aarch64-linux-android                             # 交叉编译目标

   export API=24                                                   # 最小目标SDK版本配置(24 即为 Android 7.0)

   export AR=$TOOLCHAIN/bin/llvm-ar

   export CC=$TOOLCHAIN/bin/$TARGET$API-clang

   export AS=$CC

   export CXX=$TOOLCHAIN/bin/$TARGET$API-clang++

   export LD=$TOOLCHAIN/bin/ld

   export RANLIB=$TOOLCHAIN/bin/llvm-ranlib

   export STRIP=$TOOLCHAIN/bin/llvm-strip

   make \
      AR=$AR \
      CC=$CC \
      AS=$AS \
      CXX=$CXX \
      LD=$LD \
      RANLIB=$RANLIB \
      STRIP=$STRIP \
      dtc
   ```

   注：其中的 **AR** 、 **CC** 、 **AS** 、 **CXX** 、 **LD** 、 **RANLIB** 、 **STRIP** 等决定于Makefile，我这里图方便就直接复制了 [NDK文档](https://developer.android.google.cn/ndk/guides/other_build_systems#autoconf) 中的变量，恰好已经覆盖完全，不会报错。当交叉编译报错时，请自行添加相应变量。

3. 编译产物生成于当前文件夹中

## **Python** 交叉编译 (以 [Imgextractor.py](https://github.com/xiaoxindada/SGSI-build-tool/blob/11/tool_bin/imgextractor.py) 为例)

> 利用 **Soong 编译系统** 将 Python 交叉编译为目标平台二进制ELF文件
>
> 以 **LineageOS源码** 中的 **Soong 编译系统** 为例
>
> 需要有一定 **Android源码** 编译基础
>
> 目标平台：AArch64

1. 下载 **imgextractor.py**

2. 处理第三方模块

   注意到 **imgextractor.py** 中：
   ```
   import ext4
   ```
   这是一个自定义模块，位于其同目录下 **ext4.py**
   
   我查阅了 [Soong 编译系统
文档](https://source.android.google.cn/setup/build)，暂时没有发现编译时处理第三方模块的解决方案，若同时交叉编译**imgextractor.py**、**ext4.py**并放置在同一目录下，仍会报错找不到模块 **ext4**

   因此，我们将 **ext4.py** 合并到 **imgextractor.py** 中：

   1) 复制 **ext4.py** 中所有代码到 **imgextractor.py** 的文件首
   2) 移除 **ext4**：

      将
      ```
      import ext4, string, struct
      ```
      改为
      ```
      import string, struct
      ```
   3) 处理类：

      搜索所有 **ext4.** 并删除：

      即将
      ```
      root = ext4.Volume(file).root
      ```
      改为
      ```
      root = Volume(file).root
      ```
   4) 保存

3. 编写 **Android.bp**
   ```
   // Copyright (C) 2017 The Android Open Source Project
   //
   // Licensed under the Apache License, Version 2.0 (the "License");
   // you may not use this file except in compliance with the License.
   // You may obtain a copy of the License at
   //
   //      http://www.apache.org/licenses/LICENSE-2.0
   //
   // Unless required by applicable law or agreed to in writing, software
   // distributed under the License is distributed on an "AS IS" BASIS,
   // WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   // See the License for the specific language governing permissions and
   // limitations under the License.

   //##################################################

   python_test {
      name: "imgextractor",
      main: "imgextractor.py",
      srcs: [
         "imgextractor.py",
      ],
      version: {
         py3: {
               embedded_launcher: true,
               enabled: true,
         },
      },
   }
   ```
   注：
   如果 **待编译Python脚本** 是 **Python2** 版本，则：
   ```
   // Copyright (C) 2017 The Android Open Source Project
   //
   // Licensed under the Apache License, Version 2.0 (the "License");
   // you may not use this file except in compliance with the License.
   // You may obtain a copy of the License at
   //
   //      http://www.apache.org/licenses/LICENSE-2.0
   //
   // Unless required by applicable law or agreed to in writing, software
   // distributed under the License is distributed on an "AS IS" BASIS,
   // WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   // See the License for the specific language governing permissions and
   // limitations under the License.

   //##################################################

   python_test {
      name: "xxx",
      main: "xxx.py",
      srcs: [
         "xxx.py",
      ],
      version: {
         py2: {
               embedded_launcher: true,
               enabled: true,
         },
      },
   }
   ```

4. 将 **imgextractor.py** 、**Android.bp** 放置在LineageOS源码任意位置(需包含在文件夹中)：

   以 **/home/xayah/LineageOS/system/imgextractor** 为例

   假设你现在位于LineageOS源码根目录，打开终端：
   ```
   . build/envsetup.sh

   lunch lineage_cas-userdebug

   mmma system/imgextractor
   ```

   注意： [cas(Mi 10 Ultra)](https://github.com/XayahSuSuSu/android_device_xiaomi_cas-oss) 可使用其他AArch64平台的手机的设备树

5. 编译产物生成于 **out/target/product/cas/testcases/imgextractor/arm64/imgextractor**