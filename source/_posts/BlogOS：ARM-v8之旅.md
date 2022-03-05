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
在`main.rs`中测试`println!`宏
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

#  四、中断

#  五、输入

#  六、GPIO关机

#  七、死锁与简单处理

#  八、内存管理