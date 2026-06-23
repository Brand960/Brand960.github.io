---
title: stm32h750vbt6_linker
date: 2026-06-23 10:16:49
tags: stm32
academia: true
---
# stm32h750vbt6 软件启动流程2（linker）

>本文档针对使用 arm-gcc-toolchain 工具链， stm32h750vbt6 CubeMX 生成的 `STM32H750XX_FLASH.ld`
<!-- more -->
## 零、整体流程

```
┌─────────────────────────────────────────────────────────────┐
│ ENTRY(Reset_Handler)             ← 指定ELF文件入口(e_entry)  │
│ _estack = ORIGIN(DTCMRAM)+LENGTH ← 定义栈顶（向量表首项）     │
│ _Min_Heap_Size / _Min_Stack_Size ← 堆/栈最小值（链接期校验）  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ MEMORY { DTCMRAM / RAM / RAM_D2 / RAM_D3 / ITCMRAM / FLASH }│
│          ← 声明芯片可用存储区及其属性与起始/长度               │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ SECTIONS                                                    │
│  ① .isr_vector  >FLASH          ← 向量表片段                 │
│  ② .text        >FLASH          ← 代码、glue、.init/.fini    │
│  ③ .rodata      >FLASH          ← 只读常量/字符串            │
│  ④ .ARM.extab/.ARM.exidx >FLASH ← 异常索引表（C++ 解栈）      │
│  ⑤ .preinit_array/.init_array/.fini_array >FLASH            │
│                                 ← 静态构造/析构函数表        │
│  ⑥ _sidata = LOADADDR(.data)    ← .data 在 Flash 的 LMA     │
│  ⑦ .data        >DTCMRAM AT>FLASH← VMA 在 RAM、LMA 在 Flash │
│  ⑧ .bss         >DTCMRAM        ← 零初始化段（COMMON 归此）   │
│  ⑨ ._user_heap_stack >DTCMRAM   ← 堆/栈占位（链接期校验）     │
│  ⑩ .RAM_D2      >RAM_D2         ← D2 域 SRAM 自定义段        │
│  ⑪ /DISCARD/                    ← 丢弃标准库符号             │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 启动文件据此从向量表首项取得 _estack、按 _sidata→_sdata/       │
│ _edata 复制 .data、清零 _sbss→_ebss 后进入 main()            │
└─────────────────────────────────────────────────────────────┘
```

## 一、入口与栈堆配置

### 片段

```ld
ENTRY(Reset_Handler)                                /* 程序入口符号（写入 ELF 头 e_entry 字段） */

_estack         = ORIGIN(DTCMRAM) + LENGTH(DTCMRAM); /* 栈顶 = DTCMRAM 末端（向量表首项/初始 MSP） */
_Min_Heap_Size  = 0x200;                            /* 堆最小尺寸（链接期校验） */
_Min_Stack_Size = 0x400;                            /* 栈最小尺寸（链接期校验） */
```

### 语法

| 命令/函数                | 含义                                                   | 来源                                  |
| ------------------------ | ------------------------------------------------------ | ------------------------------------- |
| `ENTRY(symbol)`          | 把 `symbol` 设为程序入口（写入 ELF 头 `e_entry` 字段） | **ld** §3.4.1 Setting the Entry Point |
| `ORIGIN(region)` / `org` | 返回 `MEMORY` 中某 region 的起始地址                   | **ld** §3.10.9 Builtin Functions      |
| `LENGTH(region)` / `len` | 返回 `MEMORY` 中某 region 的长度                       | **ld** §3.10.9 Builtin Functions      |


### 依据

- `ENTRY(Reset_Handler)` 把复位处理程序`Reset_Handler`写入 ELF 头的 `e_entry` 字段（**ELF** Book I「Header」`e_entry`）。
- `_estack = ORIGIN(DTCMRAM) + LENGTH(DTCMRAM)` = `0x20000000 + 0x20000` = `0x20020000`，正是向量表偏移 0 处的初始 MSP（**ARMv7-M ARM** Reset 伪代码读取向量表首项载入 SP），与启动文件 `ldr sp, =_estack` 吻合。
- 栈顶设在 DTCMRAM 末端、栈向下生长，对应 Cortex-M 的满递减栈（**PM0253** §2.1.3 Stack / PUSH/POP）。

