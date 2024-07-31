---
title: "xv6 教材笔记 - 2. Operating system organization"
hideSummary: true
date: 2024-07-31T00:33:23+08:00
draft: true
tags: ["xv6", "MIT 6.S081"]
series: ["xv6", "MIT 6.S081"]
author: ["xubinh"]
type: posts
math: false
---

## 2.6 Code: starting xv6, the first process and system call

运行 xv6 系统的 RISC-V 计算机启动的全过程:

1. 电脑开机并初始化自身, 然后运行存储在某个只读内存中的一个称为启动加载器 (boot loader) 的程序.
1. 启动加载器将 xv6 内核加载至内存中的物理地址 `0x80000000` (7 个零) 处, 然后 CPU 在机器模式 (machine mode) 下从 `_entry` 入口 (位于 `kernel/entry.S:7`) 开始运行 xv6.
   - RISC-V机器一开始是没有启用页表功能的.
   - 之所以不将 xv6 加载至物理地址 `0x0` 处是因为从 `0x0` 开始到 `0x80000000` 为止的这段物理内存存储着与 I/O 设备有关的信息.
1. `_entry` 的作用是为 xv6 初始化一个用于执行 C 程序的栈. 为了运行 C 程序, xv6 需要使用一个栈, 为此 xv6 在文件 `start.c` 中已经定义了一个名为 `stack0` 的栈 (位于 `kernel/start.c:11`). `_entry` 入口处的指令负责将堆栈指针寄存器 `sp` 设置为 `stack0+4096` 为 xv6 初始化该栈. 在初始化好栈之后, `_entry` 将调用 `start` 处的 C 代码 (位于 `kernel/start.c:21`).
   - 在 RISC-V 下栈顶是向低地址方向增长的.
1. `start` 处的 C 代码的主要任务是将 CPU 从特权等级最高的机器模式切换至管理员模式 (supervisor mode). RISC-V 提供了一个用于从机器模式**返回**管理员模式的指令, 即 `mret`, 但在当前情况下机器根本就没有从管理员模式进入到过机器模式, 因为机器本来就是以机器模式启动的. `start` 处的代码负责的就是为 `mret` 指令建立一个与 "机器刚刚从管理员模式进入机器模式并即将返回" 等价的环境, 其所执行的操作包括将寄存器 `mstatus` (用于指示上一个特权模式) 设置为管理员模式, 将 寄存器 `mepc` 设置为 `main` 函数 (位于 `kernel/main.c:11`) 的入口地址, 将寄存器 `satp` 的值设置为 0 以禁用管理员模式下的虚拟地址转换机制, 以及将所有中断和异常全部委托给管理员模式. 做好这些准备工作之后, `start` 还将时钟芯片 (clock chip) 设置好以便产生计时器中断 (timer interrupt), 然后执行 `mret` 使 CPU "返回" 至管理员模式, 同时程序计数器也相应变为 `main` 函数的入口地址 (此时计算机便从内核中跳出).
1. `main` 函数初始化若干个设备和子系统, 然后调用 `userinit` 函数 (位于 `kernel/proc.c:233`).
1. `userinit` 函数创建整个系统自开机以来的第一个进程. 这个进程执行一小段由 RISC-V 汇编编写的程序, 即 `initcode.S` (位于 `user/initcode.S:3`).
1. `initcode.S` 中的代码将所要传入 `exec` 的参数以及代表 `exec` 系统调用的整数代码 `SYS_EXEC` (位于 `kernel/syscall.h:8`) 分别放置在寄存器 `a0`, `a1` 以及 `a7` 中, 然后执行 `ecall` 指令.
1. `ecall` 指令将陷入内核并执行一些代码, 最终将调用 `syscall` 函数 (位于 `kernel/syscall.c:132`).
1. `syscall` 函数使用系统调用表 (system call table) `syscalls` (位于 `kernel/syscall.c:107`) 将 `SYS_EXEC` 映射至 `sys_exec` 函数的入口地址.
1. `sys_exec` 函数即为 `exec` 系统调用的实际实现, 它将调入一个新的程序 (这里即 `/init`) 替换当前进程的内存和寄存器. 当内核从 `exec` 返回后, 计算机将回到用户态并开始执行 `/init` 进程.
1. `init` 函数 (位于 `user/init.c:15`) 视情况创建一个控制台 (console) 设备文件, 并为其分配三个文件描述符, 即 0, 1 和 2. 然后 `init` 会在这个控制台上启动一个 shell. 至此整个系统便成功跑起来了.
