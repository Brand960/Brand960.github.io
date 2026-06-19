---
title: stm32h750vbt6_startupxx.s
date: 2026-06-19 23:12:28
tags: stm32
academia: true
---

>本文档针对stm32h750vbt6 cubemx生成的starupxxx.s文件流程进行分析，使用arm-gcc-toolchain工具链开发
<!-- more -->
## 整体流程

```
┌─────────────────────────────────────────────────────────────┐
│                    芯片上电 / 复位                           │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  硬件自动执行（ARMv7-M ARM Reset 伪代码）：                   │
│  1. 从向量表偏移 0  读取 MSP 初始值（_estack） → 设置 SP      │
│  2. 从向量表偏移 +4 读取 PC 初始值（Reset_Handler） → 跳转    │
│     （向量表基址由 BOOT_ADD/VTOR 决定，本工程=0x08000000）    │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Reset_Handler（startup_stm32h750xx.s）：                   │
│  ① ldr sp, =_estack        ← 设置栈指针（双重保险）          │
│  ② bl  ExitRun0Mode        ← 配置电源模式                   │
│  ③ bl  SystemInit          ← 初始化系统时钟                 │
│  ④ 复制 .data 段（_sidata → _sdata.._edata，Flash→SRAM）    │
│  ⑤ 清零 .bss 段（_sbss.._ebss）                             │
│  ⑥ bl  __libc_init_array   ← C/C++ 静态构造函数             │
│  ⑦ bl  main                ← 进入用户程序                   │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  用户程序: main() 开始执行                                   │
└─────────────────────────────────────────────────────────────┘
```

## 一、文件头声明

声明目标处理器、浮点调用约定与指令集，并选定 Thumb 编码；`.thumb` 使函数地址 LSB=1，以满足硬件复位后进入 Thumb 状态的要求。

### 片段

```asm
  .syntax unified      ; 统一汇编语法（UAL）
  .cpu cortex-m7       ; 目标 CPU 为 Cortex-M7
  .fpu softvfp         ; 软浮点调用约定（在编译阶段可指定用硬件浮点寄存器覆盖）
  .thumb               ; Thumb 指令集编码，函数地址 LSB=1
```

### 语法

| 伪指令            | 含义                                 | 来源                                              |
| ----------------- | ------------------------------------ | ------------------------------------------------- |
| `.syntax unified` | 启用统一汇编语法（UAL），ARM 专用    | **as** §9.4.4 ARM Machine Directives（`.syntax`） |
| `.cpu cortex-m7`  | 指定目标 CPU                         | **as** §9.4.4 ARM Machine Directives（`.cpu`）    |
| `.fpu softvfp`    | 软浮点调用约定                       | **as** §9.4.4 ARM Machine Directives（`.fpu`）    |
| `.thumb`          | 以 Thumb 指令集编码，等价 `.code 16` | **as** §9.4.4 ARM Machine Directives（`.thumb`）  |

### 依据

- `.thumb` 使函数地址 LSB=1、复位后进入 Thumb 状态，对应复位序列将 EPSR.T 位置为复位向量 bit[0]（**PM0253** Table 6 EPSR bit assignments）。
- 复位后硬件据此 `BranchTo` 到 `Reset_Handler`（**ARMv7-M ARM** Reset 伪代码）。

## 二、全局符号声明

导出向量表起始符号与默认中断处理函数两个全局符号，并以 `.word` 占位引用五个由链接脚本解析的下划线符号（`.data`/`.bss` 边界）。

### 片段

```asm
.global  g_pfnVectors       ; 导出向量表起始符号（在.isr_vector段中由ld指定地址）
.global  Default_Handler    ; 导出默认中断处理函数（弱函数作为默认handler）

.word  _sidata              ; .data 初值在 Flash 的起始
.word  _sdata               ; .data 在 RAM 的起始
.word  _edata               ; .data 在 RAM 的结束
.word  _sbss                ; .bss 在 RAM 的起始
.word  _ebss                ; .bss 在 RAM 的结束
```

### 语法

| 伪指令        | 含义                         | 来源                   |
| ------------- | ---------------------------- | ---------------------- |
| `.global sym` | 导出为全局符号，对 `ld` 可见 | **as** §7.43 [.global] |
| `.word expr`  | 分配并写入 1 个字（4 字节）  | **as** §7.115 [.word]  |

### 依据

- `.word` 引用的 `_sdata/_edata` 与 `_sbss/_ebss` 分别对应 `.data`（已初始化）与 `.bss`（零初始化，文件中不占空间）两类节区（**ELF** Book I「Special Sections」）。
- `.bss` 节区类型为 `SHT_NOBITS`，故其初值不存盘、需启动时清零（**ELF** Book I「Section Header」）。