```
低地址 0x20000000 ┌──────────────────┐  ← _sdata（.data 段起点）
                 │  .data（已初始化）│
                 ├──────────────────┤
                 │  .bss（未初始化） │
                 ├──────────────────┤
                 │  heap（向上生长） │  ← end / _end
                 │       ↑          │
                 │   （空闲区）      │
                 │       ↓          │
                 │  stack（向下生长）│
高地址 0x20020000 └──────────────────┘  ← _estack（栈顶，RAM 末端）
```

## 二、存储区描述

本段用 `MEMORY` 命令声明芯片可用存储区的起始地址、长度与属性供后续`SECTIONS` 中的 `>region` 即据此将各输出节区指派到对应 region

### 片段

```ld
MEMORY
{
  DTCMRAM (xrw) : ORIGIN = 0x20000000, LENGTH = 128K   /* DTCM，运行时数据/栈 */
  RAM     (xrw) : ORIGIN = 0x24000000, LENGTH = 512K   /* AXI SRAM（D1 域） */
  RAM_D2  (xrw) : ORIGIN = 0x30000000, LENGTH = 288K   /* SRAM1–3（D2 域） */
  RAM_D3  (xrw) : ORIGIN = 0x38000000, LENGTH = 64K    /* SRAM4（D3 域） */
  ITCMRAM (xrw) : ORIGIN = 0x00000000, LENGTH = 64K    /* ITCM，紧耦合指令 */
  FLASH   (rx)  : ORIGIN = 0x80000000,  LENGTH = 128K   /* 内部 AXI Flash，存代码/常量/初值 */
}
```

### 语法

| 语法                        | 含义                                                                    | 来源                       |
| --------------------------- | ----------------------------------------------------------------------- | -------------------------- |
| `MEMORY { name (attr): … }` | 描述存储区，`attr` 取自 `r`/`w`/`x`/`l`/`!`，仅作链接期检查、不影响生成 | **ld** §3.7 MEMORY Command |


### 依据

- 各 region 起始地址与 STM32H750 系统总线存储器映射一致（**RM0433** §2.3.1 Memory map）
- 内部 AXI FLASH 仅= 128 KB（**DS12556** §1 Introduction / §3.4）；片上 RAM 总量约 1 MB（192 KB TCM + 864 KB 用户 SRAM + 4 KB 备份 SRAM）（**RM0433** §2.3 Embedded SRAM）
- ITCMRAM/DTCMRAM 经 TCM 接口直连 Cortex-M7，AXI SRAM 与 D2/D3 SRAM 经各域矩阵访问（**RM0433** §2.3.1 Memory map）

## 三、向量表节区

本段把启动文件中的向量表 `.isr_vector` 段固定放在 FLASH 最起始端

### 片段

```ld
.isr_vector :
{
  . = ALIGN(4);           /* 4 字节对齐 */
  KEEP(*(.isr_vector))    /* 保留向量表，防 --gc-sections 回收 */
  . = ALIGN(4);           /* 4 字节对齐 */
} >FLASH                  /* 运行地址/加载地址均在 FLASH */
```

### 语法

| 语法                   | 含义                                                 | 来源                                                 |
| ---------------------- | ---------------------------------------------------- | ---------------------------------------------------- |
| `KEEP(*(.isr_vector))` | 即便向量表未被普通代码引用，`--gc-sections` 也不回收 | **ld** §3.6.4.4 Input Section and Garbage Collection |
| `. = ALIGN(4)`         | 4 字节对齐，保证每个 `.word` 向量位于字边界          | **ld** §3.6.8.3 Forced Output Alignment              |
| `>FLASH`               | 指派到 FLASH region（VMA /LMA）                      | **ld** §3.6.8.6 Output Section Region                |

### 依据

