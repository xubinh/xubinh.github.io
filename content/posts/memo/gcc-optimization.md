---
title: "探究 GCC 的 `-O` 选项的优化细节——以内存池为例"
description: "揭示编译器如何弥合高层抽象与底层硬件之间的差距"
summary: "揭示编译器如何弥合高层抽象与底层硬件之间的差距"
date: 2025-01-11T17:46:23+08:00
draft: false
tags: ["C++", "assembly", "GCC", "optimization"]
series: ["C++", "assembly", "GCC", "optimization"]
author: ["xubinh"]
type: posts
---

## 引言

众所周知为了避免频繁调用 `malloc` 带来的消耗, 为特定内存分配场景编写量身定制的内存池几乎是不可避免的. 我最近恰好也实现了一个内存池. 这个内存池所要解决的问题是小型字符串缓冲区的频繁分配和回收, 其实现实际上就是一个简单的块分配器. 但就是这样一个简单的内存池, 在与标准库的 `malloc` 进行比较的时候却发现仍然无法打败标准库的实现, 这个结果完全出乎我的意料. 在进一步尝试了一些优化之后, 我发现如果在编译时加入 `-O1` 优化选项, 那么内存池的执行效率就会火箭式上升. 为了探究这种令人生厌的不一致性的根源, 我决定直接查看汇编代码来理解 `-O1` 选项究竟在底层做了些什么.

## 源码

首先给出内存池的源码, 其中只展示了 `allocate` 及其辅助函数 (`deallocate` 函数同理可知, 此处暂时忽略).

```cpp
class StaticSimpleThreadLocalStringSlabAllocator {
private:
    struct LinkedListNode {
        LinkedListNode *next;
    };

    struct ChunkManager {
        ChunkManager() noexcept = default;

        ChunkManager(const ChunkManager &) = delete;
        ChunkManager &operator=(const ChunkManager &) = delete;

        ~ChunkManager() noexcept {
            for (auto chunk : _allocated_chunks) {
                ::free(chunk);
            }
        }

        void push_back(void *chunk) {
            _allocated_chunks.push_back(chunk);
        }

        std::vector<void *> _allocated_chunks;
    };

public:
    char *allocate(size_t n);

    void deallocate(char *slab, size_t n); // 定义略

private:
    LinkedListNode *_allocate_one_chunk(int power_of_2) noexcept;

    static thread_local LinkedListNode
        *_linked_lists_of_free_slabs[12]; // 2^11 = 2048
    static thread_local ChunkManager _allocated_chunks;
};

char *StaticSimpleThreadLocalStringSlabAllocator::allocate(size_t n) {
    if (n == 0) {
        throw std::bad_alloc();
    }

    if (n > 2048) {
        return reinterpret_cast<char *>(::malloc(n)); // no alignment
    }

    auto power_of_2 =
        alignment::get_next_power_of_2(std::max(n, static_cast<size_t>(32)));

    auto chosen_slab = _linked_lists_of_free_slabs[power_of_2];

    if (!chosen_slab) {
        chosen_slab = _allocate_one_chunk(power_of_2);
    }

    _linked_lists_of_free_slabs[power_of_2] = chosen_slab->next;

    return reinterpret_cast<char *>(chosen_slab);
}

StaticSimpleThreadLocalStringSlabAllocator::LinkedListNode *
StaticSimpleThreadLocalStringSlabAllocator::_allocate_one_chunk(int power_of_2
) noexcept {
    size_t slab_size = (1 << power_of_2);

    // (power_of_2, number_of_slabs) ~ (5, 2^(12-5)), ..., (11, 2^(12-11)) ==>
    // number_of_slabs = 2^(12-power_of_2)
    size_t number_of_slabs = (0x00001000 >> power_of_2);

    void *new_chunk = alignment::aalloc(4096, 4096);

    _allocated_chunks.push_back(new_chunk);

    LinkedListNode *current_node = nullptr;
    LinkedListNode *next_node = reinterpret_cast<LinkedListNode *>(new_chunk);

    for (size_t i = 0; i < number_of_slabs; i++) {
        next_node->next = current_node;

        current_node = next_node;

        next_node = reinterpret_cast<LinkedListNode *>(
            reinterpret_cast<char *>(next_node) + slab_size
        );
    }

    return current_node;
}
```

可以看到这就是一个简单的块分配器:

