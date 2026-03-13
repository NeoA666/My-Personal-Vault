# 🚀 rCore 实验预备知识清单
> **目标**：在动手写操作系统内核源码之前，系统梳理所需的理论与工具知识。
> 本文档聚焦于 **rCore-Tutorial**（基于 RISC-V + Rust 的教学操作系统实验），按"从底向上"的顺序组织。

---

## 📋 总览

| 模块 | 重要程度 | 篇幅建议 | 核心目的 |
|------|---------|---------|---------|
| 数据结构 | ⭐⭐⭐ | 简读 | 理解内核中链表、位图、堆等结构 |
| 计算机组成原理 | ⭐⭐⭐⭐ | 简读 | 理解 CPU 指令流、内存层次、中断硬件 |
| 操作系统基础 | ⭐⭐⭐⭐ | 简读 | 理解虚拟化、进程、页表等抽象 |
| RISC-V 汇编 | ⭐⭐⭐⭐⭐ | **重点** | 读懂/写 `entry.S`、`trap.S` 等关键文件 |
| Makefile | ⭐⭐⭐⭐⭐ | **重点** | 理解项目构建流程与各阶段命令 |
| Rust 核心特性 | ⭐⭐⭐⭐⭐ | **重点** | rCore 完全用 Rust 编写，裸机特性是关键 |
| 辅助工具链 | ⭐⭐⭐⭐ | 速查 | QEMU、GDB、链接脚本、objdump |

---

## 一、🗂️ 数据结构（简读）

rCore 内核中大量使用以下几种数据结构，不需要从零实现，但要能**读懂其行为**。

### 1.1 链表（Linked List）

- **内核中的用途**：空闲内存块管理（物理页帧分配器）、进程队列、设备驱动缓冲区。
- **关键变体**：
    - **侵入式链表（Intrusive Linked List）**：Linux/rCore 内核中常见，节点嵌入到结构体内部，不需要额外分配堆内存。
    - **双向循环链表**：便于 O(1) 插入/删除，常用于调度队列。
- **Rust 实现要点**：由于 Rust 的所有权，直接用 `Box<Node>` 写链表很棘手，内核中通常用裸指针或 `unsafe` 代码。

```
struct ListNode {
    next: Option<Box<ListNode>>,
    val: usize,
}
```

> ⚠️ **预警**：在 rCore 的物理内存管理器中，你会看到侵入式链表的 `unsafe` 实现。理解"节点是结构体中的一个字段"这个思想即可。

---

### 1.2 位图（Bitmap）

- **内核中的用途**：追踪哪些物理页帧已被分配（1=已用，0=空闲）。
- **关键操作**：设置位（set）、清除位（clear）、查找第一个零位（find_first_zero）。
- **Rust 中**：通常用 `Vec<usize>` 或定长数组，每个 `usize` 存 64 位状态。

---

### 1.3 栈与队列

- **内核中的用途**：
    - **栈**：每个进程有用户栈和内核栈，函数调用帧（栈帧）压栈/弹栈。
    - **队列**：就绪队列（Ready Queue）存放可调度的进程，FIFO/优先队列实现不同调度策略。
- **注意**：内核栈是一块静态分配的内存区域，不是语言层面的 `Vec<T>`，大小固定（通常 8 KB ~ 64 KB）。

---

### 1.4 哈希表与 B 树

- **内核中的用途**：文件系统目录项缓存（dcache）、进程 PID 查找。
- **rCore 中**：初级实验阶段可用 `BTreeMap`（Rust 标准库提供）。

---

## 二、🖥️ 计算机组成原理（简读）

### 2.1 CPU 流水线与指令执行

- **必知概念**：取指（IF）→ 译码（ID）→ 执行（EX）→ 访存（MEM）→ 写回（WB）。
- **与 OS 的关联**：中断发生时，CPU 必须在某个"干净"的流水线边界停下来，保存所有寄存器状态。

### 2.2 内存层次结构

```
寄存器（< 1 ns）
   ↓
L1/L2/L3 Cache（1~50 ns）
   ↓
主存 DRAM（~100 ns）
   ↓
磁盘/SSD（μs ~ ms）
```

- **与 OS 的关联**：
    - **TLB（Translation Lookaside Buffer）**：页表缓存，`sfence.vma` 指令用于刷新 TLB，在切换页表时必须执行。
    - **Cache 一致性**：多核写内存后，其他核的 Cache 可能是旧数据，需要内存屏障（Memory Barrier）指令。