- 本节区为 `SECTIONS` 中首个节区且 `>FLASH`，即启动文件中向量表 `g_pfnVectors` 所在的 section
- H7 异常 `≈` 166 条,VTOR 基址要求 `≥` 异常表长度的下一个 2 的幂字 = 256 字 = 1024 字节由其位于 FLASH 起始（0x08000000，大边界自然对齐）保证
- `KEEP` 防止向量表被 `--gc-sections` 清除（**ld** §3.6.4.4）

## 四、代码节区

- 代码
- ARM↔Thumb 桩代码(STUB)与 `.init/.fini` 入口/出口桩代码
- 异常帧信息


### 片段

```ld
.text :
{
  . = ALIGN(4);           /* 4 字节对齐 */
  *(.text)                /* 代码节 */
  *(.text*)               /* 代码派生节（如 .text.Reset_Handler） */
  *(.glue_7)              /* ARM→Thumb STUB， ARM/Thumb 混编模板继承的兼容性条目 */
  *(.glue_7t)             /* Thumb→ARM STUB， ARM/Thumb 混编模板继承的兼容性条目 */
  *(.eh_frame)            /* 异常帧信息 */
  KEEP (*(.init))         /* 保留 C 运行时 .init 入口桩(兼容性) */
  KEEP (*(.fini))         /* 保留 C 运行时 .fini 出口桩(兼容性) */
  . = ALIGN(4);           /* 4 字节对齐 */
  _etext = .;             /* 本节结束符号(常用结束位置声明约定符号,此处为兼容性保留) */
} >FLASH                  /* 运行地址/加载地址均在 FLASH */
```

### 依据

- `.init`/`.fini` 为 hosted libc 的单一入口启动/退出桩代码段(在 main() 之前执行)，不符合单片机启动调用流程，故为冗余的兼容性声明（**ELF** Book I「Special Sections」`.init`/`.fini`）
- 启动文件 `Reset_Handler` 所在输入段 `.text.Reset_Handler` 被 `*(.text*)` 通配吸收（**ld** §3.6.4.2）。

## 五、只读数据与异常索引

- 只读常量/字符串
- C++ 异常或 `-funwind-tables` 时的栈展开表 
- 运行时检索边界

### 片段

```ld
.rodata :
{
  . = ALIGN(4);           /* 4 字节对齐 */
  *(.rodata)              /* 只读常量节 */
  *(.rodata*)             /* 只读常量派生节（字符串等） */
  . = ALIGN(4);           /* 4 字节对齐 */
} >FLASH                  /* 运行地址/加载地址均在 FLASH */

.ARM.extab (READONLY) :   /* 节区类型：只读（GCC11+ 语法） */
{
  . = ALIGN(4);           /* 4 字节对齐 */
  *(.ARM.extab* .gnu.linkonce.armextab.*)   /* 异常索引扩展表 */
  . = ALIGN(4);           /* 4 字节对齐 */
} >FLASH                  /* 运行地址/加载地址均在 FLASH */

.ARM (READONLY) :         /* 节区类型：只读（GCC11+ 语法） */
{
  . = ALIGN(4);           /* 4 字节对齐 */
  __exidx_start = .;      /* 异常索引表起始 */
  *(.ARM.exidx*)          /* 异常索引主表 */
  __exidx_end = .;        /* 异常索引表结束 */
  . = ALIGN(4);           /* 4 字节对齐 */
} >FLASH                  /* 运行地址/加载地址均在 FLASH */
```

### 语法

| 语法         | 含义                                    | 来源                                |
| ------------ | --------------------------------------- | ----------------------------------- |
| `(READONLY)` | 节区类型属性，标记为只读（GCC11+ 语法） | **ld** §3.6.8.1 Output Section Type |

### 依据

- `.ARM.exidx`/`.ARM.extab` 是 ARM EHABI 的栈展开节区（**ARMv7-M ARM**「Exception Index Table」；**ELF** Book I「Special Sections」），`__exidx_start/__exidx_end` 给出运行时检索边界。
- `(READONLY)` 是 GCC11+ 引入的 ld 节区类型关键字（**ld** §3.6.8.1），若使用早期工具链需删除。GCC 11 起严格检查输出段属性与目标区域属性是否匹配，冲突即报警告，同时引入 (READONLY) 关键字让用户显式覆盖

