---
title: Git交叉编译到Android平台
date: 2021-08-02 12:05:30
tags: [ 'Git', '交叉编译', 'Android' ]
---

# 前言

这是一个很久之前的想法了，但是之前一直编译不成功。

这两天仔细研究了下，证明还是可行的。

> Android NDK 版本：r21e
>
> 编译环境：Ubuntu 21.04

### 步骤

#### 一、下载交叉编译所需源码

想要交叉编译**Git**，需要先**交叉编译Curl**和**Zlib**，**新版本**的**NDK**已经包含**预编译Zlib**，所以我们**只需要交叉编译Curl**即可。

至于为什么要交叉编译**Curl**，是因为**git clone**命令在**clone https等仓库**时，需要**依赖该库创建的git-remote-https等EFL文件**。否则，在**clone**时会发生**找不到remote-https等错误**。

下载 **[Git](https://github.com/git/git)** 和 **[Curl](https://curl.se/download.html)** 的源码并**解压**。

#### 二、交叉编译Curl

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

export NDK=/home/xayah/NDK             # NDK根目录绝对路径

export ABI=arm64-v8a                           # ABI配置(arm64-v8a 即为 AArch64)

export MINSDKVERSION=24                        # 最小目标SDK版本配置(24 即为 Android 7.0)

cmake \
   -DCMAKE_TOOLCHAIN_FILE=$NDK/build/cmake/android.toolchain.cmake \
   -DANDROID_ABI=$ABI \
   -DANDROID_NATIVE_API_LEVEL=$MINSDKVERSION \
   -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=./installed ..

cmake --build . --config Release
make install
```

**成功**后**编译产物**可见于`out/installed`。

我们将其**移动**到`/home/xayah/curl/installed`**备份**。

#### 三、交叉编译Git

接下来是我们的**重头戏**了。

进入**Git源码解压后的根目录**，可以发现**Git**有两种编译方式，一是**非Autoconf Makefile**，二是**Autoconf Makefile**，**前者**我尝试过多次，皆以**失败**告终。所以这次我们尝试**后者**。

观察源码结构，可以发现并没有`configure`文件，但存在`configure.ac`，所以我们可以**make**一个`configure`。

**cd**到源码**根目录**，输入：

```
make configure
```

此时即可生成`configure`。

让我们试试：

```
export NDK=/home/xayah/NDK                              # NDK根目录绝对路径

export TOOLCHAIN=$NDK/toolchains/llvm/prebuilt/linux-x86_64     # 交叉编译链路径

export TARGET=aarch64-linux-android                             # 交叉编译目标

export API=24                                                   # 最小目标SDK版本配置(24 即为 Android 7.0)

export AR=$TOOLCHAIN/bin/llvm-ar

export CC=$TOOLCHAIN/bin/$TARGET$API-clang

export AS=$CC

export CXX=$TOOLCHAIN/bin/$TARGET$API-clang++

export LD=$TOOLCHAIN/bin/ld

export RANLIB=$TOOLCHAIN/bin/llvm-ranlib

export READELF=$TOOLCHAIN/bin/readelf

export STRIP=$TOOLCHAIN/bin/llvm-strip

./configure --host=$TARGET --with-curl=/home/xayah/curl/installed --prefix=/home/xayah/git/install
```

可惜出现了**错误**：

```
checking whether system succeeds to read fopen'ed directory... configure: error: in `/home/xayah/下载/git-2.32.0':
configure: error: cannot run test program while cross compiling
See `config.log' for more details
```

原来是**交叉编译测试程序**出了问题，我们的**宿主机**当然**无法测试Android平台上的二进制文件**，所以我们把这段**测试代码**删掉。

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

再

```
make configure
./configure --host=$TARGET --with-curl=/home/xayah/curl/installed --prefix=/home/xayah/git/install
```

一次试试？

遇到了**类似**的**问题**：

```
checking whether snprintf() and/or vsnprintf() return bogus value... configure: error: in `/home/xayah/下载/git-2.32.0':
configure: error: cannot run test program while cross compiling
See `config.log' for more details
```

**删掉**：

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

再

```
make configure
./configure --host=$TARGET --with-curl=/home/xayah/curl/installed --prefix=/home/xayah/git/install
```

一次试试？

**Nice！**这次**成功**了：

```
configure: creating ./config.status
config.status: creating config.mak.autogen
config.status: executing config.mak.autogen commands
```

接着我们：

```
make -j128
```

**很不幸**：

```
run-command.c:520:35: error: use of undeclared identifier 'PTHREAD_CANCEL_DISABLE'
        CHECK_BUG(pthread_setcancelstate(PTHREAD_CANCEL_DISABLE, &as->cs),
```

这里有**两种解决办法**。

##### 方案一

通过**查阅资料**可知， **PTHREAD_CANCEL_DISABLE** 是 **glibc** 下 **pthread** 的一个**常量**，**值**为 **0** ，因此我们将这里**直接赋值为0**即可。

继续**编译**：

```
clang: error: linker command failed with exit code 1 (use -v to see invocation)
make: *** [Makefile:2566：git-shell] 错误 1
/home/xayah/NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/../lib/gcc/aarch64-linux-android/4.9.x/../../../../aarch64-linux-android/bin/ld: cannot find -lrt: cannot find -lrt
```

又是他！**Android由于性能及安全原因**，放弃了**glibc**在其平台上的支持，所以相应地交叉编译链也不含有这个库，但是**Android**有其**替代方案**，所以我们这里**直接**把它**去掉**即可。

打开**源码根目录**的`Makefile`，找到 **NEEDS_LIBRT** ，这里是一个 **ifdef** 判断，我们**反其道行之**，将其改为 **ifndef** 即可。

```
ifndef NEEDS_LIBRT
	EXTLIBS += -lrt
endif
```

又遇到了一个问题！

```
/home/xayah/下载/git-2.32.0/run-command.c:520: undefined reference to `pthread_setcancelstate'
libgit.a(run-command.o): In function `atfork_parent':
/home/xayah/下载/git-2.32.0/run-command.c:531: undefined reference to `pthread_setcancelstate'
libgit.a(run-command.o): In function `atfork_prepare':
/home/xayah/下载/git-2.32.0/run-command.c:520: undefined reference to `pthread_setcancelstate'
libgit.a(run-command.o): In function `atfork_parent':
/home/xayah/下载/git-2.32.0/run-command.c:531: undefined reference to `pthread_setcancelstate'
libgit.a(run-command.o): In function `atfork_prepare':
/home/xayah/下载/git-2.32.0/run-command.c:520: undefined reference to `pthread_setcancelstate'
libgit.a(run-command.o): In function `atfork_parent':
/home/xayah/下载/git-2.32.0/run-command.c:531: undefined reference to `pthread_setcancelstate'
libgit.a(run-command.o): In function `atfork_prepare':
/home/xayah/下载/git-2.32.0/run-command.c:520: undefined reference to `pthread_setcancelstate'
libgit.a(run-command.o): In function `atfork_parent':
/home/xayah/下载/git-2.32.0/run-command.c:531: undefined reference to `pthread_setcancelstate'
libgit.a(run-command.o): In function `atfork_prepare':
/home/xayah/下载/git-2.32.0/run-command.c:520: undefined reference to `pthread_setcancelstate'
libgit.a(run-command.o): In function `atfork_parent':
/home/xayah/下载/git-2.32.0/run-command.c:531: undefined reference to `pthread_setcancelstate'
libgit.a(run-command.o): In function `atfork_prepare':
/home/xayah/下载/git-2.32.0/run-command.c:520: undefined reference to `pthread_setcancelstate'
libgit.a(run-command.o): In function `atfork_parent':
/home/xayah/下载/git-2.32.0/run-command.c:531: undefined reference to `pthread_setcancelstate'
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

还是因为 **pthread** 的问题，这里我们把`run-command.c`所有的 **pthread_setcancelstate** 删掉：

```
	CHECK_BUG(pthread_setcancelstate(0, &as->cs),
		"disabling cancellation");
```

```
	CHECK_BUG(pthread_setcancelstate(as->cs, NULL),
		"re-enabling cancellation");
```

这次终于**编译成功**了！

##### 方案二

实际上我们可以不使用 **pthread**。

在`configure.ac`中可以发现：

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

```
export NDK=/home/xayah/NDK                              # NDK根目录绝对路径

export TOOLCHAIN=$NDK/toolchains/llvm/prebuilt/linux-x86_64     # 交叉编译链路径

export TARGET=aarch64-linux-android                             # 交叉编译目标

export API=24                                                   # 最小目标SDK版本配置(24 即为 Android 7.0)

export AR=$TOOLCHAIN/bin/llvm-ar

export CC=$TOOLCHAIN/bin/$TARGET$API-clang

export AS=$CC

export CXX=$TOOLCHAIN/bin/$TARGET$API-clang++

export LD=$TOOLCHAIN/bin/ld

export RANLIB=$TOOLCHAIN/bin/llvm-ranlib

export READELF=$TOOLCHAIN/bin/readelf

export STRIP=$TOOLCHAIN/bin/llvm-strip

./configure --host=$TARGET --with-curl=/home/xayah/curl/installed --prefix=/home/xayah/git/install --disable-pthreads
```

再修改**源码根目录** `Makefile`：

```
ifndef NEEDS_LIBRT
	EXTLIBS += -lrt
endif
```

然后**编译**：

```
make -j128
```

同样可以**编译成功**。

##### 安装

接下来我们把它安装到`/home/xayah/git/install`。

```
make install -j128
```

安装**成功**后：

```
xayah@xayah-virtual-machine:~/git/install$ pwd
/home/xayah/git/install
xayah@xayah-virtual-machine:~/git/install$ ls
bin  libexec  share
xayah@xayah-virtual-machine:~/git/install$ cd bin
xayah@xayah-virtual-machine:~/git/install/bin$ ls
git            gitk              git-shell           git-upload-pack
git-cvsserver  git-receive-pack  git-upload-archive
```

接下来我们推送到**Android**试试吧！



### 测试

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
cas:/data/local/tmp/bin $ ./git clone https://github.com/git/git
fatal: destination path 'git' already exists and is not an empty directory.
128|cas:/data/local/tmp/bin $ ./git clone https://github.com/git/git mGit
Cloning into 'mGit'...
warning: templates not found in /home/xayah/git/install/share/git-core/templates
fatal: unable to find remote helper for 'https'
128|cas:/data/local/tmp/bin $
```

竟然**报错**了...通过**查阅资料**发现，这是找不到**环境变量**，所以我们设置一下：

```
export PATH=$PATH:/data/local/tmp/libexec/git-core
```

输出如下：

```
cas:/data/local/tmp/bin $ export PATH=$PATH:/data/local/tmp/libexec/git-core
cas:/data/local/tmp/bin $ ./git clone https://github.com/git/git mGit
Cloning into 'mGit'...
warning: templates not found in /home/xayah/git/install/share/git-core/templates
remote: Enumerating objects: 311108, done.
remote: Counting objects: 100% (72/72), done.
remote: Compressing objects: 100% (32/32), done.
remote: Total 311108 (delta 41), reused 70 (delta 40), pack-reused 311036
Receiving objects: 100% (311108/311108), 164.19 MiB | 10.32 MiB/s, done.
Resolving deltas: 100% (232238/232238), done.
Segmentation fault
```

**Segmentation fault**！这可是个头疼的事。不过没有关系，我们可以使用 **gdbserver** 调试，看看到底是**哪里**出了**问题**。

### 调试

首先将**NDK**中相应的**gdbserver**推送到**Android平台**，我的手机是**aarch64**（现在**大多数安卓手机都是这个平台**），也就是**arm64**，所以推送**该文件夹**下的**gdbserver**并**赋予权限**即可。

```
PS C:\Users\Xayah\Desktop> adb push gdbserver /data/local/tmp/bin
gdbserver: 1 file pushed, 0 skipped. 120.6 MB/s (1343712 bytes in 0.011s)
PS C:\Users\Xayah\Desktop> adb shell chmod 777 /data/local/tmp/bin/gdbserver
```

接下来**新建**一个**终端**并**forward**一个**自定义端口**：

```
adb forward tcp:12345 tcp:12345
```

接下来在另一个**终端**中进入**adb shell**：

```
PS C:\Users\Xayah\Desktop> adb shell
cas:/ $ cd data/local/tmp/bin
cas:/data/local/tmp/bin $ export PATH=$PATH:/data/local/tmp/libexec/git-core
cas:/data/local/tmp/bin $ ./gdbserver 0.0.0.0:12345 ./git clone https://github.com/git/git mGit
Process ./git created; pid = 31451
Listening on port 12345
```

现在**gdbserver**已经启动了，我们转向另一个**终端**，**cd**到**ndk**的**gdb目录**：

```
PS C:\Users\Xayah> cd D:\Downloads\Compressed\android-ndk-r21e-windows-x86_64\android-ndk-r21e\prebuilt\windows-x86_64\bin
```

启动**gdb**：

```
PS D:\Downloads\Compressed\android-ndk-r21e-windows-x86_64\android-ndk-r21e\prebuilt\windows-x86_64\bin> ./gdb
D:\Downloads\Compressed\android-ndk-r21e-windows-x86_64\android-ndk-r21e\prebuilt\windows-x86_64\bin\gdb-orig.exe: warning: Couldn't determine a path for the index cache directory.
GNU gdb (GDB) 8.3
Copyright (C) 2019 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-w64-mingw32".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
(gdb)
```

现在**gdb**和**gdbserver**均已启动，我们**指定目标端口**：

未完待续....

