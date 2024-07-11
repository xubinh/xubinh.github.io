---
title: "笔记: 4. Cache Lab (CS:APP)"
description: "模拟实现一个高速缓存 cache, 同时在此基础上优化对矩阵的转置操作"
date: 2024-07-11T20:05:55+08:00
draft: false
tags: ["csapp"]
series: ["csapp"]
author: ["xubinh"]
type: posts
---

> - **具体代码请移步至[GitHub](https://github.com/xubinh/csapp/tree/main/4-cache-lab).**

## Cache Lab

### 实验目的

(1) 补全源文件 `csim.c` 以实现模拟一个高速缓存 cache; (2) 补全源文件 `trans.c` 以尽可能少的不命中次数实现对一个矩阵的转置.

### 实验框架

#### `csim.c` - 实现对高速缓存 cache 的模拟

在本文件中使用 C 代码模拟一个高速缓存 cache, 其以 `valgrind` 的内存跟踪信息作为输入, 模拟内存操作过程中一个 cache 的行为, 然后分别输出命中 (hit) 次数, 不命中 (miss) 次数以及驱逐 (eviction) 次数.

- 除了实现对 cache 的模拟以外还需要实现同样的参数爬取逻辑, 这可以借助 C 函数 `getopt` 实现, 需要用到的头文件包括:

  ```c
  #include <getopt.h>
  #include <stdlib.h>
  #include <unistd.h>
  ```

  `getopt` 函数的官方文档链接见 "相关资料" 小节.

- 模拟器必须对任意 $s$, $E$, $b$ 都能够得到正确结果, 这可以借助 C 函数 `malloc` 来实现, `malloc` 函数的文档可使用 Linux 命令 `man malloc` 查看.
- 本实验不要求对指令 cache 的模拟, 因此可以忽略 `xxx.trace` 文件中所有以 `I` 开头的行.
- 必须在 `main` 函数的结尾调用函数 `printSummary(hit_count, miss_count, eviction_count)` 来得到具有官方格式的输出.
- 本实验中可完全假设内存操作的 `size` 字节数据总是能够容纳在 cache 行的块中, 即 $size \leq B$.
- 建议像官方模拟器 `csim-ref` 一样实现 `-v` 选项

#### `csim-ref` - 官方实现的高速缓存 cache 参考模拟器

用法: `./csim-ref [-hv] -s <s> -E <E> -b <b> -t <tracefile>`

- `-h`: 打印帮助信息.
- `-v`: verbose 模式, 打印详细的跟踪信息.
- `-s <s>`: 组 (set) 索引位数 $s$ (组的数量 $S = 2^s$ )
- `-E <E>`: 相联度 (associativity) $E$ (即组相联中每组包含的行数)
- `-b <b>`: 块大小的对数 $b = \log_2 (B)$ (B 为块大小)
- `-t <tracefile>`: 要模拟的 valgrind 跟踪文件的文件名

- handout 中明确指出官方模拟器使用 LRU (least-recently used) 策略决定驱逐对象.
- 使用 `traces/write_allocate_or_not.trace` 可以得知官方模拟器使用 "写分配" 策略作为写不命中时的策略.

#### `traces/` - 测试样例目录

包含一系列用于测试 `csim.c` 的参考跟踪文件 (reference trace files).

其下的文件的文件名形如 `xxx.trace`, 这些文件是由一个名为 `valgrind` 的 Linux 程序生成的, 其作用是跟踪一个程序的内存使用情况. 典型的 `xxx.trace` 文件的内容格式如下:

```text
I 0400d7d4,8
 M 0421c7f0,4
 L 04f6b868,8
 S 7ff0005c8,8
```

其中每一行代表一次或两次内存读写操作, 且均具有格式 `[space]operation address,size`:

- `[space]` 表示可选的一个空格, 实际上当且仅当前缀为 `I` 时不加这个可选的空格.
- `operation` 可以是 `I`, `L`, `S` 和 `M`, 分别表示 "取指令", "加载数据", "存储数据" 和 "修改数据 (即加载数据 + 存储数据)".
- `address` 表示一个 64 位十六进制内存地址.
- `size` 表示所操作的字节数.

#### `test-csim` - 对 cache 模拟器进行评价

对所实现的 `csim.c` 进行评价.

用法:

```bash
linux> make
linux> ./test-csim
```

最终评价时将使用一系列不同的 cache 配置对编译得到的 `csim` 程序进行测试:

```bash
linux> ./csim -s 1 -E 1 -b 1 -t traces/yi2.trace
linux> ./csim -s 4 -E 2 -b 4 -t traces/yi.trace
linux> ./csim -s 2 -E 1 -b 4 -t traces/dave.trace
linux> ./csim -s 2 -E 1 -b 3 -t traces/trans.trace
linux> ./csim -s 2 -E 2 -b 3 -t traces/trans.trace
linux> ./csim -s 2 -E 4 -b 3 -t traces/trans.trace
linux> ./csim -s 5 -E 1 -b 5 -t traces/trans.trace
linux> ./csim -s 5 -E 1 -b 5 -t traces/long.trace
```

除了最后一个样例为 6 分以外, 其余样例均为 3 分, 共计 27 分.

#### `trans.c` - 实现最优化的矩阵转置

编写一个函数, 以尽可能少的不命中次数实现对矩阵的转置.

本文件中可以同时实现多个 (最多 100 个) 不同版本的矩阵转置函数, 每个函数需要使用特殊语句进行注册 (register):

```c
char my_trans_desc[] = "This is the description of `my_trans`";
void my_trans(int M, int N, int A[N][M], int B[M][N]);

registerTransFunction(my_trans, my_trans_desc);
```

将 `my_trans` 替换为任意函数名.

本文件中名为 `transpose_submit` 的函数表示提交版本, 可以将最终答案复制粘贴进这个函数. 请勿改动其对应注册字符串的内容防止注册失败.

- 每个版本的转置函数中均仅允许定义不超过 12 个 `int` 类型的局部变量 (包括转置函数本身和其调用的所有帮手函数中的所有局部变量), 这是因为本实验的打分程序并不对栈进行跟踪, 同时也因为本实验的目的就是专注于高速缓存而不是内存的访问.
- 不允许在转置函数中使用递归.
- 不允许更改所传入的矩阵 A 的内容, 不过对矩阵 B 可以随便操作.
- 不允许定义任何数组或者使用函数 `malloc` 和其任意变种.
- 如果要进行 debug 可以使用 `test-trans` 程序输出对应的 $\texttt{trace.f}i$ 然后使用 `-v` 选项运行参考 cache 模拟器.
- 因为测试时所用的 cache 配置为直接映射, 可能会发生抖动的问题, 因此在实现时要注意访问模式, 特别是沿着对角线时 (原文是 "think about the potential for conflict misses in your code, especially along the diagonal"). 尽量采取能够尽可能减少抖动的访问模式.
- 实现时可以采取 "分块" 的思想提高时间局部性, 减少 cache 不命中次数, 具体见 "相关资料" 小节.

#### `test-trans.c` - 对所实现的矩阵转置函数进行评价

对所实现的 `trans.c` 进行评价.

用法:

```bash
linux> make
linux> ./test-trans -M 32 -N 32
```

实验过程中进行测试时可以使用任意矩阵配置, 但最终打分时仅会使用如下三种不同的矩阵配置:

- $32 \times 32 ~~(M = 32, ~~N = 32)$
- $64 \times 64 ~~(M = 64, ~~N = 64)$
- $61 \times 67 ~~(M = 61, ~~N = 67)$

具体打分过程是使用 `valgrind` 对编译得到的 `trans` 程序提取其跟踪信息, 然后传入参考 cache 模拟器 (配置固定为 $(s = 5, ~~E = 1, ~~b = 5)$) 获取未命中次数 $m$, 根据 $m$ 的值进行评价. $m$ 的值越小得分越高, 共计 26 分, 其中第一个配置的未命中数小于 300 时得满分, 第二个配置小于 1300 时得满分, 而第三个配置小于 2000 时得满分, 具体关系见 handout.

- 本实验仅要求针对上述给出的三种矩阵配置以及参考模拟器的配置进行优化, 因此完全可以针对三种矩阵配置实现三个不同的函数, 然后在主函数中检查矩阵大小并分发给对应函数.
- `test-trans` 程序会对 `valgrind` 的输出进行过滤, 剔除任何和栈有关的内存访问, 这是因为 `valgrind` 的输出中关于栈的部分绝大部分都跟本实验的代码无关. 但是也因为过滤掉了和本实验有关的栈访问, 所以本实验禁用了栈 (即数组) 的使用并限制了局部变量的使用.

#### `Makefile`

- 可以使用如下命令编译所有程序:

  ```bash
  linux> make clean
  linux> make
  ```

#### `driver.py` - 最终评分工具

对整个实验的实现进行评分.

用法:

```bash
linux> ./driver.py
```

具体打分过程是首先调用 `test-csim` 程序对 cache 模拟器进行评分, 然后调用 `test-trans` 对矩阵转置函数进行评价.

#### 其他

- 程序要完全正常地通过编译, 不能够报任何 warning.

### 评价标准

- 对于 cache 模拟器, 总共有 8 个测试样例, 除了最后一个样例为 6 分以外, 其余样例均为 3 分, 共计 27 分.
- 对于所实现的矩阵转置函数, 总共有 3 个测试配置, 其中第一个配置的未命中数小于 300 时得满分, 第二个配置小于 1300 时得满分, 而第三个配置小于 2000 时得满分 (超出阈值时的失分细则见 handout), 共计 26 分.

### 实验思路与总结

#### Part A

- 主要任务是模拟一个具有组索引位数 $s$, 相联度 $E$ 和块偏移位数 $b$ 的 cache, 因为需要使用 LRU 作为驱逐策略, 所以用一个链表表示一个 cache 的组是比较合适的, 元素在链表中越靠前就代表其最近一次使用时间越晚, 然后通过遍历链表中的元素来模拟在组中查找特定行的行为. 若找到或者需要新载入一个行只需要将该行插入表头之后即可.
- 由于写不命中时的策略是写分配, 所以写不命中的效果等价于读不命中的效果. 又因为写命中的效果等价于读命中的效果, 所以整个写操作的效果等价于读操作的效果. 而修改操作又等价于一个读操作加一个写操作, 所以总的加起来实际上只需要实现一个读操作即可.
- 为了爬取 traces 文件所需的 I/O 操作以及爬取传入参数所需的 `getopt` 操作属于工具类代码, 并不是本实验考察的重点.
- 由于选择了链表 (为了方便实际上采用了双链表) 作为 cache 的实现, 在写代码时格外需要注意常规的链表操作, 例如插入删除的先后顺序等等.

#### Part B

- 首先整个 Part B 使用的都是 $s = 5, E = 1, b = 5$ 的 cache 配置, 其中 $s = 5$ 和 $b = 5$ 意味着 cache 行的块大小为 32 字节并且总共有 32 行, 对于 `int` 类型的整数而言, cache 的一行能包含 8 个整数; 而 $E = 1$ 意味着 cache 是直接映射, 每个组仅包含一行 (即相联度为 1), 于是每 8 × 32 = 256 个 `int` 整数就会占据一遍所有组, 也就是整个 cache 只能在同一时间保存 32 个组索引互不相同的 8 整数长条的副本.
- 其次本实验部分的评价标准完全是 cache 未命中 (miss) 次数. 对于一个数据块, 若要对其进行处理, 就必须要将其读入 cache (前提是写分配, 写不分配直接对内存进行写入), 因此首次读入时必然会 miss 一次, 这是不可避免的. 为了降低 miss 次数, 真正的关键在于减少 "处理一次没处理完, 下次又要再读入" 的抖动现象, 因此整个思路应该围绕着 "如何单次读取单次处理完毕" 来展开.

> [!NOTE]
>
> 在 `tracegen.c` 中可以找到矩阵 A 和 B 的定义:
>
> ```c
> static int A[256][256];
> static int B[256][256];
> ```
>
> 通过 GDB 可知 A 的初始地址为 `0x70a0`, B 的初始地址为 `0x470a0` (中间恰好相差 256 × 256 × 4 个字节). 因此 A 的初始地址以及 B 的初始地址都是和 $s = 5, ~~ E = 1, ~~ b = 5$ 配置下的 cache 对齐的, 并且 A 的初始地址对应的 cache 行号和 B 的初始地址对应的行号也相同.

##### 32 × 32

- 列数等于 32 意味着矩阵的每行等于 cache 的 4 行, 因此矩阵每 8 行就会对应一遍整个 cache.
- 由于 cache 中的一个块等于 8 个 `int`, 所以矩阵一行中的每 8 个整数长条对应一个块, 一行可以划分为连续的 4 个长条, 对应于连续的 4 个 cache 块.
- 为了尽量提高空间局部性, 最好就是一个 8 整数长条 (对应一个 cache 块) 仅加载一次就完成转置, 此后在逻辑上将该块所占用的 cache 行视作 "可用".
- 由于要做的是转置, 所以 "行" 要对应至 "列", 又由于一次要转置一个 8 整数长条, 因此将矩阵分为 8 × 8 的小块 (分块完之后 A 就成为 4 × 4 分块矩阵) 来分别进行处理是比较好的方法.
- 将 A 转置到 B 至少需要对 A 完全读取一遍, 对 B 完全写入一遍, 因此 miss 次数至少为 8 × (4 × 4) × 2 = 256 次. 满分为 300 次.
- 非对角 8 × 8 小块所占据的 cache 空间互不重合, 因此可以放心按照一般的遍历顺序进行转置. 转置一个块所产生的 miss 次数为 16 次, 即最优情况. 对角 8 × 8 小块需要利用到 12 个局部变量的帮助, 转置一个块产生的 miss 次数为 20 次. 理论 miss 总次数为 272 次, 实际为 276 次.

##### 64 × 64

- 矩阵一行现在对应于 8 个 cache 块. 同样对矩阵采取划分为 8 × 8 小块的分块方式.
- 整个 cache 现在仅对应于 4 个矩阵行, 这意味着现在即使是非对角块其上半部分和下半部分所占据的 cache 空间也会重叠. 对角块的重叠情况比非对角块还严重.
- 使用 12 个局部变量仍然能够做到对非对角块的一次处理, 一个块的 miss 次数仍然为 16 次. 对角块则直接放弃尝试一次处理, 采用对其进行 4 × 4 分块并利用 12 个局部变量尽可能减少 miss 次数, 一个块的 miss 次数为 32 次. 理论 miss 总次数为 1152 次, 实际为 1156 次.

##### 61 × 67

- 因为矩阵的维数不再与 cache 相互对齐, 所以想要做到单次读取单次处理不现实. 最好的做法是仍然采取像 64 × 64 矩阵中那样的分块方案, 在一定程度上降低 miss 总数的数学期望值.
- 对于完整的 8 × 8 对角块与非对角块, 处理方法是将其当作 64 × 64 中的对角块与非对角块来进行处理 (实际上代码就是复制粘贴 64 × 64 方案中的).
- 对于剩下的不完整的边角料, 处理方案是每 4 行 (对于右侧边角料块) 或每 4 列 (对于下方边角料块) 循环遍历, 减少抖动. 实际 miss 总次数为 2140 次 (满分阈值 2000 次).

### 实验结果展示

```text
Part A: Testing cache simulator
Running ./test-csim
                        Your simulator     Reference simulator
Points (s,E,b)    Hits  Misses  Evicts    Hits  Misses  Evicts
     3 (1,1,1)       9       8       6       9       8       6  traces/yi2.trace
     3 (4,2,4)       4       5       2       4       5       2  traces/yi.trace
     3 (2,1,4)       2       3       1       2       3       1  traces/dave.trace
     3 (2,1,3)     167      71      67     167      71      67  traces/trans.trace
     3 (2,2,3)     201      37      29     201      37      29  traces/trans.trace
     3 (2,4,3)     212      26      10     212      26      10  traces/trans.trace
     3 (5,1,5)     231       7       0     231       7       0  traces/trans.trace
     6 (5,1,5)  265189   21775   21743  265189   21775   21743  traces/long.trace
    27


Part B: Testing transpose function
Running ./test-trans -M 32 -N 32
Running ./test-trans -M 64 -N 64
Running ./test-trans -M 61 -N 67

Cache Lab summary:
                        Points   Max pts      Misses
Csim correctness          27.0        27
Trans perf 32x32           8.0         8         276
Trans perf 64x64           8.0         8        1156
Trans perf 61x67           8.6        10        2140
          Total points    51.6        53
```

### 相关资料

- [分块 (blocking) - 提升时间局部性](http://csapp.cs.cmu.edu/public/waside/waside-blocking.pdf)
- [`getopt` 函数文档](https://man7.org/linux/man-pages/man3/getopt.3.html)