## 六、初始化数组节区

静态构造/析构函数指针表

### 片段

```ld
.preinit_array (READONLY) :                      /* 节区类型：只读（GCC11+ 语法） */
{
  . = ALIGN(4);                                  /* 4 字节对齐 */
  PROVIDE_HIDDEN (__preinit_array_start = .);    /* 表起始，不导出 */
  KEEP (*(.preinit_array*))                      /* 预初始化函数表，防回收 */
  PROVIDE_HIDDEN (__preinit_array_end = .);      /* 表结束，不导出 */
  . = ALIGN(4);                                  /* 4 字节对齐 */
} >FLASH                                         /* 运行地址/加载地址均在 FLASH */

.init_array (READONLY) :                         /* 节区类型：只读（GCC11+ 语法） */
{
  . = ALIGN(4);                                  /* 4 字节对齐 */
  PROVIDE_HIDDEN (__init_array_start = .);       /* 表起始，不导出 */
  KEEP (*(SORT(.init_array.*)))                  /* 按优先级排序后保留 */
  KEEP (*(.init_array*))                         /* 无优先级后缀的项 */
  PROVIDE_HIDDEN (__init_array_end = .);         /* 表结束，不导出 */
  . = ALIGN(4);                                  /* 4 字节对齐 */
} >FLASH                                         /* 运行地址/加载地址均在 FLASH */

.fini_array (READONLY) :                         /* 节区类型：只读（GCC11+ 语法） */
{
  . = ALIGN(4);                                  /* 4 字节对齐 */
  PROVIDE_HIDDEN (__fini_array_start = .);       /* 表起始，不导出 */
  KEEP (*(SORT(.fini_array.*)))                  /* 按优先级排序后保留 */
  KEEP (*(.fini_array*))                         /* 无优先级后缀的项 */
  PROVIDE_HIDDEN (__fini_array_end = .);         /* 表结束，不导出 */
  . = ALIGN(4);                                  /* 4 字节对齐 */
} >FLASH                                         /* 运行地址/加载地址均在 FLASH */
```

### 语法

| 语法                      | 含义                                                                           | 来源                                             |
| ------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------ |
| `PROVIDE_HIDDEN(sym = .)` | 仅在被引用且未定义时提供该符号，且在 ELF 符号表中标记为 `STV_HIDDEN`（不导出） | **ld** §3.5.4 PROVIDE HIDDEN；**ELF** STV_HIDDEN |

### 依据

- `PROVIDE_HIDDEN` 在两端定义边界符号，并按 libc 约定把它们标记为隐藏符号，供启动文件中的 `bl __libc_init_array` 遍历 `__preinit_array_*` 与 `__init_array_*` 区间（**ELF** Book I「Special Sections」`.init_array`/`.preinit_array`）


## 七、数据段与加载地址

本段用 `LOADADDR` 取得 `.data` 在 Flash 的加载地址，再声明 `.data` 输出节，启动文件据此执行 Flash→SRAM 的初值复制。

### 片段

```ld
_sidata = LOADADDR(.data);   /* .data 的加载地址（LMA，源起始） */

.data :
{
  . = ALIGN(4);              /* 4 字节对齐 */
  _sdata = .;                /* .data 起始（运行地址） */
  *(.data)                   /* 已初始化数据节 */
  *(.data*)                  /* 已初始化数据派生节 */
  *(.RamFunc)                /* RAM 中执行的函数节 */
  *(.RamFunc*)               /* RAM 中执行的函数派生节 */
  . = ALIGN(4);              /* 4 字节对齐 */
  _edata = .;                /* .data 结束（运行地址） */
} >DTCMRAM AT> FLASH         /* 运行地址在 DTCMRAM，加载地址在 FLASH */
```

### 语法