- 所有分配出去的块 (slab) 的大小均为 2 的幂;
- 最小的块大小为 32 字节 (因为 `std::basic_string` 采用了 Small String Optimization (SSO), 初始的 16 字节 (实际上是 15 字节外加结尾的 `\0` 字符) 是直接在栈中进行分配的), 最大的块大小为 2048 字节;
- 所有块均是通过对原始分配得到的区块 (chunk) 进行切割得到的, 区块的大小固定为 4096 字节 (即绝大部分 Linux 系统中一个虚拟内存页面的大小);
- 使用单独的链表对所有大小相同的空闲块进行组织, 例如所有 32 字节的块挂在第一个链表下, 所有 64 字节的块挂在第二个链表下, 以此类推;
- 分配的时候从空闲链表头摘出去一块作为所要分配的块, 如果空闲链表为空则调用辅助方法 `_allocate_one_chunk` 向系统申请一个新的区块并补充空闲链表 (回收的情况同理).

正常来讲如此简单的实现理应在一开始提到的问题场景下轻松击败通用的 `malloc`, 但接下来我却发现真实情况并非如此.

## 基准测试

在将这个内存池应用到我的网络框架中后, 我发现框架的运行效率没有得到提升, 反而还下降了不少. 这一点引起了我的怀疑. 为了单独比较有内存池时和没有内存池时执行效率的差异, 我又编写了一段测试代码, 具体而言就是连续创建 10000 个 1024 字节长的字符串, 然后销毁, 重复 1000 次:

```cpp
using StringType = xubinh_server::util::string;
// using StringType = std::string;

for (int i = 0; i < 1000; i++) {
    std::vector<StringType> v;

    for (int j = 0; j < 10000; j++) {
        StringType s;

        s.resize(1023);

        v.push_back(std::move(s));
    }
}
```

我首先使用了一般的不带任何优化的配置进行编译, 基准测试结果如下表的第二列所示.

| 字符串类型                    | -O0 执行效率 | -O1 执行效率 | 加速比     |
| ----------------------------- | ------------ | ------------ | ---------- |
| `xubinh_server::util::string` | 6.131 秒     | 1.328 秒     | **4.6 倍** |
| `std::string`                 | 5.864 秒     | 5.005 秒     | 1.2 倍     |

可以看到自定义内存池的执行效率反而还要比标准库的低 (这也难怪我的网络框架的执行效率也跟着变低), 这是很反常的一件事——如果说 `malloc` 为了兼顾通用性而牺牲了一些效率, 那么我的内存池在专精一个场景的情况下理应在效率上得到质的飞跃. 正当我沮丧地想要接受这一现实的时候, 我误打误撞地打开了编译器的 `-O` 选项, 此时的基准测试结果如上表的第三列所示. 可以看到自定义内存池的执行效率真的得到了飞跃, 并且不是一倍两倍, 而是近乎 5 倍的提升, 这也让内存池顺利击败了标准库的 `malloc` 实现. 可问题是凭什么不开 `-O1` 的时候应有的提升就消失了呢?

## 汇编代码

### `allocate`

为了理解这种优化差异, 我直接查看了内存池的汇编代码并对它们在 `-O0` 和 `-O1` 情况下的版本进行了比较. 首先来看负责分配的函数 `allocate` (已对汇编代码做了详细注释).

未优化的版本:

