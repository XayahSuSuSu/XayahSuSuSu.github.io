---
title: BlogOS：ARM v8之旅
date: 2022-02-26 15:02:36
tags: [ 'BlogOS', '移植' ]
---

# 前言
**BlogOS**是**Philipp Oppermann**用**Rust语言**编写的**面向x86架构**的**简单操作系统**。

**《ARM v8之旅》**将作为 **[湖南大学2022年操作系统课程实验](https://os2022exps-doc.readthedocs.io/zh_CN/latest/index.html)** 个人参考**笔记**。

# 参考文章
- [libncursesw.so.5 is installed but a program that needs it says "No such file or directory"](https://stackoverflow.com/questions/64491112/libncursesw-so-5-is-installed-but-a-program-that-needs-it-says-no-such-file-or)
- [libpython2.7.so.1.0: cannot open shared object file: No such file or directory](https://stackoverflow.com/questions/20842732/libpython2-7-so-1-0-cannot-open-shared-object-file-no-such-file-or-directory)

#  一、环境配置
本文以**Windows Subsystem for Linux 2**为环境，可参考 **[Windows Subsystem for Linux 2 的艺术](https://acmezone.top/2022/02/12/Windows-Subsystem-for-Linux-2-%E7%9A%84%E8%89%BA%E6%9C%AF/)** 搭建。

## 1. 安装[Rust](https://www.rust-lang.org/zh-CN)
输入以下**命令**
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
若网络**正常**，则会出现以下**输出**，键入`1`，执行**默认安装**。
{% asset_img 安装RUST.png 安装RUST %}
**安装完成**后，**激活Rust环境**
```
source $HOME/.cargo/env
```
查看版本
```
rustc -V
```
{% asset_img 查看版本.png 查看版本 %}
根据**文档**，实验需要用到**Nightly版本**

```
rustup default nightly
```
{% asset_img 查看版本2.png 查看版本2 %}
安装**GCC**
```
sudo apt-get install gcc
```

> 若安装GCC后仍无法正常cargo-binutils rustfilt，请尝试将软件源更换为阿里源（参见**[Windows Subsystem for Linux 2 的艺术](https://acmezone.top/2022/02/12/Windows-Subsystem-for-Linux-2-%E7%9A%84%E8%89%BA%E6%9C%AF/)**），再重新安装一次GCC。

安装**相关工具**
```
cargo install cargo-binutils rustfilt
```

## 2. 添加ARM v8支持
键入以下**命令**
```
rustup target add aarch64-unknown-none-softfloat
```

## 3. 安装QEMU模拟器
键入以下**命令**
```
sudo apt-get install qemu qemu-system-arm
```

## 4. 下载[交叉编译工具链 (AArch64)](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads)
安装**必要环境**
```
sudo apt-get install libncursesw5 libpython2.7 axel
```
创建**交叉编译工具链**目录
```
mkdir ToolChain && cd ToolChain
```
使用**axel**多线程下载工具链 **AArch64 ELF bare-metal target (aarch64-none-elf)**
```
axel -n 32 -a https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-x86_64-aarch64-none-elf.tar.xz
```
解压
```
tar -xf gcc-arm-10.3-2021.07-x86_64-aarch64-none-elf.tar.xz
```

## 5. 创建裸机(Bare Metal)程序
> 由于我们的目标是编写一个操作系统，所以我们需要创建一个独立于操作系统的可执行程序，又称独立式可执行程序（Freestanding Executable）或裸机程序（Bare-metal Executable）。
> 这意味着所有依赖于操作系统的库我们都不能使用。比如std中的大部分内容（io, thread, file system, etc...）都需要操作系统的支持，所以这部分内容我们不能使用。
> 但是，不依赖与操作系统的Rust的语言特性我们还是可以继续使用的，比如：迭代器、模式匹配、字符串格式化、所有权系统等。这使得Rust依旧可以作为一个功能强大的高级语言，帮助我们编写操作系统。

### 新建项目
回到`Home`目录
```
cd ~
```
新建名为`rui_armv8_os`的项目
```
cargo new rui_armv8_os --bin --edition 2021
```
进入`rui_armv8_os`目录
```
cd rui_armv8_os
```
创建**实验所需文件**
```
touch src/panic.rs src/panic.rs src/start.s aarch64-qemu.ld aarch64-unknown-none-softfloat.json
```
创建`.cargo`文件夹
```
mkdir .cargo
```
创建`.cargo/config.toml`
```
touch .cargo/config.toml
```
使用[VSCode](https://code.visualstudio.com/)打开
```
code .
```
**VSCode**安装**Rust**、**Rust-Analyzer**插件
{% asset_img VSCode插件.png VSCode插件 %}
{% asset_img VSCode插件2.png VSCode插件2 %}

编辑`src/main.rs`
```
#![no_std] // 不使用标准库
#![no_main] // 不使用预定义入口点

use core::{arch::global_asm, ptr}; // 导入需要的Module

mod panic;

global_asm!(include_str!("start.s"));

#[no_mangle] // 不修改函数名
pub extern "C" fn not_main() {
    const UART0: *mut u8 = 0x0900_0000 as *mut u8;
    let out_str = b"AArch64 Bare Metal";
    for byte in out_str {
        unsafe {
            ptr::write_volatile(UART0, *byte);
        }
    }
}
```
编辑`src/panic.rs`
```
use core::panic::PanicInfo;

#[panic_handler]
fn on_panic(_info: &PanicInfo) -> ! {
    loop {}
}
```
编辑`src/start.s`
```
.globl _start
.extern LD_STACK_PTR
.section ".text.boot"

_start:
        ldr     x30, =LD_STACK_PTR
        mov     sp, x30
        bl      not_main

.equ PSCI_SYSTEM_OFF, 0x84000002
.globl system_off
system_off:
        ldr     x0, =PSCI_SYSTEM_OFF
        hvc     #0
```
编辑`aarch64-qemu.ld`
```
ENTRY(_start)
SECTIONS
{
    . = 0x40080000;
    .text.boot : { *(.text.boot) }
    .text : { *(.text) }
    .data : { *(.data) }
    .rodata : { *(.rodata) }
    .bss : { *(.bss) }

    . = ALIGN(8);
    . = . + 0x4000;
    LD_STACK_PTR = .;
}
```
编辑`aarch64-unknown-none-softfloat.json`
```
{
    "abi-blacklist": [
      "stdcall",
      "fastcall",
      "vectorcall",
      "thiscall",
      "win64",
      "sysv64"
    ],
    "arch": "aarch64",
    "data-layout": "e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128",
    "disable-redzone": true,
    "env": "",
    "executables": true,
    "features": "+strict-align,+neon,+fp-armv8",
    "is-builtin": false,
    "linker": "rust-lld",
    "linker-flavor": "ld.lld",
    "linker-is-gnu": true,
    "pre-link-args": {
      "ld.lld": ["-Taarch64-qemu.ld"]
    },
    "llvm-target": "aarch64-unknown-none",
    "max-atomic-width": 128,
    "os": "none",
    "panic-strategy": "abort",
    "relocation-model": "static",
    "target-c-int-width": "32",
    "target-endian": "little",
    "target-pointer-width": "64",
    "vendor": ""
}
```
编辑`.cargo/config.toml`
```
[unstable]
build-std = ["core", "compiler_builtins"] 

[build]
target = "aarch64-unknown-none-softfloat.json"
```
编辑`Cargo.toml`
```
[package]
name = "rui_armv8_os"
version = "0.1.0"
edition = "2021"
authors = ["Rui Li <rui@hnu.edu.cn>"]

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]

# eh_personality语言项标记的函数，将被用于实现栈展开（stack unwinding）。
# 在使用标准库的情况下，当panic发生时，Rust将使用栈展开，来运行在栈上活跃的
# 所有变量的析构函数（destructor）——这确保了所有使用的内存都被释放。
# 如果不禁用会出现错误：language item required, but not found: `eh_personality`
# 通过下面的配置禁用栈展开
# dev时禁用panic时栈展开
[profile.dev]
panic = "abort"

# release时禁用panic时栈展开
[profile.release]
panic = "abort"
```

### 编译与运行
在项目**根目录**下执行
```
cargo build
```
{% asset_img 编译成功.png 编译成功 %}
运行
```
qemu-system-aarch64 -machine virt -m 1024M -cpu cortex-a53 -nographic -kernel target/aarch64-unknown-none-softfloat/debug/rui_armv8_os
```
{% asset_img 运行.png 运行 %}

### 调试
> QEMU进入调试，启动调试服务器，默认端口1234

**关闭**之前运行的**终端**，打开一个**新的终端**，进入`rui_armv8_os`目录
```
cd rui_armv8_os
```
**启动调试**
```
qemu-system-aarch64 -machine virt -m 1024M -cpu cortex-a53 -nographic -kernel target/aarch64-unknown-none-softfloat/debug/rui_armv8_os -S -s
```
{% asset_img GDB调试.png GDB调试 %}

**重新打开**一个**终端**，进入**工具链**`bin`目录
```
cd ~/ToolChain/gcc-arm-10.3-2021.07-x86_64-aarch64-none-elf/bin
```
导出**工具链路径**临时变量
```
export ToolChainPath=`pwd`
```
进入`rui_armv8_os`目录
```
cd ~/rui_armv8_os
```
配置**临时工具链环境**（这里的`$ToolChainPath`即是刚刚导出的临时变量）
```
export PATH=$ToolChainPath:$PATH
```
启用**GDB调试客户端**
```
aarch64-none-elf-gdb target/aarch64-unknown-none-softfloat/debug/rui_armv8_os
```
设置**调试参数**，开始**调试**
{% asset_img GDB调试2.png GDB调试2 %}
连接**调试客户端**
```
target remote localhost:1234
```
查看**汇编码**
```
disassemble
```
**单步运行**
```
n
```
{% asset_img GDB调试3.png GDB调试3 %}

#  二、Hello World

#  三、设备树（可选）

#  四、中断

#  五、输入

#  六、GPIO关机

#  七、死锁与简单处理

#  八、内存管理