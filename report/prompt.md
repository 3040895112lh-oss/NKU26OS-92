以下为本实验中使用到的提示词：

提示词1：理解环境三件套与启动链路

[PROMPT]
任务：解释在做 riscv64 操作系统实验时，为什么需要"交叉编译器 + 模拟器 + 固件"这三样东西，以及它们各自在"从加电到内核运行"这条链路中的位置。
输出要求：用"我们的电脑是 x86、目标系统是 riscv64"这一前提来组织说明，不要泛泛地罗列概念。

[RELY]
- 宿主机：x86_64（我们的日常电脑）
- 目标机：riscv64（QEMU 模拟的 virt 机器）
- 固件：OpenSBI，运行在 M 态，加载基址 0x80000000
- 内核：加载基址 0x80200000，运行在 S 态

[GUARANTEE]
1. 交叉编译器的作用与必要性
2. 模拟器的作用与必要性
3. 固件（OpenSBI）在启动链路中的作用

[SPECIFICATION]
## 交叉编译器
Case 1：说明"宿主机架构 ≠ 目标架构"时，为什么必须用目标为 riscv64 的编译器，以及为什么直接用预编译（prebuilt）工具链即可。

## 模拟器
Case 1：从 CPU / 内存 / 外设三个角度说明模拟器如何"假装出一台 riscv64 计算机"。

## 固件（OpenSBI）
Case 1：说明固件为什么必须运行在最高特权级（M 态）。
Case 2：说明"固件 → 引导程序 → 操作系统"三级跳的核心思想，并指出 QEMU 的 virt 机器里谁扮演了哪一级。


提示词2：排查交叉编译报错

[PROMPT]
任务：我在 Windows 上用 xPack 的 riscv-none-elf-gcc 编译 ucore lab1，libs/printfmt.c 第 251 行报错：
  error: cast from pointer to integer of different size ([-Werror=pointer-to-int-cast])
  num = (unsigned long long)va_arg(ap, void *);
请分析根因，并给出不改动实验源码的解决办法。
输出要求：给出可直接执行的 make 命令。

[RELY]
- 报错行：num = (unsigned long long)va_arg(ap, void *);
- 目标平台应为 riscv64（64 位指针）
- Makefile 中的 CFLAGS：-mcmodel=medany -std=gnu99 -Wno-unused -Werror ...
- Makefile 通过 CFLAGS += ... $(DEFS) 引入 DEFS 变量
- 工具链前缀由 Makefile 变量 GCCPREFIX 控制

[GUARANTEE]
1. 给出根因分析
2. 给出不改动 .c/.h 源码的 make 命令

[SPECIFICATION]
## 根因分析
Case 1：若目标架构是 rv32，则指针为 32 位、unsigned long long 为 64 位，强转触发该警告；说明该工具链默认目标可能是 rv32（ilp32）。

## 解决方案
Case 1：不改源码，在编译选项里显式指定 64 位目标，并覆盖工具链前缀。
命令形如：make GCCPREFIX=riscv-none-elf- DEFS="-march=rv64gc -mabi=lp64d"


提示词3：排查内核不启动

[PROMPT]
任务：我用指导书 Makefile 里的命令启动 QEMU：
  qemu-system-riscv64 -machine virt -nographic -bios default -device loader,file=bin/ucore.img,addr=0x80200000
OpenSBI 正常启动，但内核的串口输出没有出现。请帮我定位原因，并给出可用的命令。
输出要求：解释为什么该方式下内核没被执行，并给出替代命令。

[RELY]
- QEMU 版本：11.1.0；OpenSBI 版本：1.8.1
- 现象：OpenSBI 输出中存在 "Domain0 Next Address : 0x0000000000000000"
- 内核镜像：bin/ucore.img（objcopy -O binary 生成）；ELF：bin/kernel
- 期望：OpenSBI 跳转到 0x80200000 执行内核

[GUARANTEE]
1. 定位"内核没被执行"的原因
2. 给出能正常启动内核的命令

[SPECIFICATION]
## 原因
Case 1：-device loader 只是把文件写入客户机内存的指定地址，并不改变固件的"下一级跳转地址"；因此较新的 OpenSBI 中 Next Address 为 0，固件不会跳到内核。

## 解决
Case 1：改用 -kernel bin/kernel，由 QEMU 负责加载内核并把固件跳转地址设置为内核入口 0x80200000。
命令形如：qemu-system-riscv64 -machine virt -nographic -bios default -kernel bin/kernel
