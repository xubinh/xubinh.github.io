---
title: "xv6 教材笔记 - 1. Operating system interfaces"
hideSummary: true
date: 2024-07-28T12:37:07+08:00
draft: true
tags: ["xv6", "MIT 6.s081"]
series: ["xv6", "MIT 6.s081"]
author: ["xubinh"]
type: posts
math: false
---

- xv6 借鉴了 Unix 的基本接口和内部实现.
- Unix 的接口十分精简, 但机制设计得很好, 能够组合出丰富的功能. 其他后继者如 BSD, Linux, macOS, Solaris, 甚至 Windows 都能从中看到 Unix 的影子.
- xv6 使用了传统的**内核** (kernel) 的形式, 内核是一个特殊的程序, 用于运行其他程序, 并为其他程序提供系统服务.
- 当一个程序需要使用系统服务时, 它需要执行**系统调用** (system call), 系统调用将进入内核, 执行服务, 然后从内核返回. 因此程序在**用户空间** (user space) 和**内核空间** (kernel space) 之间交替执行.
- 内核使用了由 CPU 提供的一种硬件保护机制来确保每个用户程序只能访问它们各自的内存空间. 内核自身则拥有特权 (privilege), 不收硬件保护机制的限制. 当用户程序请求系统调用时, 用户程序的特权等级将被提升以便执行内核中的预先编排好的系统程序.
- 内核所提供的系统调用就是用户程序所看到的一个操作系统暴露出的接口. xv6 提供了经典 Unix 系统所提供的服务和系统调用的一个子集.
- 相对于内核, **shell** 是一个代表用户与操作系统进行交互的命令行接口. shell 本质上就是一个普通的用户程序, 它读取用户输入的命令, 然后通过执行合适的系统调用来实现这些命令. 也正是由于 shell 与一般用户程序无异, 用户能够很容易地在不同的 shell 实现之间进行更换.
- xv6 实现的 shell 模仿的是 Unix Bourne shell (即 sh).

## 1.1 Processes and memory

- 系统调用 `fork` 用于父进程创建一个新的子进程. 子进程拥有和父进程一模一样的内存映像 (只是看上去一样, 实际上是两个物理地址不同的副本) 和文件描述符表. `fork` 子进程的 PID, 在子进程中为零.
  - 函数签名: `int fork()`
- 系统调用 `exit` 将会使得调用进程停止运行并释放所有资源. `exit` 接受一个整型参数表示退出状态 (exit status), 一般使用 0 表示成功, 使用 1 表示失败.
  - 函数签名: `int exit(int status)`
- 系统调用 `wait` 一般由一个父进程调用, 其返回值为一个由 `exit` 停止的 (或手动被 kill 掉的) 子进程的 PID, 并将子进程的退出状态复制到传入的地址中. 如果没有子进程退出, 那么 `wait` 将一直阻塞下去. 如果父进程没有任何子进程, 那么 `wait` 将立即返回 -1. 传入的地址可以是零指针, 表示父进程不关心子进程的退出状态.
  - 函数签名: `int wait(int *status)`
- 系统调用 `exec` 使用硬盘上的某个文件中所存储的内存映像 (memory image) 替换当前正在执行的程序的内存映像 (注意并不替换文件描述符表). 这个文件必须具有某种可解释的格式以指明哪些内容是指令, 哪些又是数据, 程序应该从哪一条指令开始执行等等信息. xv6 使用的格式为 ELF 格式. 一般情况下这种文件是程序员通过编译一个程序的源文件得到的. `exec` 并不会返回, 而是顺着 ELF 文件中指明的程序起始地址在新程序中继续执行下去. `exec` 接受两个参数, 第一个是所要执行的可执行文件的名称, 第二个是存储着所要传入的字符串参数的数组.
  - 函数签名: `int exec(char *file, char *argv[])`
- xv6 shell 就是通过使用 `exec` 系统调用实现代替用户执行程序的功能的. xv6 shell 程序的主要逻辑是一个 loop 循环, 每次循环都先读取用户的输入, 然后调用 `fork` 创建子进程, 子进程将负责调用 `exec` 执行用户想要执行的程序, 而父进程负责调用 `wait` 等待子进程退出.
  - 初看上去似乎 `fork` 和 `exec` 完全可以合并为一个单独的系统调用, 但此后将会了解到 shell 在实现它的 I/O 重定向功能时正是利用了 `fork` 和 `exec` 的分离设计.
  - 为了避免 `fork` 出一个子进程并分配好所有内存之后又立即调用 `exec` 使用另一个程序的内存映像将其替换掉带来的巨大浪费, 内核使用一种称为**写时复制** (copy-on-write) 的虚拟内存技术来优化 `fork` 的实现.
- 系统调用 `sbrk` 用于程序 (例如 `malloc`) 在运行时动态向系统申请内存. `sbrk` 接受一个整型参数, 表示要申请的字节数. 其返回值为所申请内存块的起始地址.
  - 函数签名: `char *sbrk(int n)`

{{< notice tip >}}

