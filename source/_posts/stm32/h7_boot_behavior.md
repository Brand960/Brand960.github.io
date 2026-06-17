---
title: stm32h750vbt6 启动流程
date: 2026-06-18 06:36:04
tags: stm32
academia: true
---

## 上电/复位

完整启动时序：稳压器/带隙稳定 → VOS3 → HSI起振 → FL_PWR Flash上电 → FL_OPTB 选项字节加载 → CPU运行（HSI时钟起振作为复位默认源，后续可SystemInit修改）
<!-- more -->

![](enable1.png)
![](enable.png)

> 参考：`stm32h750-参考手册reference.pdf`（RM0433）—— 复位与时钟控制（RCC）/ 电源控制器（PWR）章节中的“上电复位（POR/PDR、带隙稳定）/ 系统复位”

## 启动配置

锁存BOOT引脚电平，选择启动地址源，自举地址（Boot Address）是复位后向量表基地址。CPU 据此读取初始 MSP 与复位向量入口

![](bootconfig1.png)
![](bootconfig2.png)

BOOT_ADD数值来源为Optional Bytes，Flash 接口寄存器自动从 Flash 加载用户选项字节，每个字段都是 16 位位宽

![](optional_bytes0.png)
![](optional_bytes.png)
![](optional_bytes1.png)

> 参考：`stm32h750-参考手册reference.pdf`（RM0433）—— 嵌入式 Flash 存储器接口（FLASH）章节“Boot configuration（启动配置）”与“Option bytes（选项字节，BOOT_ADDx）”；另见 `stm32h750-数据手册datasheet.pdf`（DS12556）§3.4 Boot modes

### BOOT_ADD存储寄存器位置

根据Option Bytes中 BOOT_ADDx 的值，更新 FLASH_BOOT_CURR 寄存器，进行地址有效性检查

![](flashinterfacereg.png)
![](bootaddr_busmatrix.png)
![](bootaddreg0.png)
![](bootaddreg.png)

> 参考：`stm32h750-参考手册reference.pdf`（RM0433）—— Flash 存储器接口寄存器映射中的 `FLASH_BOOT_CURR`（当前启动地址寄存器，地址 0x5200_2040），及其对 BOOT_ADDx 的地址有效性检查

### 查询BOOT_ADD当前值

SWD 方式连接 JLink 和开发板，Boot(pin)=0 & BOOT_ADD0=0x1FF0（或 Boot(pin)=1 & BOOT_ADD1=0x1FF0）模式下

```bash
$ openocd -f interface/jlink.cfg -c "transport select swd" -f target/stm32h7x.cfg
xPack Open On-Chip Debugger 0.12.0+dev-02228-ge5888bda3-dirty (2025-10-04-22:44)
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : J-Link Pro V4 complied Jun 21 2023 09:20:55
Info : Hardware version: 13.00
Info : VTarget = 3.322 V
Info : clock speed 1800 kHz
Info : SWD DPIDR 0x6ba02477
Info : [stm32h7x.ap2] Examination succeed
Info : [stm32h7x.cpu0] Cortex-M7 r1p1 processor detected
Warn : [stm32h7x.cpu0] Erratum 3092511: Cortex-M7 can halt in an incorrect address when breakpoint and exception occurs simultaneously
Info : [stm32h7x.cpu0] The erratum 3092511 workaround will resume after an incorrect halt
Info : [stm32h7x.cpu0] target has 8 breakpoints, 4 watchpoints
Info : [stm32h7x.cpu0] Examination succeed
Info : [stm32h7x.ap2] gdb port disabled
Info : [stm32h7x.cpu0] starting gdb server on 3333
Info : Listening on port 3333 for gdb connections
```

另开终端执行，可见bit[15:0](即 BOOT_ADD0 这个字段)= 0x0080 是 Flash 地址;0x1FF0(bootloader)属于 BOOT_ADD1,在寄存器高半字，符合选项字节出厂默认值

```bash
$ telnet localhost 4444
Open On-Chip Debugger                  
$ mdw 0x52002040                   
0x52002040: 1ff0080
# FLASH_BOOT_CURR: BOOT_ADD1[15:0]=0x1FF0(系统bootloader 0x1FF00000),
#                  BOOT_ADD0[15:0]=0x0080(Flash 0x08000000)
```

> 参考：基于 `stm32h750-参考手册reference.pdf`（RM0433）`FLASH_BOOT_CURR` 寄存器地址（0x5200_2040）

## CPU核心寄存器

![](core_registers.png)

> 参考：`arm-v7-m编程手册programming.pdf`（PM0253，STM32F7/H7 Cortex®-M7 processor programming manual）—— 第 2 章“Programmer's model”核心寄存器汇总