```assembly
0x0000000000003962 <+0>:     endbr64

# 进入栈帧前的准备工作
0x0000000000003966 <+4>:     push   %rbp
0x0000000000003967 <+5>:     mov    %rsp,%rbp
0x000000000000396a <+8>:     push   %rbx
0x000000000000396b <+9>:     sub    $0x38,%rsp
0x000000000000396f <+13>:    mov    %rdi,-0x38(%rbp) # `this` 指针
0x0000000000003973 <+17>:    mov    %rsi,-0x40(%rbp) # 大小 `n`
0x0000000000003977 <+21>:    mov    %fs:0x28,%rax    # 金丝雀值 (canary values) 用于
                                                     # 检测栈溢出, 保护栈帧
0x0000000000003980 <+30>:    mov    %rax,-0x18(%rbp)

# if (n == 0) ...
0x0000000000003984 <+34>:    xor    %eax,%eax
0x0000000000003986 <+36>:    mov    -0x40(%rbp),%rax
0x000000000000398a <+40>:    test   %rax,%rax
0x000000000000398d <+43>:    sete   %al
0x0000000000003990 <+46>:    movzbl %al,%eax
0x0000000000003993 <+49>:    test   %rax,%rax
0x0000000000003996 <+52>:    je     0x39c9 <+103>
0x0000000000003998 <+54>:    mov    $0x8,%edi
0x000000000000399d <+59>:    call   0x2400 <__cxa_allocate_exception@plt>
0x00000000000039a2 <+64>:    mov    %rax,%rbx
0x00000000000039a5 <+67>:    mov    %rbx,%rdi
0x00000000000039a8 <+70>:    call   0x3c86 <_ZNSt9bad_allocC2Ev>
0x00000000000039ad <+75>:    mov    0xe61c(%rip),%rax
0x00000000000039b4 <+82>:    mov    %rax,%rdx
0x00000000000039b7 <+85>:    lea    0xe22a(%rip),%rax
0x00000000000039be <+92>:    mov    %rax,%rsi
0x00000000000039c1 <+95>:    mov    %rbx,%rdi
0x00000000000039c4 <+98>:    call   0x2620 <__cxa_throw@plt>

# if (n > 2048) ...
0x00000000000039c9 <+103>:   mov    -0x40(%rbp),%rax
0x00000000000039cd <+107>:   cmp    $0x800,%rax
0x00000000000039d3 <+113>:   jbe    0x39e3 <+129>
0x00000000000039d5 <+115>:   mov    -0x40(%rbp),%rax
0x00000000000039d9 <+119>:   mov    %rax,%rdi
0x00000000000039dc <+122>:   call   0x25b0 <malloc@plt>
0x00000000000039e1 <+127>:   jmp    0x3a55 <+243>

# auto power_of_2 = ...
0x00000000000039e3 <+129>:   movq   $0x20,-0x30(%rbp) # 将字面量 `32` 放入内存中 (无意义)
0x00000000000039eb <+137>:   lea    -0x30(%rbp),%rdx  # 字面量 `32` 的引用 (地址)
0x00000000000039ef <+141>:   lea    -0x40(%rbp),%rax  # 大小 `n` 的引用 (地址)
0x00000000000039f3 <+145>:   mov    %rdx,%rsi         # 重复 (无意义)
0x00000000000039f6 <+148>:   mov    %rax,%rdi         # 重复 (无意义)
0x00000000000039f9 <+151>:   call   0x31f5            # 调用 `std::max` 函数
0x00000000000039fe <+156>:   mov    (%rax),%rax
0x0000000000003a01 <+159>:   mov    %rax,%rdi
0x0000000000003a04 <+162>:   call   0x3f4b
0x0000000000003a09 <+167>:   mov    %rax,-0x20(%rbp)  # 将 `power_of_2` 存入内存

# auto chosen_slab = ...
0x0000000000003a0d <+171>:   call   0x4089              # 调用 TLS 的 wrapper 函数
0x0000000000003a12 <+176>:   mov    -0x20(%rbp),%rdx    # 获取 `power_of_2`
0x0000000000003a16 <+180>:   mov    (%rax,%rdx,8),%rax  # 计算 `chosen_slab`
0x0000000000003a1a <+184>:   mov    %rax,-0x28(%rbp)    # 将 `chosen_slab` 存入内存

# if (!chosen_slab) ...
0x0000000000003a1e <+188>:   cmpq   $0x0,-0x28(%rbp)
0x0000000000003a23 <+193>:   jne    0x3a3d <+219>
0x0000000000003a25 <+195>:   mov    -0x20(%rbp),%rax    # 获取 `power_of_2`
0x0000000000003a29 <+199>:   mov    %eax,%edx           # 重复 (无意义)
0x0000000000003a2b <+201>:   mov    -0x38(%rbp),%rax    # 获取 `this` 指针
0x0000000000003a2f <+205>:   mov    %edx,%esi           # 重复 (无意义)
0x0000000000003a31 <+207>:   mov    %rax,%rdi           # 重复 (无意义)
0x0000000000003a34 <+210>:   call   0x3b56
0x0000000000003a39 <+215>:   mov    %rax,-0x28(%rbp)    # 将结果存入内存 (无意义)
0x0000000000003a3d <+219>:   mov    -0x28(%rbp),%rax    # 再将结果从内存中取出来 (无意义)
0x0000000000003a41 <+223>:   mov    (%rax),%rbx         # 即 `chosen_slab->next`
0x0000000000003a44 <+226>:   call   0x4089              # 再次获取 TLS 链表头
0x0000000000003a49 <+231>:   mov    -0x20(%rbp),%rdx    # 获取 `power_of_2`
0x0000000000003a4d <+235>:   mov    %rbx,(%rax,%rdx,8)  # 更新链表头

# 退出栈帧前的准备工作
0x0000000000003a51 <+239>:   mov    -0x28(%rbp),%rax
0x0000000000003a55 <+243>:   mov    -0x18(%rbp),%rdx
0x0000000000003a59 <+247>:   sub    %fs:0x28,%rdx       # 检查金丝雀值是否发生改变
0x0000000000003a62 <+256>:   je     0x3a69 <+263>
0x0000000000003a64 <+258>:   call   0x2570 <__stack_chk_fail@plt>
0x0000000000003a69 <+263>:   mov    -0x8(%rbp),%rbx
0x0000000000003a6d <+267>:   leave
0x0000000000003a6e <+268>:   ret
```