### 2.3 中断与异常机制（硬件层面）

> 这是 OS 与硬件的最核心接口，需重点理解。

| 类型 | 触发来源 | 是否可屏蔽 | 示例 |
|------|---------|-----------|------|
| 外部中断（Interrupt） | 外设（定时器、网卡） | 可屏蔽 | 时钟中断（驱动进程调度） |
| 同步异常（Exception） | CPU 自身 | 不可屏蔽 | 缺页异常、非法指令 |
| 陷阱（Trap） | 软件主动触发 | 不可屏蔽 | `ecall`（系统调用） |

- **RISC-V 中的对应**：三者统称为 **Trap**，由 `stvec` 寄存器指向处理入口。

### 2.4 RISC-V 特权级架构

```
M-Mode（Machine Mode）   ← 最高权限，固件/SBI 运行在这里
   ↓（委托机制 medeleg/mideleg）
S-Mode（Supervisor Mode） ← OS 内核运行在这里
   ↓（用户态请求 ecall）
U-Mode（User Mode）       ← 应用程序运行在这里
```

> 📌 rCore 的大部分内核代码运行在 **S-Mode**。理解三个特权级和它们之间的切换是读源码的基础。

---

## 三、🐧 操作系统基础知识（简读）

> 若已读过《操作系统导论》或王道，以下内容应已熟悉，可快速复习。

### 3.1 进程与线程

- **进程**：资源分配的基本单位，拥有独立的地址空间（页表）。
- **线程**：CPU 调度的基本单位，同进程内的线程共享地址空间。
- **rCore 中的 TaskControlBlock（TCB）**：等价于 PCB，存储进程的所有状态（寄存器上下文、页表、栈指针等）。

### 3.2 虚拟内存与分页

- **页（Page）**：虚拟内存的基本单位，RISC-V 上为 4 KB（Sv39 模式）。
- **页表（Page Table）**：从虚拟页号（VPN）到物理页号（PPN）的映射表，每个进程一张。
- **Sv39**：rCore 使用的 RISC-V 三级页表方案，支持 39 位虚拟地址空间（512 GB）。

```
虚拟地址 (39 bit):  VPN[2] | VPN[1] | VPN[0] | Page Offset
                     9 bit    9 bit    9 bit      12 bit
```

### 3.3 系统调用流程

```
用户态 app  →  ecall 指令  →  [硬件保存上下文, 跳转 stvec]
           →  内核 trap_handler  →  syscall 分发
           →  sret 指令  →  返回用户态
```

### 3.4 调度算法（了解即可）

- rCore 初始实现简单轮转（Round-Robin）。
- 后期实验会引入优先级、时间片机制。

---

## 四、🔧 Makefile（重点）

Makefile 是 rCore 构建系统的骨架，读懂 Makefile 才能理解"一条 `make run` 命令背后发生了什么"。

### 4.1 Makefile 基础语法

#### 规则结构（三要素）

```makefile
目标(target): 依赖(prerequisites)
[TAB]  命令(recipe)
```

> ⚠️ **关键坑**：命令行**必须以 Tab 缩进**，不能用空格！这是 Makefile 语法的硬性要求。

```makefile
# 示例：编译一个 C 文件
main.o: main.c
    gcc -c main.c -o main.o
```

#### 变量

```makefile
# 定义变量（两种赋值方式）
CC      := gcc          # 立即赋值（推荐，不会被递归展开）
CFLAGS   = -Wall -O2   # 延迟赋值

# 使用变量
build: main.c
    $(CC) $(CFLAGS) main.c -o main
```

#### 自动变量（最常用的三个）

| 变量 | 含义 | 示例 |
|------|------|------|
| `$@` | 当前目标名 | `main.o` |
| `$<` | 第一个依赖 | `main.c` |
| `$^` | 所有依赖（去重）| `main.c utils.c` |

```makefile
%.o: %.c
    $(CC) -c $< -o $@   # 将 $< (xxx.c) 编译为 $@ (xxx.o)
```

#### 模式规则

```makefile
%.o: %.c           # % 是通配符，匹配任意字符串
    $(CC) -c $< -o $@
```

#### 伪目标（.PHONY）

```makefile
.PHONY: clean run  # 声明这些目标不是真实文件

clean:
    rm -f *.o main

run: main
    ./main
```

