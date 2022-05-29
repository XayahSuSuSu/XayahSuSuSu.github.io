---
title: BlogOS：ARM v8之旅
date: 2022-02-26 15:02:36
tags: [ 'BlogOS', '移植' ]
---

# 前言
**BlogOS**是**Philipp Oppermann**用**Rust语言**编写的**面向x86架构**的**简单操作系统**。

**《ARM v8之旅》**将作为 **[湖南大学2022年操作系统课程实验](https://os2022exps-doc.readthedocs.io/zh_CN/latest/index.html)** 个人参考**笔记**。
更**详细的解析**请参考 **[rust写个操作系统：课程实验blogos移至armV8深度解析](https://noionion.top/16433.html)** 。

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
只保留一个`println!宏`以及**中断初始化函数**`init_gicv2()`即可。
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
> 这里的五句 **asm!** 其实**前四句**是**无效操作**，可以**仅执行最后一句**。
> 另外更严谨地说，该 **loop {}** 亦可放在 **not_main()** 函数中，调用 **init_gicv2()** 初始化后。

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
这里其实是因为`catch()`函数在调用第一个参数`ctx`时会**发生阻塞**，~~**具体原因不详**。~~
> 若想**修复**这个**问题**，可以编辑`.cargo/config.toml`，**清空**然后改为**以下内容**
> ```
> [unstable]
> build-std = ["core", "compiler_builtins"] 
> 
> [build]
> target = "aarch64-unknown-none-softfloat"
> rustflags = ["-C","link-arg=-Taarch64-qemu.ld", "-C", "target-cpu=cortex-a53", "-D", "warnings"]
> ```
> 

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
> 参考代码：{% asset_link ScienceFive.tar.gz 下载 %}
> 
> **QEMU**的**virt机器**默认没有**键盘**作为**输入设备**，但当我们执行**QEMU**使用`-nographic`参数（Disable graphical output and redirect serial I/Os to console）时**QEMU**会将**串口重定向**到**控制台**，因此我们可以使用**UART**作为**输入设备**。
> 

## 1. 安装tock-registers库
> 在**实验四**中，针对**GICD**，**GICC**，**TIMER**等**硬件**我们定义了**大量**的**常量**和**寄存器值**，这在使用时过于**繁琐**也**容易出错**。因此我们决定采用**tock-registers**库。
> 

在`Cargo.toml`中的`[dependencies]`处中加入**依赖**：
```
tock-registers = "0.7.0"
```
{% asset_img 添加tr依赖.png 添加tr依赖 %}

## 2. 重构
> 为了不至于使`src/uart_console.rs`文件过长，我们选择**重构**`uart_console.rs`。

首先进入项目**根目录**，创建`src/uart_console`目录
```
mkdir src/uart_console
```
将原`uart_console.rs`**更名**为`mod.rs`，且置于`src/uart_console`目录下
```
mv src/uart_console.rs src/uart_console/mod.rs
```
最后**新建**`src/uart_console/pl011.rs`
```
touch src/uart_console/pl011.rs
```
依据**tock_registers**库的要求对`pl011`所涉及到的**寄存器**进行描述。
编辑`src/uart_console/pl011.rs`
```
use tock_registers::{registers::{ReadOnly, ReadWrite, WriteOnly}, register_bitfields, register_structs};

pub const PL011REGS: *mut PL011Regs = (0x0900_0000) as *mut PL011Regs;

register_bitfields![
    u32,

    pub UARTDR [
        DATA OFFSET(0) NUMBITS(8) []
    ],
    /// Flag Register
    pub UARTFR [
        /// Transmit FIFO full. The meaning of this bit depends on the
        /// state of the FEN bit in the UARTLCR_ LCRH Register. If the
        /// FIFO is disabled, this bit is set when the transmit
        /// holding register is full. If the FIFO is enabled, the TXFF
        /// bit is set when the transmit FIFO is full.
        TXFF OFFSET(6) NUMBITS(1) [],

        /// Receive FIFO empty. The meaning of this bit depends on the
        /// state of the FEN bit in the UARTLCR_H Register. If the
        /// FIFO is disabled, this bit is set when the receive holding
        /// register is empty. If the FIFO is enabled, the RXFE bit is
        /// set when the receive FIFO is empty.
        RXFE OFFSET(4) NUMBITS(1) []
    ],

    /// Integer Baud rate divisor
    pub UARTIBRD [
        /// Integer Baud rate divisor
        IBRD OFFSET(0) NUMBITS(16) []
    ],

    /// Fractional Baud rate divisor
    pub UARTFBRD [
        /// Fractional Baud rate divisor
        FBRD OFFSET(0) NUMBITS(6) []
    ],

    /// Line Control register
    pub UARTLCR_H [
        /// Parity enable. If this bit is set to 1, parity checking and generation
        /// is enabled, else parity is disabled and no parity bit added to the data frame.
        PEN OFFSET(1) NUMBITS(1) [
            Disabled = 0,
            Enabled = 1
        ],
        /// Two stop bits select. If this bit is set to 1, two stop bits are transmitted
        /// at the end of the frame.
        STP2 OFFSET(3) NUMBITS(1) [
            Stop1 = 0,
            Stop2 = 1
        ],
        /// Enable FIFOs.
        FEN OFFSET(4) NUMBITS(1) [
            Disabled = 0,
            Enabled = 1
        ],

        /// Word length. These bits indicate the number of data bits
        /// transmitted or received in a frame.
        WLEN OFFSET(5) NUMBITS(2) [
            FiveBit = 0b00,
            SixBit = 0b01,
            SevenBit = 0b10,
            EightBit = 0b11
        ]
    ],

    /// Control Register
    pub UARTCR [
        /// Receive enable. If this bit is set to 1, the receive
        /// section of the UART is enabled. Data reception occurs for
        /// UART signals. When the UART is disabled in the middle of
        /// reception, it completes the current character before
        /// stopping.
        RXE    OFFSET(9) NUMBITS(1) [
            Disabled = 0,
            Enabled = 1
        ],

        /// Transmit enable. If this bit is set to 1, the transmit
        /// section of the UART is enabled. Data transmission occurs
        /// for UART signals. When the UART is disabled in the middle
        /// of transmission, it completes the current character before
        /// stopping.
        TXE    OFFSET(8) NUMBITS(1) [
            Disabled = 0,
            Enabled = 1
        ],

        /// UART enable
        UARTEN OFFSET(0) NUMBITS(1) [
            /// If the UART is disabled in the middle of transmission
            /// or reception, it completes the current character
            /// before stopping.
            Disabled = 0,
            Enabled = 1
        ]
    ],

    pub UARTIMSC [
        RXIM OFFSET(4) NUMBITS(1) [
            Disabled = 0,
            Enabled = 1
        ]
    ],
    /// Interupt Clear Register
    pub UARTICR [
        /// Meta field for all pending interrupts
        ALL OFFSET(0) NUMBITS(11) [
            Clear = 0x7ff
        ]
    ]
];

register_structs! {
    pub PL011Regs {
        (0x00 => pub dr: ReadWrite<u32, UARTDR::Register>),                   // 0x00
        (0x04 => __reserved_0),               // 0x04
        (0x18 => pub fr: ReadOnly<u32, UARTFR::Register>),      // 0x18
        (0x1c => __reserved_1),               // 0x1c
        (0x24 => pub ibrd: WriteOnly<u32, UARTIBRD::Register>), // 0x24
        (0x28 => pub fbrd: WriteOnly<u32, UARTFBRD::Register>), // 0x28
        (0x2C => pub lcr_h: WriteOnly<u32, UARTLCR_H::Register>), // 0x2C
        (0x30 => pub cr: WriteOnly<u32, UARTCR::Register>),     // 0x30
        (0x34 => __reserved_2),               // 0x34
        (0x38 => pub imsc: ReadWrite<u32, UARTIMSC::Register>), // 0x38
        (0x44 => pub icr: WriteOnly<u32, UARTICR::Register>),   // 0x44
        (0x48 => @END),
    }
}
```
> **register_bitfields!宏**按照**寄存器**的**位结构**进行**描述**，注意最后要加**分号";"**，只要**注册**自己想**处理**的**位**即可。
> 
> **register_structs!宏**最后需加上 **(0x\*\* => @END)** ，表示结束。

## 3. 数据接收中断
编辑`src/uart_console/mod.rs`，修改**Writer**的**初始化方式**
```
// ······
use core::{fmt, ptr};

use lazy_static::lazy_static;
use spin::Mutex;

use tock_registers::interfaces::Writeable;

pub mod pl011;
use pl011::*;

lazy_static! {
    /// A global `Writer` instance that can be used for printing to the VGA text buffer.
    ///
    /// Used by the `print!` and `println!` macros.
    pub static ref WRITER: Mutex<Writer> = Mutex::new(Writer::new());
}

/// Like the `print!` macro in the standard library, but prints to the VGA text buffer.
// ······
```
{% asset_img 修改初始化.png 修改初始化 %}

编辑`src/uart_console/mod.rs`，为**Writer**结构实现**构造函数**
```
//嵌入式系统使用串口，而不是vga，直接输出，没有颜色控制，不记录列号，也没有frame buffer，所以采用空结构
pub struct Writer;

//往串口寄存器写入字节和字符串进行输出
impl Writer {
    pub fn write_byte(&mut self, byte: u8) {
        // ······
    }

    pub fn write_string(&mut self, s: &str) {
        // ······
    }

    pub fn new() -> Writer{
        unsafe {
            // pl011 device registers
            let pl011r: &PL011Regs = &*PL011REGS;
    
            // 禁用pl011
            pl011r.cr.write(UARTCR::TXE::Disabled + UARTCR::RXE::Disabled + UARTCR::UARTEN::Disabled);
            // 清空中断状态
            pl011r.icr.write(UARTICR::ALL::Clear);
            // 设定中断mask，需要使能的中断
            pl011r.imsc.write(UARTIMSC::RXIM::Enabled);
            // IBRD = UART_CLK / (16 * BAUD_RATE)
            // FBRD = ROUND((64 * MOD(UART_CLK,(16 * BAUD_RATE))) / (16 * BAUD_RATE))
            // UART_CLK = 24M
            // BAUD_RATE = 115200
            pl011r.ibrd.write(UARTIBRD::IBRD.val(13));
            pl011r.fbrd.write(UARTFBRD::FBRD.val(1));
            // 8N1 FIFO enable
            pl011r.lcr_h.write(UARTLCR_H::WLEN::EightBit + UARTLCR_H::PEN::Disabled + UARTLCR_H::STP2::Stop1
                + UARTLCR_H::FEN::Enabled);
            // enable pl011
            pl011r.cr.write(UARTCR::UARTEN::Enabled + UARTCR::RXE::Enabled + UARTCR::TXE::Enabled);
        }
    
        Writer
    }
}

impl core::fmt::Write for Writer {
// ······
```
{% asset_img 构造函数.png 构造函数 %}

继续编辑`src/uart_console/mod.rs`，修改`write_byte()`函数，使用我们通过**宏**描述的**寄存器**
```
//嵌入式系统使用串口，而不是vga，直接输出，没有颜色控制，不记录列号，也没有frame buffer，所以采用空结构
pub struct Writer;

//往串口寄存器写入字节和字符串进行输出
impl Writer {
    pub fn write_byte(&mut self, byte: u8) {
        // const UART0: *mut u8 = 0x0900_0000 as *mut u8;
        unsafe {
            // pl011 device registers
            let pl011r: &PL011Regs = &*PL011REGS;
    
            // ptr::write_volatile(UART0, byte);
            pl011r.dr.write(UARTDR::DATA.val(byte as u32));
        }
    }

    pub fn write_string(&mut self, s: &str) {
    // ······
```
{% asset_img 修改wb函数.png 修改wb函数 %}

编辑`src/interrupts.rs`，修改`init_gicv2()`函数，对**UART**的**数据接收中断**进行**初始化**：
```
// ······
    //配置timer
    unsafe {
        // ······
    }

    // 初始化UART0 中断
    // interrupts = <0x00 0x01 0x04>; SPI, 0x01, level
    set_config(UART0_IRQ, ICFGR_LEVEL); //电平触发
    set_priority(UART0_IRQ, 0); //优先级设定
    // set_core(TIMER_IRQ, 0x1); // 单核实现无需设置中断目标核
    clear(UART0_IRQ); //清除中断请求
    enable(UART0_IRQ); //使能中断

    loop {
        // ······
    }
// ······
```
{% asset_img 初始化设备中断.png 初始化设备中断 %}
编辑`src/interrupts.rs`，接下来我们定义`UART0_IRQ`**全局常量**，同时把`TIMER_IRQ`也修改为**全局常量**
```
// 时钟中断号
const TIMER_IRQ: u32 = 30;
// 设备中断号
const UART0_IRQ: u32 = 33;
```
{% asset_img 全局常量.png 全局常量 %}

继续编辑`src/interrupts.rs`，对**UART**的**数据接收中断**进行**处理**，并修改**timer中断**的**处理方法**，使之**每隔2秒输出一个点**。
在**文末**添加以下**三个函数**。
```
use tock_registers::interfaces::Readable;
fn handle_irq_lines(ctx: &mut ExceptionCtx, _core_num: u32, irq_num: u32) {
    if irq_num == TIMER_IRQ {
        handle_timer_irq(ctx);
    }else if irq_num == UART0_IRQ {
        handle_uart0_rx_irq(ctx);
    }
    else{
        catch(ctx, EL1_IRQ);
    }
}

fn handle_timer_irq(_ctx: &mut ExceptionCtx){

    crate::print!(".");

    // 每2秒产生一次中断
    unsafe {
        asm!("mrs x1, CNTFRQ_EL0");
        asm!("add x1, x1, x1");
        asm!("msr CNTP_TVAL_EL0, x1");
    }

}

fn handle_uart0_rx_irq(_ctx: &mut ExceptionCtx){
    use crate::uart_console::pl011::*;

    crate::print!("\nInput interrupt: ");
    unsafe{
        // pl011 device registers
        let pl011r: &PL011Regs = &*PL011REGS;

        let mut flag = pl011r.fr.read(UARTFR::RXFE);
        while flag != 1 {
            let value = pl011r.dr.read(UARTDR::DATA);

            crate::print!("{}", value as u8 as char);
            flag = pl011r.fr.read(UARTFR::RXFE);
        }
    }
}
```

继续编辑`src/interrupts.rs`，修改`el1_irq()`函数
```
#[no_mangle]
unsafe extern "C" fn el1_irq(ctx: &mut ExceptionCtx) {
    // reads this register to obtain the interrupt ID of the signaled interrupt.
    // This read acts as an acknowledge for the interrupt.
    // 中断确认
    const GICC_IAR: *mut u32 = (GICC_BASE + 0x0c) as *mut u32;
    const GICC_EOIR: *mut u32 = (GICC_BASE + 0x10) as *mut u32;
    let value: u32 = ptr::read_volatile(GICC_IAR);
    let irq_num: u32 = value & 0x1ff;
    let core_num: u32 = value & 0xe00;

    // 实际处理中断
    handle_irq_lines(ctx, core_num, irq_num);
    // catch(ctx, EL1_IRQ);

    // A processor writes to this register to inform the CPU interface either:
    // • that it has completed the processing of the specified interrupt
    // • in a GICv2 implementation, when the appropriate GICC_CTLR.EOImode bit is set to 1, to indicate that the interface should perform priority drop for the specified interrupt.
    // 标记中断完成，清除相应中断位
    ptr::write_volatile(GICC_EOIR, core_num | irq_num);
    clear(irq_num);
}
```
{% asset_img 修改irq函数.png 修改irq函数 %}

之前我们修改了`catch()`函数，没有调用`ctx`参数，所以会有一个**warning**，这里我们选择再次将其**忽略**
编辑`src/main.rs`
```
#![allow(dead_code, unused_variables)] // 忽略dead_code
```
{% asset_img 忽略未使用变量.png 忽略未使用变量 %}

并且在`src/uart_console/mod.rs`中，有一个未使用的`core::ptr`引用，将其移除
{% asset_img 移除ptr.png 移除ptr %}

接下来**编译**并**运行**
```
cargo build && qemu-system-aarch64 -machine virt -m 1024M -cpu cortex-a53 -nographic -kernel target/aarch64-unknown-none-softfloat/debug/rui_armv8_os
```
{% asset_img 每两秒输出一次.png 每两秒输出一次 %}
可以看到每过**2s**就会**打一个点**。
如果我们按**顺序**输入**a**、**b**、**c**，则会**触发输入中断**
{% asset_img 输入中断.png 输入中断 %}

#  六、GPIO关机
> 参考代码：{% asset_link ScienceSix.tar.gz 下载 %}

## 1. 原理
查看`virt.dts`，可以**发现**：
```
gpio-keys {
	#address-cells = <0x01>;
	#size-cells = <0x00>;
	compatible = "gpio-keys";

	poweroff {
		gpios = <0x8003 0x03 0x00>;
		linux,code = <0x74>;
		label = "GPIO Key Poweroff";
	};
};

pl061@9030000 {
	phandle = <0x8003>;
	clock-names = "apb_pclk";
	clocks = <0x8000>;
	interrupts = <0x00 0x07 0x04>;
	gpio-controller;
	#gpio-cells = <0x02>;
	compatible = "arm,pl061\0arm,primecell";
	reg = <0x00 0x9030000 0x00 0x1000>;
};
```
> 其中**gpio-keys**中定义了一个**poweroff**键， **gpios = <0x8003 0x03 0x00>** 中的第一项**0x8003**表示它的**phandle**是**0x8003**， 即**pl061@9030000**，也即**gpio-keys**是设备**pl061**的组成部分，第二项**0x03**表示该键是**pl061**的**第三根GPIO线**，第三项是**flag**，且**pl061**的寄存器映射到了内存**0x9030000**开始的位置。如下图所示。
> {% asset_img gpio-poweroff.png gpio-poweroff %}

## 2. 实现
新建`src/pl061.rs`
```
touch src/pl061.rs
```
编辑`src/pl061.rs`，通过**tock-registers**描述**寄存器**
```
use tock_registers::{registers::{ReadWrite, WriteOnly}, register_bitfields, register_structs};

pub const PL061REGS: *mut PL061Regs = (0x0903_0000) as *mut PL061Regs;

register_bitfields![
    u32,
    pub GPIOIE [
        IO3 OFFSET(3) NUMBITS(1) [
            Disabled = 0,
            Enabled = 1
        ]
    ],
];

register_structs! {
    pub PL061Regs {
        (0x000 => __reserved_0),                                               // 0x000
        (0x410 => pub ie: ReadWrite<u32, GPIOIE::Register>),                   // 0x410
        (0x414 => __reserved_1),                                               // 0x414
        (0x41C => pub ic: WriteOnly<u32>),                                     // 0x41C
        (0x420 => @END),                                                       // 0x420
    }
}
```
编辑`src/main.rs`，引入**pl061**模块。
```
// ······
use core::arch::global_asm; // 导入需要的Module

mod panic;
mod uart_console;
mod interrupts;
mod pl061;

global_asm!(include_str!("start.s"));
// ······
```
{% asset_img mod_pl061.png mod_pl061 %}
编辑`src/interrupts.rs`，在`init_gicv2`函数中**初始化pl061的GPIO中断**
```
// ······
// 时钟中断号
const TIMER_IRQ: u32 = 30;
// 设备中断号
const UART0_IRQ: u32 = 33;

const GPIO_IRQ: u32 = 39; // virt.dts interrupts = <0x00 0x07 0x04>; 32 + 0x07 = 39

pub fn init_gicv2() {
    // 初始化Gicv2的distributor和cpu interface
    // 禁用distributor和cpu interface后进行相应配置
    // ······

    // 初始化GPIO中断
    set_config(GPIO_IRQ, ICFGR_LEVEL); //电平触发
    set_priority(GPIO_IRQ, 0); //优先级设定
    // set_core(TIMER_IRQ, 0x1); // 单核实现无需设置中断目标核
    clear(GPIO_IRQ); //清除中断请求
    enable(GPIO_IRQ); //使能中断

    // 使能GPIO的poweroff key中断
    use crate::pl061::*;
    unsafe{
        let pl061r: &PL061Regs = &*PL061REGS;

        // 启用pl061 gpio中的3号线中断
        pl061r.ie.write(GPIOIE::IO3::Enabled);
    }

    loop {
        // ······
    }
}
// ······
```
{% asset_img 初始化关机中断.png 初始化关机中断 %}
编辑`src/interrupts.rs`，引入`tock_registers::interfaces::Writeable`
```
// ······
use tock_registers::interfaces::Readable;
use tock_registers::interfaces::Writeable;
fn handle_irq_lines(ctx: &mut ExceptionCtx, _core_num: u32, irq_num: u32) {
    // ······
}
// ······
```
{% asset_img 引入Writeable.png 引入Writeable %}
编辑`src/interrupts.rs`，处理`pl061`**3号GPIO线**引发的**中断**
```
// ······
fn handle_irq_lines(ctx: &mut ExceptionCtx, _core_num: u32, irq_num: u32) {
    if irq_num == TIMER_IRQ {
        handle_timer_irq(ctx);
    }else if irq_num == UART0_IRQ {
        handle_uart0_rx_irq(ctx);
    }else if irq_num == GPIO_IRQ {
        handle_gpio_irq(ctx);
    }
    else{
        catch(ctx, EL1_IRQ);
    }
}

fn handle_timer_irq(_ctx: &mut ExceptionCtx){
    // ······
}
// ······
fn handle_gpio_irq(_ctx: &mut ExceptionCtx){
    use crate::pl061::*;
    crate::println!("Power off!\n");
    unsafe {
        let pl061r: &PL061Regs = &*PL061REGS;

        // 清除中断信号
        pl061r.ic.set(pl061r.ie.get());
        // 关机
        asm!("mov w0, #0x18");
        asm!("hlt #0xF000");
    }
}
```
{% asset_img 关机中断处理1.png 关机中断处理1 %}
{% asset_img 关机中断处理2.png 关机中断处理2 %}
> 在 **handle_gpio_irq()** 函数里通过**内联汇编**执行了指令**hlt #0xF000**，这里用到了**ARM**的**Semihosting**功能。
> 
> - **Semihosting**的作用：能够让**bare-metal**的**ARM**设备通过**拦截指定的SVC指令**，在连**操作系统**都没有的环境中实现**POSIX**中的许多**标准函数**，比如**printf**、**scanf**、**open**、**read**、**write**等等。这些**IO**操作将被**Semihosting**协议转发到**Host主机**上，然后由**主机代为执行**。

编辑`src/interrupts.rs`，停止打点，方便后续测试观察
```
// ······
fn handle_timer_irq(_ctx: &mut ExceptionCtx){

    // crate::print!(".");

    // 每2秒产生一次中断
    unsafe {
        asm!("mrs x1, CNTFRQ_EL0");
        asm!("add x1, x1, x1");
        asm!("msr CNTP_TVAL_EL0, x1");
    }

}
// ······
```
{% asset_img 停止打点.png 停止打点 %}

## 3. 执行
> 为了启用**Semihosting**功能，在**QEMU**执行时需要加入 **-semihosting** 参数
```
cargo build && qemu-system-aarch64 -machine virt,gic-version=2 -cpu cortex-a57 -nographic -kernel target/aarch64-unknown-none-softfloat/debug/rui_armv8_os -semihosting
```
先按`Ctrl + A`，松手再按`C`，然后输入`system_powerdown`执行关机。
{% asset_img 执行关机.png 执行关机 %}

#  七、死锁与简单处理
> 参考代码：{% asset_link ScienceSeven.tar.gz 下载 %}
> 
> 当**多个任务**访问**同一个资源（数据）**时就会引发**竞争条件问题**，这不仅在**进程间**会出现，在**操作系统**和**进程间**也会出现。由**竞争条件**引发的问题很难**复现**和**调试**，这也是其**最困难**的地方。本**实验**的目的在于**了解竞争条件**和**死锁现象**，并**掌握**处理这些问题的**初步方法**等。

## 勘误
> 在**src/interrupts.rs**的**init_gicv2()**函数中，我们之前使用了一个**循环**并使用**内联汇编asm!("wfi")**来**等待中断**，实际上在之前的实验中，这里所有的**内联汇编**都是没有必要的。当去掉这个**循环**，我们的**OS**会**串行执行完成**然后**自动关机**，从而导致后续的测试**无效**。因此我们**需要且仅需要**一个**空循环**来**使OS持续运行**，以便后续的**中断测试**。

编辑`src/interrupts.rs`，在`init_gicv2()`函数中移除`loop{}`循环。
{% asset_img 去除无效内联汇编.png 去除无效内联汇编 %}

编辑`src/main.rs`，在`not_main()`函数尾部添加`loop{}`空循环。
```
// ······
#[no_mangle] // 不修改函数名
pub extern "C" fn not_main() {
    println!("\n[0] Hello from Rust!\n");
    interrupts::init_gicv2();
    loop {}
}
```
{% asset_img 添加空循环.png 添加空循环 %}

## 死锁的复现
首先编辑`src/main.rs`，在`not_main()`函数的空循环中调用`print!`宏
```
// ······
global_asm!(include_str!("start.s"));

#[no_mangle] // 不修改函数名
pub extern "C" fn not_main() {
    println!("\n[0] Hello from Rust!\n");
    interrupts::init_gicv2();
    loop {
        print!("-");
    }
}
```
{% asset_img loop1.png loop1 %}

这里有两种方式复现死锁现象。

### 1. loop{}中`print!`宏与`handle_uart0_rx_irq()`中`print!`宏竞争
检查`src/interrupts.rs`中的`handle_uart0_rx_irq()`函数，可以看到我们之前写了一个输入中断回调函数，在函数中调用了`print!`宏输出信息。
{% asset_img 输入中断函数.png 输入中断函数 %}

直接编译并运行，预期在输入时触发死锁。
```
cargo build && qemu-system-aarch64 -machine virt,gic-version=2 -cpu cortex-a57 -nographic -kernel target/aarch64-unknown-none-softfloat/debug/rui_armv8_os -semihosting
```

不停地乱序敲击键盘，此时有概率出现卡死，按键无法再次输入内容，即触发死锁现象。
{% asset_img 死锁.png 死锁 %}

### 2. loop{}中`print!`宏与`handle_timer_irq()`中`print!`宏竞争
检查`src/interrupts.rs`中的`handle_timer_irq()`函数，可以看到我们之前写了一个时间中断回调函数，在函数中调用了`print!`宏打点。

但它之前被我们注释掉了，因此我们取消注释
{% asset_img 重新打点.png 重新打点 %}

然后我们编译并运行，预期在打第一个点时会触发死锁。
```
cargo build && qemu-system-aarch64 -machine virt,gic-version=2 -cpu cortex-a57 -nographic -kernel target/aarch64-unknown-none-softfloat/debug/rui_armv8_os -semihosting
```

实验按预期触发了死锁。
{% asset_img 打点死锁.png 打点死锁 %}
有时会在打第二个点时触发死锁。
{% asset_img 打点死锁2.png 打点死锁2 %}

## 死锁的简单处理
编辑`src/uart_console/mod.rs`，引入`asm!`宏
```
// ······
impl core::fmt::Write for Writer {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        // ······
    }
}

use core::{fmt, arch::asm};

use lazy_static::lazy_static;
use spin::Mutex;

use tock_registers::interfaces::Writeable;

pub mod pl011;
use pl011::*;
// ······
```
{% asset_img 引入asm.png 引入asm %}

编辑`src/uart_console/mod.rs`中的`_print()`函数，在处理输入时先关闭中断，再打开。
```
// ······
/// Prints the given formatted string to the VGA text buffer through the global `WRITER` instance.
#[doc(hidden)]
pub fn _print(args: fmt::Arguments) {
    use core::fmt::Write;
    unsafe {
        // 关闭d a i f类型的中断
        asm!("msr daifset, #0xf");
    }

    WRITER.lock().write_fmt(args).unwrap();

    unsafe {
        // 仅打开i类型的中断，不支持嵌套，嵌套应该保存状态，然后再恢复之前的状态
        asm!("msr daifclr, #2");
    }
}
```
{% asset_img 修复死锁.png 修复死锁 %}

## 验证
此时再用上述两种方式测试死锁，发现死锁现象消失了~

#  八、内存管理
> 分页内存管理是内存管理的基本方法之一。本实验的目的在于全面理解分页式内存管理的基本方法以及访问页表，完成地址转换等的方法。

## ARM v8的地址转换
> **[ARM Cortex-A Series Programmer's Guide for ARMv8-A](https://developer.arm.com/documentation/den0024/a/The-Memory-Management-Unit/Context-switching)** 中提到：
> 
> For EL0 and EL1, there are two translation tables. TTBR0_EL1 provides translations for the bottom of Virtual Address space, which is typically application space and TTBR1_EL1 covers the top of Virtual Address space, typically kernel space. This split means that the OS mappings do not have to be replicated in the translation tables of each task.
> 
> 即**TTBR0**指向**虚拟空间下半部分**通常用于**应用程序**的空间，**TTBR1**指向**虚拟空间上半部分**通常用于**内核**的空间。其中**TTBR0**除了在**EL1**中存在外，也在**EL2**和**EL3**中存在，但**TTBR1**只在**EL1**中存在。
> 
> **TTBR0_ELn**和**TTBR1_ELn**是页表**基地址寄存器**，**地址转换**的过程如下所示 
> {% asset_img v2p-translate.svg v2p-translate %}

## 一、使用Identity Mapping映射
> 参考代码：{% asset_link ScienceEight1.tar.gz 下载 %}
> 
> **虚拟地址转换**很**容易出错**也**很难调试**，所以我们从**最简单的方式**开始，即采用**Identity Mapping**，将**虚拟地址**映射到**相同**的**物理地址**。

编辑`src/start.s`，初始化MMU、页表以及启用页表。
```
.globl _start
.extern LD_STACK_PTR
.section ".text.boot"

_start:
        ldr     x30, =LD_STACK_PTR
        mov     sp, x30

        // Initialize exceptions
        ldr     x0, =exception_vector_table
        msr     vbar_el1, x0
        isb

_setup_mmu:
        // 初始化TCR控制寄存器
        ldr     x0, =TCR_EL1_VALUE
        msr     tcr_el1, x0
        ldr     x0, =MAIR_EL1_VALUE
        msr     mair_el1, x0            // 内存属性间接寄存器，作用是预先定义好属性，然后通过索引来访问这些预定义的属性

_setup_pagetable:
        // 因为采用的36位地址空间，所以是一级页表
        ldr     x1, =LD_TTBR0_BASE
        msr     ttbr0_el1, x1           //页表基地址TTBR0
        ldr     x2, =LD_TTBR1_BASE
        msr     ttbr1_el1, x2           //页表基地址TTBR1

        // 一级页表部分
        // 虚拟地址空间的下半部分采用Identity Mapping
        // 第一项 虚拟地址0 - 1G，根据virt的定义为flash和外设，参见virt.c
        ldr     x3, =0x0
        lsr     x4, x3, #30             // 除以1G
        lsl     x5, x4, #30             // 乘以1G，并且将表索引保存在x0
        ldr     x6, =PERIPHERALS_ATTR
        orr     x5, x5, x6              // 添加符号
        str     x5, [x1], #8
        // 第二项 虚拟地址1G - 2G，_start部分
        ldr     x3, =_start
        lsr     x4, x3, #30             // 除以1G
        lsl     x5, x4, #30             // 乘以1G，并且将表索引保存在x0
        ldr     x6, =IDENTITY_MAP_ATTR
        orr     x5, x5, x6              // 添加符号
        str     x5, [x1], #8

_enable_mmu:
        // 启用MMU.
        mrs     x0, sctlr_el1
        orr     x0, x0, #0x1
        msr     sctlr_el1, x0
        dsb     sy                      // Programmer’s Guide for ARMv8-A chapter13.2 Barriers
        isb

_start_main:
        bl      not_main

.equ PSCI_SYSTEM_OFF, 0x84000002
.globl system_off
system_off:
        ldr     x0, =PSCI_SYSTEM_OFF
        hvc     #0

.equ TCR_EL1_VALUE, 0x1B55C351C
// ---------------------------------------------
// IPS   | b001    << 32 | 36bits address space - 64GB
// TG1   | b10     << 30 | 4KB granule size for TTBR1_EL1
// SH1   | b11     << 28 | 页表所在memory: Inner shareable
// ORGN1 | b01     << 26 | 页表所在memory: Normal, Outer Wr.Back Rd.alloc Wr.alloc Cacheble
// IRGN1 | b01     << 24 | 页表所在memory: Normal, Inner Wr.Back Rd.alloc Wr.alloc Cacheble
// EPD   | b0      << 23 | Perform translation table walk using TTBR1_EL1
// A1    | b1      << 22 | TTBR1_EL1.ASID defined the ASID
// T1SZ  | b011100 << 16 | Memory region 2^(64-28) -> 0xffffffexxxxxxxxx
// TG0   | b00     << 14 | 4KB granule size
// SH0   | b11     << 12 | 页表所在memory: Inner Sharebale
// ORGN0 | b01     << 10 | 页表所在memory: Normal, Outer Wr.Back Rd.alloc Wr.alloc Cacheble
// IRGN0 | b01     << 8  | 页表所在memory: Normal, Inner Wr.Back Rd.alloc Wr.alloc Cacheble
// EPD0  | b0      << 7  | Perform translation table walk using TTBR0_EL1
// 0     | b0      << 6  | Zero field (reserve)
// T0SZ  | b011100 << 0  | Memory region 2^(64-28)

.equ MAIR_EL1_VALUE, 0xFF440C0400
// ---------------------------------------------
//                   INDX         MAIR
// DEVICE_nGnRnE    b000(0)     b00000000
// DEVICE_nGnRE         b001(1)         b00000100
// DEVICE_GRE               b010(2)     b00001100
// NORMAL_NC                b011(3)     b01000100
// NORMAL               b100(4)         b11111111

.equ PERIPHERALS_ATTR, 0x60000000000601
// -------------------------------------
// UXN   | b1      << 54 | Unprivileged eXecute Never
// PXN   | b1      << 53 | Privileged eXecute Never
// AF    | b1      << 10 | Access Flag
// SH    | b10     << 8  | Outer shareable
// AP    | b01     << 6  | R/W, EL0 access denied
// NS    | b0      << 5  | Security bit (EL3 and Secure EL1 only)
// INDX  | b000    << 2  | Attribute index in MAIR_ELn，参见MAIR_EL1_VALUE
// ENTRY | b01     << 0  | Block entry

.equ IDENTITY_MAP_ATTR, 0x40000000000711
// ------------------------------------
// UXN   | b1      << 54 | Unprivileged eXecute Never
// PXN   | b0      << 53 | Privileged eXecute Never
// AF    | b1      << 10 | Access Flag
// SH    | b11     << 8  | Inner shareable
// AP    | b00     << 6  | R/W, EL0 access denied
// NS    | b0      << 5  | Security bit (EL3 and Secure EL1 only)
// INDX  | b100    << 2  | Attribute index in MAIR_ELn，参见MAIR_EL1_VALUE
// ENTRY | b01     << 0  | Block entry
```
{% asset_img 编辑start.png 编辑start %}
(如果预览不清晰，可以在新标签页中打开图片，或者下载图片，然后放大)

编辑`aarch64-qemu.ld`，定义前文中用到的LD_TTBR0_BASE和LD_TTBR1_BASE符号
```
ENTRY(_start)
SECTIONS
{
    . = 0x40080000;
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
    .rodata : { *(.rodata) }
    .bss : { *(.bss) }

    . = ALIGN(8);
    . = . + 0x4000;
    LD_STACK_PTR = .;

    . = ALIGN(4096);
    /*页表基地址TTBR0*/
    LD_TTBR0_BASE = .;
    . = . + 0x1000;

    /*页表基地址TTBR1*/
    LD_TTBR1_BASE = .;
    . = . + 0x1000;
}
```
{% asset_img 编辑ld.png 编辑ld %}

编译并运行，测试能否正常工作。
```
cargo clean && cargo build && qemu-system-aarch64 -machine virt,gic-version=2 -cpu cortex-a57 -nographic -kernel target/aarch64-unknown-none-softfloat/debug/rui_armv8_os -semihosting
```
{% asset_img 第一次测试.png 第一次测试 %}
正常运行！

## 二、使用Identity Mapping映射 - 偏移映射与页面共享
> 参考代码：{% asset_link ScienceEight2.tar.gz 下载 %}
> 
> 修改代码，将**虚拟地址**2G - 3G处映射到**物理地址**0 - 1G，从而对**0x89000000**地址的写入将通过**pl011**串口输出，因为此时**0x89000000**映射到了**物理地址pl011@9000000**。

编辑`src/start.s`，空白映射**虚拟地址**0 - 1G，将**虚拟地址**2G - 3G处映射到**物理地址**0 - 1G。
```
// ······
_setup_pagetable:
        // 因为采用的36位地址空间，所以是一级页表
        ldr     x1, =LD_TTBR0_BASE
        msr     ttbr0_el1, x1           //页表基地址TTBR0
        ldr     x2, =LD_TTBR1_BASE
        msr     ttbr1_el1, x2           //页表基地址TTBR1

        // 一级页表部分
        // 虚拟地址空间的下半部分采用Identity Mapping
        // 第一项 虚拟地址0 - 1G
        ldr     x5, =0x0
        str     x5, [x1], #8
        // 第二项 虚拟地址1G - 2G，_start部分
        ldr     x3, =_start
        lsr     x4, x3, #30             // 除以1G
        lsl     x5, x4, #30             // 乘以1G，并且将表索引保存在x0
        ldr     x6, =IDENTITY_MAP_ATTR
        orr     x5, x5, x6              // 添加符号
        str     x5, [x1], #8
        // 第三项 虚拟地址2 - 3G，根据virt的定义为flash和外设，参见virt.c
        ldr     x3, =0x0
        lsr     x4, x3, #30             // 除以1G
        lsl     x5, x4, #30             // 乘以1G，并且将表索引保存在x0
        ldr     x6, =PERIPHERALS_ATTR
        orr     x5, x5, x6              // 添加符号
        str     x5, [x1], #8
// ······
```
{% asset_img 映射23G.png 映射23G %}

编辑`src/interrupts.rs`，修改其基址（2G+原基址）
```
use core::ptr;

// GICD和GICC寄存器内存映射后的起始地址
const GICD_BASE: u64 = 0x8000_0000 + 0x08000000;
const GICC_BASE: u64 = 0x8000_0000 + 0x08010000;

// Distributor
// ······
```
{% asset_img 修改中断.png 修改中断 %}

编辑`src/pl061.rs`，修改其基址（2G+原基址）
```
use tock_registers::{registers::{ReadWrite, WriteOnly}, register_bitfields, register_structs};

pub const PL061REGS: *mut PL061Regs = (0x8000_0000u32 + 0x0903_0000) as *mut PL061Regs;
// ······
```
{% asset_img 修改pl061.png 修改pl061 %}

编辑`src/uart_console/pl011.rs`，修改其基址（2G+原基址）
```
use tock_registers::{registers::{ReadOnly, ReadWrite, WriteOnly}, register_bitfields, register_structs};

pub const PL011REGS: *mut PL011Regs = (0x8000_0000u32 +0x0900_0000) as *mut PL011Regs;
// ······
```
{% asset_img 修改pl011.png 修改pl011 %}

编译并运行，测试能否正常工作。
```
cargo clean && cargo build && qemu-system-aarch64 -machine virt,gic-version=2 -cpu cortex-a57 -nographic -kernel target/aarch64-unknown-none-softfloat/debug/rui_armv8_os -semihosting
```
{% asset_img 第二次测试.png 第二次测试 %}
正常运行！

## 三、使用非Identity Mapping映射 - 块级映射
> 参考代码：{% asset_link ScienceEight3.tar.gz 下载 %}

编辑`src/start.s`，处理虚拟地址空间的上半部分。
```
// ······
_setup_pagetable:
        // 因为采用的36位地址空间，所以是一级页表
        ldr     x1, =LD_TTBR0_BASE
        msr     ttbr0_el1, x1           //页表基地址TTBR0
        ldr     x2, =LD_TTBR1_BASE
        msr     ttbr1_el1, x2           //页表基地址TTBR1

        // 一级页表部分
        // 虚拟地址空间的下半部分采用Identity Mapping
        // 第一项 虚拟地址0 - 1G
        ldr     x5, =0x0
        str     x5, [x1], #8
        // 第二项 虚拟地址1G - 2G，_start部分
        ldr     x3, =0x40010000
        lsr     x4, x3, #30             // 除以1G
        lsl     x5, x4, #30             // 乘以1G，并且将表索引保存在x0
        ldr     x6, =IDENTITY_MAP_ATTR
        orr     x5, x5, x6              // 添加符号
        str     x5, [x1], #8

        // 虚拟地址空间的上半部分采用非Identity Mapping
        // 第一项 虚拟地址0 - 1G，根据virt的定义为flash和外设，参见virt.c
        ldr     x3, =0x0 //
        lsr     x4, x3, #30             // 除以1G
        lsl     x5, x4, #30             // 乘以1G，并且将表索引保存在x0
        ldr     x6, =PERIPHERALS_ATTR
        orr     x5, x5, x6              // 添加符号
        str     x5, [x2], #8

        // 第二项， 映射到内存（块级映射）
        ldr     x3, =0x40010000
        lsr     x4, x3, #30             // 除以1G
        lsl     x5, x4, #30             // 乘以1G，并且将表索引保存在x0
        ldr     x6, =KERNEL_ATTR
        orr     x5, x5, x6              // 添加符号
        str     x5, [x2], #8

_enable_mmu:
// ······

.equ KERNEL_ATTR, 0x40000000000711
// -------------------------------------
// UXN   | b1      << 54 | Unprivileged eXecute Never
// PXN   | b0      << 53 | Privileged eXecute Never
// AF    | b1      << 10 | Access Flag
// SH    | b11     << 8  | Inner shareable
// AP    | b00     << 6  | R/W, EL0 access denied
// NS    | b0      << 5  | Security bit (EL3 and Secure EL1 only)
// INDX  | b100    << 2  | Attribute index in MAIR_ELn，参见MAIR_EL1_VALUE
// ENTRY | b01     << 0  | Block entry
```
{% asset_img start1.png start1 %}

重构`aarch64-qemu.ld`
```
__KERN_VMA_BASE = 0xfffffff000000000;
__PHY_DRAM_START_ADDR = 0x40000000;
__PHY_START_LOAD_ADDR = 0x40010000;

ENTRY(__PHY_START_LOAD_ADDR)
SECTIONS
{
    . = __KERN_VMA_BASE + __PHY_START_LOAD_ADDR;
    .text.boot : AT(__PHY_START_LOAD_ADDR) { KEEP(*(.text.boot)) }
    .text :
    {
        *(.text*)
    }
    . = ALIGN(0x1000);

    LD_DATA_BASE = .;
    .data : { *(.data*) }
    . = ALIGN(0x1000);

    LD_RODATA_BASE = .;
    .rodata : { *(.rodata*) }
    . = ALIGN(0x1000);

    LD_BSS_BASE = .;
    .bss :
    { 
        *(.bss*)
        . = ALIGN(4096);
        . += (4096 * 100); /* 栈的大小 */
        stack_top = .;
        LD_STACK_PTR = .;
    }

    /* 页表 */
    .pt :
        {
        . = ALIGN(4096);

        /* 页表基地址TTBR0 */
        LD_TTBR0_BASE = . - __KERN_VMA_BASE;
        . = . + 0x1000;

        /* 页表基地址TTBR1 */
        LD_TTBR1_BASE = . - __KERN_VMA_BASE;
        . = . + 0x1000;
        }

    . = . + 0x1000;
    LD_KERNEL_END = . - __KERN_VMA_BASE;
}
```

编辑`src/interrupts.rs`，修改其基址（0xfffffff000000000+原基址）
```
use core::ptr;

// GICD和GICC寄存器内存映射后的起始地址
const GICD_BASE: u64 = 0xfffffff000000000 + 0x08000000;
const GICC_BASE: u64 = 0xfffffff000000000 + 0x08010000;

// Distributor
// ······
```
{% asset_img 修改中断3.png 修改中断3 %}

编辑`src/pl061.rs`，修改其基址（0xfffffff000000000+原基址）
```
use tock_registers::{registers::{ReadWrite, WriteOnly}, register_bitfields, register_structs};

pub const PL061REGS: *mut PL061Regs = (0xfffffff000000000u64 + 0x0903_0000) as *mut PL061Regs;
// ······
```
{% asset_img 修改pl0613.png 修改pl0613 %}

编辑`src/uart_console/pl011.rs`，修改其基址（0xfffffff000000000+原基址）
```
use tock_registers::{registers::{ReadOnly, ReadWrite, WriteOnly}, register_bitfields, register_structs};

pub const PL011REGS: *mut PL011Regs = (0xfffffff000000000u64 + 0x0900_0000) as *mut PL011Regs;
// ······
```
{% asset_img 修改pl0113.png 修改pl0113 %}

编译并运行，测试能否正常工作。
```
cargo clean && cargo build && qemu-system-aarch64 -machine virt,gic-version=2 -cpu cortex-a57 -nographic -kernel target/aarch64-unknown-none-softfloat/debug/rui_armv8_os -semihosting
```
{% asset_img 第三次测试.png 第三次测试 %}
~~正常运行！~~

### 修复异常现象
> 参考代码：{% asset_link ScienceEight3Fix.tar.gz 下载 %}

乍一看能正常运行，但是运行一段时间后居然卡死了！这是为什么呢？
第一时间想到的是互斥锁可能出问题了，但是仔细看互斥锁的代码发现和内存映射关系应该不大。
仔细观察输出发现打点是正常的，而且在没有卡死之前如果触发输入中断则会立刻卡死，因此判断是输入中断出了问题，[noionion](https://noionion.top/)认为是链接脚本的问题，而事实也如他所说。

编辑`aarch64-qemu.ld`，修改.text : {}部分：
```
/* ······ */
.text.boot : AT(__PHY_START_LOAD_ADDR) { KEEP(*(.text.boot)) }
.text :
{
    KEEP(*(.text.boot))
    *(.text.exceptions)
    . = ALIGN(4096); /* align for exceptions_vector_table*/
    *(.text.exceptions_vector_table)
    *(.text)
}
. = ALIGN(0x1000);
/* ······ */
```
{% asset_img 修复ld.png 修复ld %}

编译并运行，测试能否正常工作。
```
cargo clean && cargo build && qemu-system-aarch64 -machine virt,gic-version=2 -cpu cortex-a57 -nographic -kernel target/aarch64-unknown-none-softfloat/debug/rui_armv8_os -semihosting
```
{% asset_img 第三点五次测试.png 第三点五次测试 %}
正常运行！

## 四、使用非Identity Mapping映射 - 页表映射
> 未完待续。