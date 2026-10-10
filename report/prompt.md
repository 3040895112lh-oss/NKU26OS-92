以下为本实验中使用到的提示词：

练习1：理解 entry.S 的入口操作

[PROMPT]
任务：阅读 kern/init/entry.S，解释入口点 kern_entry 中的 la sp, bootstacktop 与 tail kern_init 两条指令各自完成了什么操作、目的是什么。
操作要求：以工程中的真实定义为准，不要泛泛而谈。
输出要求：结合 RISC-V 调用约定与 ucore 的内存布局给出结论并说明依据，不要只复述指令的字面含义。

[RELY]
- tools/kernel.ld: BASE_ADDRESS = 0x80200000，ENTRY(kern_entry)
- kern/mm/mmu.h: PGSIZE = 4096, PGSHIFT = 12
- kern/mm/memlayout.h: KSTACKPAGE = 2, KSTACKSIZE = KSTACKPAGE * PGSIZE
- entry.S 的 .data 段中 bootstack 为 .space KSTACKSIZE，其后紧跟 bootstacktop
- kern_init 在 kern/init/init.c 中声明为 __attribute__((noreturn))

[GUARANTEE]
1. la sp, bootstacktop 完成的操作与目的
2. tail kern_init 完成的操作与目的

[SPECIFICATION]
## la sp, bootstacktop
Pre-Condition：CPU 刚进入 kern_entry，sp 仍为 OpenSBI 遗留的值。
Post-Condition：sp 指向内核栈顶 bootstacktop，即 sp = &bootstack + KSTACKSIZE。
Case 1：la 是伪指令，作用是把符号的地址装入寄存器。
Case 2：RISC-V 栈向下增长，故空栈的栈顶取栈区域的最高地址。
Case 3：若不设置 sp（沿用固件栈），会破坏固件内存。

## tail kern_init
Pre-Condition：内核栈已就绪。
Post-Condition：控制权交给 kern_init，且不在栈上留下返回地址。
Case 1：tail = 跳转且丢弃返回地址（jalr x0 / j，写在零寄存器上）。
Case 2：与 call（jal ra）的区别，并与 kern_init 的 noreturn 相对应。


练习2：用 GDB 跟踪启动流程

[PROMPT]
任务：使用 QEMU + GDB 跟踪 RISC-V 从加电到执行内核第一条指令（跳转到 0x80200000）的整个过程，并回答：
(1) 加电后最初执行的几条指令位于什么地址？
(2) 它们主要完成了哪些功能？
(3) 控制权是如何最终交给内核的？
操作要求：给出可直接复现的操作步骤（命令 + GDB 命令序列），并记录关键观察结果（PC / SP / 反汇编）。
输出要求：以真实调试输出为依据，不要臆测。

[RELY]
- 复位地址：0x1000（QEMU 模拟的 RISC-V 处理器复位向量）
- 固件：QEMU 内置 OpenSBI，加载基址 0x80000000（M 态）
- 内核：加载基址 0x80200000（S 态），入口符号 kern_entry
- 内核符号表：obj/kernel.sym（含 kern_entry / kern_init / bootstack / bootstacktop）

[GUARANTEE]
产出一份可复现的调试流程，至少包含：
- 启动 QEMU 使其暂停并开放 GDB 端口（-S -s）
- GDB 连接后查看 PC 与复位地址处的指令；在 0x80200000 下断点；命中后查看寄存器与反汇编；单步执行入口两条指令

[SPECIFICATION]
## 步骤 A：连接并观察复位位置
Pre-Condition：QEMU 以 -S -s 启动，内核镜像已加载。
Post-Condition：GDB 报告 PC = 0x1000。
Case 1：反汇编 0x1000 处若干条指令，说明它们如何准备启动寄存器。

## 步骤 B：中断到内核入口
Pre-Condition：已连接目标。
Post-Condition：在 0x80200000 停下，pc = 0x80200000 <kern_entry>。
Case 1：记录 ra、sp 的值，说明它们此刻仍属于 OpenSBI。

## 步骤 C：单步入口两条指令
Pre-Condition：停在 kern_entry 第一条指令。
Post-Condition：执行 la sp 后 sp = bootstacktop；执行 tail 后 pc 指向 kern_init。
Case 1：用 objdump / obj/kernel.sym 交叉验证 sp 的新值确实等于 bootstacktop。

Requirements：每一步都给出实际看到的数值，而不是"应该是多少"。