可以看到未做优化的版本的汇编代码显得冗长且低效. 总结起来有如下几点:

- 栈帧的初始化工作过于繁琐: 每次进入 `allocate` 函数时都要新建一次栈帧. ❌
- 过度依赖栈内存: 像 `power_of_2` 和 `chosen_slab` 这样的临时变量被反复写入到内存中再从内存中读取出来. ❌
- 冗余的移动指令: 同一个变量被一条指令移出去后又被紧接着的下一条指令移回来. ❌
- 冗余的函数调用: 诸如 `std::max` 和 `alignment::get_next_power_of_2` 等简单的函数仍然通过调用的方式进行使用, 增加不必要的开销. ❌
- 线程本地变量的获取: 获取线程本地变量也需要通过函数调用实现, 进一步增加开销. ❌

然后是 `-O1` 优化的版本:

```assembly
0x0000000000002c16 <+0>:     endbr64

# 进入栈帧前的准备工作
0x0000000000002c1a <+4>:     push   %rbx

# if (n == 0) ...
0x0000000000002c1b <+5>:     test   %rsi,%rsi
0x0000000000002c1e <+8>:     je     0x2c64 <+78>

# if (n > 2048) ...
0x0000000000002c20 <+10>:    cmp    $0x800,%rsi
0x0000000000002c27 <+17>:    ja     0x2c8e <+120>

# auto power_of_2 = ...
0x0000000000002c29 <+19>:    mov    $0x20,%eax
0x0000000000002c2e <+24>:    cmp    %rax,%rsi
0x0000000000002c31 <+27>:    cmovae %rsi,%rax
0x0000000000002c35 <+31>:    sub    $0x1,%eax
0x0000000000002c38 <+34>:    bsr    %eax,%eax
0x0000000000002c3b <+37>:    xor    $0x1f,%eax
0x0000000000002c3e <+40>:    mov    $0x20,%esi
0x0000000000002c43 <+45>:    sub    %eax,%esi
0x0000000000002c45 <+47>:    movslq %esi,%rbx

# if (!chosen_slab) ...
0x0000000000002c48 <+50>:    mov    %fs:-0x260(,%rbx,8),%rax   # 直接读取 TLS 链表头
0x0000000000002c51 <+59>:    test   %rax,%rax
0x0000000000002c54 <+62>:    je     0x2c98 <+130>              # 如果为空指针则跳转
0x0000000000002c56 <+64>:    mov    (%rax),%rdx
0x0000000000002c59 <+67>:    mov    %rdx,%fs:-0x260(,%rbx,8)   # 直接写入 TLS 链表头

# 退出栈帧前的准备工作
0x0000000000002c62 <+76>:    pop    %rbx
0x0000000000002c63 <+77>:    ret

# `throw std::bad_alloc();` 部分入口
0x0000000000002c64 <+78>:    mov    $0x8,%edi
0x0000000000002c69 <+83>:    call   0x2300 <__cxa_allocate_exception@plt>
0x0000000000002c6e <+88>:    mov    %rax,%rdi
0x0000000000002c71 <+91>:    lea    0x6fc8(%rip),%rax
0x0000000000002c78 <+98>:    mov    %rax,(%rdi)
0x0000000000002c7b <+101>:   mov    0x7346(%rip),%rdx
0x0000000000002c82 <+108>:   lea    0x6fcf(%rip),%rsi
0x0000000000002c89 <+115>:   call   0x2490 <__cxa_throw@plt>

# `malloc` 函数入口
0x0000000000002c8e <+120>:   mov    %rsi,%rdi
0x0000000000002c91 <+123>:   call   0x2420 <malloc@plt>
0x0000000000002c96 <+128>:   jmp    0x2c62 <+76>

# `_allocate_one_chunk` 函数入口
0x0000000000002c98 <+130>:   call   0x2b36
0x0000000000002c9d <+135>:   jmp    0x2c56 <+64>
```

