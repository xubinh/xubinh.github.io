---
title: "MIT 6.S081 操作系统 xv6 教材笔记 - 第 2 章"
description: "系统梳理教材知识点, 结合代码深入理解 xv6 核心工作原理"
summary: "系统梳理教材知识点, 结合代码深入理解 xv6 核心工作原理"
date: 2025-02-17T16:07:23+08:00
draft: false
tags: ["MIT 6.S081", "Operating System Engineering", "xv6"]
series: ["MIT 6.S081", "Operating System Engineering", "xv6"]
author: ["xubinh"]
type: posts
math: true
---

## 2.6 Code: starting xv6, the first process and system call

运行 xv6 系统的 RISC-V 计算机启动的全过程:

1. 电脑开机并初始化自身, 然后运行存储在某个只读内存中的一个称为启动加载器 (boot loader) 的程序.
1. 启动加载器将 xv6 内核加载至内存中的物理地址 `0x80000000` (7 个零) 处, 然后 CPU 在机器模式 (machine mode) 下从 `_entry` 入口 (位于文件 `kernel/entry.S` 中) 开始运行 xv6.
   - RISC-V 机器一开始是没有启用页表功能的.
   - 之所以不将 xv6 加载至物理地址 `0x0` 处是因为从 `0x0` 开始到 `0x80000000` 为止的这段物理内存中存储着与 I/O 设备有关的信息.
1. `_entry` 的作用是为 xv6 初始化一个用于执行 C 程序的栈. 为了运行 C 程序, xv6 需要使用一个栈, 为此 xv6 在文件 `start.c` 中已经定义了一个名为 `stack0` 的栈 (位于文件 `kernel/start.c` 中). `_entry` 入口处的指令负责将堆栈指针寄存器 `sp` 设置为 `stack0 + (hartid * 4096)` 来为 xv6 初始化该栈. 在初始化好栈之后, `_entry` 将调用 `start` 处的 C 代码 (位于文件 `kernel/start.c` 中).
   - 注: 在 RISC-V 下栈顶是向低地址方向增长的.
1. `start` 处的 C 代码的主要任务是将 CPU 从特权等级最高的机器模式切换至管理员模式 (supervisor mode). RISC-V 提供了一个用于从机器模式**返回**管理员模式 (也有可能是其他模式, 具体由 MPP (Machine Previous Privilege) 寄存器字段指定) 的指令, 即 `mret`, 但在当前情况下机器根本就没有从管理员模式进入到过机器模式, 因为机器本来就是以机器模式启动的. `start` 处的代码负责的就是为 `mret` 指令建立一个与 "机器刚刚从管理员模式进入机器模式并即将返回" 等价的环境, 其所执行的操作包括将寄存器 `mstatus` (用于指示上一个特权模式) 设置为管理员模式, 将 寄存器 `mepc` 设置为 `main` 函数 (位于文件 `kernel/main.c` 中) 的入口地址, 将寄存器 `satp` 的值设置为 `0` 以暂时禁用虚拟地址转换机制, 最后将所有中断和异常全部委托给管理员模式. 做好这些准备工作之后, `start` 还将时钟芯片 (clock chip) 设置好以便产生计时器中断 (timer interrupt), 然后执行 `mret` 使 CPU "返回" 至管理员模式, 同时程序计数器也相应变为 `main` 函数的入口地址.
1. `main` 函数进行诸多初始化过程 (内容较多, 此处省略, 具体请查看代码), 然后调用 `userinit` 函数 (位于文件 `kernel/proc.c` 中).
1. `userinit` 函数将创建整个系统自开机以来的第一个进程 (通过调用 `allocproc` 函数), 这个进程将执行 `initcode` 数组中所包含的机器指令, 这些机器指令实际上就是文件 `user/initcode.S` 中的汇编代码所对应的机器指令.
   - 可以将 `userinit` 函数看作是 "没有父进程" 的 `fork` 函数, 因为 `userinit` 的作用正相当于创建子进程, 而该子进程负责调用 `exec` 系统调用, 加载 `/init` 程序并执行. 所不同的是在 `userinit` 中子进程的代码是直接复制于 `initcode` 数组中的, 毕竟此时根本就没有父进程.
1. `initcode.S` 的工作是调用 `exec("/init")`, 具体操作为将字符串 `"/init"` 以及代表 `exec` 系统调用的整数代码 `SYS_EXEC` (位于文件 `kernel/syscall.h` 中) 放置在寄存器中然后执行 `ecall` 指令.
1. `ecall` 指令将触发一次陷入, 经过蹦床 (trampoline) 代码的路由后最终将调用 `syscall` 函数 (位于文件 `kernel/syscall.c` 中).
1. `syscall` 函数使用系统调用表 (system call table) `syscalls` (位于文件 `kernel/syscall.c` 中) 将 `SYS_EXEC` 映射至 `sys_exec` 函数的入口地址.
1. `sys_exec` 函数即为 `exec` 系统调用的实际实现, 它将加载指定的目标程序 (此处即 `init`) 替换当前进程的内存和寄存器. 当内核从 `exec` 返回后, 计算机将回到用户态并开始执行 `init` 程序.
1. `init` 程序 (位于文件 `user/init.c` 中) 将分配三个文件描述符, 即 stdin, stdout, 以及 stderr, 然后启动 shell. 至此整个系统便成功跑起来了.