| 语法                 | 含义                                 | 来源                                                |
| -------------------- | ------------------------------------ | --------------------------------------------------- |
| `LOADADDR(.data)`    | 返回 `.data` 的绝对加载地址（LMA）   | **ld** §3.10.9 Builtin Functions                    |
| `>DTCMRAM AT> FLASH` | 运行地址在 DTCMRAM，加载地址在 FLASH | **ld** §3.6.8.6 Region；§3.6.8.2 Output Section LMA |

### 依据

- `_sdata/_edata` 即启动文件 `ldr r0,=_sdata` / `ldr r1,=_edata` 的边界
- `_sidata = LOADADDR(.data)` 对应 `ldr r2,=_sidata` 源起始 LMA（**ld** §3.10.9）
- `*(.RamFunc*)` 是 HAL/CubeMX 中约定的收集需在 RAM 中执行的对内部 Flash 操作的关键函数（如 Flash 擦写例程）。内部 Flash 是 基于电荷擦写的物理阵列，擦写时需要在字线上施加高压（~10V），同一 Bank 的字线被占据，读出放大器也处于擦写偏置状态，物理上无法同时给出有效读数据
- `*(.RamFunc*)` 被并入 .data，启动文件复制 .data 时自动完成 Flash→DTCMRAM 搬移，无需额外处理

## 八、零初始化数据节区

本段声明 `.bss` 零初始化段。

### 片段

```ld
. = ALIGN(4);                /* 4 字节对齐 */
.bss :
{
  _sbss = .;                 /* .bss 起始 */
  __bss_start__ = _sbss;     /* 兼容 C 库（newlib）的别名 */
  *(.bss)                    /* 零初始化数据节 */
  *(.bss*)                   /* 零初始化数据派生节 */
  *(COMMON)                  /* 未初始化全局变量（COMMON 块） */
  . = ALIGN(4);              /* 4 字节对齐 */
  _ebss = .;                 /* .bss 结束 */
  __bss_end__ = _ebss;       /* 兼容 C 库（newlib）的别名 */
} >DTCMRAM                   /* 运行地址/加载地址均在 DTCMRAM，不占 Flash */
```

### 语法

| 语法                   | 含义                                               | 来源                               |
| ---------------------- | -------------------------------------------------- | ---------------------------------- |
| `>DTCMRAM`（无 `AT>`） | 仅运行地址、无独立加载地址——`.bss` 不占 Flash 空间 | **ld** §3.6.8.2 Output Section LMA |

### 依据

- `.bss` 节区类型为 `SHT_NOBITS`（**ELF** Book I「Section Header」），故映像不存初值
- `_sbss/_ebss` 即启动文件 `ldr r2,=_sbss` / `ldr r4,=_ebss` 的边界（**ld** §3.10.5）。

## 九、堆栈空间校验

本段在 `.bss` 之后预留堆与栈的最小尺寸，使链接器在 RAM 不足以容纳它们时直接报错

### 片段

```ld
._user_heap_stack :
{
  . = ALIGN(8);              /* 8 字节对齐 */
  PROVIDE ( end = . );       /* 堆起始符号（兼容） */
  PROVIDE ( _end = . );      /* 堆起始符号 */
  . = . + _Min_Heap_Size;    /* 预留堆空间 */
  . = . + _Min_Stack_Size;   /* 预留栈空间 */
  . = ALIGN(8);              /* 8 字节对齐 */
} >DTCMRAM                   /* 运行地址/加载地址均在 DTCMRAM */
```

### 语法

| 语法                                     | 含义                                                           | 来源                                    |
| ---------------------------------------- | -------------------------------------------------------------- | --------------------------------------- |
| `PROVIDE(end = .)` / `PROVIDE(_end = .)` | 仅在 C 库引用且未定义时提供堆起始符号提供缺省值                | **ld** §3.5.3 PROVIDE                   |
| `. = . + _Min_Heap_Size`                 | 用位置计数器在输出映像中为堆/栈占位                            | **ld** §3.10.5 Location Counter         |
| `. = ALIGN(8)`                           | 8 字节对齐（满足 Cortex-M7 的 8 字节栈对齐与 double 访问要求） | **ld** §3.6.8.3 Forced Output Alignment |