### 控制寄存器

bit[1]控制SP寄存器为MSP或PSP

![](MSP&PSP.png)

控制寄存器bit[2]表示是否激活了浮点上下文（复位为 0）。使用 MSR 指令将 CONTROL.SPSEL 位（当前活动堆栈指针位）设置为 1 可将线程模式中使用的堆栈指针切换到 PSP(Handler 模式下写 SPSEL=1 被忽略，仅 Thread 模式有效)

![](control_register.png)

> 参考：`arm-v7-m编程手册programming.pdf`（PM0253）—— CONTROL register / Figure 7 “Control bit assignments”（bit[1] SPSEL 选 MSP/PSP；bit[2] FPCA 浮点上下文，复位为 0）；Handler 模式写 SPSEL=1 被忽略，仅 Thread 模式有效

### EPSR

EPSR（Execution Program Status Register，执行程序状态寄存器）是 ARMv7-M 核心寄存器中 xPSR 的一部分， 包含 T 位和重叠的 ICI/IT 字段：
- T 位：指示执行 Thumb 指令。Armv7-M 只支持 Thumb 指令集，T 位必须始终为 1，一旦为 0，执行首条指令即触发 INVSTATE UsageFault。
- ICI/IT 字段：支持中断可继续的加载/存储与 IT 指令块。
复位时 EPSR 有两个动作：① 把 T 位置为复位向量 bit[0] 的值（必须为 1）；② 把 IT/ICI 位清 0。这两步直接决定了下面复位序列的取值与跳转方式。

在.s启动文件中开头使用`.thumb`声明thumb指令集编译即可确保Thumb 函数地址的 LSB 置 1，用于指示 CPU 进入 Thumb 状态

> 参考：`arm-v7-m编程手册programming.pdf`（PM0253）—— Table 6 “EPSR bit assignments”（T 位 / ICI·IT 字段）；复位流程见 `cortex-armv7-m文档.pdf`（ARMv7-M Architecture Reference Manual / DDI 0403）Reset 伪代码

### VTOR

向量表（偏移）寄存器VTOR结构说明

![](VTOR.png)

> 参考：`arm-v7-m编程手册programming.pdf`（PM0253）—— Vector table offset register（VTOR）/ Table 54 “VTOR bit assignments”

## reset behavior

在 armv7-m 架构手册中有关于 reset 流程的伪代码,控制寄存器设置 bit[2]=0(FP inactive)与 bit[0]=0(privileged)

![](reset_behavior.png)
![](reset_behavior1.png)

1  控制寄存器bit[1]设为0，即栈寄存器SP使用MSP
2 `bits(32) vectortable = VTOR<31:7>:'0000000'` 取 VTOR向量表偏移寄存器 的高 25 位、低位补 0，得到向量表基址。
3 `SP_main = MemA_with_priv[vectortable, 4, AccType_VECTABLE] AND 0xFFFFFFFC<31:0>;`根据向量表地址开头（已置0）连续读取 4 个字节，并将其赋值给主堆栈指针SP_main（即 MSP）
4 `tmp = MemA_with_priv[vectortable+4, 4, AccType_VECTABLE];`从向量表偏移 +4 处读取 4 字节作为复位异常处理程序入口地址临时变量 tmp
5 `BranchTo(tmp AND 0xFFFFFFFE<31:0>);`将刚才读取的复位异常处理程序的入口物理地址写入 PC，执行硬件级别的跳转（BranchTo）。处理器正式进入异常处理模式，开始执行开发者写在汇编文件（.s）中的 Reset_Handler 第一条指令。

以下为向量表的内容结构及对应偏移量

![](vector_table.png)

**注意** 手册中提到 Reset Exception 是最高优先级的异常,走专属复位序列（不复栈、不保存现场）,实际上复位不经过 NVIC 嵌套向量中断控制器的异常优先级仲裁，是异步的特权复位序列。

![](reset_exception.png)

总而言之，在 STM32H7 上，硬件依据 BOOT_ADD（选项字节）把 VTOR 复位为自举存储区的物理地址， CPU 直接在该物理地址读取初始 MSP 与复位向量入口。.s和.ld文件会固定在开头指定地址烧入Reset_Handler的逻辑，进入 Reset_Handler 后，系统在软件执行初始化阶段（通常在 SystemInit() 函数中）会显式地将实际物理地址写入 VTOR 寄存器 `SCB->VTOR = FLASH_BASE;`，后续所有的中断响应（如定时器、串口中断）都会直接去FLASH_BASE物理空间寻找。