## 三、复位处理程序

复位处理程序完成七步软件初始化：设栈、配置电源与时钟（含写 `SCB->VTOR`）、复制 `.data`、清零 `.bss`、调用静态构造函数，最后跳转 `main()`。

### 片段

```asm
  .section  .text.Reset_Handler       ; 输入段 .text.Reset_Handler
  .weak  Reset_Handler                ; 弱符号，可被用户同名强符号覆盖
  .type  Reset_Handler, %function     ; 标记为函数（STT_FUNC）
Reset_Handler:
  ldr   sp, =_estack                  ; 载入栈顶到 SP

  bl  ExitRun0Mode                    ; 调用电源模式配置
  bl  SystemInit                      ; 调用系统时钟初始化

  ; 复制 .data 段初值（Flash → SRAM）
  ldr   r0, =_sdata                   ; r0 = 写入目标起始
  ldr   r1, =_edata                   ; r1 = 写入目标结束
  ldr   r2, =_sidata                  ; r2 = 读取源起始
  movs  r3, #0                        ; r3 = 偏移计数器清 0
  b     LoopCopyDataInit              ; 跳到条件判断
CopyDataInit:                         ; 循环体
  ldr   r4, [r2, r3]                  ; 从源读取 1 字
  str   r4, [r0, r3]                  ; 写入目标
  adds  r3, r3, #4                    ; 偏移 +4
LoopCopyDataInit:                     ; 条件判断
  adds  r4, r0, r3                    ; r4 = 当前写入地址
  cmp   r4, r1                        ; 比较是否到达结束
  bcc   CopyDataInit                  ; 未到达则继续

  ; 清零 .bss 段
  ldr   r2, =_sbss                    ; r2 = 写入起始
  ldr   r4, =_ebss                    ; r4 = 写入结束
  movs  r3, #0                        ; r3 = 填充值 0
  b     LoopFillZerobss               ; 跳到条件判断
FillZerobss:                          ; 循环体
  str   r3, [r2]                      ; 写入 0
  adds  r2, r2, #4                    ; 指针 +4
LoopFillZerobss:                      ; 条件判断
  cmp   r2, r4                        ; 比较是否到达结束
  bcc   FillZerobss                   ; 未到达则继续

  ; 调用 C/C++ 静态构造函数
  bl    __libc_init_array             ; 遍历 .init_array/.preinit_array

  ; 跳转到用户程序
  bl    main                          ; 进入 main()
  bx    lr                            ; main 返回（裸机下通常 HardFault）

  .size  Reset_Handler, .-Reset_Handler  ; 符号大小 = 当前位置 − 起始
```

### 语法

| 伪指令                         | 含义                        | 来源                                                          |
| ------------------------------ | --------------------------- | ------------------------------------------------------------- |
| `.section .text.Reset_Handler` | 定义输入段                  | **as** §7.88 [.section]                                       |
| `.weak Reset_Handler`          | 弱符号，可被同名强符号覆盖  | **as** §7.113 [.weak]；**ELF** Book I「Symbol Table」STB_WEAK |
| `.type …, %function`           | 标记为函数（`STT_FUNC`）    | **as** §7.105 [.type]；**ELF** STT_FUNC                       |
| `.size …, .-Reset_Handler`     | 用位置计数器 `.` 设符号大小 | **as** §7.92 [.size]                                          |

指令要点：`ldr/str [Rn, Rm]` 基址偏移寻址；`bl` 带链接跳转（返回地址存 `lr`）；`cmp/bcc` 比较后无符号小于则跳转（`C=0`）；`adds` 更新标志位。

### 依据

- 设栈（`ldr sp, =_estack`）对应硬件复位序列已从向量表首项自动载入 MSP（**ARMv7-M ARM** Reset 伪代码），此处为软件层面的重复确认。
- `__libc_init_array` 遍历的构造函数表对应 `.init_array`/`.preinit_array` 节区（**ELF** Book I「Special Sections」）。

## 四、默认中断处理程序

未实现中断的兜底处理函数，直接进入死循环保留现场，供调试器查看。

### 片段

```asm
  .section  .text.Default_Handler,"ax",%progbits  ; 可执行输入段（a=可分配，x=可执行）
Default_Handler:                                   ; 默认处理函数入口
Infinite_Loop:                                     ; 死循环标号
  b  Infinite_Loop                                 ; 无条件跳回自身
  .size  Default_Handler, .-Default_Handler        ; 符号大小
```

### 语法