> 💡 **为什么需要 .PHONY？** 如果目录下恰好有个叫 `clean` 的文件，make 会认为目标已是最新而不执行命令。用 `.PHONY` 强制告诉 make "这是一个动作，不是文件"。

---

### 4.2 rCore Makefile 核心片段解析

rCore 的 Makefile 涉及以下关键操作：

#### ① 交叉编译目标设置

```makefile
TARGET     := riscv64gc-unknown-none-elf  # 目标平台：64位 RISC-V，无 OS 环境
MODE       := release
KERNEL_ELF := target/$(TARGET)/$(MODE)/os
KERNEL_BIN := target/$(TARGET)/$(MODE)/os.bin
```

- **`riscv64gc-unknown-none-elf`** 含义：
    - `riscv64gc`：RISC-V 64位，支持通用扩展（G）和压缩指令（C）
    - `unknown-none-elf`：无操作系统（裸机），ELF 格式输出

#### ② 构建内核二进制

```makefile
kernel:
    @cargo build --release --target $(TARGET)
    @rust-objcopy --strip-all -O binary $(KERNEL_ELF) $(KERNEL_BIN)
    # ↑ 用 rust-objcopy 把 ELF 文件剥离符号表，生成纯二进制镜像
```

#### ③ 启动 QEMU

```makefile
QEMU_ARGS := -machine virt                  \
             -nographic                     \
             -bios $(BOOTLOADER)            \
             -device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)

run: kernel
    qemu-system-riscv64 $(QEMU_ARGS)
```

- **`-machine virt`**：使用 QEMU 提供的虚拟 RISC-V 开发板
- **`-bios`**：指定 SBI 固件（rCore 用 RustSBI），运行在 M-Mode
- **`-device loader`**：将内核镜像加载到指定物理地址

#### ④ GDB 调试

```makefile
gdbserver: kernel
    qemu-system-riscv64 $(QEMU_ARGS) -s -S   
    # -s: 开启 GDB server 监听 localhost:1234
    # -S: 启动时暂停，等待 GDB 连接

gdbclient:
    riscv64-unknown-elf-gdb -ex 'file $(KERNEL_ELF)' \
                            -ex 'set arch riscv:rv64' \
                            -ex 'target remote localhost:1234'
```

---

### 4.3 常用 Makefile 技巧速查

```makefile
# 条件判断
ifeq ($(MODE), debug)
    CFLAGS += -g
endif

# 打印信息（@ 抑制命令回显）
info:
    @echo "Building for $(TARGET)..."

# 包含其他 Makefile
include config.mk

# 获取文件列表
SRCS := $(wildcard src/*.c)
OBJS := $(patsubst src/%.c, build/%.o, $(SRCS))

# 强制重新构建（删除所有目标文件）
clean:
    rm -rf build/
```

---

## 五、⚙️ 汇编语言（重点：RISC-V 汇编）

rCore 中有两个关键汇编文件：**`entry.asm`**（内核入口）和 **`trap.S`**（陷阱处理）。读懂这两个文件，你就掌握了 rCore 最硬核的部分。

### 5.1 RISC-V 寄存器速查表

RISC-V 有 32 个通用整数寄存器（`x0`~`x31`），每个都有 ABI 别名：

| 寄存器 | ABI 名 | 约定用途 | 是否调用者保存 |
|--------|-------|---------|-------------|
| `x0` | `zero` | 恒为 0，写入无效 | — |
| `x1` | `ra` | 返回地址（Return Address） | 调用者保存 |
| `x2` | `sp` | 栈指针（Stack Pointer） | 被调用者保存 |
| `x3` | `gp` | 全局指针（Global Pointer） | — |
| `x4` | `tp` | 线程指针（Thread Pointer） | — |
| `x5`~`x7` | `t0`~`t2` | 临时寄存器 | 调用者保存 |
| `x8` | `s0`/`fp` | 帧指针/保存寄存器 | 被调用者保存 |
| `x9` | `s1` | 保存寄存器 | 被调用者保存 |
| `x10`~`x11` | `a0`~`a1` | 函数参数/返回值 | 调用者保存 |
| `x12`~`x17` | `a2`~`a7` | 函数参数 | 调用者保存 |
| `x18`~`x27` | `s2`~`s11` | 保存寄存器 | 被调用者保存 |
| `x28`~`x31` | `t3`~`t6` | 临时寄存器 | 调用者保存 |