### 依据

- 若 DTCMRAM 装不下 `.bss` + `_Min_Heap_Size` + `_Min_Stack_Size`，`>DTCMRAM` region 溢出即报错（**ld** §3.7 MEMORY Command 区域检查）。
- Cortex-M7 的 EABI 要求 8 字节栈对齐（**PM0253** §2.1 Stack / AAPCS；**ARMv7-M ARM** SP alignment），故此处 `ALIGN(8)`。
- end/_end 标记的是 bss 段结束之后、heap 预留区之前的地址，也就是"堆真正的起始地址"，为 C 运行时库（newlib）的动态内存管理 _sbrk 提供堆开始的基准地址
  
## 十、D2 域 SRAM 节区

本段预留一个自定义输入节 `.RAM_D2`，用于显式存放 SRAM 大块缓冲（如 DMA 缓冲、LCD 帧缓冲）。

### 片段

```ld
.RAM_D2 :
{
  . = ALIGN(4);              /* 4 字节对齐 */
  *(.RAM_D2)                 /* 用户自定义节（DMA 缓冲等） */
  *(.RAM_D2*)                /* 派生节 */
  . = ALIGN(4);              /* 4 字节对齐 */
} >RAM_D2                    /* 运行地址/加载地址均在 D2 域 SRAM */
```

### 依据

- D2 域 SRAM（SRAM1/SRAM2/SRAM3）可被多个 AHB 主设备（DMA1/DMA2、USB、ETH 等）直接访问，适合做 DMA 缓冲区（**RM0433** §2.3 Embedded SRAM / §7 DMA）。
- 用户在 C 源码中用 `__attribute__((section(".RAM_D2")))` 即可把变量定位到此区，无需改动链接脚本（**as** 段属性；**ld** §3.6.4.2）。

## 十一、节区丢弃

本段用特殊输出节名 `/DISCARD/` 把标准库（`libc.a`/`libm.a`/`libgcc.a`）中所有未使用符号显式丢弃，避免被链接器拖入映像、占用空间。

### 片段

```ld
/DISCARD/ :
{
  libc.a ( * )               /* 丢弃 libc.a 所有成员节区 */
  libm.a ( * )               /* 丢弃 libm.a 所有成员节区 */
  libgcc.a ( * )             /* 丢弃 libgcc.a 所有成员节区 */
}
```

### 语法

| 语法                        | 含义                                                 | 来源                                    |
| --------------------------- | ---------------------------------------------------- | --------------------------------------- |
| `/DISCARD/ { archive (*) }` | 特殊输出节名：分配到此处的输入节区一律不写入输出文件 | **ld** §3.6.7 Output Section Discarding |

### 依据

- `/DISCARD/` 是 ld 保留的输出节名，输入节名不被前面通配匹配即未被吸收的残余节即被丢弃（**ld** §3.6.7）。
- 用通配 `( * )` 把三大标准库的所有成员节区显式排除，配合 `--gc-sections` 进一步缩减体积。
- `libc.a ( * )` 强制丢弃全部标准库成员，会与 newlib 的 _sbrk/malloc/printf 等冲突。此写法通常配合 -nostdlib/-nodefaultlibs 用于纯裸机裁剪；若使用 newlib 的标准 IO/动态内存，不应整体丢弃 libc.a

## 十二、附录

- **as** = `Using as`（GNU Binutils v2.46）
- **ld** = `GNU linker ld`（GNU Binutils v2.46）
- **ELF** = `ELF - v4.2 格式规范手册`（Xinuos ELF v4.2）
- **PM0253** = `STM32F7/H7 Cortex-M7 编程手册`（PM0253 Rev 7）
- **ARMv7-M ARM** = `Arm®v7-M 架构参考手册`（DDI 0403E.e）
- **RM0433** = `STM32H750 参考手册`（RM0433）
- **DS12556** = `STM32H750 数据手册`（DS12556 Rev 8）