仅仅经过简单的 `-O1` 优化, 汇编代码的质量便发生了巨大变化:

- 取消了栈帧的准备工作. ✔️
- 对临时变量的存储位置进行了内联. ✔️
- 消除了冗余的指令, 变得精确且高效. ✔️
- 对诸如 `std::max` 和 `alignment::get_next_power_of_2` 等简单的函数进行了内联化处理. ✔️
- 对线程本地变量的获取进行了内联化处理. ✔️

### `_allocate_one_chunk`

除了 `allocate`, 我们还可以从其辅助函数 `_allocate_one_chunk` 中进一步了解优化细节.

未优化的版本:

```assembly
0x0000000000003b56 <+0>:     endbr64

# 进入栈帧前的准备工作
0x0000000000003b5a <+4>:     push   %rbp
0x0000000000003b5b <+5>:     mov    %rsp,%rbp
0x0000000000003b5e <+8>:     sub    $0x40,%rsp
0x0000000000003b62 <+12>:    mov    %rdi,-0x38(%rbp) # `this` 指针
0x0000000000003b66 <+16>:    mov    %esi,-0x3c(%rbp) # `power_of_2`

# `slab_size = (1 << power_of_2)`
0x0000000000003b69 <+19>:    mov    -0x3c(%rbp),%eax # 获取 `power_of_2`
0x0000000000003b6c <+22>:    mov    $0x1,%edx
0x0000000000003b71 <+27>:    mov    %eax,%ecx        # 重复 (无意义)
0x0000000000003b73 <+29>:    shl    %cl,%edx
0x0000000000003b75 <+31>:    mov    %edx,%eax
0x0000000000003b77 <+33>:    cltq                    # 将 `int` 符号扩展为 `size_t`
0x0000000000003b79 <+35>:    mov    %rax,-0x18(%rbp) # 将 `slab_size` 存入内存

# `number_of_slabs = (0x00001000 >> power_of_2)`
0x0000000000003b7d <+39>:    mov    -0x3c(%rbp),%eax
0x0000000000003b80 <+42>:    mov    $0x1000,%edx
0x0000000000003b85 <+47>:    mov    %eax,%ecx
0x0000000000003b87 <+49>:    sar    %cl,%edx
0x0000000000003b89 <+51>:    mov    %edx,%eax
0x0000000000003b8b <+53>:    cltq
0x0000000000003b8d <+55>:    mov    %rax,-0x10(%rbp) # 将 `number_of_slabs` 存入内存

# `new_chunk = alignment::aalloc(4096, 4096);`
0x0000000000003b91 <+59>:    mov    $0x1000,%esi
0x0000000000003b96 <+64>:    mov    $0x1000,%edi
0x0000000000003b9b <+69>:    call   0xacf0
0x0000000000003ba0 <+74>:    mov    %rax,-0x8(%rbp)  # 将 `new_chunk` 存入内存

# `_allocated_chunks.push_back(new_chunk);`
0x0000000000003ba4 <+78>:    call   0x40a2
0x0000000000003ba9 <+83>:    mov    %rax,%rdx
0x0000000000003bac <+86>:    mov    -0x8(%rbp),%rax
0x0000000000003bb0 <+90>:    mov    %rax,%rsi
0x0000000000003bb3 <+93>:    mov    %rdx,%rdi
0x0000000000003bb6 <+96>:    call   0x401e

# 进入循环体前的准备工作
0x0000000000003bbb <+101>:   movq   $0x0,-0x30(%rbp) # 初始化 `current_node`
0x0000000000003bc3 <+109>:   mov    -0x8(%rbp),%rax  # 获取 `new_chunk`
0x0000000000003bc7 <+113>:   mov    %rax,-0x28(%rbp) # 存入并初始化 `next_node`
0x0000000000003bcb <+117>:   movq   $0x0,-0x20(%rbp) # 初始化循环下标 `i`
0x0000000000003bd3 <+125>:   jmp    0x3bf5 <+159>

# 循环体
0x0000000000003bd5 <+127>:   mov    -0x28(%rbp),%rax # 获取 `next_node`
0x0000000000003bd9 <+131>:   mov    -0x30(%rbp),%rdx # 获取 `current_node`
0x0000000000003bdd <+135>:   mov    %rdx,(%rax)      # 赋值 `next_node->next`
0x0000000000003be0 <+138>:   mov    -0x28(%rbp),%rax # 获取 `next_node` (无意义)
0x0000000000003be4 <+142>:   mov    %rax,-0x30(%rbp) # `current_node = next_node;`
0x0000000000003be8 <+146>:   mov    -0x18(%rbp),%rax
0x0000000000003bec <+150>:   add    %rax,-0x28(%rbp) # `next_node += slab_size;`
0x0000000000003bf0 <+154>:   addq   $0x1,-0x20(%rbp) # `i++;`
0x0000000000003bf5 <+159>:   mov    -0x20(%rbp),%rax
0x0000000000003bf9 <+163>:   cmp    -0x10(%rbp),%rax
0x0000000000003bfd <+167>:   jb     0x3bd5 <+127>
0x0000000000003bff <+169>:   mov    -0x30(%rbp),%rax # 返回 `current_node`

# 退出栈帧前的准备工作
0x0000000000003c03 <+173>:   leave
0x0000000000003c04 <+174>:   ret
```

