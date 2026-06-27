---
title: stm32h750vbt6_fpu
date: 2026-06-27 23:22:13
tags: stm32
academia: true
---
# STM32H750VBT6 FPU启用与编译选项

>本文档包含：`cmake/gcc-arm-none-eabi.cmake` 中启用硬件浮点运算相关编译选项的依据，以及硬件浮点寄存器的说明
<!-- more -->
## 1 芯片 FPU 硬件规格

STM32H750VBT6 采用 Cortex-M7 内核，其浮点单元为双精度 FPU

数据手册 DS12556 Rev 8 第 1 页 Features 段：

> 32-bit Arm® Cortex®-M7 core with double precision FPU and L1 cache …

编程手册 PM0253 Rev 5 第 4.7 节（第 233 页）：

> The Cortex®-M7 Floating-Point Unit (FPU) implements the FPv5 floating-point extensions. The FPU fully supports single-precision and double-precision add, subtract, multiply, divide, multiply and accumulate, and square root operations … The FPU contains 32 single-precision extension registers, which can be also accessed as 16 doubleword registers …

Cortex-M7 有两种 FPU 实现，须按芯片型号选择：

| FPU 变体 | 架构名 | GCC `-mfpu=` 取值 | 适用芯片 |
|---|---|---|---|
| 单精度 | FPv5-SP | `fpv5-sp-d16` | 部分 STM32F7 |
| 双精度 | FPv5-DP | `fpv5-d16` | STM32H750 |

取值后缀 `d16` 表示实现 16 个 64 位双字寄存器 D0–D15，等价于 32 个单精度寄存器 S0–S31。Cortex-M 系列只实现 D16。

## 2 硬件浮点寄存器位置

PM0253 第 2.1.3 节的 Table 2 只列整数(核心)寄存器（R0–R12、SP、LR、PC、PSR），不含浮点数据寄存器。浮点寄存器的逐位说明见 ARMv7-M 架构参考手册第 A2.5 节。

### 2.1 数据寄存器 S0–S31 与 D0–D15

出处：ARMv7-M 架构参考手册 DDI 0403E.e，第 A2.5.2 节 The FP extension registers，含 Figure A2-1。

> Software can access the FP extension register bank as:
> • Thirty-two 32-bit single-precision registers, S0-S31.
> • Sixteen 64-bit double-precision registers, D0-D15.
> The extension can use the two views simultaneously. Figure A2-1 shows the relationship between the two views.

![Figure A2-1 寄存器映射图](fpu_regis.png)

图：S0–S31 与 D0–D15 对应关系

### 2.2 状态控制寄存器 FPSCR

出处：ARMv7-M 架构参考手册第 A2.5.3 节 Floating-point Status and Control Register, FPSCR。PM0253 第 4.7.4 节 Table 98（FPSCR bit assignments）给出相同位赋值。

![FPSCR 位域图](FPSCR.png)

图：FPSCR 位赋值（AHP/DN/FZ/RMode 及异常标志）

### 2.3 PM0253 中的相关内容

PM0253 第 4.7 节（第 233–239 页）重点描述 FPU 控制与状态寄存器，不含 S/D 数据寄存器详述表：

| PM0253 章节 | 内容 | 表号 |
|---|---|---|
| 4.7.1 | CPACR，CP10/CP11 访问使能 | Table 95 |
| 4.7.2 | FPCCR，自动入栈与出栈控制 | Table 96 |
| 4.7.3 | FPCAR，浮点上下文地址 | Table 97 |
| 4.7.4 | FPSCR 位赋值 | Table 98 |
| 4.7.5 | FPDSCR，默认状态控制 | Table 99 |
| 3.11 | 浮点指令（VCMP、VMOV 等，操作数写作 `<Sd>`/`<Dd>`） | — |

![PM0253 FPU 系统寄存器总览](FPU_overview.png)

图：Cortex-M7 浮点系统寄存器总览

## 3 编译选项依据

### 3.1 `-mfpu=fpv5-d16`

出处：GCC v15.2.0 手册第 3.20.5 节 ARM Options（p.369–370）。

> -mfpu=name
> This specifies what floating-point hardware (or hardware emulation) is available on the target. Permissible names are: 'auto', 'vfpv2', … 'fpv4-sp-d16', 'neon-vfpv4', 'fpv5-d16', 'fpv5-sp-d16', 'fp-armv8' …

同节对 cortex-m7 双精度的反向佐证（p.369）：

> '+nofp.dp' Disables the double-precision component of the floating-point instructions on … 'cortex-m7'.

GCC 把 cortex-m7 的浮点描述为含双精度分量，故取 `fpv5-d16` 而非 `fpv5-sp-d16`。选错会导致 `double` 运算退化为软件模拟。

### 3.2 `-mfloat-abi=hard`

出处：GCC v15.2.0 手册第 3.20.5 节 ARM Options（p.359）。

> -mfloat-abi=name
> … Permissible values are: 'soft', 'softfp' and 'hard'.
> Specifying 'soft' causes GCC to generate output containing library calls for floating-point operations. 'softfp' allows the generation of code using hardware floating-point instructions, but still uses the soft-float calling conventions. 'hard' allows generation of floating-point instructions and uses FPU-specific calling conventions.
> Note that the hard-float and soft-float ABIs are not link-compatible; you must compile your entire program with the same ABI, and link with a compatible set of libraries.

`hard` 表示浮点形参直接经 FPU 寄存器传递，省去整数寄存器与 FPU 寄存器之间的搬运。由于硬浮点与软浮点 ABI 不兼容，工程内所有库（含 newlib、libm）必须同为 hard ABI；`arm-none-eabi-gcc` 通过 multilib 按 `fpv5-d16/hard` 自动匹配对应库。

## 4 工程配置

`cmake/gcc-arm-none-eabi.cmake` 第 36 行：

```cmake
set(MCU_FLAGS "-mcpu=cortex-m7 -mthumb -mfpu=fpv5-d16 -mfloat-abi=hard")
set(CMAKE_C_FLAGS          "${MCU_FLAGS}" CACHE STRING "" FORCE)
set(CMAKE_ASM_FLAGS        "${MCU_FLAGS}" CACHE STRING "" FORCE)
set(CMAKE_EXE_LINKER_FLAGS "${MCU_FLAGS}" CACHE STRING "" FORCE)
```

三处 `FORCE` 保证 C、汇编、链接及 `try_compile` 探测阶段统一带上 FPU 标志，因为 `CMAKE_*_FLAGS_INIT` 在部分 CMake 版本下不可靠。

## 5 文档出处汇总

| 文档（Zotero 附件） | 章节 / 页码 | 用途 |
|---|---|---|
| STM32H750 数据手册 DS12556 Rev 8（UCJWBMYD） | Features，p.1 | 确认双精度 FPU |
| STM32H7 Cortex-M7 编程手册 PM0253 Rev 5（X7IHELX2） | §4.7，p.233–239；§2.1.3，p.20–22 | FPU 为 FPv5；FP 系统寄存器；整数核寄存器 |
| ARMv7-M 架构参考手册 DDI 0403E.e（B6NMIVWU） | §A2.5.2、§A2.5.3 | 浮点数据寄存器 S/D 与 FPSCR 位域详述 |
| GCC v15.2.0 编译器手册（FGII9T65） | §3.20.5 ARM Options，p.359、p.369–370 | `-mfloat-abi` 与 `-mfpu` 定义 |