> 💡 **调用者保存 vs 被调用者保存**：
> - **调用者保存（Caller-saved）**：调用函数前，调用者自己负责把这些寄存器压栈保存。
> - **被调用者保存（Callee-saved）**：函数内部如果要用这些寄存器，**必须**先保存原值，返回前恢复。

---

### 5.2 关键 CSR 寄存器（控制状态寄存器）

OS 内核通过读写 CSR 控制 CPU 行为，以下是 rCore 中最常见的：

| CSR 名 | 全称 | 用途 |
|--------|------|------|
| `stvec` | Supervisor Trap Vector | Trap 处理入口地址 |
| `sepc` | Supervisor Exception PC | 发生 Trap 时的 PC 值（用于返回） |
| `scause` | Supervisor Cause | Trap 原因（中断/异常类型） |
| `stval` | Supervisor Trap Value | 附加信息（如缺页地址） |
| `sstatus` | Supervisor Status | 全局中断使能、特权级状态位 |
| `satp` | Supervisor ATP | 页表根地址 + 分页模式（Sv39） |
| `sscratch` | Supervisor Scratch | 临时寄存器（陷阱处理时暂存用户栈指针） |

**读写 CSR 的指令**：

```asm
csrr  a0, sepc        # 读 sepc → a0
csrw  stvec, a0       # 写 a0 → stvec
csrrw a0, sscratch, a0 # 原子交换：a0 ↔ sscratch（上下文切换常用！）
```

---

### 5.3 RISC-V 常用指令

#### 基础整数运算

```asm
add   a0, a1, a2    # a0 = a1 + a2
addi  a0, a0, 4     # a0 = a0 + 4（立即数加法）
sub   a0, a1, a2    # a0 = a1 - a2
li    a0, 100       # a0 = 100（加载立即数，伪指令）
mv    a0, a1        # a0 = a1（移动，伪指令）
```

#### 内存访问（load/store）

```asm
# 加载（从内存读到寄存器）
ld   a0, 0(sp)      # a0 = *(sp + 0)，加载 8 字节（双字）
lw   a0, 0(sp)      # 加载 4 字节（字）
lb   a0, 0(sp)      # 加载 1 字节，符号扩展

# 存储（从寄存器写到内存）
sd   a0, 0(sp)      # *(sp + 0) = a0，存储 8 字节
sw   a0, 0(sp)      # 存储 4 字节
```

#### 跳转与分支

```asm
j     label          # 无条件跳转（伪指令，= jal zero, label）
jal   ra, func       # 跳转到 func，同时 ra = PC + 4（调用函数）
jalr  zero, ra, 0    # 跳转到 ra（函数返回，伪指令 ret）
beq   a0, a1, label  # if a0 == a1, jump to label
bne   a0, a1, label  # if a0 != a1, jump to label
blt   a0, a1, label  # if a0 < a1, jump to label（有符号）
```

#### 栈操作（没有专门的 push/pop，手动操作 sp）

```asm
# 压栈（分配 8 字节）
addi sp, sp, -8
sd   ra, 0(sp)

# 弹栈（释放 8 字节）
ld   ra, 0(sp)
addi sp, sp, 8
```

#### 特权级切换指令

```asm
ecall    # 从 U-Mode/S-Mode 向上层发出调用请求（系统调用用这条！）
sret     # 从 S-Mode 返回（恢复 sepc → PC，恢复特权级）
mret     # 从 M-Mode 返回
wfi      # Wait For Interrupt，让 CPU 进入低功耗等待
sfence.vma  # 刷新 TLB，切换页表后必须执行
```

---

### 5.4 理解 `entry.asm`（内核启动代码）

```asm
# os/src/entry.asm （简化版）
    .section .text.entry    # 声明代码段，段名 .text.entry
    .globl _start           # 声明全局符号，让链接器能找到它

_start:
    # 设置内核栈：sp 指向内核栈顶
    la   sp, boot_stack_top  # la = load address（加载地址到寄存器）
    call rust_main           # 跳转到 Rust 写的主函数

    # 内核栈空间定义
    .section .bss.stack      # BSS 段（未初始化数据）
    .globl boot_stack        # 栈底
boot_stack:
    .space 4096 * 16         # 分配 64KB 栈空间（4096 * 16 字节）
    .globl boot_stack_top    # 栈顶（RISC-V 栈向下增长，所以栈顶在高地址）
boot_stack_top:
```

