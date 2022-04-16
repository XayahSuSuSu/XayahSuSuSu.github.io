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
> 参考代码：{% asset_link ScienceOne.tar.gz 下载 %}

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

安装**相关工具**
```
cargo install cargo-binutils rustfilt
```
> 若安装GCC后仍无法正常`cargo install cargo-binutils rustfilt`，请尝试将**软件源**更换为**阿里源**（参见 **[Windows Subsystem for Linux 2 的艺术](https://acmezone.top/2022/02/12/Windows-Subsystem-for-Linux-2-%E7%9A%84%E8%89%BA%E6%9C%AF/)** ），再重新安装一次**GCC**。

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
> 由于我们的目标是编写一个**操作系统**，所以我们需要创建一个**独立于操作系统**的**可执行程序**，又称**独立式可执行程序**（Freestanding Executable）或**裸机程序**（Bare-metal Executable）。
> 
> 这意味着所有**依赖于操作系统的库**我们都**不能使用**。比如`std`中的大部分内容（`io`, `thread`, `file system`, etc...）都需要操作系统的支持，所以这部分内容我们不能使用。
> 
> 但是，**不依赖于操作系统**的**Rust**的**语言特性**我们还是可以继续使用的，比如：**迭代器**、**模式匹配**、**字符串格式化**、**所有权系统**等。这使得**Rust**依旧可以作为一个**功能强大**的**高级语言**，帮助我们编写**操作系统**。

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
> 参考代码：{% asset_link ScienceTwo.tar.gz 下载 %}

> `print`函数是学习几乎任何一种软件开发语言时**最先**学习使用的函数，同时该函数也是最基本和原始的**程序调试手段**，但该函数的实现却并**不简单**。本实验的目的在于**理解操作系统与硬件的接口方法**，并实现一个**可打印字符的宏**（**非系统调用**），用于后续的**调试**和**开发**。

## 1. 了解virt机器
> **操作系统**介于**硬件**和**应用程序**之间，**向下管理硬件资源**，**向上提供应用编程接口**。设计并实现**操作系统**需要熟悉**底层硬件**的**组成**及其**操作方法**。
> 
> 本系列实验都会在**QEMU模拟器**上完成，首先来了解一下模拟的**机器信息**。可以通过下列**两种方法**：

### 1) 文档或源码方式
> 查看**QEMU**关于**virt**的[描述](https://www.qemu.org/docs/master/system/arm/virt.html)， 或者查看**QEMU**的**源码**，如**GitHub**上的[virt.h](https://github.com/qemu/qemu/blob/master/include/hw/arm/virt.h)和[virt.c](https://github.com/qemu/qemu/blob/master/hw/arm/virt.c)。`virt.c`中可见如下有关**内存映射**的内容。

```
/* Addresses and sizes of our components.
 * 0..128MB is space for a flash device so we can run bootrom code such as UEFI.
 * 128MB..256MB is used for miscellaneous device I/O.
 * 256MB..1GB is reserved for possible future PCI support (ie where the
 * PCI memory window will go if we add a PCI host controller).
 * 1GB and up is RAM (which may happily spill over into the
 * high memory region beyond 4GB).
 * This represents a compromise between how much RAM can be given to
 * a 32 bit VM and leaving space for expansion and in particular for PCI.
 * Note that devices should generally be placed at multiples of 0x10000,
 * to accommodate guests using 64K pages.
 */
static const MemMapEntry base_memmap[] = {
    /* Space up to 0x8000000 is reserved for a boot ROM */
    [VIRT_FLASH] =              {          0, 0x08000000 },
    [VIRT_CPUPERIPHS] =         { 0x08000000, 0x00020000 },
    /* GIC distributor and CPU interfaces sit inside the CPU peripheral space */
    [VIRT_GIC_DIST] =           { 0x08000000, 0x00010000 },
    [VIRT_GIC_CPU] =            { 0x08010000, 0x00010000 },
    [VIRT_GIC_V2M] =            { 0x08020000, 0x00001000 },
    [VIRT_GIC_HYP] =            { 0x08030000, 0x00010000 },
    [VIRT_GIC_VCPU] =           { 0x08040000, 0x00010000 },
    /* The space in between here is reserved for GICv3 CPU/vCPU/HYP */
    [VIRT_GIC_ITS] =            { 0x08080000, 0x00020000 },
    /* This redistributor space allows up to 2*64kB*123 CPUs */
    [VIRT_GIC_REDIST] =         { 0x080A0000, 0x00F60000 },
    [VIRT_UART] =               { 0x09000000, 0x00001000 },
    [VIRT_RTC] =                { 0x09010000, 0x00001000 },
    [VIRT_FW_CFG] =             { 0x09020000, 0x00000018 },
    [VIRT_GPIO] =               { 0x09030000, 0x00001000 },
    [VIRT_SECURE_UART] =        { 0x09040000, 0x00001000 },
    [VIRT_SMMU] =               { 0x09050000, 0x00020000 },
    [VIRT_PCDIMM_ACPI] =        { 0x09070000, MEMORY_HOTPLUG_IO_LEN },
    [VIRT_ACPI_GED] =           { 0x09080000, ACPI_GED_EVT_SEL_LEN },
    [VIRT_NVDIMM_ACPI] =        { 0x09090000, NVDIMM_ACPI_IO_LEN},
    [VIRT_PVTIME] =             { 0x090a0000, 0x00010000 },
    [VIRT_SECURE_GPIO] =        { 0x090b0000, 0x00001000 },
    [VIRT_MMIO] =               { 0x0a000000, 0x00000200 },
    /* ...repeating for a total of NUM_VIRTIO_TRANSPORTS, each of that size */
    [VIRT_PLATFORM_BUS] =       { 0x0c000000, 0x02000000 },
    [VIRT_SECURE_MEM] =         { 0x0e000000, 0x01000000 },
    [VIRT_PCIE_MMIO] =          { 0x10000000, 0x2eff0000 },
    [VIRT_PCIE_PIO] =           { 0x3eff0000, 0x00010000 },
    [VIRT_PCIE_ECAM] =          { 0x3f000000, 0x01000000 },
    /* Actual RAM size depends on initial RAM and device memory settings */
    [VIRT_MEM] =                { GiB, LEGACY_RAMLIMIT_BYTES },
};
```

### 2) 设备树（Device Tree）方式
首先安装**DTC**
```
sudo apt-get install device-tree-compiler
```
新建一个**设备树目录**并**进入**
```
mkdir ~/device && cd ~/device
```
导出**DT**
```
qemu-system-aarch64 -machine virt,dumpdtb=virt.dtb -cpu cortex-a53 -nographic
```
> `-machine virt`指明**机器类型**为**virt**，这是**QEMU**仿真的**虚拟机器**。

用**DTC**将导出的**Device Tree Blob**转换为**Device Tree Source**
```
dtc -I dtb -O dts -o virt.dts virt.dtb
```
用**文本编辑器**打开`virt.dts`，可以发现如下内容
```
pl011@9000000 {
	clock-names = "uartclk\0apb_pclk";
	clocks = <0x8000 0x8000>;
	interrupts = <0x00 0x01 0x04>;
	reg = <0x00 0x9000000 0x00 0x1000>;
	compatible = "arm,pl011\0arm,primecell";
};
/* ······ */
chosen {
	stdout-path = "/pl011@9000000";
};
```
> 由上可以看出，**virt**机器包含有**pl011**的设备，该设备的**寄存器**在`0x9000000`开始处。**pl011**实际上是一个**UART设备**，即**串口**。可以看到**virt**选择使用**pl011**作为**标准输出**，这是因为**与PC不同**，大部分**嵌入式系统**默认情况下**并不包含VGA设备**。

## 2. 实现println!宏
> 我们参照[Writing an OS in Rust - VGA Text Mode](https://os.phil-opp.com/vga-text-mode/) （[使用Rust编写操作系统（三）：VGA字符模式](https://github.com/rustcc/writing-an-os-in-rust/blob/master/03-vga-text-mode.md)）来实现`println!`宏，但与之不同的是，我们使用**串口**来输出，而不是通过操作**VGA**的**Frame Buffer**。

### 1) 用串口实现println!宏
进入`rui_armv8_os`目录
```
cd ~/rui_armv8_os
```
新建`src/uart_console.rs`
```
touch src/uart_console.rs
```
编辑`src/uart_console.rs`，定义一个**Writer结构**，实现**字节写入**和**字符串写入**。
```
//嵌入式系统使用串口，而不是vga，直接输出，没有颜色控制，不记录列号，也没有frame buffer，所以采用空结构
pub struct Writer;

//往串口寄存器写入字节和字符串进行输出
impl Writer {
    pub fn write_byte(&mut self, byte: u8) {
        const UART0: *mut u8 = 0x0900_0000 as *mut u8;
        unsafe {
            ptr::write_volatile(UART0, byte);
        }
    }

    pub fn write_string(&mut self, s: &str) {
        for byte in s.chars() {
            self.write_byte(byte as u8)
        }
    }
}
```
> 如何操作硬件通常需要阅读**硬件制造商**提供的**技术手册**。如**pl011**串口设备（PrimeCell UART）是**arm**设计的，其**技术参考手册**可以通过其[官网](https://developer.arm.com/documentation/ddi0183/latest/)查看。
> 
> 依据之前`virt.dts`中的描述，**pl011**的**寄存器**在**virt**机器中被**映射**到了`0x9000000`的**内存位置**。通过访问**pl011**的**技术参考手册**中`Chapter 3. Programmers Model`中的`Summary of registers`一节可知：**第0号寄存器**是**pl011**串口的**数据寄存器**，用于**数据**的**收发**。其**详细描述**参见 [这里](https://developer.arm.com/documentation/ddi0183/g/programmers-model/register-descriptions/data-register--uartdr?lang=en)。
> 
> 注意到我们只是向**UART0**写入，而没从**UART0**读出（如果读出会读出**其他设备**通过**串口**发送过来的数据，而**不是**刚才**写入**的数据，这与**读写内存**时是不一样的，详情参见**pl011**的**技术手册**），**编译器**在**优化**时可能对这部分代码进行**错误**的**优化**，如**把这些操作都忽略掉**。
> 
> 使用`ptr::write_volatile`库的**目的**是告诉**编译器**，这些**写入**有**特定目的**，不应将其**优化**（也就是告诉编译器**不要瞎优化**，这些**写入**和**读出**都有**特定用途**。
> 
> 比如**连续两次读**，**编译器**可能认为**第二次读**就是**前次的值**，所以**优化**掉**第二次读**，但对**外设寄存器**的**连续读**可能返回**不同的值**。
> 
> 比如写，**编译器**可能认为**写**后没有**读**所以**写**没有作用，或者**连续的写**会**覆盖**前面的**写**，但对这些**寄存器**的**写入**对**外设**都有**特定作用**）。

在`src/uart_console.rs`中为**Write结构**实现`core::fmt::Write`**trait**，该**trait**会自动实现`write_fmt`方法，支持**格式化**。
```
//嵌入式系统使用串口，而不是vga，直接输出，没有颜色控制，不记录列号，也没有frame buffer，所以采用空结构
pub struct Writer;

//往串口寄存器写入字节和字符串进行输出
impl Writer {
    // ······
}

impl core::fmt::Write for Writer {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        self.write_string(s);

        Ok(())
    }
}
```
> 基于**Rust**的`core::fmt`实现**格式化控制**，可以使我们方便地**打印不同类型**的**变量**。实现`core::fmt::Write`后，我们就可以使用**Rust**内置的**格式化**宏`write!`和`writeln!`，这使你瞬间具有**其他语言**运行时所提供的**格式化控制能力**。

### 2) 测试
在`main.rs`末尾加入以下代码
```
#![no_std] // 不使用标准库
#![no_main] // 不使用预定义入口点

// ······

#[no_mangle] // 不修改函数名
pub extern "C" fn not_main() {
    // ······
}

include!("uart_console.rs");
use core::fmt;

pub fn print_something() {
    // 一定要引用core::fmt::Write;否则报错：no method named `write_fmt` found for struct `Writer` in the current scope。
    pub use core::fmt::Write;

    let mut writer = Writer {};
    let display: fmt::Arguments = format_args!("hello arguments!\n");

    writer.write_string("\n-----My writer-----\n");
    writer.write_byte(b'H');
    writer.write_string("ello ");
    writer.write_string("World!\n");
    writer.write_string("[0] Hello from Rust!\n");

    // 通过实现core::fmt::Write自动实现的方法
    writer.write_fmt(display).unwrap();
    // 使用write!宏
    write!(writer, "The numbers are {} and {} \n", "42", "1.0").unwrap();

    writer.write_string("-----My writer-----");
}
```
编辑`main.rs`中`not_main`函数：
```
pub extern "C" fn not_main() {
    // ······
    print_something(); // 调用测试函数
}
```
**编译**并**运行**
```
cargo build && qemu-system-aarch64 -machine virt -m 1024M -cpu cortex-a53 -nographic -kernel target/aarch64-unknown-none-softfloat/debug/rui_armv8_os
```
{% asset_img Writer.png Writer %}

按住`CTRL + A`，然后松手按`C`，输入`quit`即可退出**QEMU模拟器**。

### 3) 全局实现
> 现在我们已经可以采用`print_something`函数通过**串口输出**字符了。但若要实现**输出**，我们需要**两个**步骤：
> （1）创建**Writer类型**的**实例**。
> （2）调用**实例**的`write_byte`或`write_string`等**函数**。
> 
> 为了方便在**其他模块**中**调用**，我们希望可以**直接执行步骤（2）**而不是**先执行步骤（1）**再**执行步骤（2）**。
> 
> 一般情况下可以通过将**步骤（1）**中的**实例**定义为`static`类型来实现，但**Rust**暂不支持**Write**r这样类型的**静态（编译时）初始化**，需要使用`lazy_static`来解决。此外，为了保证访问**安全**还引入了**自旋锁（spin）**。

编辑`Cargo.toml`：
```
# ······
# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
spin = "0.9.2"

[dependencies.lazy_static]
version = "1.0"
features = ["spin_no_std"]

# eh_personality语言项标记的函数，将被用于实现栈展开（stack unwinding）。
# ······
```
编辑`src/uart_console.rs`，实现`print!`和`println!`宏。
```
impl core::fmt::Write for Writer {
    // ······
}

use core::{fmt, ptr};

use lazy_static::lazy_static;
use spin::Mutex;

lazy_static! {
    /// A global `Writer` instance that can be used for printing to the VGA text buffer.
    ///
    /// Used by the `print!` and `println!` macros.
    pub static ref WRITER: Mutex<Writer> = Mutex::new(Writer { });
}

/// Like the `print!` macro in the standard library, but prints to the VGA text buffer.
#[macro_export]
macro_rules! print {
    ($($arg:tt)*) => ($crate::uart_console::_print(format_args!($($arg)*)));
}

/// Like the `println!` macro in the standard library, but prints to the VGA text buffer.
#[macro_export]
macro_rules! println {
    () => ($crate::print!("\n"));
    ($($arg:tt)*) => ($crate::print!("{}\n", format_args!($($arg)*)));
}

/// Prints the given formatted string to the VGA text buffer through the global `WRITER` instance.
#[doc(hidden)]
pub fn _print(args: fmt::Arguments) {
    use core::fmt::Write;

    WRITER.lock().write_fmt(args).unwrap();
}
```
在`main.rs`中**注释或删除之前的`print_something()`函数及其调用**，测试`println!`宏
```
// ······
mod uart_console;
// ······

pub extern "C" fn not_main() {
    // ······
    println!("\n[0] Hello from Rust!");
}
```
**编译**并**运行**
```
cargo build && qemu-system-aarch64 -machine virt -m 1024M -cpu cortex-a53 -nographic -kernel target/aarch64-unknown-none-softfloat/debug/rui_armv8_os
```
{% asset_img println.png println %}

#  三、设备树（可选）
> 参考[湖南大学2022年操作系统课程实验 - 实验三 设备树（可选）](https://os2022exps-doc.readthedocs.io/zh_CN/latest/exp3/index.html)

#  四、中断
> 参考代码：{% asset_link ScienceFour.tar.gz 下载 %}
> 
> **中断**、**异常**和**陷阱**指令是**操作系统**的**基石**，现代操作系统就是由**中断驱动**的。本实验的目的在于**深刻理解中断的原理和机制**，**掌握CPU访问设备控制器的方法**，**掌握ARM体系结构的中断机制和规范**，**实现时钟中断服务和部分异常处理**等。

## 1. 概念
### 1) 陷入操作系统
> 如下图所示，**操作系统**是一个**多入口**的**程序**，执行**陷阱（Trap）指令**，出现**异常**、**发生中断**时都会**陷入**到**操作系统**。
> 
> {% asset_img enter_into_os.png enter_into_os %}

### 2) ARM的中断系统
> **中断**是一种**硬件机制**。借助于**中断**，**CPU**可以不必再采用**轮询**这种**低效**的方式**访问外部设备**。将所有的**外部设备**与**CPU直接相连**是**不现实**的，**外部设备**的**中断请求**一般经由**中断控制器**，由**中断控制器**仲裁后再转发给**CPU**。如下图所示**ARM**的**中断系统**。
> 
> {% asset_img ARMGIC.png ARMGIC %}
> 
> 其中**nIRQ**是**普通中断**，**nFIQ**是**快速中断**。**ARM**采用的**中断控制器**叫做**GIC**，即**General Interrupt Controller**。**GIC**包括多个版本，如**GICv1（已弃用）**，**GICv2**，**GICv3**，**GICv4**。简单起见，我们实验将选用**GICv2**版本。
> 
> 为了配置好**GICv2中断控制器**，与**pl011串口**一样，我们需要阅读其**技术参考手册**。
> 
> 访问**ARM**官网下载[ARM Generic Interrupt Controller Architecture version 2.0 - Architecture Specification](https://developer.arm.com/documentation/ihi0048/latest)。
> 
> {% asset_img gicv2-logic.png gicv2-logic %}
> 
> 从上图（来源于[ARM Generic Interrupt Controller Architecture version 2.0 - Architecture Specification](https://developer.arm.com/documentation/ihi0048/latest)中的**Chapter 2 GIC Partitioning**）可以看出：
> - **GICv2**最多支持**8核**的**中断管理**。
> - **GIC**包括**两大主要部分**（由图中**蓝色虚竖线**分隔，**Distributor**和**CPU Interface**由**蓝色虚矩形框**标示），分别是：
> 	- **Distributor**，其通过`GICD_`开头的寄存器进行控制（**蓝色实矩形框**标示）
> 	- **CPU Interface**，其通过`GICC_`开头的寄存器进行控制（**蓝色实矩形框**标示）
> -  **中断类型**分为以下几类（由图中**红色虚线椭圆**标示）：
> 	- **SPI：（Shared Peripheral Interrupt）**，**共享外设中断**。该**中断**来源于**外设**，通过**Distributor**分发给特定的**Core**，其**中断编号**为**32-1019**。从图中可以看到所有核**共享SPI**。
> 	- **PPI：（Private Peripheral Interrupt）**，**私有外设中断**。该**中断**来源于**外设**，但只对指定的**Core**有效，**中断信号**只会发送给**指定的Core**，其**中断编号**为**16-31**。从图中可以看到每个**Core**都有自己的**PPI**。
> 	- **SGI：（Software-Generated Interrupt）**，**软中断**。**软件产生**的**中断**，用于给其他的**Core**发送**中断信号**，其**中断编号**为**0-15**。
> 	- **Virtual Interrupt**，**虚拟中断**，用于支持**虚拟机**。图中也可以看到，因为我们**暂时不关心**，所以没有标注。
> 	- 此外可以看到**（FIQ，IRQ）**可通过**b**进行**旁路**，我们也不关心。如感兴趣可以查看**技术手册**了解细节。
> 
> 此外，由[ARM Generic Interrupt Controller Architecture version 2.0 - Architecture Specification](https://developer.arm.com/documentation/ihi0048/latest)(Section 1.4.2)可知，**外设中断**可由**两种方式**触发：
> - **Edge-Triggered**：**边沿触发**，当检测到**中断信号上升沿**时**中断有效**。
> - **Level-Sensitive**：**电平触发**，当**中断源**为**指定电平**时**中断有效**。
> 因为**SOC**中**中断**有很多，为了方便对**中断的管理**，对每个**中断**附加了**中断优先级**。在**中断仲裁**时，**高优先级的中断**，会**优于低优先级的中断**，发送给**CPU处理**。当**CPU**在**响应低优先级中断**时，如果此时来了**高优先级中断**，那么**高优先级中断**会**抢占低优先级中断**，而被**处理器响应**。
> 由[ARM Generic Interrupt Controller Architecture version 2.0 - Architecture Specification](https://developer.arm.com/documentation/ihi0048/latest)(Section 3.3)可知，**GICv2**最多支持**256**个**中断优先级**。**GICv2**中规定，所支持的**中断优先级别数**与**GIC**的具体实现有关，如果支持的**中断优先级数**比**256**少（最少为**16**），则**8位优先级**的**低位**为**0**，且遵循**RAZ/WI（Read-As-Zero, Writes Ignored）**原则。


### 3) GICv2初始化
```
/* ······ */
intc@8000000 {
    phandle = <0x8001>;
    reg = <0x00 0x8000000 0x00 0x10000 0x00 0x8010000 0x00 0x10000>;
    compatible = "arm,cortex-a15-gic";
    ranges;
    #size-cells = <0x02>;
    #address-cells = <0x02>;
    interrupt-controller;
    #interrupt-cells = <0x03>;

    v2m@8020000 {
        phandle = <0x8002>;
        reg = <0x00 0x8020000 0x00 0x1000>;
        msi-controller;
        compatible = "arm,gic-v2m-frame";
    };
};
/* ······ */
timer {
    interrupts = <0x01 0x0d 0x104 0x01 0x0e 0x104 0x01 0x0b 0x104 0x01 0x0a 0x104>;
    always-on;
    compatible = "arm,armv8-timer\0arm,armv7-timer";
};
```
> 由`virt.dts`中`intc`和`timer`的部分并结合**kernel.org**中关于[ARM Generic Interrupt Controller](https://www.kernel.org/doc/Documentation/devicetree/bindings/interrupt-controller/arm%2Cgic.txt)和[ARM architected timer](https://www.kernel.org/doc/Documentation/devicetree/bindings/arm/arch_timer.txt)的**DeviceTree**的说明可知：
> - `intc`中的`reg`指明**GICD寄存器**映射到内存的位置为`0x8000000`，长度为`0x10000`，**GICC寄存器**映射到内存的位置为`0x8010000`，长度为`0x10000`。
> - `intc`中的`#interrupt-cells`指明**interrupts**包括**3**个**cells**。[第一个文档](https://www.kernel.org/doc/Documentation/devicetree/bindings/interrupt-controller/arm%2Cgic.txt)指明：第一个**cell**为**中断类型**，**0**表示**SPI**，**1**表示**PPI**；第二个**cell**为**中断号**，**SPI**范围为**[0-987]**，**PPI**为**[0-15]**；第三个**cell**为**flags**，其中**[3:0]**位表示**触发类型**，**[4]**表示**高电平触发**，**[15:8]**为**PPI**的**CPU中断掩码**，每**1**位对应一个**CPU**，为**1**表示该**中断**会连接到对应的**CPU**。
> - 以`timer`设备为例，其中包括**4**个**中断**。以第**2**个**中断**的参数`0x01 0x0e 0x104`为例，其指明该**中断**为**PPI**类型的**中断**，**中断号14**， 路由到第一个**CPU**，且**高电平触发**。但注意到**PPI**的**起始中断号**为**16**，所以实际上该**中断**在**GICv2**中的**中断号**应为**16 + 14 = 30**。
> 阅读[ARM Generic Interrupt Controller Architecture version 2.0 - Architecture Specification](https://developer.arm.com/documentation/ihi0048/latest)，在**Chapter 4 Programmers’ Model**部分有关于**GICD**和**GICC寄存器**的**描述，以及如何使能**Distributor**和**CPU Interfaces**的方法。

### 4) ARMv8的中断与异常处理
> 访问**ARM官网**下载并阅读[ARM Cortex-A Series Programmer's Guide for ARMv8-A](https://developer.arm.com/documentation/den0024/a/AArch64-Exception-Handling/Exception-handling-registers)和[AArch64 Exception and Interrupt Handling](https://developer.arm.com/documentation/100933/0100/AArch64-exception-vector-table)等**技术参考手册**。
> **ARMv8架构**定义了**两种执行状态([Execution States](https://developer.arm.com/documentation/den0024/a/Fundamentals-of-ARMv8/Execution-states))**：**AArch64**和**AArch32**。分别对应使用**64位宽通用寄存器**或**32位宽通用寄存器**的执行。
> {% asset_img aarch64_exception_levels_2.png aarch64_exception_levels_2 %}
> 上图所示为**AArch64**中的**异常级别(Exception levels)**的组织。可见**AArch64**中共有**4**个**异常级别**，分别为**EL0**，**EL1**，**EL2**和**EL3**。在**AArch64**中，**Interrupt**是**Exception**的**子类型**，称为**异常**。**AArch64**中有**四种类型**的**异常**：
> - **Sync（Synchronous exceptions，同步异常）**。在**执行时触发**的**异常**，例如在尝试**访问不存在**的**内存地址**时。
> - **IRQ （Interrupt requests，中断请求）**。由**外部设备**产生的**中断**。
> - **FIQ （Fast Interrupt Requests，快速中断请求）**。类似于**IRQ**，但具有**更高**的**优先级**，因此**FIQ**中断服务程序不能被其他**IRQ**或**FIQ**中断。
> - **SError （System Error，系统错误）**。用于**外部数据**中止的**异步中断**。
> 当**异常**发生时，**处理器**将执行与该**异常**对应的**异常处理代码**。在**ARM架构**中，这些**异常处理代码**将会被保存在**内存**的**异常向量表**中。每一个**异常级别（EL0，EL1，EL2和EL3）**都有其对应的**异常向量表**。需要注意的是，与**x86等架构**不同，该表包含的是要执行的**指令**，而不是**函数地址**。
> **异常向量表**的**基地址**由`VBAR_ELn`给出，然后每个表项都有一个从该**基地址**定义的**偏移量**。 每个表有**16**个表项，每个表项的大小为**128（0x80）**字节（**32**条**指令**）。 该表实际上有**4**组，每组**4**个表项。 分别是：
> - 发生于**当前异常级别**的**异常**且**SPSel寄存器**选择**SP0**，**Sync**、**IRQ**、**FIQ**、**SError**对应的**4个异常处理**。
> - 发生于**当前异常级别**的**异常**且**SPSel寄存器**选择**SPx**，**Sync**、**IRQ**、**FIQ**、**SError**对应的**4个异常处理**。
> - 发生于**较低异常级别**的**异常**且**执行状态**为**AArch64**，**Sync**、**IRQ**、**FIQ**、**SError**对应的**4个异常处理**。
> - 发生于**较低异常级别**的**异常**且**执行状态**为**AArch32**，**Sync**、**IRQ**、**FIQ**、**SError**对应的**4个异常处理**。


## 2. 实现
### 1) 编写代码
新建`src/interrupts.rs`，`src/exceptions.s`
```
touch src/interrupts.rs src/exceptions.s
```
编辑`src/interrupts.rs`，定义各种**常量**，如**寄存器地址**和**寄存器值**等，然后定义`init_gicv2`函数对**GICD**和**GICC**进行**初始化**，最后定义若干**辅助函数**用于**中断配置**。
```
use core::ptr;

// GICD和GICC寄存器内存映射后的起始地址
const GICD_BASE: u64 = 0x08000000;
const GICC_BASE: u64 = 0x08010000;

// Distributor
const GICD_CTLR: *mut u32 = (GICD_BASE + 0x0) as *mut u32;
const GICD_ISENABLER: *mut u32 = (GICD_BASE + 0x0100) as *mut u32;
const GICD_ICENABLER: *mut u32 = (GICD_BASE + 0x0180) as *mut u32;
const GICD_ICPENDR: *mut u32 = (GICD_BASE + 0x0280) as *mut u32;
const GICD_IPRIORITYR: *mut u32 = (GICD_BASE + 0x0400) as *mut u32;
const GICD_ICFGR: *mut u32 = (GICD_BASE + 0x0c00) as *mut u32;

const GICD_CTLR_ENABLE: u32 = 1;  /* Enable GICD */
const GICD_CTLR_DISABLE: u32 = 0;     /* Disable GICD */
const GICD_ISENABLER_SIZE: u32 = 32;
const GICD_ICENABLER_SIZE: u32 = 32;
const GICD_ICPENDR_SIZE: u32 = 32;
const GICD_IPRIORITY_SIZE: u32 = 4;
const GICD_IPRIORITY_BITS: u32 = 8;
const GICD_ICFGR_SIZE: u32 = 16;
const GICD_ICFGR_BITS: u32 = 2;


// CPU Interface
const GICC_CTLR: *mut u32 = (GICC_BASE + 0x0) as *mut u32;
const GICC_PMR: *mut u32 = (GICC_BASE + 0x0004) as *mut u32;
const GICC_BPR: *mut u32 = (GICC_BASE + 0x0008) as *mut u32;

const GICC_CTLR_ENABLE: u32 = 1;
const GICC_CTLR_DISABLE: u32 = 0;
// Priority Mask Register. interrupt priority filter, Higher priority corresponds to a lower Priority field value.
const GICC_PMR_PRIO_LOW: u32 = 0xff;
// The register defines the point at which the priority value fields split into two parts,
// the group priority field and the subpriority field. The group priority field is used to
// determine interrupt preemption. NO GROUP.
const GICC_BPR_NO_GROUP: u32 = 0x00;

pub fn init_gicv2() {
    // 初始化Gicv2的distributor和cpu interface
    // 禁用distributor和cpu interface后进行相应配置
    unsafe {
        ptr::write_volatile(GICD_CTLR, GICD_CTLR_DISABLE);
        ptr::write_volatile(GICC_CTLR, GICC_CTLR_DISABLE);
        ptr::write_volatile(GICC_PMR, GICC_PMR_PRIO_LOW);
        ptr::write_volatile(GICC_BPR, GICC_BPR_NO_GROUP);
    }

    // 启用distributor和cpu interface
    unsafe {
        ptr::write_volatile(GICD_CTLR, GICD_CTLR_ENABLE);
        ptr::write_volatile(GICC_CTLR, GICC_CTLR_ENABLE);
    }

}

// 使能中断号为interrupt的中断
pub fn enable(interrupt: u32) {
    unsafe {
        ptr::write_volatile(
            GICD_ISENABLER.add((interrupt / GICD_ISENABLER_SIZE) as usize),
            1 << (interrupt % GICD_ISENABLER_SIZE)
        );
    }
}

// 禁用中断号为interrupt的中断
pub fn disable(interrupt: u32) {
    unsafe {
        ptr::write_volatile(
            GICD_ICENABLER.add((interrupt / GICD_ICENABLER_SIZE) as usize),
            1 << (interrupt % GICD_ICENABLER_SIZE)
        );
    }
}

// 清除中断号为interrupt的中断
pub fn clear(interrupt: u32) {
    unsafe {
        ptr::write_volatile(
            GICD_ICPENDR.add((interrupt / GICD_ICPENDR_SIZE) as usize),
            1 << (interrupt % GICD_ICPENDR_SIZE)
        );
    }
}

// 设置中断号为interrupt的中断的优先级为priority
pub fn set_priority(interrupt: u32, priority: u32) {
    let shift = (interrupt % GICD_IPRIORITY_SIZE) * GICD_IPRIORITY_BITS;
    unsafe {
        let addr: *mut u32 = GICD_IPRIORITYR.add((interrupt / GICD_IPRIORITY_SIZE) as usize);
        let mut value: u32 = ptr::read_volatile(addr);
        value &= !(0xff << shift);
        value |= priority << shift;
        ptr::write_volatile(addr, value);
    }
}

// 设置中断号为interrupt的中断的属性为config
pub fn set_config(interrupt: u32, config: u32) {
    let shift = (interrupt % GICD_ICFGR_SIZE) * GICD_ICFGR_BITS;
    unsafe {
        let addr: *mut u32 = GICD_ICFGR.add((interrupt / GICD_ICFGR_SIZE) as usize);
        let mut value: u32 = ptr::read_volatile(addr);
        value &= !(0x03 << shift);
        value |= config << shift;
        ptr::write_volatile(addr, value);
    }
}
```
编辑`src/exceptions.s`，参照[AArch64 exception table](https://developer.arm.com/documentation/den0024/a/AArch64-Exception-Handling/AArch64-exception-table)定义异常向量表。
```
// SPDX-License-Identifier: MIT OR Apache-2.0
//
// Copyright (c) 2018-2021 Andre Richter <andre.o.richter@gmail.com>

.extern el1_sp0_sync
.extern el1_sp0_irq
.extern el1_sp0_fiq
.extern el1_sp0_error
.extern el1_sync
.extern el1_irq
.extern el1_fiq
.extern el1_error
.extern el0_sync
.extern el0_irq
.extern el0_fiq
.extern el0_error
.extern el0_32_sync
.extern el0_32_irq
.extern el0_32_fiq
.extern el0_32_error

//--------------------------------------------------------------------------------------------------
// Definitions
//--------------------------------------------------------------------------------------------------

/// Call the function provided by parameter `\handler` after saving the exception context. Provide
/// the context as the first parameter to '\handler'.
.equ CONTEXT_SIZE, 264

.section .text.exceptions

.macro EXCEPTION_VECTOR handler
    sub sp, sp, #CONTEXT_SIZE

// store general purpose registers
    stp x0, x1, [sp, #16 * 0]
    stp x2, x3, [sp, #16 * 1]
    stp x4, x5, [sp, #16 * 2]
    stp x6, x7, [sp, #16 * 3]
    stp x8, x9, [sp, #16 * 4]
    stp x10, x11, [sp, #16 * 5]
    stp x12, x13, [sp, #16 * 6]
    stp x14, x15, [sp, #16 * 7]
    stp x16, x17, [sp, #16 * 8]
    stp x18, x19, [sp, #16 * 9]
    stp x20, x21, [sp, #16 * 10]
    stp x22, x23, [sp, #16 * 11]
    stp x24, x25, [sp, #16 * 12]
    stp x26, x27, [sp, #16 * 13]
    stp x28, x29, [sp, #16 * 14]

// store exception link register and saved processor state register
    mrs x0, elr_el1
    mrs x1, spsr_el1
    stp x0, x1, [sp, #16 * 15]

// store link register which is x30
    str x30, [sp, #16 * 16]
    mov x0, sp

// call exception handler
    bl \handler

// exit exception
    b .exit_exception
.endm


//--------------------------------------------------------------------------------------------------
// Private Code
//--------------------------------------------------------------------------------------------------

//------------------------------------------------------------------------------
// The exception vector table.
//------------------------------------------------------------------------------
/** When an exception occurs, the processor must execute handler code that corresponds to the exception.
The location in memory where the handler is stored is called the exception vector. In the ARM architecture,
exception vectors are stored in a table, called the exception vector table.

Each Exception level has its own vector table, that is, there is one for each of EL3, EL2, and EL1. The table contains
instructions to be executed, rather than a set of addresses. These would normally be branch instructions that direct the
core to the full exception handler.

The exception vector table for EL1, for example, holds instructions for handling all types of exception that can occur at EL1,
Vectors for individual exceptions are at fixed offsets from the beginning of the table. The virtual address of each table base
is set by the Vector Based Address Registers: VBAR_EL3, VBAR_EL2 and VBAR_EL1.

Each entry in the vector table is 16 instructions long (in ARMv7-A and AArch32, each entry is only 4 bytes). This means that in
AArch64 the top-level handler can be written directly in the vector table.

The base address is given by VBAR_ELn and each entry has a defined offset from this base address. Each table has 16 entries,
with each entry being 128 bytes (32 instructions) in size. The table effectively consists of 4 sets of 4 entries. Which entry
is used depends on several factors:

The type of exception (SError, FIQ, IRQ, or Synchronous)
If the exception is being taken at the same Exception level, the stack pointer to be used (SP0 or SPn)
If the exception is being taken at a lower Exception level, the Execution state of the next lower level (AArch64 or AArch32).
*/



.section .text.exceptions_vector_table
// Export a symbol for the Rust code to use.
.globl exception_vector_table
exception_vector_table:

.org 0x0000
    EXCEPTION_VECTOR el1_sp0_sync

.org 0x0080
    EXCEPTION_VECTOR el1_sp0_irq

.org 0x0100
    EXCEPTION_VECTOR el1_sp0_fiq

.org 0x0180
    EXCEPTION_VECTOR el1_sp0_error

.org 0x0200
    EXCEPTION_VECTOR el1_sync

.org 0x0280
    EXCEPTION_VECTOR el1_irq

.org 0x0300
    EXCEPTION_VECTOR el1_fiq

.org 0x0380
    EXCEPTION_VECTOR el1_error

.org 0x0400
    EXCEPTION_VECTOR el0_sync

.org 0x0480
    EXCEPTION_VECTOR el0_irq

.org 0x0500
    EXCEPTION_VECTOR el0_fiq

.org 0x0580
    EXCEPTION_VECTOR el0_error

.org 0x0600
    EXCEPTION_VECTOR el0_32_sync

.org 0x0680
    EXCEPTION_VECTOR el0_32_irq

.org 0x0700
    EXCEPTION_VECTOR el0_32_fiq

.org 0x0780
    EXCEPTION_VECTOR el0_32_error

.org 0x0800

.exit_exception:
// restore link register
    ldr x30, [sp, #16 * 16]

// restore exception link register and saved processor state register
    ldp x0, x1, [sp, #16 * 15]
    msr elr_el1, x0
    msr spsr_el1, x1

// restore general purpose registers
    ldp x28, x29, [sp, #16 * 14]
    ldp x26, x27, [sp, #16 * 13]
    ldp x24, x25, [sp, #16 * 12]
    ldp x22, x23, [sp, #16 * 11]
    ldp x20, x21, [sp, #16 * 10]
    ldp x18, x19, [sp, #16 * 9]
    ldp x16, x17, [sp, #16 * 8]
    ldp x14, x15, [sp, #16 * 7]
    ldp x12, x13, [sp, #16 * 6]
    ldp x10, x11, [sp, #16 * 5]
    ldp x8, x9, [sp, #16 * 4]
    ldp x6, x7, [sp, #16 * 3]
    ldp x4, x5, [sp, #16 * 2]
    ldp x2, x3, [sp, #16 * 1]
    ldp x0, x1, [sp, #16 * 0]

// restore stack pointer
    add sp, sp, #CONTEXT_SIZE
    eret
```

编辑`src/interrupts.rs`，文末引入`exceptions.s`，同时定义结构`ExceptionCtx`，与`src/exceptions.s`中`EXCEPTION_VECTOR`宏保存的**寄存器数据**对应。
```
// ······
// 注意：这里的······代表承接并省略上文代码
use core::arch::global_asm;
global_asm!(include_str!("exceptions.s"));

#[repr(C)]
pub struct ExceptionCtx {
    regs: [u64; 30],
    elr_el1: u64,
    spsr_el1: u64,
    lr: u64,
}
```
继续编辑`src/interrupts.rs`，在`EXCEPTION_VECTOR`宏中，每一类**中断**都对应一个**处理函数**。
```
// ······
const EL1_SP0_SYNC: &'static str = "EL1_SP0_SYNC";
const EL1_SP0_IRQ: &'static str = "EL1_SP0_IRQ";
const EL1_SP0_FIQ: &'static str = "EL1_SP0_FIQ";
const EL1_SP0_ERROR: &'static str = "EL1_SP0_ERROR";
const EL1_SYNC: &'static str = "EL1_SYNC";
const EL1_IRQ: &'static str = "EL1_IRQ";
const EL1_FIQ: &'static str = "EL1_FIQ";
const EL1_ERROR: &'static str = "EL1_ERROR";
const EL0_SYNC: &'static str = "EL0_SYNC";
const EL0_IRQ: &'static str = "EL0_IRQ";
const EL0_FIQ: &'static str = "EL0_FIQ";
const EL0_ERROR: &'static str = "EL0_ERROR";
const EL0_32_SYNC: &'static str = "EL0_32_SYNC";
const EL0_32_IRQ: &'static str = "EL0_32_IRQ";
const EL0_32_FIQ: &'static str = "EL0_32_FIQ";
const EL0_32_ERROR: &'static str = "EL0_32_ERROR";

// 调用我们的print!宏打印异常信息，你也可以选择打印异常发生时所有寄存器的信息
fn catch(ctx: &mut ExceptionCtx, name: &str) {
    crate::print!(
        "\n  \
        {} @ 0x{:016x}\n\n ",
        name,
        ctx.elr_el1,
    );
}

#[no_mangle]
unsafe extern "C" fn el1_sp0_sync(ctx: &mut ExceptionCtx) {
    catch(ctx, EL1_SP0_SYNC);
}

#[no_mangle]
unsafe extern "C" fn el1_sp0_irq(ctx: &mut ExceptionCtx) {
    catch(ctx, EL1_SP0_IRQ);
}

#[no_mangle]
unsafe extern "C" fn el1_sp0_fiq(ctx: &mut ExceptionCtx) {
    catch(ctx, EL1_SP0_FIQ);
}

#[no_mangle]
unsafe extern "C" fn el1_sp0_error(ctx: &mut ExceptionCtx) {
    catch(ctx, EL1_SP0_ERROR);
}

#[no_mangle]
unsafe extern "C" fn el1_sync(ctx: &mut ExceptionCtx) {
    catch(ctx, EL1_SYNC);
}

#[no_mangle]
unsafe extern "C" fn el1_irq(ctx: &mut ExceptionCtx) {
    catch(ctx, EL1_IRQ);
}

#[no_mangle]
unsafe extern "C" fn el1_fiq(ctx: &mut ExceptionCtx) {
    catch(ctx, EL1_FIQ);
}

#[no_mangle]
unsafe extern "C" fn el1_error(ctx: &mut ExceptionCtx) {
    catch(ctx, EL1_ERROR);
}

#[no_mangle]
unsafe extern "C" fn el0_sync(ctx: &mut ExceptionCtx) {
    catch(ctx, EL0_SYNC);
}

#[no_mangle]
unsafe extern "C" fn el0_irq(ctx: &mut ExceptionCtx) {
    catch(ctx, EL0_IRQ);
}

#[no_mangle]
unsafe extern "C" fn el0_fiq(ctx: &mut ExceptionCtx) {
    catch(ctx, EL0_FIQ);
}

#[no_mangle]
unsafe extern "C" fn el0_error(ctx: &mut ExceptionCtx) {
    catch(ctx, EL0_ERROR);
}

#[no_mangle]
unsafe extern "C" fn el0_32_sync(ctx: &mut ExceptionCtx) {
    catch(ctx, EL0_32_SYNC);
}

#[no_mangle]
unsafe extern "C" fn el0_32_irq(ctx: &mut ExceptionCtx) {
    catch(ctx, EL0_32_IRQ);
}

#[no_mangle]
unsafe extern "C" fn el0_32_fiq(ctx: &mut ExceptionCtx) {
    catch(ctx, EL0_32_FIQ);
}

#[no_mangle]
unsafe extern "C" fn el0_32_error(ctx: &mut ExceptionCtx) {
    catch(ctx, EL0_32_ERROR);
}
```

编辑`src/start.s`，载入**异常向量表**`exception_vector_table`
```
// ······
        mov     sp, x30

        // Initialize exceptions
        ldr     x0, =exception_vector_table
        msr     vbar_el1, x0
        isb

        bl      not_main
// ······
```
{% asset_img start.s.png start.s %}

编辑`aarch64-qemu.ld`，处理链接脚本，为`exceptions.s`中定义的`exceptions_vector_table`选择位置，同时满足**4K对齐**。
```
// ······
    .text.boot : { *(.text.boot) }
    .text :
    {
        KEEP(*(.text.boot))
        *(.text.exceptions)
        . = ALIGN(4096); /* align for exceptions_vector_table*/
        *(.text.exceptions_vector_table)
        *(.text)
    }
    .data : { *(.data) }
// ······
```
{% asset_img aarch64-qemu.ld.png aarch64-qemu.ld %}

编辑`src/main.rs`，引入`interrupts.rs`模块，并在`not_main()`函数中注释掉之前的输出代码，调用`init_gicv2()`函数
```
// ······
mod panic;
mod uart_console;
mod interrupts;

global_asm!(include_str!("start.s"));
// ······
#[no_mangle] // 不修改函数名
pub extern "C" fn not_main() {
    // const UART0: *mut u8 = 0x0900_0000 as *mut u8;
    // let out_str = b"AArch64 Bare Metal";
    // for byte in out_str {
    //     unsafe {
    //         ptr::write_volatile(UART0, *byte);
    //     }
    // }
    // // print_something(); // 调用测试函数
    // println!("\n[0] Hello from Rust!");
    interrupts::init_gicv2();
}
// ······
```
{% asset_img main.rs.png main.rs %}
至此，我们已经在**EL1级别**定义了完整的**中断处理框架**，可以开始处理实际的**中断**了。

### 2) 使能时钟中断
编辑`src/interrupts.rs`，在`init_gicv2`函数中添加**使能时钟中断**，同时配置**时钟**每秒产生一次**中断**。
```
// ······
use core::arch::asm;
pub fn init_gicv2() {
    // ······
    // 启用distributor和cpu interface
    unsafe {
        ptr::write_volatile(GICD_CTLR, GICD_CTLR_ENABLE);
        ptr::write_volatile(GICC_CTLR, GICC_CTLR_ENABLE);
    }
    
    // 电平触发
    const ICFGR_LEVEL: u32 = 0;
    // 时钟中断号30
    const TIMER_IRQ: u32 = 30;
    set_config(TIMER_IRQ, ICFGR_LEVEL); // 电平触发
    set_priority(TIMER_IRQ, 0); // 优先级设定
    clear(TIMER_IRQ); // 清除中断请求
    enable(TIMER_IRQ); // 使能中断
    
    //配置timer
    unsafe {
        asm!("mrs x1, CNTFRQ_EL0"); // 读取系统频率
        asm!("msr CNTP_TVAL_EL0, x1");  // 设置定时寄存器
        asm!("mov x0, 1");
        asm!("msr CNTP_CTL_EL0, x0"); // enable=1, imask=0, istatus= 0,
        asm!("msr daifclr, #2");
    }
}
// ······
```
{% asset_img interrupts.rs.png interrupts.rs %}

### 3) 调试
**编译**并以**调试模式**运行
```
cargo build && qemu-system-aarch64 -machine virt -m 1024M -cpu cortex-a53 -nographic -kernel target/aarch64-unknown-none-softfloat/debug/rui_armv8_os -S -s
```
**保持**此**终端会话**，**新开一个终端**，配置**GDB环境**
```
cd ~/ToolChain/gcc-arm-10.3-2021.07-x86_64-aarch64-none-elf/bin
export ToolChainPath=`pwd`
cd ~/rui_armv8_os
export PATH=$ToolChainPath:$PATH
```
启动**GDB调试客户端**
```
aarch64-none-elf-gdb target/aarch64-unknown-none-softfloat/debug/rui_armv8_os
```
连接**远程客户端**
```
target remote localhost:1234
```
在`not_main()`函数处**设置断点**
```
b not_main
```
运行到`interrupts::init_gicv2();`语句之前。
```
n
```
{% asset_img init_gicv2().png init_gicv2() %}
我们之前在`init_gicv2()`函数中加入了以下代码
```
// 电平触发
const ICFGR_LEVEL: u32 = 0;
// 时钟中断号30
const TIMER_IRQ: u32 = 30;
set_config(TIMER_IRQ, ICFGR_LEVEL); //电平触发
set_priority(TIMER_IRQ, 0); //优先级设定
clear(TIMER_IRQ); //清除中断请求
enable(TIMER_IRQ); //使能中断
```
因此，当我们运行`init_gicv2()`函数后，其中的`enable(TIMER_IRQ);`会产生**使能中断**。我们查看`enable()`函数的代码
```
// 使能中断号为interrupt的中断
pub fn enable(interrupt: u32) {
    unsafe {
        ptr::write_volatile(
            GICD_ISENABLER.add((interrupt / GICD_ISENABLER_SIZE) as usize),
            1 << (interrupt % GICD_ISENABLER_SIZE)
        );
    }
}
```
由此可知，该函数对`GICD_ISENABLER + interrupt / GICD_ISENABLER_SIZE`对应的地址**易失性**写入`1 << (interrupt % GICD_ISENABLER_SIZE)`。

我们之前在`src/interrupts.rs`中定义`GICD`寄存器内存映射`GICD_BASE`的起始地址为`0x08000000`，而`GICD_ISENABLER`的地址为`GICD_BASE + 0x0100 = 0x08000100`，`GICD_ISENABLER_SIZE`为`32`，`TIMER_IRQ`为`30`。
```
const GICD_BASE: u64 = 0x08000000;
const GICD_ISENABLER: *mut u32 = (GICD_BASE + 0x0100) as *mut u32;
const GICD_ISENABLER_SIZE: u32 = 32;
const TIMER_IRQ: u32 = 30;
```

因此，对于`enable(TIMER_IRQ);`，我们可以理解为在`0x08000100`中**易失性**写入**1左移30位**后的**二进制数**。

查看`0x08000100`地址中的值
```
x/t 0x08000100
```
我们得到了`0000000000000000111111111111111`，继续运行，执行`interrupts::init_gicv2();`，再次查看`0x08000100`地址中的值，此时变为了`0100000000000000111111111111111`
{% asset_img init_gicv2()2.png init_gicv2()2 %}
~~由此证明**中断**产生了。~~
实际上这里**并没有产生中断**！我们只是**初始化**了**GICV2**并且写入**TIMER_IRQ中断号**，如果**时钟中断**生效了，那么理论上来说**每隔一秒**都会**调用一次**`el1_irq()`回调函数并且**打印相应的中断信息**，**哪里出问题了呢？**

# 四*、实现真正的时钟中断
## 1. 整理代码
> 参考代码：{% asset_link ScienceFourClean.tar.gz 下载 %}
> 

在实现真正的**时钟中断**之前，我们的**代码**已经有**亿**点乱了，并且还会有很多**恼人**的**unused warnings**，因此我们先**整理**一下**代码**。
首先打开`src/main.rs`：
{% asset_img 4.1.1.png 4.1.1 %}
注意到这里的`core::ptr`并没有被使用。
{% asset_img 4.1.2.png 4.1.2 %}
因此我们将其**移除**。
{% asset_img 4.1.3.png 4.1.3 %}
`not_main()`函数中**移除不需要的代码**
{% asset_img 4.1.4.png 4.1.4 %}
只保留一个``println!宏``以及**中断初始化函数**``init_gicv2()`即可。
`print_something()`函数我们亦不再用到，**移除其相关代码**。
{% asset_img 4.1.5.png 4.1.5 %}
现在看起来就**清爽**多了~
接下来在`src/interrupts.rs`中有很多**没有用到的常量和函数**，通常称为`dead_code`，但是为了**保证完整性**我们**不选择删除它们**，而是**忽略**掉。
在`src/main.rs`中加入
```
#![allow(dead_code)] // 忽略dead_code
```
{% asset_img 4.2.1.png 4.2.1 %}
最后一个**warning**在`aarch64-unknown-none-softfloat.json`中
```
"abi-blacklist": [
      "stdcall",
      "fastcall",
      "vectorcall",
      "thiscall",
      "win64",
      "sysv64"
    ],
```
这个`abi-blacklist`推测是**屏蔽一些接口**，我们**并没有调用这些接口**，所以**直接移除**。
{% asset_img 4.3.1.png 4.3.1 %}
至此我们的**warnings**已经 **全部处理(忽略)** 完了。
{% asset_img 4.3.2.png 4.3.2 %}



## 2. 实现
> 参考代码：{% asset_link ScienceFourPlus.tar.gz 下载 %}
> 
在**查阅大量的资料**后，我找到了本次实验的**原型(?)**[LeOS](https://github.com/lowenware/leos-kernel)以及其对应的**时钟中断**部分的[博客](https://lowenware.com/blog/osdev/aarch64-gic-and-timer-interrupt/)。仔细阅读可以发现他实现**时钟中断**的[Commit](https://github.com/lowenware/leos-kernel/commit/7e89a52f91a98bdcbc1357091159e9391aff2d8d#diff-b02a7b840232145efa38636f47d5c4e8e2ea0cd3d98b449ffd3908e38d2dadc1L1)。
在与[noionion](https://noionion.top/)的**合作**及其**帮助**下，我们发现了**LeOS**关于**时钟中断**的实现与[实验四 中断](https://os2022exps-doc.readthedocs.io/zh_CN/latest/exp4/index.html)中有一些**不一样**的地方：

在**初始化中断**时，**LeOS**还多了以下**代码**:
{% asset_img Loop.png Loop %}
因此我们在`src/interrupts.rs`下的`init_gicv2()`函数**尾部**添加以下代码：
```
loop {
    unsafe {
        asm!("mrs x0, CNTPCT_EL0"); // 系统计数器
        asm!("mrs x0, CNTP_CTL_EL0"); // 控制计数器
        asm!("mrs x0, CNTP_TVAL_EL0"); // 定时计数器
        asm!("mrs x0, CNTP_CVAL_EL0"); // 比较计数器
        asm!("wfi"); // Wait for Interrupt 等待中断，下一次中断发生前都停止在此处
    }
}
```
{% asset_img 开启中断回调.png 开启中断回调 %}
**编译**并**运行**
```
cargo build && qemu-system-aarch64 -machine virt -m 1024M -cpu cortex-a53 -nographic -kernel target/aarch64-unknown-none-softfloat/debug/rui_armv8_os
```
{% asset_img 第一次触发.png 第一次触发 %}
在**运行**后，首先会**输出**
```
[0] Hello from Rust!
```
大约**1s**后会**输出**
```
EL1_IRQ @ 0x
```
说明这次我们**成功**调用了`el1_irq()`**回调函数**！
但**问题**是，**时钟中断**的**理想状态**应是每隔**1s**就会调用一次`el1_irq()`**回调函数**，这里调用一次后却**不再变化**了。
这里其实是因为`catch()`函数在调用第一个参数`ctx`时会**发生阻塞**，**具体原因不详**。
因此我们**修改该函数**，编辑`interrupts.rs`：
```
// ······
// 调用我们的print!宏打印异常信息，你也可以选择打印异常发生时所有寄存器的信息
fn catch(ctx: &mut ExceptionCtx, name: &str) {
    crate::print!("{}\n", name);
}
// ······
```
{% asset_img 修改catch.png 修改catch %}
然后**编译运行**
{% asset_img 不停输出.png 不停输出 %}
可以发现现在确实能够**一直触发中断**并且**输出回调函数名**了。但这每次**输出间隔**的**时间太短**了吧！
这里我们需要了解一些**概念**：
- 在**ARM体系结构**中，**处理器内部**有**通用计时器**，**通用计时器**包含一组**比较器**，用来与**系统计数器(CNTPCT_EL0)**进行比较，一旦**通用计时器**的值**小于等于系统计数器**时便会产生**时钟中断**。
- **比较寄存器(CNTP_CVAL_EL0)** 有64位，如果**设置**了之后，当**系统计数器达到或超过**了这个**值**之后，就会**触发定时器中断**。
- **定时寄存器(CNTP_TVAL_EL0)** 有32位，如果**设置**了之后，会将**比较寄存器**设置成当前**系统计数器**加上设置的**定时寄存器**的值。
>> 详见[此处](https://github.com/2X-ercha/blogOS-armV8/blob/44d7bf5b7296be0e01398459456b265fa68e4e6d/src/main.rs#L64)

因此我们若想要有**延时效果**，需要在调用`el1_irq()`**回调函数**时**再次写入定时寄存器**。
```
asm!("mrs x1, CNTFRQ_EL0");
asm!("msr CNTP_TVAL_EL0, x1");
```
{% asset_img 写入定时寄存器.png 写入定时寄存器 %}
此时再**编译运行**，我们就已经**成功**做到**每1s**处理一次**时钟中断**了！
{% asset_img 每一秒触发一次.png 每一秒触发一次 %}





#  五、输入

#  六、GPIO关机

#  七、死锁与简单处理

#  八、内存管理