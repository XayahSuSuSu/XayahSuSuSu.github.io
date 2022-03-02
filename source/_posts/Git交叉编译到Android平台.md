---
title: Git交叉编译到Android平台
date: 2021-08-02 12:05:30
tags: [ 'Git', '交叉编译', 'Android' ]
---

# 前言

这是一个很久之前的想法了，但是之前一直编译不成功。

这两天仔细研究了下，证明还是可行的。

> Android NDK 版本：r23b
>
> 编译环境：Ubuntu 20.04
> 
> NDK目录：`~/NDK`

# 步骤

## 一、下载交叉编译所需源码

想要交叉编译**Git**，需要先**交叉编译Curl**和**Zlib**，**新版本**的**NDK**已经包含**预编译Zlib**，所以我们**只需要交叉编译Curl**即可。

至于为什么要交叉编译**Curl**，是因为**git clone**命令在**clone https等仓库**时，需要**依赖该库创建的git-remote-https等EFL文件**。否则，在**clone**时会发生**找不到remote-https等错误**。

而若**Curl**依赖**OpenSSL**，因此还得先交叉编译**OpenSSL**。

下载 **[Git](https://github.com/git/git)** 、 **[Curl](https://curl.se/download.html)** 和 **[OpenSSL](https://github.com/openssl/openssl)** 的源码并**解压**。

## 二、交叉编译OpenSSL并安装到NDK
在**OpenSSL**官方仓库中，可以找到编译到**Android**平台的 **[文档](https://github.com/openssl/openssl/blob/master/NOTES-ANDROID.md)** 。
导出**NDK**临时变量
```
export ANDROID_NDK_ROOT=~/NDK
```
导出**PATH**临时变量
```
export PATH=$ANDROID_NDK_ROOT/toolchains/llvm/prebuilt/linux-x86_64/bin:$ANDROID_NDK_ROOT/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin:$PATH
```
配置`Makefile`
```
./Configure android-arm64 -D__ANDROID_API__=26 --prefix=$ANDROID_NDK_ROOT/toolchains/llvm/prebuilt/linux-x86_64/sysroot
```
编译
```
make -j128
```
安装到`prefix`目录
```
make install -j128
```

## 三、交叉编译Curl并安装到NDK

打开**Curl**的源码，我们可以发现它提供了两种编译方式，**Autoconf Makefile** 和 **Cmake**。

使用**Autoconf Makefile**方式可以编译**带OpenSSL**的**Curl**，但是由于某种未知原因，在后面在**编译Git**时**无法识别**。

因此我们选择**Cmake**方式。

在`INSTALL.cmake`中我们可以得到一些编译的信息：

```
Current flaws in the curl CMake build
=====================================

   Missing features in the cmake build:

   - Builds libcurl without large file support
   - Does not support all SSL libraries (only OpenSSL, Schannel,
     Secure Transport, and mbed TLS, NSS, WolfSSL)
   - Doesn't allow different resolver backends (no c-ares build support)
   - No RTMP support built
   - Doesn't allow build curl and libcurl debug enabled
   - Doesn't allow a custom CA bundle path
   - Doesn't allow you to disable specific protocols from the build
   - Doesn't find or use krb4 or GSS
   - Rebuilds test files too eagerly, but still can't run the tests
   - Doesn't detect the correct strerror_r flavor when cross-compiling (issue #1123)
```

由此可知，利用**Cmake**编译出来的**Curl不带SSL库**，但这并**不影响**我们后续**编译Git**和**相应的git-remote-https等二进制文件**。

**cd**到**解压后**的**Curl根目录**，输入：

```
mkdir out && cd out

export NDK=~/NDK                     # NDK根目录绝对路径

export ABI=arm64-v8a                           # ABI配置(arm64-v8a 即为 AArch64)

export MINSDKVERSION=26                        # 最小目标SDK版本配置(26 即为 Android 8.0)

cmake \
   -DCMAKE_TOOLCHAIN_FILE=$NDK/build/cmake/android.toolchain.cmake \
   -DANDROID_ABI=$ABI \
   -DANDROID_NATIVE_API_LEVEL=$MINSDKVERSION \
   -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=$NDK/toolchains/llvm/prebuilt/linux-x86_64/sysroot ..

cmake --build . --config Release
make install
```

## 四、交叉编译Git

接下来是我们的**重头戏**了。

进入**Git源码解压后的根目录**，可以发现**Git**有两种编译方式，一是**非Autoconf Makefile**，二是**Autoconf Makefile**，**前者**我尝试过多次，皆以**失败**告终。所以这次我们尝试**后者**。

我们的**宿主机**因为**无法测试Android平台上的二进制文件**，所以我们把**测试代码**删掉。

进入`configure.ac`，**删除**：
```
#
# Define FREAD_READS_DIRECTORIES if your are on a system which succeeds
# when attempting to read from an fopen'ed directory.
AC_CACHE_CHECK([whether system succeeds to read fopen'ed directory],
 [ac_cv_fread_reads_directories],
[
AC_RUN_IFELSE(
	[AC_LANG_PROGRAM([AC_INCLUDES_DEFAULT],
		[[
		FILE *f = fopen(".", "r");
		return f != NULL;]])],
	[ac_cv_fread_reads_directories=no],
	[ac_cv_fread_reads_directories=yes])
])
if test $ac_cv_fread_reads_directories = yes; then
	FREAD_READS_DIRECTORIES=UnfortunatelyYes
else
	FREAD_READS_DIRECTORIES=
fi
GIT_CONF_SUBST([FREAD_READS_DIRECTORIES])
```
**再删除**：
```
#
# Define SNPRINTF_RETURNS_BOGUS if your are on a system which snprintf()
# or vsnprintf() return -1 instead of number of characters which would
# have been written to the final string if enough space had been available.
AC_CACHE_CHECK([whether snprintf() and/or vsnprintf() return bogus value],
 [ac_cv_snprintf_returns_bogus],
[
AC_RUN_IFELSE(
	[AC_LANG_PROGRAM([AC_INCLUDES_DEFAULT
		#include "stdarg.h"

		int test_vsnprintf(char *str, size_t maxsize, const char *format, ...)
		{
		  int ret;
		  va_list ap;
		  va_start(ap, format);
		  ret = vsnprintf(str, maxsize, format, ap);
		  va_end(ap);
		  return ret;
		}],
		[[char buf[6];
		  if (test_vsnprintf(buf, 3, "%s", "12345") != 5
		      || strcmp(buf, "12")) return 1;
		  if (snprintf(buf, 3, "%s", "12345") != 5
		      || strcmp(buf, "12")) return 1]])],
	[ac_cv_snprintf_returns_bogus=no],
	[ac_cv_snprintf_returns_bogus=yes])
])
if test $ac_cv_snprintf_returns_bogus = yes; then
	SNPRINTF_RETURNS_BOGUS=UnfortunatelyYes
else
	SNPRINTF_RETURNS_BOGUS=
fi
GIT_CONF_SUBST([SNPRINTF_RETURNS_BOGUS])
```

观察源码结构，可以发现并没有`configure`文件，但存在`configure.ac`，所以我们可以**make**一个`configure`。

**cd**到源码**根目录**，输入：
```
make configure
```

此时即可生成`configure`。

**git**编译时会默认编译**pthread**，而**Android由于性能及安全原因**，放弃了**glibc**在其平台上的支持，所以相应地交叉编译链也不含有这个库。

**Android**有其**替代方案**，所以我们在`make`时将`NEEDS_LIBRT`赋值为空从而不编译它即可。
```
ifdef NEEDS_LIBRT
	EXTLIBS += -lrt
endif
```

而在`configure.ac`中可以发现：

```
AC_ARG_ENABLE([pthreads],
 [AS_HELP_STRING([--enable-pthreads=FLAGS],
  [FLAGS is the value to pass to the compiler to enable POSIX Threads.]
  [The default if FLAGS is not specified is to try first -pthread]
  [and then -lpthread.]
  [--disable-pthreads will disable threading.])],
[
if test "x$enableval" = "xyes"; then
   AC_MSG_NOTICE([Will try -pthread then -lpthread to enable POSIX Threads])
elif test "x$enableval" != "xno"; then
   PTHREAD_CFLAGS=$enableval
   AC_MSG_NOTICE([Setting '$PTHREAD_CFLAGS' as the FLAGS to enable POSIX Threads])
else
   AC_MSG_NOTICE([POSIX Threads will be disabled.])
   NO_PTHREADS=YesPlease
   USER_NOPTHREAD=1
fi],
[
   AC_MSG_NOTICE([Will try -pthread then -lpthread to enable POSIX Threads.])
])
```

所以我们可以使用 **--disable-pthreads** 参数来**取消**编译 **phread** 相关部分。

于是接下来我们输入以下命令

```
export NDK=~/NDK                                      # NDK根目录绝对路径

export TOOLCHAIN=$NDK/toolchains/llvm/prebuilt/linux-x86_64     # 交叉编译链路径

export TARGET=aarch64-linux-android                             # 交叉编译目标

export API=26                                                   # 最小目标SDK版本配置(26 即为 Android 8.0)

export AR=$TOOLCHAIN/bin/llvm-ar

export CC=$TOOLCHAIN/bin/$TARGET$API-clang

export AS=$CC

export CXX=$TOOLCHAIN/bin/$TARGET$API-clang++

export LD=$TOOLCHAIN/bin/ld

export RANLIB=$TOOLCHAIN/bin/llvm-ranlib

export READELF=$TOOLCHAIN/bin/readelf

export STRIP=$TOOLCHAIN/bin/llvm-strip

./configure --host=$TARGET --prefix=/data/local/tmp --disable-pthreads
```

注意这里的`--prefix`，需要定义为**Android**环境下**git运行目录**，否则会出现**找不到templates**等问题。

然后**编译**：

```
make NEEDS_LIBRT= NO_TCLTK=1 -j128
```

注意这里的`NO_TCLTK`，如果不定义它的话，会默认编译`git-gui`，这不是我们需要的，所以将其定义为`1`。

## 五、推送并测试
先安装到宿主机`~/git/install`
```
make install NEEDS_LIBRT= NO_TCLTK=1 DESTDIR=~/git/install -j128
```
可得如下**产物**
```
xayah@xayah-virtual-machine:~/git/install/data/local/tmp$ ls
bin  libexec  share
```
将`~/git/install/data/local/tmp` **打包**并**推送**到**Android** `/data/local/tmp`目录
由于一些**魔法因素**，**打包**为**zip**竟足足有**200M+**！可能是因为**压缩算法**不同，**打包**为**tar.xz**就会**小很多**，大概**10M**左右。

```
PS C:\Users\Xayah\Desktop> adb push git.zip /data/local/tmp
git.zip: 1 file pushed, 0 skipped. 38.1 MB/s (648532238 bytes in 16.219s)
PS C:\Users\Xayah\Desktop> adb shell
cas:/ $ cd /data/local/tmp
cas:/data/local/tmp $ unzip git.zip >/dev/null
cas:/data/local/tmp $ cd bin
cas:/data/local/tmp/bin $ ls
git  git-cvsserver  git-receive-pack  git-shell  git-upload-archive  git-upload-pack  gitk
```
让我们`git clone`试试
```
cas:/data/local/tmp/bin $ ./git clone https://github.com/git/git mGit
Cloning into 'mGit'...
remote: Enumerating objects: 311108, done.
remote: Counting objects: 100% (72/72), done.
remote: Compressing objects: 100% (32/32), done.
remote: Total 311108 (delta 41), reused 70 (delta 40), pack-reused 311036
Receiving objects: 100% (311108/311108), 164.19 MiB | 10.32 MiB/s, done.
Resolving deltas: 100% (232238/232238), done.
Segmentation fault
```

**Segmentation fault**！这可是个头疼的事。不过没有关系，我们可以使用 **gdbserver** 调试，看看到底是**哪里**出了**问题**。

首先将**NDK**中相应的**gdbserver**推送到**Android平台**，我的手机是**aarch64**（现在**大多数安卓手机都是这个平台**），也就是**arm64**，所以推送**该文件夹**下的**gdbserver**即可。

```
adb push gdbserver /data/local/tmp/bin
```

接下来**新建**一个**终端**并**forward**一个**自定义端口**：

```
adb forward tcp:12345 tcp:12345
```

接下来在另一个**终端**中进入**adb shell**：

```
./gdbserver 0.0.0.0:12345 ./git clone https://github.com/git/git mGit
```

现在**gdbserver**已经启动了，我们转向另一个**终端**，启动**gdb**：
{% asset_img gdb.png gdb %}

现在**gdb**和**gdbserver**均已启动，我们**指定目标端口**：
{% asset_img remote.png remote %}

输入`c`继续运行
{% asset_img sigsegv.png sigsegv %}
段错误出现了！可以看出错误是出在`copy_gecos()`这个函数。
在源码中搜索这个函数，可以在`ident.c`中找到。
定位至相应位置
{% asset_img copy_gecos.png copy_gecos %}
这个函数看起来貌似和`git config`有一丝关系，难道是因为我们没有定义**git用户信息**？
让我们试试
```
./git config --global user.name "Xayah"
```
{% asset_img config.png config %}

果然报错了！看来是`.gitconfig`没有创建成功，在源码中搜索`.gitconfig`，可以发现**git**获取`.gitconfig`的路径是`$HOME/.gitconfig`

那么我们试试把`$HOME`定义为**当前目录**
```
export HOME=/data/local/tmp/bin
```
再配置用户信息
```
./git config --global user.name "Xayah"
./git config --global user.email zds1249475336@gmail.com
```
这次没有报错了，并且当前目录也成功生成了`.gitconfig`
{% asset_img config2.png config2 %}

这次我们再试试`git clone`
```
./git clone https://github.com/git/git mGit
```

{% asset_img success.png success %}
成功！