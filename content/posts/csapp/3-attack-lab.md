---
title: "笔记: 3. Attack Lab (CS:APP)"
description: "利用缓冲区溢出实施代码注入攻击, 以及利用地址无关的代码零件实施返回导向编程攻击"
date: 2024-07-11T20:04:55+08:00
draft: false
tags: ["csapp"]
series: ["csapp"]
author: ["xubinh"]
type: posts
---

> - **具体代码请移步至[GitHub](https://github.com/xubinh/csapp/tree/main/3-attack-lab).**

## Attack Lab

### 实验目的

对两个可执行文件 `ctarget` 和 `rtarget` 成功实施共计 5 次攻击, 包括针对 `ctarget` 的 3 次攻击 level#1 至 level#3 以及对应的针对 `rtarget` 的 2 次升级版攻击 level#2 和 level#3.

攻击类型包括 "代码注入 (code-injection, CI)" 攻击和 "返回导向编程 (return-oriented-programming, ROP)" 攻击两种, 其中前者通过在攻击字符串中包含 (注入) 攻击代码来直接实现攻击, 后者通过在被攻击者的代码中寻找所谓 "零件 (gadget)" 并将其组合起来实现攻击.

### 实验框架

#### `ctarget`, `rtarget` - 可执行文件

待实施攻击的可执行文件. `ctarget` 和 `rtarget` 均从标准输入 `stdin` 或者文件 `file` (使用选项 `-i file`) 中读取一个字符串, 代码如下所示:

```c
unsigned getbuf(){
    char buf[BUFFER_SIZE];
    Gets(buf);
    return 1;
}
```

其中函数 `Gets` 的作用和 C 标准库函数 `gets` 类似, 均为读取字符串至缓冲区 `buf` (并在末尾添加 `\0`). 由于函数 `Gets` 只是单纯地读入字符串, 并不检查缓冲区是否能够容纳得下该字符串, 因此如果输入的字符串足够长便会造成缓冲区溢出, 此时该字符串便可称为 "攻击字符串 (exploit strings)". 使用函数 `getbuf` 读入一个字符串有三种可能的结果:

1. 缓冲区能够容纳得下所读入的字符串, 此时函数 `getbuf` 将能够正常返回;
2. 缓冲区无法容纳字符串, 造成缓冲区溢出并导致普通的段错误, 此时函数 `getbuf` 无法正常返回;
3. 缓冲区无法容纳字符串, 造成缓冲区溢出, 但由于攻击字符串中包含精心构造的代码从而使得攻击者成功接管程序并实施攻击.

#### `hex2raw` - 帮手程序

用于生成攻击字符串的帮手程序. 通过标准输入传入使用空白符 (空格或者换行符) 分隔的十六进制字符对, 输出与之对应的二进制字节, 例如传入 `30 31 32 33 34 35 00` 将输出二进制字节 `0x30313233343500`. 传入的内容中允许包含形如 C 语言块注释的注释, 即 `/* some comment */`. 注意需要确保在 `/*` 和 `*/` 的两边保留空格以便程序能够正确解析.

#### `farm.c` - 零件仓库

这是用于实施 ROP 攻击的可用零件的源代码文件, 仅用于参考. 源代码已同 `rtarget` 一同编译形成二进制文件, 因此可在 `rtarget` 中找到对应机器代码.

#### 其他

- 在生成攻击字符串时可以将汇编代码存放在文件 (例如 `exploit.s`) 中 (汇编代码中字符 `#` 表示一行注释的开始), 使用命令 `gcc -c` 汇编生成二进制机器代码文件 (`exploit.o`), 再使用命令 `objdump -d` 反汇编生成包含汇编代码和机器代码的文件 (`exploit.d`), 最后配合使用 `hex2raw` 生成攻击字符串.
- 为了能够在离线环境下运行 `ctarget` 与 `rtarget`, 需要在命令行中使用选项 `-q` 禁止程序连接 CMU 服务器.
- 本实验可参考教材的 3.10.3 到 3.10.4 小节.
- 不允许使用攻击来绕过 validation 程序. 实际上实验要求攻击过程中跳转至的地址必须为下列地址中的其中一个:
  - 函数 `touch1`, `touch2`, 和 `touch3` 的起始地址.
  - 注入代码的起始地址.
  - 零件仓库中的任意函数的起始地址.
- 在对 `rtarget` 实施返回导向编程攻击时只允许使用位于函数 `start_farm` 和 `end_farm` 之间的代码生成零件.

{{< notice caution >}}
在运行 `ctarget` 时可能会报如下错误:

```text
Program received signal SIGSEGV, Segmentation fault.
0x00007ffff7dfe0d0 in __vfprintf_internal (s=0x7ffff7fa4780 <_IO_2_1_stdout_>, format=0x4032b4 "Type string:", ap=ap@entry=0x5561dbd8, mode_flags=mode_flags@entry=2) at ./stdio-common/vfprintf-internal.c:1244
```

对应位置的汇编代码为:

```text
 0x00007ffff7dfe0c0  __vfprintf_internal+144 movdqu (%rax),%xmm1
 0x00007ffff7dfe0c4  __vfprintf_internal+148 movups %xmm1,0x118(%rsp)
 0x00007ffff7dfe0cc  __vfprintf_internal+156 mov    0x10(%rax),%rax
 0x00007ffff7dfe0d0  __vfprintf_internal+160 movaps %xmm1,0x10(%rsp)
 0x00007ffff7dfe0d5  __vfprintf_internal+165 mov    %rax,0x128(%rsp)
```

调试过程具体见[debug.md](https://github.com/xubinh/csapp/tree/main/3-attack-lab/debug.md).

查阅网络资料后得知报错的直接原因是汇编代码

```text
0x00007ffff7dfe0d0  __vfprintf_internal+160 movaps %xmm1,0x10(%rsp)
```

中的指令 `movaps` 要求目的地址 `0x10(%rsp)` 必须与 16 字节对齐, 否则会导致段错误. 而报错的根本原因可能是 `ctarget` 程序编译时的环境和当前执行的环境不同, 导致调用 `printf` 函数时无法保证 `movaps` 指令的目的地址总是与 16 字节对齐.

由于上述问题, 在运行 `ctarget` 与 `rtarget` 的过程中要尽可能避免程序调用 `printf` 函数, 例如在运行前应使用选项 `-i <input-file>` 指定输入文件路径, 因为若使用标准输入会导致程序调用 `printf` 函数输出提示信息 "Type string:" 从而造成段错误; 此外在执行完攻击代码结束并返回时也会调用 `printf` 函数输出攻击成功的提示信息, 尽管此时无法避免调用 `printf` 函数, 但可以通过在攻击代码中增加对齐 `$rsp` 的功能的方法成功调用 `printf` 函数.
{{< /notice >}}

### 评价标准

| Phase | Program | Level | Method | Function | Points |
|-------|---------|-------|--------|----------|--------|
| 1     | CTARGET | 1     | CI     | touch1   | 10     |
| 2     | CTARGET | 2     | CI     | touch2   | 25     |
| 3     | CTARGET | 3     | CI     | touch3   | 25     |
| 4     | RTARGET | 2     | ROP    | touch2   | 35     |
| 5     | RTARGET | 3     | ROP    | touch3   | 5      |

- 总分为 10 + 25 + 25 + 35 + 5 = 100 分.

### 实验思路与总结

#### `ctarget` - level#1

目标: 利用栈溢出漏洞使函数 `test` 在返回时跳转至函数 `touch1` 处.

- 通过反汇编 `test` 函数可知 `test` 函数的工作是调用 `getbuf` 函数, 若 `getbuf` 成功返回则打印字符串 "No exploit. Getbuf returned 0x%x\n". 因此攻击目标调整为使函数 `getbuf` 返回至 `touch1` 处.
- 通过反汇编 `getbuf` 函数可知字符串的缓冲区的大小固定为 `0x28` 字节, 紧随缓冲区之后的 8 个字节内存存放的即为函数 `getbuf` 的返回地址.
- 通过反汇编代码内容可知函数 `touch1` 的入口地址为 `0x00000000004017c0`.

因此最终的攻击策略是将攻击字符串的前 40 字节设置为任意内容用于填充缓冲区的 40 字节, 然后将接下来的 8 个字节设置为函数 `touch1` 的入口地址. 具体见文件[c1.txt](https://github.com/xubinh/csapp/tree/main/3-attack-lab/c1.txt).

#### `ctarget` - level#2

目标: 利用栈溢出漏洞使函数 `test` 在返回时跳转**并调用**函数 `touch2`.

> - 在使用攻击代码进行函数调用时不需要使用 `jmp` 或 `call` 指令, 因为这些指令可能要求使用偏移地址而非绝对地址, 而偏移地址较为难以计算. 若要实现函数跳转, 直接使用 `ret` 指令配合绝对地址即可, 由于函数 `touch2` 并不包含返回语句 (其使用 C 库函数 `exit()` 直接终止程序的运行), 因此不需要考虑是否需要使用 `call` 指令压入返回地址等问题.

- 由于需要调用函数 `touch2` 并传入参数, 直接使用地址跳转至特定函数的方法此时便不再适用, 因为并不存在现有的函数实现 "跳转至函数 `touch2` 并传入给定参数" 的功能, 因此需要在攻击字符串中直接注入攻击代码.
- 若要使函数 `getbuf` 执行攻击字符串中包含的攻击代码, 需要使其原地返回至攻击代码的入口地址 (该地址仍位于字符串中, 因此称之为 "原地返回"). 通过反汇编代码内容可知进入 `getbuf` 时 (同时在分配缓冲区局部变量前) 寄存器 `$rsp` 的值为 `0x000000005561dca0`, 因此攻击字符串中的返回地址应设置为 `$rsp + 8` 即 `0x000000005561dca8`.
- 若要调用函数 `touch2`, 汇编代码首先需要将 cookie 作为参数传入 `touch2`, 然后通过移动 `$rsp` 至存储有函数 `touch2` 的入口地址的内存上并执行 `ret` 指令的方法调用函数 `touch2`. 函数 `touch2` 的入口地址为 `0x00000000004017ec`.
- 此外还需要对齐指针的 8 字节以及对齐前述的 `movaps` 指令的 16 字节, 因此需要额外注意移动 `$rsp` 的距离的设置.

综上可知攻击策略为 40 字节 padding + 8 字节攻击代码入口地址 + 攻击代码 (传入参数, 移动 `$rsp`, 执行 `ret`) + 16 字节对齐 padding + 函数 `touch2` 的入口地址. 具体见文件[c2.txt](https://github.com/xubinh/csapp/tree/main/3-attack-lab/c2.txt).

#### `ctarget` - level#3

思路:

- 将 cookie 字符串存储在攻击字符串中
- 将 cookie 字符串起始地址传入函数 `touch3`, 注意此处应使用 `lea` 指令
- 将 `touch3` 的入口地址包含在攻击字符串中
- 移动 `$rsp`
- 执行 `ret` 命令

方案:

- 攻击字符串 = 40 字节 padding + 攻击代码入口地址 + 攻击代码 (传入 cookie 字符串起始地址, 移动 `$rsp`, 执行 `ret`) + 对齐 padding + `touch3` 入口地址 + cookie 字符串.

数据:

- 攻击代码入口地址和 level#2 一样, 为 `0x000000005561dca8`
- `touch3` 入口地址为 `0x00000000004018fa`
- cookie 为 `0x59b997fa`, 转换为小写十六进制字符串为 `35 39 62 39 39 37 66 61`
- 对齐 padding 的长度取决于攻击代码的长度和位置, 由于攻击代码的长度同 level#2 一样, 因此对齐 padding 的长度也同样不变, 为 16 - 2 = 14 字节
- cookie 字符串的起始地址取决于对齐 padding 的长度和位置, 为 `$rsp + 10 + 14 + 8`
- 攻击代码中移动 `$rsp` 的距离取决于对齐 padding 的长度和位置, 为 `10 + 14`

具体见文件[c3.txt](https://github.com/xubinh/csapp/tree/main/3-attack-lab/c3.txt).

#### `rtarget` - level#2

> - level#2 只允许使用 `farm.c` 中位于 `start_farm` 和 `mid_farm` 之间的零件.
> - 尽管可以使用在 `start_farm` 和 `mid_farm` 之间的所有零件, 本阶段只需使用 `movq`, `popq`, `ret` 和 `nop` 四个类型的指令以及前 8 个寄存器 `$rax`, `$rbx`, `$rcx`, `$rdx`, `$rbp`, `$rsp`, `$rsi`, `$rdi` 即可构建出成功攻击 `rtarget` - level#2 所需的零件. (在 level#3 中可能还要用到包含有 `lea` 指令的零件[待办])
> - handout 中明确指出解决 level#2 只需要 2 个零件.
> - 由于 `mov` 指令并不改变 `$rsp`, 而 `pop` 指令仅将栈中数据弹出至寄存器, 因此攻击字符串的结构应为入口地址与数据的交替组合.

思路:

- 无法再通过在攻击字符串中包含代码的方式执行攻击, 因为包含栈的内存区域被标记为了 `nonexecutable`, 同时栈的位置也被随机化了, 因此既无法跳转至攻击代码, 即使跳转到了也无法执行.
- 虽然由于栈随机化而无法执行攻击者自己希望注入的代码, 但是代码的相对位置不会轻易发生改变, 而被攻击者的代码中也可能包含有一些虽然功能简单但组合起来也许能够实现攻击目的指令模式, 称为 "零件 (gadget)", 并且返回指令 `ret` 仍然有效, 于是可以利用返回指令配合巧妙的零件组合, 通过在不同零件之间跳转来实现攻击.
- 为了重新实现在 `ctarget` 中的 level#2 攻击, 需要想办法将 cookie 的值传送至 `$rdi` 并将 `$rsp` 指向存储有函数 `touch2` 入口地址的内存地址. 此时攻击字符串中只有存储数据才是有意义的, 因此可以使用 `popq` 配合在攻击字符串中包含数据来实现自定义数据的传送, 使用 `movq` 实现特定方向的数据传送, 使用 `popq` 和 `ret` 配合在攻击字符串中包含地址来实现跳转.

指令模式归纳:

- `movq` 指令的字节模式是 `48 89` 加上 `c0`-`ff` 中的某一个数, 其中 `c0`-`ff` 总共有 64 种可能, 对应于 8 个寄存器之间两两配对.
- `popq` 指令的字节模式是 `58`-`5f`, 共 8 种可能, 对应 8 个寄存器.
- `ret` 指令的字节模式是单字节 `c3`.
- `nop` 指令的字节模式是单字节 `90`.

找到的零件:

| 零件    | 汇编代码         | 机器代码      | 入口地址   | 位于函数                           |
| ------- | ---------------- | ------------- | ---------- | ---------------------------------- |
| 零件 #1 | `movq %rax %rdi` | `48 89 c7 c3` | `0x4019a2` | (1) `addval_273`, (2) `setval_426` |
| 零件 #2 | `popq %rax`      | `58 90 c3`    | `0x4019ab` | (1) `addval_219`, (2) `getval_280` |

方案:

- 使用零件 #2 将 cookie 值出栈至 `$rax`
- 使用零件 #1 将 `$rax` 传送至 `$rdi`
- 攻击字符串 = 40 字节 padding + 零件 #2 的入口地址 + cookie 值 + 零件 #1 的入口地址 + 函数 `touch2` 的入口地址

具体见文件[r2.txt](https://github.com/xubinh/csapp/tree/main/3-attack-lab/r2.txt).

#### `rtarget` - level#3

> - level#3 允许使用整个位于 `start_farm` 和 `end_farm` 之间的所有零件.
> - handout 中明确指出解决 level#3 共需要 8 个零件, 并且零件可重复使用.

指令模式归纳:

- `movl` 指令的字节模式是 `89` 加上 `c0`-`ff` 中的某一个数, 同 `movq` 指令类似.
- 功能性 (functional) `nop` 指令, 即逻辑上不是 `nop` 指令但实际执行起来相当于 `nop` 指令的指令:

  - `andb`: 以 `20` 开头.
  - `orb`: 以 `08` 开头.
  - `cmpb`: 以 `38` 开头.
  - `testb`: 以 `84` 开头.

  这四个指令头需要配合字节 `c0`, `c9`, `d2`, `db` (分别对应 `$al`, `$cl`, `$dl`, `$bl`) 来构成完整的指令. 例如 `20 c0` 代表指令 `andb %al`.

找到的零件:

| 零件              | 汇编代码                 | 机器代码         | 入口地址   | 位于函数                                                               |
| ----------------- | ------------------------ | ---------------- | ---------- | ---------------------------------------------------------------------- |
| 零件 #1 (level#2) | `movq %rax %rdi`         | `48 89 c7 c3`    | `0x4019a2` | (1) `addval_273`, (2) `setval_426`                                     |
| 零件 #2 (level#2) | `popq %rax`              | `58 90 c3`       | `0x4019ab` | (1) `addval_219`, (2) `getval_280`                                     |
| 零件 #3           | `movq %rsp %rax`         | `48 89 e0 c3`    | `0x401a06` | (1) `addval_190`, (2) `setval_350`                                     |
| 零件 #4           | `movl %eax %edi`         | `89 c7 c3`       | `0x4019a3` | (1) `addval_273`, (2) `setval_426`                                     |
| 零件 #5           | `movl %eax %edx`         | `89 c2 90 c3`    | `0x4019dd` | (1) `getval_481`, (2) `addval_487`                                     |
| 零件 #6           | `movl %esp %eax`         | `89 e0 c3`       | `0x401a07` | (1) `addval_190`, (2) `addval_110`, (3) `addval_358`, (4) `setval_350` |
| 零件 #7           | `movl %ecx %esi`         | `89 ce 90 90 c3` | `0x401a13` | (1) `addval_436`, (2) `addval_187`                                     |
| 零件 #8           | `movl %edx %ecx`         | `89 d1 38 c9 c3` | `0x401a34` | (1) `getval_159`, (2) `getval_311`                                     |
| 零件 #9           | `lea (%rdi,%rsi,1) %rax` | `48 8d 04 37 c3` | `0x4019d6` | (1) `add_xy`                                                           |

思路:

- 由于 `movq %rsp` 指令仅仅记录当前的 `$rsp` 值, 而不是攻击字符串中 cookie 所处的位置, 因此仅仅使用 `movq` 或 `movl` 指令是无法攻击成功的. 直接使用 `popq` 指令试图将 cookie 的地址弹出到 `$rax` 中也是不现实的, 因为程序使用了栈随机化技术, cookie 字符串的地址无法提前硬编码至攻击字符串中. 整个攻击过程中唯一固定不变的是攻击字符串本身, 或者说攻击字符串中各个要素之间的偏移. 因此要想攻击成功必须使用 `movq $rsp` 指令记录栈的基位置 (base position), 消除栈随机化的影响, 然后使用算数指令计算出 cookie 位于攻击字符串中的确切位置. 而能够找到的最明显的算数运算指令就是零件 #9 `lea (%rdi,%rsi,1) %rax`.
- cookie 在攻击字符串中的偏移由其位于各个零件中的位置决定.

方案:

- 使用零件 #3 存储栈的基位置至 `$rax`
- 使用零件 #1 将栈的基位置由 `$rax` 传送至 `$rdi`
- 使用零件 #2 将 cookie 的偏移弹出至 `$rax`
- 使用零件 #5, #8, #7 将 cookie 的偏移由 `$rax` 传送至 `$rsi`
- 使用零件 #9 计算 cookie 字符串的首地址, 存储至 `$rax`
- 使用零件 #1 将指针传入函数 `touch3`
- 攻击字符串 = 40 字节 padding + 零件 #3 的入口地址 + 零件 #1 的入口地址 + 零件 #2 的入口地址 + cookie 字符串的偏移 + 零件 #5, #8, #7 的入口地址 + 零件 #9 的入口地址 + 零件 #1 的入口地址 + 函数 `touch3` 的入口地址 + cookie 字符串

具体见文件[r3.txt](https://github.com/xubinh/csapp/tree/main/3-attack-lab/r3.txt).

### 实验结果展示

具体见文件:

- CTARGET#1: [c1.txt](https://github.com/xubinh/csapp/tree/main/3-attack-lab/c1.txt);
- CTARGET#2: [c2.txt](https://github.com/xubinh/csapp/tree/main/3-attack-lab/c2.txt);
- CTARGET#3: [c3.txt](https://github.com/xubinh/csapp/tree/main/3-attack-lab/c3.txt);
- RTARGET#2: [r2.txt](https://github.com/xubinh/csapp/tree/main/3-attack-lab/r2.txt);
- RTARGET#3: [r3.txt](https://github.com/xubinh/csapp/tree/main/3-attack-lab/r3.txt).

### 相关资料

无