> **关键理解**：
> 1. CPU 上电后从固定地址开始执行（SBI 初始化后跳转到内核入口）。
> 2. Rust 代码运行前，必须先用汇编设置好 `sp`（栈指针），否则函数调用无法工作。
> 3. `.section`、`.globl`、`.space` 是**汇编伪指令**，不产生机器码，指示汇编器如何组织内存。

---

### 5.5 理解 `trap.S`（陷阱上下文保存/恢复）

这是 rCore 中**最复杂的汇编代码**，目标是在发生 Trap 时保存所有寄存器，处理完后恢复：

```asm
# 简化的 __alltraps 和 __restore

    .section .text.trampoline
    .globl __alltraps
__alltraps:
    # 交换 sp 和 sscratch（sp 变成内核栈，sscratch 保存了用户栈地址）
    csrrw sp, sscratch, sp

    # 在内核栈上分配 TrapContext 结构体空间（34 个寄存器 × 8 字节）
    addi sp, sp, -34*8

    # 保存所有通用寄存器 x0~x31（x0 固定为 0 可以不保存，但保留位置）
    sd x1,  1*8(sp)
    sd x3,  3*8(sp)
    # ... 省略 x4~x30 ...
    sd x31, 31*8(sp)

    # 保存 CSR: sstatus 和 sepc（用于 sret 恢复）
    csrr t0, sstatus
    csrr t1, sepc
    sd   t0, 32*8(sp)
    sd   t1, 33*8(sp)

    # 将用户栈地址（原 sp，现在在 sscratch 里）也保存
    csrr t2, sscratch
    sd   t2, 2*8(sp)

    # 调用 Rust 写的 trap_handler(cx: &mut TrapContext)
    mv a0, sp        # a0 = TrapContext 指针（第一个参数）
    call trap_handler

__restore:
    # 恢复 CSR
    ld t0, 32*8(sp)
    ld t1, 33*8(sp)
    csrw sstatus, t0
    csrw sepc, t1

    # 恢复通用寄存器
    ld x1,  1*8(sp)
    # ... 省略 ...
    ld x31, 31*8(sp)

    # 恢复用户栈，并释放 TrapContext 空间
    ld sp, 2*8(sp)

    sret    # 返回用户态（PC = sepc，特权级降为 U-Mode）
```

> **核心思想**：陷阱处理 = **保存现场** → **处理事务** → **恢复现场** → **返回**。
> 汇编的作用就是自动保存那些 Rust 编译器不会为你保存的寄存器。

---

## 六、🦀 Rust 裸机编程核心特性（重点）

rCore 不是普通的 Rust 应用程序，它是一个**裸机（bare metal）程序**，运行在没有标准库的环境中。

### 6.1 `#![no_std]` 与 `#![no_main]`

```rust
// os/src/main.rs
#![no_std]   // 不使用 Rust 标准库（std），因为 std 依赖 OS，而我们就是 OS！
#![no_main]  // 不使用默认的 main 函数入口（入口是汇编的 _start）

use core::panic::PanicInfo;

// 必须手动定义 panic 处理函数
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}  // 简单实现：死循环（实际中会打印错误并关机）
}
```

> **`core` vs `std`**：
> - `std`：完整标准库，依赖操作系统（文件、网络、线程...）
> - `core`：标准库的无 OS 子集（整数、Option、Result、迭代器...），裸机可用

---

### 6.2 `unsafe` 代码

内核开发中无法完全避免 `unsafe`，必须理解何时使用：

```rust
// 读写任意内存地址（如映射 MMIO 寄存器）
unsafe {
    let uart_ptr = 0x1000_0000 as *mut u8;
    *uart_ptr = b'H';  // 直接写内存地址
}

// 内联汇编（最常用！）
use core::arch::asm;

unsafe {
    asm!(
        "ecall",                     // 执行 ecall 指令
        in("a7") syscall_id,         // a7 = 系统调用号
        in("a0") arg0,               // a0 = 第一个参数
        lateout("a0") ret,           // a0 = 返回值（执行后写入 ret）
    );
}
```

---

### 6.3 内联汇编语法（`core::arch::asm!`）