未做优化的 `_allocate_one_chunk` 同样存在以下问题:

- 移动指令冗余. ❌
- 使用复杂且冗余的指令实现简单的移位操作. ❌
- 临时变量被反复存入内存并从内存中读取. ❌
- 使用函数获取线程局部变量. ❌
- 像 `std::vector<T>::push_back()` 这样的函数被重复调用, 增加了不必要的开销. ❌

`-O1` 优化的版本:

```assembly
0x0000000000002b36 <+0>:     endbr64

# 进入栈帧前的准备工作
0x0000000000002b3a <+4>:     push   %r12
0x0000000000002b3c <+6>:     push   %rbp
0x0000000000002b3d <+7>:     push   %rbx
0x0000000000002b3e <+8>:     sub    $0x10,%rsp
0x0000000000002b42 <+12>:    mov    %esi,%ecx      # `power_of_2`
0x0000000000002b44 <+14>:    mov    %fs:0x28,%rax  # 金丝雀值
0x0000000000002b4d <+23>:    mov    %rax,0x8(%rsp)
0x0000000000002b52 <+28>:    xor    %eax,%eax      # 清空寄存器中的金丝雀值

# `slab_size = (1 << power_of_2)`
0x0000000000002b54 <+30>:    mov    $0x1,%ebx
0x0000000000002b59 <+35>:    shl    %cl,%ebx
0x0000000000002b5b <+37>:    movslq %ebx,%rbx      # 将 `int` 符号扩展为 `size_t`

# `number_of_slabs = (0x00001000 >> power_of_2)`
0x0000000000002b5e <+40>:    mov    $0x1000,%r12d
0x0000000000002b64 <+46>:    sar    %cl,%r12d
0x0000000000002b67 <+49>:    movslq %r12d,%r12     # 将 `int` 符号扩展为 `size_t`

# `new_chunk = alignment::aalloc(4096, 4096);`
0x0000000000002b6a <+52>:    mov    $0x1000,%esi
0x0000000000002b6f <+57>:    mov    $0x1000,%edi
0x0000000000002b74 <+62>:    call   0x58d4
0x0000000000002b79 <+67>:    mov    %rax,%rbp      # 将 `new_chunk` 放入寄存器

# 直接对 TLS 数据 `std::vector` 进行内联化的 `push_back()` 操作
0x0000000000002b7c <+70>:    call   0x2ac9 <__tls_init()>
0x0000000000002b81 <+75>:    mov    %rbp,(%rsp)
0x0000000000002b85 <+79>:    mov    %fs:0xfffffffffffffd90,%rsi
0x0000000000002b8e <+88>:    cmp    %fs:0xfffffffffffffd98,%rsi
0x0000000000002b97 <+97>:    je     0x2bef <+185>
0x0000000000002b99 <+99>:    mov    %rbp,(%rsi)
0x0000000000002b9c <+102>:   addq   $0x8,%fs:0xfffffffffffffd90

# 检查是否需要进入循环体
0x0000000000002ba6 <+112>:   test   %r12,%r12     # 确认 `number_of_slabs` 为非零
0x0000000000002ba9 <+115>:   je     0x2c09 <+211>

# 进入循环体前的准备工作
0x0000000000002bab <+117>:   mov    %rbp,%rdx     # 初始化 `next_node` 为 `new_chunk`
0x0000000000002bae <+120>:   mov    $0x0,%eax     # 初始化循环下标 `i`
0x0000000000002bb3 <+125>:   mov    $0x0,%ecx     # 初始化 `current_node` 为 `nullptr`

# 循环体
0x0000000000002bb8 <+130>:   mov    %rcx,(%rdx)   # `next_node->next = current_node;`
0x0000000000002bbb <+133>:   mov    %rdx,%rcx     # `current_node = next_node;`
0x0000000000002bbe <+136>:   add    %rbx,%rdx     # `next_node += slab_size`
0x0000000000002bc1 <+139>:   mov    %rax,%rsi     # 备份当前 `i` 的值
0x0000000000002bc4 <+142>:   add    $0x1,%rax     # `i++`
0x0000000000002bc8 <+146>:   cmp    %rax,%r12
0x0000000000002bcb <+149>:   jne    0x2bb8 <+130>
0x0000000000002bcd <+151>:   imul   %rsi,%rbx
0x0000000000002bd1 <+155>:   lea    0x0(%rbp,%rbx,1),%rax # 使用所备份的 `i` 的值现场
                                                          # 计算并返回 `current_node`

# 退出栈帧前的准备工作
0x0000000000002bd6 <+160>:   mov    0x8(%rsp),%rdx        # 校验金丝雀值
0x0000000000002bdb <+165>:   sub    %fs:0x28,%rdx
0x0000000000002be4 <+174>:   jne    0x2c10 <+218>
0x0000000000002be6 <+176>:   add    $0x10,%rsp
0x0000000000002bea <+180>:   pop    %rbx
0x0000000000002beb <+181>:   pop    %rbp
0x0000000000002bec <+182>:   pop    %r12
0x0000000000002bee <+184>:   ret

# `std::vector<T>::_M_realloc_insert`
0x0000000000002bef <+185>:   mov    %rsp,%rdx
0x0000000000002bf2 <+188>:   mov    %fs:0x0,%rax
0x0000000000002bfb <+197>:   lea    -0x278(%rax),%rdi
0x0000000000002c02 <+204>:   call   0x3f1a
0x0000000000002c07 <+209>:   jmp    0x2ba6 <+112>

# `number_of_slabs` 为零时的分支入口
0x0000000000002c09 <+211>:   mov    $0x0,%eax             # 返回空指针
0x0000000000002c0e <+216>:   jmp    0x2bd6 <+160>

# 金丝雀值校验失败的分支入口
0x0000000000002c10 <+218>:   call   0x23e0 <__stack_chk_fail@plt>
```