| 语法                        | 含义                                                                                     | 来源                                                      |
| --------------------------- | ---------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| `.section …,"ax",%progbits` | `"a"`=可分配（`SHF_ALLOC`），`"x"`=可执行（`SHF_EXECINSTR`），`%progbits`=`SHT_PROGBITS` | **as** §7.88 [.section]；**ELF** Book I「Section Header」 |

## 五、中断向量表

上电后 CPU 查询的异常/中断向量表：首项为初始 MSP，次项为复位向量，其后按异常编号顺序排列系统异常与外设 IRQ。

### 片段

```asm
  .section  .isr_vector,"a",%progbits   ; 可分配输入段 .isr_vector
  .type  g_pfnVectors, %object          ; 标记为数据对象（STT_OBJECT）
g_pfnVectors:                            ; 向量表起始
  .word  _estack                         ; 偏移 0x00: 初始 MSP
  .word  Reset_Handler                   ; 偏移 0x04: 复位向量
  .word  NMI_Handler                     ; 偏移 0x08: NMI
  .word  HardFault_Handler               ; 偏移 0x0C: 硬故障
  .word  MemManage_Handler               ; 偏移 0x10: 内存管理故障
  .word  BusFault_Handler                ; 偏移 0x14: 总线故障
  .word  UsageFault_Handler              ; 偏移 0x18: 用法故障
  .word  0                               ; 偏移 0x1C: 保留
  .word  0                               ; 偏移 0x20: 保留
  .word  0                               ; 偏移 0x24: 保留
  .word  0                               ; 偏移 0x28: 保留
  .word  SVC_Handler                     ; 偏移 0x2C: 系统服务调用
  .word  DebugMon_Handler                ; 偏移 0x30: 调试监控
  .word  0                               ; 偏移 0x34: 保留
  .word  PendSV_Handler                  ; 偏移 0x38: PendSV
  .word  SysTick_Handler                 ; 偏移 0x3C: SysTick
  …                                      ; 偏移 0x40 起: 外设 IRQ
```

### 语法

| 语法               | 含义                           | 来源                                      |
| ------------------ | ------------------------------ | ----------------------------------------- |
| `.type …, %object` | 标记为数据对象（`STT_OBJECT`） | **as** §7.105 [.type]；**ELF** STT_OBJECT |

### 依据

- 偏移 0 为 MSP、偏移 +4 为复位向量的取值顺序，依据 ARMv7-M 异常向量布局（**ARMv7-M ARM** Exception Model）。
- 向量表物理位于 `0x08000000`，由 BOOT 引脚/选项字节 BOOT_ADD 经 FLASH 接口重映射，本工程 `BOOT_ADD0=0x0080`（**RM0433** FLASH 章节 Boot configuration / FLASH_BOOT_CURR；**DS12556** §3.4 Boot modes）。

## 六、弱别名表

把向量表中所有未实现的中断处理函数弱绑定到 `Default_Handler`；用户定义同名强符号即可覆盖默认实现，无需修改启动文件。

### 片段

```asm
   .weak      NMI_Handler                     ; 声明为弱符号
   .thumb_set NMI_Handler,Default_Handler     ; 设为 Default_Handler 的 Thumb 别名
   .weak      HardFault_Handler               ; 逐个 IRQ 重复
   .thumb_set HardFault_Handler,Default_Handler
   …
   .weak      WAKEUP_PIN_IRQHandler
   .thumb_set WAKEUP_PIN_IRQHandler,Default_Handler
```

### 语法

| 伪指令            | 含义                                             | 来源                                                          |
| ----------------- | ------------------------------------------------ | ------------------------------------------------------------- |
| `.weak X`         | 声明 `X` 为弱符号（`STB_WEAK`）                  | **as** §7.113 [.weak]；**ELF** Book I「Symbol Table」STB_WEAK |
| `.thumb_set X, Y` | 等价 `.set X, Y` 并标记 Thumb 函数（地址 LSB=1） | **as** §7.89 [.set]；§9.4.4 ARM Machine Directives            |



## 附录

- **as** = `Using as`（GNU Binutils v2.46）
- **ld** = `GNU linker ld`（GNU Binutils v2.46）
- **ELF** = `ELF ABI 规范手册`（TIS ELF v1.2）
- **PM0253** = `STM32F7/H7 Cortex-M7 编程手册`（PM0253 Rev 5）
- **ARMv7-M ARM** = `Arm®v7-M 架构参考手册`（DDI 0403E.e）
- **RM0433** = `STM32H750 参考手册`（RM0433）
- **DS12556** = `STM32H750 数据手册`（DS12556 Rev 8）