```rust
use core::arch::asm;

// 基本格式
asm!(
    "指令模板",          // 汇编代码字符串（用 {} 作占位符）
    in("寄存器") 变量,   // 输入：Rust 变量 → 寄存器
    out("寄存器") 变量,  // 输出：寄存器 → Rust 变量
    lateout("寄存器") 变量, // 延迟输出（优化提示）
    options(nostack),   // 选项：nostack=不触及栈
);

// 读取 CSR 示例
fn read_sstatus() -> usize {
    let sstatus: usize;
    unsafe {
        asm!("csrr {}, sstatus", out(reg) sstatus);
    }
    sstatus
}
```

---

### 6.4 所有权与生命周期在内核中的特殊性

```rust
// 静态全局变量（需要 unsafe 访问可变静态）
static mut KERNEL_HEAP: [u8; 0x30_0000] = [0; 0x30_0000];

// 使用 Mutex 保护全局状态（内核中常见模式）
use spin::Mutex;  // spin crate 提供无 OS 版的互斥锁

static SOME_MANAGER: Mutex<SomeManager> = Mutex::new(SomeManager::new());

fn use_manager() {
    let mut m = SOME_MANAGER.lock();  // 获取锁
    m.do_something();
}  // 锁自动释放
```

---

### 6.5 `repr(C)` 与内存布局

```rust
// 当你需要确保结构体内存布局与 C/汇编 兼容时
#[repr(C)]  // 使用 C 语言的内存对齐规则
pub struct TrapContext {
    pub x: [usize; 32],   // 32 个通用寄存器
    pub sstatus: usize,   // 偏移量 = 32 * 8 = 256
    pub sepc: usize,      // 偏移量 = 33 * 8 = 264
}
// 汇编中用 sd x1, 1*8(sp) 访问的就是这个结构体的字段
```

---

## 七、🛠️ 辅助工具链（速查）

### 7.1 QEMU：模拟器

| 命令/选项 | 含义 |
|---------|------|
| `qemu-system-riscv64` | 启动 64 位 RISC-V 系统模拟器 |
| `-machine virt` | 使用 QEMU 的 virt 通用开发板 |
| `-nographic` | 无图形界面，串口输出到终端 |
| `-kernel xxx.bin` | 加载内核镜像 |
| `-bios xxx.bin` | 加载 BIOS/SBI 固件 |
| `-s` | 启动 GDB Server（端口 1234）|
| `-S` | 启动时暂停，等待 GDB 连接 |
| `Ctrl+A, X` | 退出 QEMU（-nographic 模式）|

---

### 7.2 GDB：调试器

```bash
# 启动 RISC-V 版 GDB
riscv64-unknown-elf-gdb os/target/riscv64gc-unknown-none-elf/release/os

# 常用 GDB 命令
(gdb) target remote :1234   # 连接 QEMU GDB Server
(gdb) break rust_main        # 在 rust_main 处设断点
(gdb) continue               # 继续运行到断点
(gdb) stepi                  # 单步执行（机器指令级）
(gdb) info registers         # 查看所有寄存器
(gdb) x/10i $pc              # 查看 PC 处的 10 条反汇编指令
(gdb) x/4gx $sp              # 查看栈顶的 4 个 64 位值（十六进制）
(gdb) p/x $sstatus           # 打印 CSR（十六进制格式）
```

---

### 7.3 `objdump`：二进制分析

```bash
# 反汇编内核 ELF
rust-objdump -d os/target/riscv64gc-unknown-none-elf/release/os | less

# 查看段信息（各段的起始地址、大小）
rust-objdump -h os/target/riscv64gc-unknown-none-elf/release/os

# 查看符号表
rust-nm os/target/riscv64gc-unknown-none-elf/release/os | grep -i main
```

---

### 7.4 链接脚本（Linker Script）

链接脚本控制最终 ELF 文件的内存布局，这是 OS 开发中**极为关键**却经常被忽视的知识点。

```ld
/* os/src/linker.ld（简化版）*/

/* 指定目标架构 */
OUTPUT_ARCH(riscv)

/* 程序入口符号 */
ENTRY(_start)

/* 各段（Section）的加载地址与布局 */
SECTIONS {
    /* 内核从物理地址 0x80200000 开始（RustSBI 之后） */
    . = 0x80200000;

    /* 代码段 */
    stext = .;
    .text : {
        *(.text.entry)   /* 首先放置入口代码 */
        *(.text .text.*)
    }
    etext = .;

    /* 只读数据段 */
    srodata = .;
    .rodata : { *(.rodata .rodata.*) }
    erodata = .;

    /* 可读写数据段 */
    sdata = .;
    .data : { *(.data .data.*) }
    edata = .;

    /* BSS 段（清零的未初始化数据） */
    sbss = .;
    .bss : {
        *(.bss.stack)    /* 内核栈 */
        *(.bss .bss.*)
    }
    ebss = .;
}
```