关于为什么 `sbrk` 要叫 `sbrk` 可以参考[文档](https://linux.die.net/man/2/sbrk):

- **brk**() and **sbrk**() change the location of the *program break*, which defines the end of the process's data segment (i.e., the program break is the first location after the end of the uninitialized data segment). Increasing the program break has the effect of allocating memory to the process; decreasing the break deallocates memory.
- **brk**() sets the end of the data segment to the value specified by *addr*, when that value is reasonable, the system has enough memory, and the process does not exceed its maximum data size.
- **sbrk**() increments the program's data space by *increment* bytes. Calling **sbrk**() with an *increment* of 0 can be used to find the current location of the program break.

{{< /notice >}}

## 1.2 I/O and File descriptors

- 一个文件描述符就是一个整数, 这个整数代表了一个由内核管理的, 能够由进程读取或写入的文件对象. 文件描述符可以通过打开一个文件, 目录或设备 (device) 来获得, 也可以通过创建一个管道 (pipe), 或者通过复制一个已存在的文件描述符来获得. 文件描述符的最大作用是对不同的文件, 管道以及设备进行抽象, 使得它们个个看起来都像是字节流.
- xv6 内部为每个进程单独维护一个表格, 并使用文件描述符充当下标对表格中的文件对象进行索引, 这样每个进程都拥有一组互不相关的文件描述符. 依照惯例, 一个进程从文件描述符 0 读取输入 (此即标准输入 (stdin)), 向文件描述符 1 写入输出 (此即标准输出 (stdout)), 并向文件描述符 2 写入错误信息 (此即标准错误 (stderr)).
- 系统调用 `read` 用于从由文件描述符索引的文件对象中读取数据. 执行 `read(fd, buf, n)` 将从文件描述符 `fd` 中读取**至多** (注意不一定刚好) `n` 个字节至缓冲区 `buf` 中, 并返回真实读取的字节数. `read` 读取的字节数少于 `n` 当且仅当执行过程中出错. 每个文件通过一个偏移 (offset) 标识当前已读取内容的位置, 调用 `read` 将读取并向前移动这个偏移, 因此本次 `read` 的结束位置将是下一次 `read` 的起始位置.
  - 函数签名: `int read(int fd, char *buf, int n)`
- 系统调用 `write` 用于向由文件描述符索引的文件对象中写入数据. 执行 `write(fd, buf, n)` 将从=向文件描述符 `fd` 中写入**至多** (注意不一定刚好) `n` 个字节至缓冲区 `buf` 中, 并返回真实写入的字节数. `write` 写入的字节数少于 `n` 当且仅当执行过程中出错. 和 `read` 类似, 每个文件通过另一个偏移标识当前已写入内容的位置, 调用 `write` 将读取并向前移动这个偏移, 因此本次 `write` 的结束位置将是下一次 `write` 的起始位置.
  - 函数签名: `int write(int fd, char *buf, int n)`
- 系统调用 `close` 用于释放一个文件描述符, 以便所释放的描述符能够被后续的 `open`, `pipe`, 以及 `dup` 等系统调用重新使用. 新分配的文件描述符总是当前未使用的文件描述符中序号最小的那一个.
  - 函数签名: `int close(int fd)`
- 由于 `exec` 仅替换内存映像, 不替换文件打开表, shell 实现 I/O 重定向的方法是在 `fork` 和 `exec` 之间 (即子进程创建但还未开始执行用户所指定的程序时) 对标准输入所关联的文件进行更换, 这样在执行 `exec` 后新程序虽仍然从标准输入进行读取, 但读取的文件已经从终端变为了所重定向至的文件.
  - 题外话. Unix 的设计哲学就是解耦, 以最少的基础部件组合出最丰富的功能, 有点类似于数学中的公理系统. 如果不将 `fork` 和 `exec` 实现为两个独立的系统调用, 而是一个统一的 (例如) 称为 `forkexec` 的调用, 那么虽然 shell 仍然能够实现 I/O 重定向, 但是它只能要么在调用 `forkexec` 前将父进程和子进程的标准输入同时改掉 (并在子进程返回后再改回来), 要么将重定向的要求以参数形式传入 `forkexec`, 要么必须在 shell 的文档中加上这么一句话: "所有要写给 shell 调用的程序都必须自己实现 I/O 重定向", 三个选项一个比一个奇怪.
- 系统调用 `open` 用于打开一个文件并返回内核为该文件分配的文件描述符. `open` 接受两个参数, 一个是文件路径, 另一个是文件的打开模式. 所有可用的打开模式均定义在 `kernel/fcntl.h` (file control) 头文件中. 共有 5 个不同的打开模式, 分别是 `O_RDONLY`, `O_WRONLY`, `O_RDWR`, `O_CREATE`, 以及 `O_TRUNC`. 它们的意义分别是 "只读", "只写", "既读又写", "如果文件不存在则创建", 以及 "打开时将文件长度截断为 0".
- 尽管 `fork` 会复制父进程的文件描述符表给子进程, 底层的文件偏移却是在父进程和子进程之间共享的, 这就为 shell 顺序执行多个程序并顺序输出各个程序的执行结果至同一个文件中提供了设计空间 (例如考虑像 `(echo hello; echo world) >output.txt` 这样的命令).
- 系统调用 `dup` 用于复制一个给定的文件描述符, 并返回一个新的文件描述符, 新描述符与原描述符共享同一个底层文件对象 (也因此共享文件的偏移), 这一点类似于 `fork`, 只不过 `fork` 是在父进程和子进程之间复制文件描述符. `dup` 允许 shell 实现类似于 `2>&1` 这样的重定向, 其原理是关闭文件描述 2 并复制文件描述符 1, 这样文件描述符 2 将自动分配至文件描述符 1 所指向的文件.
  - 函数签名: `int dup(int fd)`
- 除非使用 `fork` 和 `dup` 进行文件描述符的复制, 其他任何方式得到的文件描述符都不共享底层的文件偏移, 即使是像 `open` 这样的系统调用也不是 (即连续两次调用 `open` 将打开两个独立的文件描述符).
- 文件描述符最大的作用就是向程序员提供了对文件, 设备 (例如控制台 (console)) 以及管道等等不同的概念的高级抽象.
