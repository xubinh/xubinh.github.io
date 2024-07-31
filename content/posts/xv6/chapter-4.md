---
title: "xv6 教材笔记 - 4. Traps and system calls"
hideSummary: true
date: 2024-07-31T00:33:46+08:00
draft: true
tags: ["xv6", "MIT 6.S081"]
series: ["xv6", "MIT 6.S081"]
author: ["xubinh"]
type: posts
math: false
---

## 4.3 Code: Calling system calls

第二章讲到 `initcode.S` 调用了 `exec` 系统调用, 下面展开介绍一下从用户代码开始到系统调用执行的过程.

1. `initcode.S` 首先将 `exec` 所需要的两个参数放在寄存器 `a0` 和 `a1` 中, 然后将代表 `exec` 的系统调用代码 `SYS_exec` 放在寄存器 `a7` 中, 然后调用 `ecall` 指令.
1. `ecall` 函数将陷入内核并依次执行 `uservec`, `usertrap` 以及 `syscall`, 正如上面所看到的.
1. `syscall` 使用 `a7` 中的系统调用代码来索引系统调用表 `syscalls` 得到系统调用的实际实现代码的入口地址 (本例中即 `sys_exec`) 并执行. 当 `sys_exec` 返回时, `syscall` 再将其返回值放置在寄存器 `a0` 中, 这也就是系统调用 `exec` 的最终返回值.

## 4.4 Code: System call arguments

本小节主要介绍系统调用的参数传递过程和内核如何安全获取由用户传入的指针类型参数所指向的数据.

- 用户代码执行系统调用之前会先将参数存储在 C 语言调用规则约定的地方, 即寄存器中. 内核的陷入代码需要将寄存器中的参数转移至当前进程的陷入栈帧 (trap frame) 中以便后续调用的内核代码能够获取到.
  - 内核函数 `argraw` (位于 `kernel/syscall.c:34`) 用于从用户寄存器中获取参数.
  - 内核函数 `argint`, `argaddr` 和 `argfd` 用于从陷入栈帧中分别获取第 n 个系统调用的整型, 指针类型和文件描述符类型参数.
- 一些系统调用要求用户以指针的形式传入所需数据, 内核必须负责从这些用户指针指向的内存中读取或写入数据. 为了能够做到这一点, 内核有两个问题需要解决:
  1. 确保用户指针是合法且安全的 (即不会指向其他例如内核地址空间等区域);
  1. 解决用户页表映射与内核页表映射不一致的问题.

  内核使用某些专门编写的函数来安全地从用户给出的地址中读取或向其中写入数据. 一个例子便是 `fetchstr` 函数 (位于 `kernel/syscall.c:25`). 顾名思义, `fetchstr` 函数的作用是从用户地址空间中读取字符串. `fetchstr` 内部调用的是 `copyinstr` 函数 (位于 `kernel/vm.c:403`). `copyinstr` 函数从用户页表 `pagetable` 下的虚拟地址 `srcva` 处的位置复制至多 `max` 字节至 `dst` 处. 由于用户页表与内核页表并不相同, `copyinstr` 进一步调用 `walkaddr` (位于 `kernel/vm.c:109`) 来从用户页表 `pagetable` 中获取 `srcva` 对应的物理地址 `pa0`. 在 `copyinstr` 复制的时候内核会自动将物理地址转换为内核虚拟地址, 因此 `copyinstr` 可以毫无知觉地将数据从 `pa0` 复制到 `dst`. 这就解决了上述第二个问题. 至于第一个问题, 由于 `walkaddr` 同时还负责检查用户传入的指针是否位于进程的用户地址空间, 因此第一个问题也解决了.

  与 `copyinstr` 相反, 另一个类似的函数 `copyout` 将数据从内核空间复制到用户指定的地址.