优化后的 `_allocate_one_chunk` 更加精简且快速:

- 移动指令被精简. ✔️
- 正确使用三条指令实现移位操作 (准备操作数, 移位, 符号扩展). ✔️
- 对临时变量的存储位置进行了内联. ✔️
- 对线程本地变量的获取进行了内联化处理. ✔️
- 对诸如 `std::vector<T>::push_back()` 这样的函数进行内联化处理. ✔️

## 结语

分析到这里可以看到其实我的内存池的实现本身是完全没问题的, 真正拖累执行效率的是反复的内存读写和函数调用, 以及缺少对冗余指令的折叠或短路. 造成这一设想与现实相背离的现象的本质原因是程序员和编译器所处的层次不同——程序员所看到的是一个经过抽象的平整的内存映像, 而编译器看到的则是一个包含了寄存器, 主存, 以及夹杂在其中的高速缓存 (Cache) 共同构成的层次化的内存结构. 从源码到真正的机器代码之前仍然要走一段很长的路, 因此有时候机器代码的执行可能并不会直接按我们所编写的逻辑逐行展开, 而是会受限于底层硬件架构的约束. 诸如寄存器的利用效率, 高速缓存的命中率, 分支的预判准确率, 甚至指令流水线的优化策略等等均有可能对程序最终的执行效率产生显著影响. 因此我们在优化代码的时候不能只关注高层逻辑的正确性和清晰度, 还应当深入理解底层硬件的行为, 在这种硬件层次的认知的保驾护航下我们便能够编写出更加稳定, 更加可预测的程序.
