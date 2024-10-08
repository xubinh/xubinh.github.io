---
title: "CSAPP 实验笔记 - 6. Malloc Lab"
description: "实现动态内存分配器"
summary: "实现动态内存分配器"
date: 2024-07-11T20:07:55+08:00
draft: false
tags: ["CSAPP", "CMU 15-213", "ICS"]
series: ["CSAPP", "CMU 15-213", "ICS"]
author: ["xubinh"]
type: posts
math: true
---

{{< notice note >}}

- **具体代码请移步至[GitHub](https://github.com/xubinh/csapp/tree/main/6-malloc-lab).**

{{< /notice >}}

## Malloc Lab

### 实验目的

模仿官方 `malloc` 包, 自由探索可能的实现动态内存分配器的方案, 填充文件 `mm.c`, 在预定义的抽象内存模型上实现一个动态内存分配器.

### 实验框架

#### `memlib.c` - 预定义的内存模型

`memlib.c` 定义了一个简单的内存模型抽象, 其中暴露在外的可供调用的函数有:

- `void *mem_sbrk(int incr)`: 模仿 Unix `sbrk` 函数, 但限制了参数 `incr` 为正整数.
- `void *mem_heap_lo()`: 返回堆空间的第一个字节的地址.
- `void *mem_heap_hi()`: 返回堆空间的最后一个字节的地址 (等价于 `brk - 1`).
- `size_t mem_heapsize()`: 返回堆空间的大小 (单位: 字节).
- `size_t mem_pagesize()`: 返回系统的页大小 (单位: 字节).

内存模型的抽象方法是使用官方 `malloc` 函数在真实堆内存中申请一大块内存用于模拟主存, 并使用 `mem_sbrk` 函数来模拟堆空间的扩充.

#### `mm.c` - 待实现的分配器的骨架

`mm.c` 中包含四个待填充的函数:

- `int mm_init(void)`: 根据分配器的具体实现做对应的初始化.

  - 评分工具 `mdriver.c` 会在调用任何 `mm_malloc`, `mm_realloc` 和 `mm_free` 函数之前调用本函数以对堆进行必要的初始化.
  - 如果初始化失败则返回 -1, 否则返回 0.

  初始实现是什么也不做:

  ```c
  int mm_init(void) {
      return 0;
  }
  ```

- `void *mm_malloc(size_t size)`: 模仿官方 `malloc` 函数.

  - 函数 `mm_malloc` 需要返回一个指向已分配块中的有效载荷的首字节的指针, 并且该有效载荷的大小必须至少为 `size`.
  - 整个已分配块 (包含有效载荷以及可能的头部或尾部) 必须位于堆内存范围之内, 并且不允许与其他已分配块发生重叠.
  - 由于评分工具会将本函数与官方 `malloc` 函数进行比较, 而官方函数总是返回 8 字节对齐的指针, 因此本函数同样需要确保返回 8 字节对齐的指针.

  初始实现只是简单地线性扩充堆空间并返回相应有效载荷的首字节地址:

  ```c
  #define ALIGNMENT 8  // 对齐大小
  #define ALIGN(size) (((size) + (ALIGNMENT - 1)) & ~0x7)  // 对齐操作
  #define SIZE_T_SIZE (ALIGN(sizeof(size_t)))  // 对 size_t 类型变量的长度进行对齐

  void *mm_malloc(size_t size) {
      // 块大小:
      int newsize = ALIGN(SIZE_T_SIZE + size);

      // 分配空间:
      void *p = mem_sbrk(newsize);

      if (p == (void *)-1){
          return NULL;
      }

      else {
          // 将有效载荷大小存储至头部:
          *(size_t *)p = size;

          // 返回有效载荷的首地址:
          return (void *)((char *)p + SIZE_T_SIZE);
      }
  }
  ```

- `void mm_free(void *ptr)`: 模仿官方 `free` 函数.

  - 函数 `mm_free` 负责释放指针 `ptr` 所指向的已分配块. 本函数只需要确保在指针 `ptr` 指向先前调用 `mm_malloc` 或 `mm_realloc` 所返回的未被释放的已分配块的前提下正常工作即可.

  初始实现什么也不做:

  ```c
  void mm_free(void *ptr) {
  }
  ```

- `void *mm_realloc(void *ptr, size_t size)`: 模仿官方 `realloc` 函数.

  - 函数 `mm_realloc` 需要返回一个指向已分配的且大小至少为 `size` 的块, 其中:
    - 如果 `ptr` 等于 `NULL` (即不需要复制任何已有数据), 那么本函数等价于 `mm_malloc(size)`;
    - 如果 `size` 等于 0 (即指定新分配块的有效载荷大小为 0), 那么本函数等价于 `mm_free(ptr)`;
    - 如果 `ptr` 不等于 `NULL` 并且 `size` 不等于 0, 那么在 `ptr` 为先前调用 `mm_malloc` 或 `mm_realloc` 所返回的结果的前提下, 本函数的作用是返回一个新的有效载荷大小至少为 `size` 的已分配块, 其中:
      - 所返回的指针和 `ptr` 不必相同, 因为如果 `size` 过大就必须申请一个新的内存块.
      - 完成重分配之后记得释放 `ptr` 所指向的旧的已分配块.
      - 如果 `size` 小于 `ptr` 所指向的已分配块的有效载荷大小, 那么确保前 `size` 个字节的内容被复制到新分配块即可; 如果 `size` 大于该有效载荷大小, 那么只需要确保有效载荷的内容全部被复制到新分配块即可, 多余的区域继续保持未初始化的状态, 不要做任何改动.

  初始实现直接请求新的块, 复制内容, 并释放旧的块:

  ```c
  void *mm_realloc(void *ptr, size_t size) {
      void *oldptr = ptr;
      void *newptr;
      size_t copySize;

      // 直接分配新的块:
      newptr = mm_malloc(size);

      if (newptr == NULL) {
          return NULL;
      }

      copySize = *(size_t *)((char *)oldptr - SIZE_T_SIZE);

      // 如果新载荷小于旧载荷, 则只复制新载荷长度的内容:
      if (size < copySize) {
          copySize = size;
      }

      // 将旧载荷内容复制进新载荷中:
      memcpy(newptr, oldptr, copySize);

      // 释放旧载荷:
      mm_free(oldptr);

      return newptr;
  }
  ```

- 可以在 `mm.c` 中根据需要定义其他的静态 (私有) 帮手函数.

除了上述四个函数以外, 还可以选择实现一个可选的堆空间合理性检查工具 `int mm_check(void)` 来检查任何重要的不变量或一致性条件. 因为动态内存分配器的实现需要大量的无类型指针操作, 因此实现一个合理性检查工具能够极大地降低开发难度. 一些可能的不变量或一致性条件包括:

- 空闲链表中的指针是否均指向合法的块?
- 空闲链表中的所有块是否均被标记为空闲块?
- 是否所有空闲块均位于空闲链表中?
- 是否存在连续的空闲块没有被合并?
- 是否存在已分配块相互重叠?

函数 `mm_check` 返回非零值 (true) 当且仅当通过一致性检查. 记得在进行最终评分的时候注释掉任何 `mm_check` 语句 (以及其他任何不必要的语句, 例如 `printf` 语句), 避免对吞吐量造成影响.

#### `traces` - 测试样例

根目录下包含 2 个初始测试样例 `short1-bal.rep` 和 `short2-bal.rep`, 主要用于前期验证分配器实现的合理性.

在 `traces` 目录下还包含 11 个正式测试样例:

- `amptjp-bal.rep`
- `cccp-bal.rep`
- `cp-decl-bal.rep`
- `expr-bal.rep`
- `coalescing-bal.rep`
- `random-bal.rep`
- `random2-bal.rep`
- `binary-bal.rep`
- `binary2-bal.rep`
- `realloc-bal.rep`
- `realloc2-bal.rep`

上述正式测试样例中的前 9 个仅测试 `malloc` 和 `free` 两个函数, 后 2 个则还额外测试 `realloc` 函数.

#### `mdriver.c` - 评分驱动程序

`mdriver.c` 用于评估所实现的动态内存分配器的总体性能. 使用 `make` 命令对其进行编译, 然后使用命令行 `./mdriver` 执行评估.

参数:

- `-t <trace-dir>`: 使用指定的 trace 文件目录, 覆盖 `config.h` 中的相应配置.
- `-f <trace-file>`: 使用指定的 trace 文件. 可以结合本选项与 2 个初始测试样例在开发初期进行调试.
- `-h`: 打印帮助信息.
- `-l`: 除了所实现的分配器以外同时也对官方 `libc` 的 `malloc` 包进行评估.
- `-v`: 打印详细信息.
- `-V`: 打印更加详细的信息.

#### 其他

##### 注意事项

- 不允许改变 `mm.c` 中的任何接口.
- 不允许调用任何已有的与内存管理相关的系统函数或库函数, 例如 `malloc`, `calloc`, `free`, `realloc`, `sbrk`, `brk`, 以及这些函数的其他变种等等.
- 不允许定义任何全局或静态**复合**数据结构, 例如数组, 结构体, 树, 链表等等 (可能是因为全局或静态的对象位于堆区域之外, 所定义的复合数据结构并不是 scalable 的).
- 允许定义全局标量变量, 如整数, 浮点数与指针等等.

##### 一些提示

- 遇到困难时应使用 `gcc -g` 以及 GDB 进行调试.
- 在开始动手之前应确保理解了 CS:APP 书中那个隐式空闲链表的实现中的每一行代码.
- 模仿书中的实现, 将各种关于指针的运算封装在宏 (macro) 中.
- 建议将整个实验分为三个循序渐进的开发阶段:
  1. 正确实现 `mm_malloc` 和 `mm_free` 函数, 确保通过前 9 个正式测试样例;
  2. 基于对 `mm_malloc` 和 `mm_free` 函数的调用实现 `mm_realloc` 函数, 并通过后 2 个正式测试样例;
  3. 不基于 `mm_malloc` 和 `mm_free` 函数, 实现独立的 `mm_realloc` 函数, 冲击更好的性能.
- 使用 profiler 对分配器实现进行性能上的调试优化, 例如 `gprof` 工具.

##### 关于评分工具 `./mdriver` 所使用的默认吞吐量配置

- 评分工具 `./mdriver` 所使用的默认 libc `malloc` 的吞吐量为 600 Kops/sec, 虽然这个吞吐量大小不太能反映真实的机器性能 (例如在实现过程中某次测试 `binary-bal.rep` 时 libc `malloc` 的吞吐量达到了 19228 Kops/sec), 但是设置这个吞吐量目标的目的并不是类似于 "这个目标非常难达到, 只要使得你的程序追上这个目标就说明你的程序非常棒", 而是类似于 "这个目标是平均难度, 不是很容易但也没有多难, 你应该在达到这个吞吐量目标之后转而追求内存利用率的提升". 因此尽管该默认值无法反映真实机器速率, 但仍然是符合本实验所要传达的理念的 (即设计一个在吞吐率和内存利用率之间达到良好平衡的分配器).
- 实在感觉到不舒服可以在运行 `./mdriver` 时使用 `-l` 选项测试本机上的 libc `malloc` 并肉眼比较吞吐率.
- 理论上 libc 的 `malloc` 会在执行过程中进行许多额外的工作, 因此本实验中实现的简单分配器的吞吐量应当很容易超过 libc 的 `malloc` 的吞吐量. 实际测试结果也证明了这一点.

##### 用于自动化测试的脚本

下面的脚本对测试过程进行自动化:

```bash
#!/usr/bin/env bash

trace_files=(
    "amptjp-bal.rep"
    "cccp-bal.rep"
    "cp-decl-bal.rep"
    "expr-bal.rep"
    "coalescing-bal.rep"
    "random-bal.rep"
    "random2-bal.rep"
    "binary-bal.rep"
    "binary2-bal.rep"
    "realloc-bal.rep"
    "realloc2-bal.rep"
)

# 运行总测试 (提供概览):
echo '--------------------------------'
./mdriver -t traces/ -V
echo ""
echo ""

# 单独运行每个测试 (提供详细信息):
for trace_file in "${trace_files[@]}"; do
    echo '--------------------------------'
    ./mdriver -f traces/"$trace_file" -V -l
    echo ""
    echo ""
done
```

使用方法为 `./run.sh > some_result.txt`.

### 评价标准

正确性 (20 分): 通过一个 trace 文件得一定分数, 全部通过即得满分.

性能 (35 分): 分为**内存利用率**和**吞吐量**两个维度, 最终的性能指标 $P$ 的计算公式为

$$
\begin{equation}
P = \omega U + (1 - \omega) \min \bigg( 1, \frac{T}{T_{libc}} \bigg),
\end{equation}
$$

其中 $U$ 为峰值利用率, $T$ 为吞吐率, $T_{\text{libc}}$ 为官方 `malloc` 包的吞吐率 (可在 `config.h` 中配置). $\omega$ 决定内存利用率与吞吐量在 $P$ 中的权重, $\omega$ 默认为 $0.6$.

格式 (10 分):

- `mm.c` 中的实现应该解耦为若干个函数, 并使用尽可能少的全局变量. `mm.c` 应以一个头部注释 (header comment) 开始, 该头部注释应解释空闲块与已分配块的结构, 空闲链表的组织方式, 以及分配器如何操作空闲链表. 此外 `mm.c` 中的每个函数也应该具有一个头部注释用于解释该函数的作用以及工作原理.
- 合理性检查工具 `mm_check` 函数功能齐全并具有良好的文档.

上述两点各占 5 分.

### 实验思路与总结

#### 定义

- 假设**页大小**为 2 的幂.
- 假设**堆空间的起始地址**为 8 的倍数.
- 假设**块头部** (`size_t` 类型变量) 的大小为 4 的倍数.
- 假设**空闲链表指针** (`void *` 类型变量) 的大小为 4 的倍数.
- **块头部**的大小等于 `size_t` 类型变量的大小, 用于存储块大小以及在确保块大小必然为 4 的倍数的前提下在 (必然为零的) 低 2 位存储当前块的分配状态以及前一个块的分配状态.
- **空闲块尾部**的大小等于头部的大小, 并且当且仅当一个块空闲时设置尾部, 用于边界标记.
- 一个合法的**空闲块**由一个右对齐的头部, 中间的由若干个 8 字节区间构成的延伸区域 (因为并不是分配状态, 不存在有效载荷的概念, 所以命名为延伸区域), 以及一个左对齐的尾部共三部分构成. 延伸区域中包含用于形成双向空闲链表的两个指针, 因此延伸区域的大小至少为 8 字节.
- 一个合法的**已分配块**由一个合法的空闲块在保持块大小不变的前提下转化而来, 其中头部大小和位置不变, 延伸区域和尾部合并构成有效载荷区域.
- 一个**合法的块的大小**必然为 8 的倍数, 并且已分配块的有效载荷的起始地址也必然为 8 的倍数.
- 一个**堆**的组成部分从左到右分别为: 一个必要的首部 padding, 若干个空闲链表的头结点 (形式为空闲块), 若干个已分配块与空闲块, 以及最后的结尾块.
- 一个 **(双向) 链表** 由一个固定的头结点 (形式为空闲块) 和若干个空闲块相互连接构成.
- 采用**分离适配**的策略:
  - 将空闲块大小按 2 的幂分为不同的等价类, 最小的等价类的上界为 "不小于最小块大小的 2 的幂的最小值", 从最小的等价类开始, 每个等价类的上界不断二乘, 直至达到页大小时停止, 所有超过页大小的块统一归类至最后一个等价类中;
  - 每个等价类均与一个空闲链表对应, 该空闲链表中的每个空闲块的大小均落于该等价类中;
  - 每个空闲链表均与一个等价类对应;
- 采用**最佳适配**的策略:
  - 将每个请求块大小按照等价类进行分类, 从该请求块所属的等价类对应的空闲链表开始查找合适的块;
  - 若没有找到合适的块则转向下一个等价类;
  - 若没有在任何链表中找到合适的块则向系统请求内存空间.

#### 实现 `mm_init`, `mm_malloc` 和 `mm_free`

函数 `mm_init` 的执行逻辑:

- 对堆设置一个必要的首部 padding 执行对齐;
- 初始化链表头结点;
- 初始化结尾块.

函数 `mm_alloc` 的执行逻辑:

- 将载荷大小转换为块大小;
- 使用最佳适配的策略在空闲链表中寻找合适的块;
  - 调用函数 `void *_find_in_free_lists(size_t block_size)`
- 若没有找到合适的块则向系统请求额外的内存空间;
  - 调用函数 `void *_extend(size_t new_free_block_size)`
- 将载荷分配至所得的空闲块中.
  - 调用函数 `void *_place(size_t block_size, void *bp)`

函数 `mm_free` 的执行逻辑:

- 将载荷地址转换为块地址;
- 将已分配块格式化为空闲块;
  - 调用函数 `void _format_free_block(void *bp)`
- 尝试合并空闲块;
  - 调用函数 `void *_coalesce(void *bp)`
- 将合并得到的大空闲块插入空闲链表.
  - 调用函数 `void _insert_to_free_lists(void *bp)`

函数 `_find_in_free_lists` 的执行逻辑:

- 使用最佳适配策略, 从小到大查找每个等价类所对应的空闲链表;
- 若有合适的空闲块, 则将其从链表中摘出并返回;
  - 调用函数 `void _pick_from_free_list(void *bp)`
- 否则返回 `NULL`.

函数 `_extend` 的执行逻辑:

- 申请额外内存空间;
- 格式化新的结尾块;
- 格式化新申请的空闲块;
  - 调用函数 `void _format_free_block(void *bp)`
- 返回新空闲块 (不执行合并).

函数 `_place` 的执行逻辑:

- 检查所需块大小和传入的块大小;
- 若分割剩余的内存空间足够形成一个空闲块, 则进行分割;
  - 调用函数 `void _format_free_block(void *bp)`
  - 调用函数 `void _insert_to_free_lists(void *bp)`
- 格式化已分配块;
- 返回载荷指针.

函数 `_format_free_block` 的执行逻辑:

- 假设块的大小已经设置好;
- 设置块的分配状态为 "空闲";
- 将头部复制至尾部;
- 将本块的分配状态同步至下一个块的头部;
- 若下一个块为空闲块则还需要将该块的头部复制至尾部.

函数 `_coalesce` 的执行逻辑:

- 假设传入的块为空闲块;
- 检查后一个块是否为空闲块, 若是则返回该块;
  - 调用函数 `void *_get_next_free_block(void *bp)`
- 若后一个块为空闲块:
  - 将该块从链表中摘出;
    - 调用函数 `void _pick_from_free_list(void *bp)`
  - 合并当前块与该空闲块;
    - 调用函数 `void _coalesce_current_and_next_free_block(void *bp, void *next_bp)`
- 检查前一个块是否为空闲块, 若是则返回该块;
  - 调用函数 `void *_get_previous_free_block(void *bp)`
- 若前一个块为空闲块:
  - 将该块从链表中摘出;
    - 调用函数 `void _pick_from_free_list(void *bp)`
  - 合并当前块与该空闲块;
    - 调用函数 `void _coalesce_current_and_next_free_block(void *bp, void *next_bp)`
- 返回合并得到的大空闲块.

#### 实现 `mm_check`

函数 `mm_check` 的执行逻辑:

- 检查堆的格式是否正确, 并返回空闲块数量;
  - 调用函数 `int _check_heap_format(void)`
- 检查空闲链表的格式是否正确, 并返回空闲块数量;
  - 调用函数 `int _check_free_lists(void)`
- 检查堆中空闲块是否和空闲链表中的块一致;
  - 调用函数 `int _check_free_list_and_heap_consistency(size_t heap_free_block_number, size_t free_list_block_number)`
- 若一切正常则返回非零值, 否则返回零.

函数 `_check_heap_format` 的执行逻辑:

- 跳过所有链表头结点;
- 对遍历到的每一个块:
  - 是否连续两个块都是空闲块, 若是则返回 -1;
  - 检查当前块的头部中记录的前一个块的分配状态是否与前一个块的真实分配状态一致, 若不一致则返回 -1;
  - 如果是空闲块则检查头部和尾部是否相同, 若不相同则返回 -1.
- 返回空闲块数量 (非负数).

函数 `_check_free_lists` 的执行逻辑:

- 遍历每个空闲链表;
- 对每个结点:
  - 检查该结点是否为空闲块, 若不是则返回 -1;
  - 检查该结点所连接的前一个结点是否为真实的前一个结点, 若不是则返回 -1;
- 返回空闲块数量 (非负数).

函数 `_check_free_list_and_heap_consistency` 的执行逻辑:

- 收集堆中所有空闲块的地址;
- 收集空闲链表中所有块的地址;
- 对两个地址数组进行排序;
- 检查两个集合是否一致, 若一致则返回零, 否则返回非零值.

#### DEBUG

##### `mm_free` 中忘记接住合并函数 `_coalesce` 的返回值

在出错的最后一次释放操作之前的空闲链表情况如下 (方括号代表即将从链表中摘出并参与合并的前后两个空闲块):

```text
空闲块数量: 9
链表内部情况:
16
32
64
128
256
512
1024
2048
4096 -> 0xf694bc94   -> 0xf6958bc4   -> 0xf695db74   -> 0xf6962b24   -> 0xf696fa54   -> [0xf69779d4]   -> 0xf696aaa4
∞   -> [0xf69799b4]   -> 0xf6980944
```

当前即将释放的块的地址: 0xf69789c4, 块大小: 4080

当前块的后一个块为空闲块, 块地址: 0xf69799b4, 该空闲块在链表中的前一个块地址: 0xf68ab0a4, 该空闲块在链表中的后一个块地址: 0xf6980944
当前块的前一个块为空闲块, 块地址: 0xf69779d4, 该空闲块在链表中的前一个块地址: 0xf696fa54, 该空闲块在链表中的后一个块地址: 0xf696aaa4

释放块之后的空闲链表情况 (圆括号代表合并完成并插入链表的新空闲块):

```text
空闲块数量: 8
链表内部情况:
16
32
64
128
256
512
1024
2048
4096 -> 0xf694bc94   -> 0xf6958bc4   -> 0xf695db74   -> 0xf6962b24   -> 0xf696fa54   -> 0xf696aaa4
∞   -> (0xf69789c4)   -> 0xf6980944
```

可以看到本应是地址为 `0xf69779d4` 的新空闲块被插入, 结果最后插入的是原来的被释放的块的地址 `0xf69789c4`. 这说明是函数 `_coalesce` 或是 `mm_free` 本身出了问题. 经过检查发现是因为函数 `_coalesce` 的逻辑是对所要释放的块进行合并, 然后返回**合并得到的块的地址**, 结果函数 `mm_free` 中忘了接住合并得到的块的地址,

```c
// 尝试合并空闲块:
_coalesce(bp);  // 忘记接住 "合并得到的块" 的地址
```

导致最后插入的仍旧是所要释放的块的地址. 将其修改为

```c
// 尝试合并空闲块:
bp = _coalesce(bp);
```

消除了 BUG.

##### 放置函数 `_place` 中对不分割的情况的处理存在错误

最后一次 (函数 `mm_malloc` 中的) 放置函数的运行情况如下:

```text
[mm_malloc]
请求载荷大小: 24
所需块大小: 32
[_find_in_free_lists]
当前等价类上界: 32
当前等价类上界: 64
[_pick_from_free_list]
当前块地址: 0xf69fc3bc
前一个块地址: 0xf68bb034
后一个块地址: 0xf699e834
成功将前块连接至后块
成功将后块连接至前块
已找到空闲块, 块地址: 0xf69fc3bc
[_place]
所需的块大小: 32
传入的块大小: 40
块地址: 0xf69fc3bc
Segmentation fault
```

第一反应是像 `_place` 这样的简单函数不应该出现错误, 肯定是自己哪里粗心大意写错了十分明显的逻辑. 检查之后发现是错误处理了不分割情况下传入的块的大小仍然大于所需的块大小时真正的块的大小. 例如在上述运行情况中, 所需的块大小为 32 字节, 而传入的块大小为 40 字节, 由于 8 字节的多余空间不足以分割, 这 8 字节将会以内部碎片的方式保留在最后分配的块中, 因此真正的块大小应为 40 字节. 但程序中忘记了这一点, 因此导致最后将所分配的块大小设置为 32 字节 (然而实际上该块仍然长 40 字节). 于是在函数 `_place` 返回之后在检查程序 `mm_check` 中尝试遍历堆时由于错误的指针运算而报错.

最后在程序中稍加修改消除了 BUG.

##### 忘记在检查程序 `mm_check` 中释放内存

之所以能够发现这个 BUG 是因为程序 `./mdriver` 在指定使用所有 trace 文件并且运行了非常久之后报了段错误, 而单拎出来运行一个 trace 文件则一切正常, 并且还有个异常现象, 那就是吞吐量巨低. 运行时间久说明必然是 `printf` 或者 `mm_check` 忘记关闭了, 单拎出来运行一个 trace 文件一切正常说明程序本身是没有问题的, 而最后依然报了段错误那就只能是内存泄漏了. 经过检查发现是函数 `mm_check` 中在检查堆中的空闲块集合与链表中的空闲块集合的一致性时使用 `malloc` 动态请求了内存但忘了在检查完成之后释放.

#### 优化 `mm_malloc` 和 `mm_free`

##### 测试样例 `binary-bal.rep` 和 `binary2-bal.rep` 的内存利用率过低

测试时发现其他的都挺好, 就这两个测试样例 `binary-bal.rep` 和 `binary2-bal.rep` 的内存利用率非常低, 仅为 55% 左右.

通过查看 `binary-bal.rep` 的内容发现其请求模式 (即连续 2000 次一前一后地请求有效载荷长度分别为 `64` 字节和 `448` 字节的块, 然后释放所有有效载荷长度为 `448` 字节的块, 最后再连续请求 2000 个有效载荷长度为 `512` 字节的块) 倾向于生成外部碎片. 在最简单的线性分配的策略 (每次请求均向系统申请相等大小的内存空间) 下, 这样的请求模式会造成内存利用率降至 (64 + 512) / (512 + 512) = 56.25% 左右.

由于当前的分配器实现方案采用的是 "申请后分配前执行合并" 的策略, 之前的操作中剩下的空闲块在本次分配前被合并到所申请的新的空闲块中, 导致本次分配起始地址向前移动到了前一个分配块的末尾地址, 即当前的分配器实现方案在 `binary-bal.rep` 的请求模式下等价于简单的线性分配策略. 实际上测试得到的内存利用率也位于上述理论值的附近 (加上头部以及对齐造成的额外空间分配, 实际的内存利用率约为 55%).

简单注释掉函数 `_extend` 末尾调用函数 `_coalesce` 的语句, 再次运行测试发现 `binary-bal.rep` 的内存利用率提升到了 73%. 这是因为当前分配器采用的是 "对于小于等于页大小的块, 为其分配两倍于其大小的空间, 而对于超过页大小的块则分配与其大小相等的空间" 的分配方案, 同时由于取消了 "分配前合并" 的措施, 每个 64 字节的块均位于一个 128 字节的两倍的空间中, 两两配对并均分这个 128 字节的空间, 而每个 448 字节的块同样均位于一个 896 字节的两倍的空间中并且两两均分该空间. 这样释放掉所有 448 字节的块之后就剩下了数量为 448 字节快的一半的 896 字节的空闲块. 等到 512 字节的块的申请到来时前面一半的申请落在这些 896 字节的块中, 后面一半的申请则由于外部碎片的缘故只能新开内存. 最后的理论内存利用率 (不考虑头部以及对齐所占用的额外空间) 约为

$$
\begin{equation}
\frac{(64 + 64 + 512) \times 500 + 512 \times 500}{(64 + 64 + 448 + 448) \times 500 + 512 \times 500} = 0.75.
\end{equation}
$$

这一结果与实际观察到的 73% 的内存利用率相吻合. 但是 73% 的内存利用率还是不够好, 堆中仍然存在 500 个长度为 896 - 512 = 384 的外部碎片无法被用于分配.

之所以在 `binary-bal.rep` 的请求模式下会产生大量外部碎片, 根本原因还是未来的分配和释放请求的不可知性, 或者反过来说是分配器策略的先验性. 无论分配器的实现如何, 一个不怀好意的人总是能够找到一个最大化外部碎片并最小化内存利用率的请求序列. 外部碎片问题无法解决, 只能通过寻找一个好的分配器策略最小化最大外部碎片率 (即寻找 min-max 问题的一个解).

根据前面的论述, 实际上可以通过微调分配器中的如下代码中的超参求解针对当前给出的 trace file 的最佳参数:

```c
void *mm_malloc(size_t size) {
    // ...
        // 对于不超过页面大小的块, 一次性请求一定倍数大小的空间, 对于大于页面大小的块则按原样大小请求:
        size_t allocated_block_size = block_size <= PAGE_SIZE ? (<hyper-parameter> * block_size) : ALIGN(block_size);
    // ...
}
```

通过命令行 `./run.sh > results/result_extend_no_coalesce_adapt_456_to_520.txt` 对不同超参配置下的分配器进行评测得到如下表格:

| hyper-parameter | binary-bal.rep | binary2-bal.rep |
| --------------- | -------------- | --------------- |
| 2               | 73%            | 68%             |
| 3               | 82%            | 76%             |
| 4               | 88%            | 81%             |
| 5               | 91%            | 84%             |
| 6               | 94%            | 51%             |
| 7               | 95%            | 51%             |
| 8               | 53%            | 55%             |
| 9               | 63%            | 58%             |

其中超参从 5 到 6 时测试样例 `binary2-bal.rep` 的结果发生突变, 这是因为 `binary2-bal.rep` 中采用的是 (16, 112) 的组合, 其中 16 字节的载荷生成 24 字节的块大小, 112 字节的载荷生成 120 字节的块大小, 而超参等于 6 时, 为 24 字节的块多分配的 5 个块恰好组成一个 120 字节的空闲块, 从而造成该空闲块重新进入 112 字节载荷的搜索范围中 (与一开始的 "申请后分配前执行合并" 的效果等价), 导致最终堆内的分配布局又回到了大小块交叉而外部碎片巨多的情况. 在超参从 7 到 8 时 `binary-bal.rep` 结果的突变同理.

对上述超参进行微调的意义是找到一个整数 a, 使得外部碎片 (448 * a) % 512 最小. 按照这个想法如果将 a 的选择范围拓展至实数轴效果肯定会更好, 但实际上不行, 因为调整 a 的比例不仅会影响 448 的块的分配, 还会影响 64 的块的分配, 而后者同样会产生外部碎片.

最终选择将超参设置为 5.

#### 实现 `mm_realloc`

实现 `mm_realloc` 的初始方案是直接调用 `mm_alloc` 和 `mm_free`, 代码如下:

```c
void *mm_realloc(void *ptr, size_t size) {
    // 分配新的块:
    void *new_ptr = mm_malloc(size);

    // 复制内容:
    memcpy(new_ptr, ptr, size);

    // 释放原有块:
    mm_free(ptr);

    // 返回新的块:
    return new_ptr;
}
```

这么做除了性能低下 (最后两个测试样例的内存利用率分别为 28% 和 26%) 以外其他都挺好. 之所以原始实现的性能不好是因为每次重分配都固定进行空间分配以及数据迁移. 实际上如果重分配的大小比原大小来得小, 那么完全可以在原已分配块的基础上进行原地收缩, 同理如果原分配块之后存在空闲空间那么完全有可能进行原地拓展. 而由于数据迁移永远是一个 $O(n)$ 的操作, 因此应当尽可能降低数据迁移的次数. 围绕 "尽可能减少数据的复制" 的观点对 `mm_realloc` 进行了独立设计.

函数 `mm_realloc` 的执行逻辑:

- 获取旧分配块的大小, 并通过新的载荷大小计算新的分配块的大小;
- 若新旧分配块大小相等则直接返回;
- 若旧分配块需要收缩:
  - 若旧分配块的下一块为已分配块:
    - 若旧分配块收缩空出的内存空间不足以形成新的空闲块:
      - 以内部碎片的形式保留在新分配块中并返回.
    - 否则分割.
  - 若旧分配块的下一块为空闲块:
    - 分割, 并将空出的内存空间纳入下一块中.
- 若旧分配块需要拓展:
  - 若旧分配块的下一块为已分配块, 或者旧分配块的下一块为空闲块但不足以容纳拓展的空间:
    - 调用 `mm_alloc` 重新分配空间.
  - 若旧分配块的下一块是空闲块并且能够容纳拓展的空间:
    - 如果拓展之后原空闲块剩下的空间不足以形成新的空闲块:
      - 以内部碎片的形式保留在新分配块中.
    - 否则分割.

#### 优化 `mm_realloc`

实现之后发现两个测试用例的结果还是改进前一样糟糕. 观察发现分配器在 `realloc-bal.rep` 的请求模式下会在结尾块之前维护一个不断 realloc 的块, 每当这个块拓展到结尾块时便会触发一次重分配以及数据迁移, 但显然此时完全可以直接通过分配空间来原地拓展 (因为结尾块同一般已分配块不一样, 并非不可移动的). 解决办法是额外添加针对恰好位于结尾块之前的块的重分配机制. 除了添加该机制以外, 这一版本为了尝试提高内存利用率还将放置策略从首次适配更改为最佳适配. 更改后的函数 `mm_realloc` 的执行策略:

- ...
- 若旧分配块需要拓展:
  - 若旧分配块的下一块为已分配块, 或者旧分配块的下一块为空闲块但不足以容纳拓展的空间:
    - 若下一块虽然为已分配块但是为结尾块:
      - 直接拓展堆.
    - 若下一块空闲块虽然不足以容纳拓展的空间, 但下下一块为结尾块:
      - 拓展堆, 然后合并空闲块.
    - 其余情况:
      - 调用 `mm_alloc` 重新分配空间.
  - 若旧分配块的下一块是空闲块并且能够容纳拓展的空间:
    - ...

测试之后发现测试样例 `random-bal.rep` 和 `random2-bal.rep` 的结果都得到了改进, 从原来的 89% 和 86% 分别提高到了 95% 和 95%, `realloc-bal.rep` 的结果也提高到了 98%, 但奇怪的是测试用例 `realloc2-bal.rep` 的测试效果还是没得到改进. `realloc2-bal.rep` 和 `random-bal.rep` 的模式类似, 都会在结尾块前维护一个不断拓展的已分配块. 尽管已经添加了直接移动结尾块进行堆拓展的机制, 但观察发现每当结尾块前的已分配块即将拓展至结尾块时就会被一个 16 字节的小块占据住空间使其无法拓展并触发重分配. 但问题是为什么之前的测试样例 `realloc-bal.rep` 没有出现同样的问题呢? 再次观察发现与 `realloc-bal.rep` 相比, `realloc2-bal.rep` 的模式是预先分配的块较大 (4092 字节), 而 realloc 的步进较小 (5 字节为单位), 因此在 realloc 不断 "吞噬" 空闲块的时候产生的一系列大小逐渐递减空闲块序列的粒度也更小, 只要空闲块递减至位于 16 字节的小块的最佳适配范围内就会被其占据; 而对于 `realloc-bal.rep`, 其空闲块的递减粒度很大, 恰好错开了 128 字节的块的最佳适配范围的窗口期, 因此不会被其所占据.

解决方法和优化 `mm_alloc` 和 `mm_free` 时类似, 都是面向测试样例优化, 具体是将分配策略 "对小于等于页大小的块分配 5 倍空间, 对于超过页大小的块按原样分配空间" 修改为 "对小于页大小的块分配 5 倍空间, 对于大于等于页大小的块按原样分配空间", 也即单单修改了针对大小为 4096 字节的初始块的分配策略, 这样就确保了一开始就没有外部碎片, 也就不会出现外部碎片被不断侵蚀而逐渐减小并在减小到一定程度的时候被较小的块提前找到并占据的情况.

### 实验结果展示

```text
Results for mm malloc:
trace  valid  util     ops      secs  Kops
 0       yes   99%    5694  0.000151 37684
 1       yes   99%    5848  0.000157 37177
 2       yes   99%    6648  0.000183 36388
 3       yes   99%    5380  0.000145 37129
 4       yes   98%   14400  0.000254 56738
 5       yes   95%    4800  0.000693  6923
 6       yes   95%    4800  0.000701  6846
 7       yes   91%   12000  0.000553 21692
 8       yes   84%   24000  0.001509 15906
 9       yes   98%   14401  0.000200 72005
10       yes   87%   14401  0.000191 75437
Total          95%  112372  0.004737 23721

Perf index = 57 (util) + 40 (thru) = 97/100
```

### 相关资料

- 完整的 trace 文件:

  - [patlewis/malloc-lab](https://github.com/patlewis/malloc-lab)
  - [Fanziyang-v/CSAPP-Lab](https://github.com/Fanziyang-v/CSAPP-Lab)
  - [jon-whit/malloc-lab](https://github.com/jon-whit/malloc-lab)

  官方的 handout tarball 里面并没有包含 `config.h` 中提到的 11 个 trace 文件, 上述 repo 中的 trace 文件仅供参考.

- C 语言语法摘要 (K&R)
  - P120: `sizeof` 运算符
    - `sizeof` 是一个编译时一元运算符, 用于返回对象或类型的大小, 可以作用于对象 (`sizeof object`) 和类型 (`sizeof(type_name)`), 前者不需要括号, 后者需要.
    - `sizeof` 的返回值的类型为 `size_t`, 该类型定义于头文件 `<stddef.h>`.
    - 一个 `sizeof` 不能用于 `#if` 行, 因为 `#if` 行是由预处理器解析的, 而预处理器并不解析类型名.
    - 一个 `sizeof` **可以**用于 `#define` 行, 因为 `#define` 行并不是由预处理器处理的.
  - P31: 外部变量
    - 外部变量是定义于所有函数的外部的变量.
    - 外部变量必须被定义, 必须在任何函数外定义, 并且只能被定义一次. 这为该外部变量创建了存储空间.
    - 任何想要使用某个外部变量的函数必须在其函数体中声明该外部变量. 这说明了该外部变量的类型.
    - 如果外部变量定义在想要使用它的函数之前, 那么函数中可以不使用 `explicit` 关键字声明; 否则函数必须使用 `explicit` 关键字声明该外部变量.
    - `explicit` 声明可以在函数体中, 也可以移至函数体外.
    - 一般做法是在一个文件中存放对外部变量的 `explicit` 声明 (这类文件有个约定的名称, 即**头文件**, 头文件的后缀名约定为 `.h`), 在另一个与之对应的源文件中定义外部变量, 然后在余下的其他所有源文件中通过 `#include` 将头文件包括进来.
  - P76: 初始化
    - 如果不存在任何显式定义/初始化, 外部变量和静态变量会默认初始化为 0; 自动变量和寄存器变量的值则是未定义的.
    - 外部变量和静态变量的初始化器必须是常量表达式.
    - 标量变量完全可以在声明的时候顺手被初始化, 例如可以写成 `int i = 0;` 而不是 `int i; i = 0;`.
    - 对自动变量的初始化其实只是对赋值语句的简写, 选择哪种风格完全是个人口味的问题. K&R 中一般使用赋值语句而不是显式初始化, 因为他们认为对变量在声明时初始化离用这个变量的地方太远, 找起来不太方便.