> **关键知识点**：
> - `stext`、`etext` 等符号会被 Rust 代码引用（用 `extern "C"` 声明），用于清零 BSS 段等操作。
> - `*(.text.entry)` 确保入口代码 `_start` 位于整个内核的最开头，否则 CPU 跳过来执行的第一条指令就是乱码。
> - `.` 是**位置计数器（Location Counter）**，代表当前的地址。

---

### 7.5 Cargo 交叉编译配置

```toml
# os/.cargo/config.toml

[build]
target = "riscv64gc-unknown-none-elf"  # 默认编译目标

[target.riscv64gc-unknown-none-elf]
rustflags = [
    "-C", "link-arg=-Tsrc/linker.ld",  # 指定链接脚本
]
```

```bash
# 安装 RISC-V 工具链
rustup target add riscv64gc-unknown-none-elf
cargo install cargo-binutils          # 提供 rust-objcopy, rust-objdump 等
rustup component add llvm-tools-preview

# 安装 QEMU（Ubuntu）
sudo apt install qemu-system-misc
```

---

## 八、📚 推荐学习路径

```
阶段 1：环境准备（1~2 天）
  └─ 安装 Rust 工具链 + QEMU + RISC-V GDB
  └─ 阅读 rCore-Tutorial-Book 第 0 章：实验环境配置

阶段 2：基础扫盲（3~5 天）
  ├─ RISC-V 汇编：《RISC-V 手册》第 1~3 章（免费 PDF）
  ├─ Makefile：《跟我一起写 Makefile》陈皓版（在线免费）
  └─ 链接脚本：《程序员的自我修养》第 3~4 章（或在线博客）

阶段 3：rCore 实战（4~6 周）
  ├─ ch1：独立可执行程序（裸机 Hello World）
  ├─ ch2：批处理系统（特权级切换、ecall）
  ├─ ch3：多道程序与分时多任务（上下文切换）
  ├─ ch4：地址空间（页表、Sv39）
  ├─ ch5：进程（fork/exec/waitpid）
  ├─ ch6：文件系统
  └─ ch7：进程间通信与 I/O 重定向

阶段 4：深入与扩展
  └─ 参考 xv6-riscv 源码（MIT，C 语言实现，与 rCore 思路一致）
  └─ 阅读《操作系统：原理与实现》（陈海波）
```

---

## 九、⚡ 快速参考：阅读 rCore 源码时的"解码器"

当你在 rCore 源码中看到以下内容时，对照此表查询：

| 看到的代码 | 含义 |
|-----------|------|
| `#[no_std]` | 裸机程序，不用标准库 |
| `global_asm!(include_str!("entry.asm"))` | 将汇编文件内联到 Rust 编译单元 |
| `csrr!` / `csrw!` 宏 | 读/写 CSR 控制寄存器 |
| `sstatus::read().spp()` | 读取 sstatus 的 SPP 字段（上次特权级）|
| `satp::write(...)` | 切换页表（同时需要 `sfence.vma`） |
| `PageTable::from_token(token)` | 从 satp 值恢复页表对象 |
| `PhysPageNum` / `VirtPageNum` | 物理/虚拟页号类型 |
| `MapPermission::R \| W \| X` | 页表项权限位（读/写/执行）|
| `TaskContext` | 进程上下文（内核寄存器快照）|
| `TrapContext` | 陷阱上下文（用户寄存器快照）|
| `__switch(curr, next)` | 汇编实现的进程切换函数 |
| `SBI_CALL!(...)` | 调用 SBI（即 `ecall` 进 M-Mode）|

---

> 🎯 **最后的建议**：
> 1. **不要期望第一遍全懂**。先跑通实验，再回头理解原理，再看源码。循环 3 遍。
> 2. **汇编是门槛**。如果 `trap.S` 看不懂，先把 5.4 节的注释逐行过一遍。
> 3. **Makefile 是地图**。每次 `make` 报错，先读 Makefile 理解它在做什么，再看错误信息。
> 4. **GDB 是最好的老师**。遇到难以理解的行为，用 GDB 单步跑一遍，远胜过看一小时文档。
