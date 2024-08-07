---
title: "volatile 与多线程编程"
description: "介绍 volatile 与内存屏障等一系列多线程编程中的常见概念"
summary: "介绍 volatile 与内存屏障等一系列多线程编程中的常见概念"
date: 2024-08-03T00:28:51+08:00
draft: false
tags: ["xv6", "MIT 6.S081", "CSAPP", "CMU 15-213", "Multithreading", "C", "C++", "Java"]
series: ["xv6", "MIT 6.S081", "CSAPP", "CMU 15-213", "Multithreading", "C", "C++", "Java"]
author: ["xubinh"]
type: posts
---

> 起因是我在阅读 xv6 源码的时候遇到了如下两行代码:
>
> ```c
> // kernel/spinlock.c:32
> while(__sync_lock_test_and_set(&lk->locked, 1) != 0);
> 
> // kernel/riscv.h:53
> asm volatile("csrr %0, sstatus" : "=r" (x) );
> ```

## 并发三大特征: 原子性, 可见性, 以及有序性

在多线程编程中确保程序的正确性和效率是一个极具挑战性的任务, 其复杂性主要源于线程之间对共享资源的访问和操作. 为了理解多线程编程中的问题及其解决方案, 我们首先得需要了解并发的三大核心特征: 原子性, 可见性和有序性. 这些特征决定了多线程程序如何正确且安全地执行所期望的操作.

- 原子性: 原子性即多线程下操作的互斥性. 程序员使用多个 CPU 执行一个误以为具有但实际不具有原子性的操作时可能会互相重叠, 从而产生与预期不符的结果.
  - 例如像 `int x; x += 1` 这样的操作有可能被编译为多条机器指令, 从而有可能在执行到一半的时候被信号中断或是 CPU 被抢占, 导致未修改完成的值提前暴露给其他线程.
  - 此外即便是单条机器指令也有可能被分解为多条**微指令** (microcode), 同样有可能导致其他线程读取到当前线程未修改完成的值.
- 可见性: 编译器在编译时默认是没有多线程概念的, 一切都是以在单线程下正确执行为目标. 在单线程下如果能够在不影响程序执行正确性的前提下取消将某个变量的修改写回内存的步骤, 那么编译器很有可能使用寄存器缓存该变量的值以实现更快的运行效率. 但是如果有其他线程依赖于对该变量的修改, 由于变量的修改被寄存器拦截, 其他线程将看不到该变量修改后的值, 程序便会出错. 反过来如果本线程依赖于其他线程对某个变量的修改, 由于本线程将该变量使用寄存器缓存起来并在此后的读取操作中都使用寄存器中的值, 那么其他线程对该变量的修改同样也不为本线程可见. 可见性与**缓存一致性** (Cache coherence) 问题 (例如 MESI 协议) 密切相关.
- 有序性: 和可见性类似, 在不影响单线程执行正确性的情况下如果对指令进行重排序能够优化执行性能, 那么编译器很有可能会对指令进行重排序. 但如果此时有其他线程依赖于本线程的代码执行的顺序, 那么指令重排序就有可能导致多线程下执行出错.
  - 例如假设其他线程需要等待本线程**先**初始化某个变量**再**设置标志位表示该变量已被初始化, 如果本线程的指令被重排为先设置标志位再对变量进行初始化, 这在单线程下是完全正确的, 但在多线程下其他线程将会在收到 "变量已被初始化" 的信号后见到一个根本没有被初始化的变量, 这就会导致错误.

## `volatile` 与可见性

C 语言下的 `volatile` 关键字的作用是防止编译器在**无可见副作用** (assumption of non visible side effects) 的假设下对代码进行省略优化. 常见的使用场景包括:

1. 内存映射 I/O (Memory-mapped I/O, MMIO). 使用 MMIO 将使得外部设备的内存和寄存器将被映射至用户的地址空间以便程序员使用内存地址直接访问设备. 在这种情况下如果外部设备改变了其内存中的数据, 程序员必须使用 `volatile` 关键字修饰变量以便使这种改变总是对程序可见.

1. 信号处理函数. 和 MMIO 类似, 信号处理函数也并不是编译器对程序进行编译时需要考虑的对象, 因此程序员必须使用 `volatile` 禁止编译器优化以使得信号处理函数作出的修改总是对程序可见.

   例如考虑 C 程序:

   ```c
   int finished = 0;
   
   while(!finished) {
       // do something
   }
   ```

   其中标志位 `finished` 将由某个异步的信号处理函数进行修改以通知当前进程某个事件已经发生. 如果不使用 `volatile` 关键字对 `finished` 进行修饰, 那么编译器将有可能因为认为 `finished` 变量的值在当前进程中没有任何修改而将其内联化, 于是循环语句会被省略优化为 `while(true)`, 这就会导致出错.

在使用**非本地跳转** (non-local jumps) 时也需要使用 `volatile` 对 `setjump` 和 `longjump` 之间定义的局部变量进行修饰, 避免跳转后结果丢失.

C 语言下的 `volatile` 关键字只保证可见性, 并不保证原子性和有序性. 原子性的实现需要硬件层面的配合, 单独使用高级语言无法完全实现原子性. 有序性则与内存屏障有关, 下面将会讲到.

## 原子性与锁

在并发编程中原子性是确保线程安全和数据一致性的重要概念. **原子性**指的是一个操作在执行过程中不可被中断, 即要么全部执行成功要么完全不执行, 没有任何线程能够观察到执行的中间状态. **读-改-写** (read-modify-write, RMW) 操作是实现原子性的重要手段之一. 典型的 RMW 操作包括 Test-and-set, Fetch-and-add 和 Compare-and-swap. 这些操作能够确保在多线程访问同一共享资源时的原子性.

### Test-and-set

Test-and-set (TAS) 操作的作用是将某一内存地址下的数据置 1, 同时返回置 1 前该内存地址下的数据的值. 整个过程原子化不可分割, 不会受到任何外部事件的中断.

- x86 下的含有 LOCK 前缀的 `BTS` 指令便是一个 TAS 操作. 这里 LOCK 前缀是使得 `BTS` 指令成为原子操作的关键. LOCK 前缀的作用是在当前指令执行期间对**数据总线** (data bus) 进行上锁, 这样除了当前抢占到数据总线的 CPU 在指令执行期间可以访问内存, 其他 CPU 都无法访问内存, 从而确保了指令的原子性. x86 下可以对 `ADD`, `ADC`, `AND`, `BTC`, `BTR`, `BTS`, `CMPXCHG`, `DEC`, `INC`, `NEG`, `NOT`, `OR`, `SBB`, `SUB`, `XOR`, `XADD` 以及 `XCHG` 等一系列指令添加 LOCK 前缀. 其中 `XCHG` 指令默认对数据总线上锁, 因此可以省略 LOCK 前缀.
- 需要指出的是 LOCK 前缀与如 MESI 等缓存一致性协议是两个概念. 即使实现了缓存一致性协议也无法避免两个 RMW 操作的重叠.

### Fetch-and-add

Fetch-and-add (FAA) 操作的作用是对某一内存地址下的数据进行增加 (increment), 同时返回增加前该内存地址下的数据的值. 整个过程原子化不可分割, 不会受到任何外部事件的中断.

- x86 下的含有 LOCK 前缀的 `XADD` 指令便是一个 FAA 操作.

### Compare-and-swap

Compare-and-swap (CAS) 操作的作用是将某一内存地址下的数据的值与一个给定的值进行比较, 当且仅当比较相等时将另一个新值写入该内存地址, 同时无论比较是否相等均返回比较前该内存地址下的数据的值. 整个过程原子化不可分割, 不会受到任何外部事件的中断.

- x86 下的含有 LOCK 前缀的 `CMPXCHG` 指令便是一个 CAS 操作.
- CAS 操作比 FAA 操作多一个自由度. FAA 只能控制增量的大小, 而 CAS 既可以控制增量的大小, 也可以指定所要增加的数据的初始值.
- 实际应用时如果每个线程在每一次 CAS 操作失败后立即执行下一次 CAS 操作, 有可能出现有的线程总是抢占不到期望的数据的初始值的情况导致饥饿现象. 研究表明使用**指数退避** (exponential backoff) 方法能够提高多线程下 CAS 操作的效率.

#### CAS 操作与 ABA 问题

尽管 CAS 操作本身是原子性的, 在使用 CAS 实现其他常见算法时仍然有可能导致微妙的错误, 其中最著名的便是 **ABA 问题** (ABA problem).

假设程序员想要实现这样一种逻辑: 在使用前一次 CAS 操作检测到某个变量的当前值为 A 后将在本次 CAS 操作中期望将 A 交换为 B, 反之如果前一次操作检测到该变量为 B 则将在本次操作中期望交换为 A. 初看上去这个逻辑似乎没有什么问题, 但程序员遗漏了一个重要事实: 从前一次 CAS 操作结束并读出该变量的值到本次 CAS 操作即将开始的这一段时间里这个逻辑并不是原子性的. 这意味着有可能存在 2 个线程同时读到了 A, 并打算在本次 CAS 中将其与 B 交换, 但第 1 个线程的 CPU 被抢占并因此进入睡眠, 于是第 2 个线程成功进行交换, 交换后变量的值为 B. 如果到这里为止则程序实际上还是正确的, 因为睡眠的线程恢复后将由于变量的值已经不是 A 而导致 CAS 失败. 不幸的是恰好有第 3 个线程检测到此时变量的值为 B 并又在下一次 CAS 操作中将其交换回了 A. 如果此时睡眠的线程恢复执行, 它理所当然会将 A 再一次交换为 B, 但这次交换是冗余的 (因此是错误的), 因为从一开始线程 1 和线程 2 所执行的便是同一个交换工作 (A -> B) 的两个副本而线程 2 已经帮线程 1 做完工作了. 如果没有线程 3 的加入并且线程 2 执行完 A -> B 的 CAS 操作后便不再进行 CAS 操作, 那么线程 1 的交换最终还是会失败, 此时冗余被消除, 于是程序还是正确的. 正是由于线程 3 (或者也可以是线程 2 本身) 所做的更改使得变量的值复原到了线程 1 睡眠前的状态, 让线程 1 误以为自己的交换操作仍然是必要的, 因此导致交换成功并产生冗余操作.

ABA 问题本质上和一般 RMW 指令的原子性问题一样, 都是由于非原子性导致的对同一个操作需求的多次冗余执行, 区别在于一般 RMW 指令是直接写入的, 不需要比较, 而 CAS 操作本身虽然是原子性的并且写入 (交换) 前需要比较, 却有可能因为 CAS 操作所属的外部较大逻辑的非原子性而导致其错误通过了比较.

### 互斥锁与自旋锁

多线程下的锁是程序在多个 CPU 之间进行互斥与同步的重要机制, 程序员基于锁提供的**获取** (acquire) 和**释放** (release) 操作可进一步确保位于获取操作与释放操作之间的**临界区域** (critical section) 中的代码在不同 CPU 之间的互斥与同步关系. 在众多类型的锁中互斥锁 (mutual exclusion, mutex) 和自旋锁 (spinlock) 是两类最为重要的锁. 互斥锁与自旋锁自身的实现均是利用了 RMW 操作的原子性, 二者的区别在于不同情况下为实现锁而引入的额外开销不同. 一个简单的使用 TAS 操作实现的自旋锁的C语言代码如下:

```c
volatile int lock = 0;

void acquire() {
    while (__sync_lock_test_and_set(&lock, 1) != 0)
        ;

    __sync_synchronize();
}

void release() {
    __sync_synchronize();

    __sync_lock_release(&lock);
}
```

其中函数 `__sync_lock_test_and_set` 为 TAS 操作, 函数 `__sync_lock_release` 为原子化的置 0 操作, 函数 `__sync_synchronize` 用于设置完全屏障 (下面将会讲到), 确保临界区域中的代码不会被重排序至锁的保护范围外. 互斥锁的实现与自旋锁类似, 只不过为了能够使未能成功抢占到锁的线程休眠还需要额外执行特定的系统调用.

## 有序性与内存屏障

程序员在进行多线程编程时不仅考虑代码在当前文件中的执行效果, 还会考虑代码与其他线程以及外部设备之间的相互影响, 而编译器在编译时一般只考虑前者, 并且默认不会对后者产生任何影响; 同时现代处理器在执行机器指令时为了达到更高的执行效率会在不破坏原有的前后指令之间数据的可见性依赖的前提下在执行层面对指令进行重排序, 即**乱序执行** (Out-of-order execution, OoOE). 因此有序性同时存在于两个层面, 一个是编译时 (例如使用 GCC 将源码编译为机器指令时可能发生重排序), 另一个是运行时 (即 CPU 的乱序执行).

考虑某个进程下的线程 1 和线程 2, 假设线程 2 需要等待线程 1 设置好线程 2 所需的数据后才能开始执行, 并且线程 1 和 2 之间通过某个共享变量作为标志位确定同步关系:

```c
// 线程 1:
int x = 123; // 准备数据
flag_prepared = true; // 提醒线程 2 开始

// 线程 2:
while(!flag_prepared); // 等待线程 1 准备好数据
printf("%d\n", x); // 开始执行
```

无论是编译器在编译上述代码的过程中还是 CPU 执行编译后的可执行文件的过程中均有可能将线程 1 对 `flag_prepared` 的设置提前到对数据 `x` 的准备之前, 这就会导致线程 2 误以为线程 1 已经准备好了数据而照常执行 (`x` 此时的值未定义). 另一方面其也有可能将线程 2 对 `x` 的读取 (位于 `printf` 函数中) 提前到对 `flag_prepared` 的测试之前, 这同样会导致未定义的行为.

前面提到的由于没有确保有序性而导致的问题究其根本是因为编译器或 CPU 在不知道两条指令之前存在高层顺序依赖关系的情况下对它们进行了位置的互换. 因此我们可以从一个更加底层的视角来看待有序性问题. 固定构成一个可执行文件的顺序机器指令流中的任意一点, 在这个点之前执行的指令与在这个点之后执行的指令在没有任何对有序性的限制的前提下由于编译时和运行时重排序将有可能以各种合适的方案被移动并跨越该点. **内存屏障** (memory fence) 的作用便是通过在该点插入某种特殊的高级语言表达式或者某种特殊的机器指令对编译器或 CPU 能否将该点前后的普通指令进行移动并跨越该点作出限制. 如果将位于该点前的所有指令分为读指令和写指令两个部分, 同时将位于该点后的所有指令也分为读指令和写指令两个部分, 不同的内存屏障的区别仅在于对这四个部分之间的执行顺序的要求不同. 几种常见的内存屏障如下:

- 读屏障 (read barrier): 位于读屏障前的读操作不能向后跨越读屏障, 位于读屏障后的读操作不能向前跨越读屏障.
- 写屏障 (write barrier): 位于写屏障前的写操作不能向后跨越写屏障, 位于写屏障后的写操作不能向前跨越写屏障.
- 释放屏障 (release barrier): 位于释放屏障前的读操作不能向后跨越释放屏障, 位于释放屏障后的读操作和写操作不能向前跨越释放屏障.
- 获取屏障 (acquire barrier): 位于获取屏障前的读操作和写操作不能向后跨越获取屏障, 位于获取屏障后的写操作不能向前跨越获取屏障.
- 完全屏障 (full barrier): 位于完全屏障前的读操作和写操作不能向后跨越完全屏障, 位于完全屏障后的读操作和写操作不能向前跨越完全屏障.

内存屏障的存在使得人们对机器指令的编译时和运行时执行顺序拥有了主动权. 程序员可以通过在他们自己的代码中设置内存屏障来为代码引入有序性.

- x86 处理器提供了 `lfence` (能够实现释放屏障), `sfence` (能够实现获取屏障) 以及 `mfence` (完全屏障) 三种起到内存屏障作用的指令. 值得一提的是尽管 x86 提供了 `lfence` 指令, 但它自身的内存模型已经具有足够强的有序性, `lfence` 指令实际上是可有可无的.
- C语言下可以使用如下内联汇编语句设置**编译时**完全屏障:

  ```c
  asm volatile("" ::: "memory");
  __asm__ __volatile__ ("" ::: "memory");
  ```

  - `volatile` 关键字的作用是禁止对本内联汇编代码进行编译时重排序, 真正起到内存屏障作用的是括号内部的内容. 关于更多有关内联汇编的信息可以参考[这篇教程](https://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html#ss5.4).

  另一方面可以使用 GCC 的内置函数 `__sync_synchronize()` 同时设置**编译时和运行时**完全屏障.
- C++11 提供了 `std::atomic`, `std::memory_order`, 以及 `std::atomic_thread_fence` 等机制实现各类内存屏障.

## 例子: 双重检查锁

在单例模式中, 程序员想要初始化一个全局唯一的对象 (首先以 Java 为例):

```java
class Foo {
    private static Helper helper;
    public Helper getHelper() {
        if (helper == null) {
            helper = new Helper();
        }
        return helper;
    }

    // 其他成员...
}
```

这么做的缺点是在多线程情况下可能有多个线程同时调用 `getHelper` 函数尝试进行初始化. 最简单的优化可以是对初始化函数本身加锁:

```java
class Foo {
    private Helper helper;
    public synchronized Helper getHelper() {
        if (helper == null) {
            helper = new Helper();
        }
        return helper;
    }

    // 其他成员...
}
```

但这么做粒度太大, 因为逻辑上只需要为首次初始化的那一小段时间内的竞争加锁即可, 然而上述代码在此后每次获取对象时都要进行一次锁的获取. 为了进一步降低锁的粒度, 人们提出了**双重检查锁** (Double-checked locking) 模式:

```java
class Foo {
    private Helper helper;
    public Helper getHelper() {
        if (helper == null) {
            synchronized (this) {
                if (helper == null) {
                    helper = new Helper();
                }
            }
        }
        return helper;
    }

    // 其他成员...
}
```

双检锁机制扎扎实实地将锁的粒度缩小为仅针对首次初始化的那一小段时间内的竞争, 此后每次调用均只引入一个 if 语句的开销, 可以忽略不计.

乍看上去似乎双检锁已经完全实现了单例模式, 实则不然. 上述代码在多线程模式下运行时仍然有可能出错, 原因正是上述代码忽略了语句 `helper = new Helper();` 并不具有有序性, 可能遭到编译时重排序导致尚未初始化完全的引用 `helper` 提前暴露给其他线程. 在 Java 中修复该问题的方法也很简单, 就是使用 `volatile` 关键字对 `helper` 对象进行修饰 (注意在 Java 下 `volatile` 关键字同时保证可见性和有序性 (底层通过内存屏障实现有序性)):

```java
class Foo {
    private volatile Helper helper;
    public Helper getHelper() {
        Helper localRef = helper;
        if (localRef == null) {
            synchronized (this) {
                localRef = helper;
                if (localRef == null) {
                    helper = localRef = new Helper();
                }
            }
        }
        return localRef;
    }

    // 其他成员...
}
```

在 C++ 下需要使用 `std::atomic` 提供的有序性实现指针+锁方案的单例模式 (实际上存在更加简单的单例模式方案, 例如静态局部变量方案的单例模式):

```cpp
#include <atomic>
#include <mutex>

class Singleton {
public:
    static Singleton *GetInstance();

private:
    Singleton() = default;

    static std::atomic<Singleton *> s_instance;
    static std::mutex s_mutex;
};

Singleton *Singleton::GetInstance() {
    Singleton *p = s_instance.load(std::memory_order_acquire);
    if (p == nullptr) {
        std::lock_guard<std::mutex> lock(s_mutex);
        p = s_instance.load(std::memory_order_relaxed);
        if (p == nullptr) {
            p = new Singleton();
            s_instance.store(p, std::memory_order_release);
        }
    }
    return p;
}
```

其中使用了 `std::atomic` 类配合 `std::memory_order_release` 和 `std::memory_order_acquire` 来实现有序性, 后者在写者线程和读者线程之间建立起**释放-获取顺序** (Release-Acquire ordering).

- 尽管对于如 x86 等强调有序性的处理器绝大部分情况下不需要显式禁止运行时重排序, 上述代码仍然能够禁止如 GCC 等编译器进行编译时重排序.

## 参考资料

- [declaration - Why is volatile needed in C? - Stack Overflow](https://stackoverflow.com/questions/246127/why-is-volatile-needed-in-c)
- [Memory-mapped I/O and port-mapped I/O - Wikipedia](https://en.wikipedia.org/wiki/Memory-mapped_I/O_and_port-mapped_I/O)
- [c++ - Concurrency: Atomic and volatile in C++11 memory model - Stack Overflow](https://stackoverflow.com/questions/8819095/concurrency-atomic-and-volatile-in-c11-memory-model)
- [linux - How to do an atomic increment and fetch in C? - Stack Overflow](https://stackoverflow.com/questions/2353371/how-to-do-an-atomic-increment-and-fetch-in-c)
- [Atomic Builtins - Using the GNU Compiler Collection (GCC)](https://gcc.gnu.org/onlinedocs/gcc-4.5.3/gcc/Atomic-Builtins.html)
- [__atomic Builtins - Using the GNU Compiler Collection (GCC)](https://gcc.gnu.org/onlinedocs/gcc-4.8.2/gcc/_005f_005fatomic-Builtins.html)
- [Fetch-and-add - Wikipedia](https://en.wikipedia.org/wiki/Fetch-and-add)
- [Compare-and-swap - Wikipedia](https://en.wikipedia.org/wiki/Compare-and-swap)
- [c++ - What does the "lock" instruction mean in x86 assembly? - Stack Overflow](https://stackoverflow.com/questions/8891067/what-does-the-lock-instruction-mean-in-x86-assembly)
- [ABA problem - Wikipedia](https://en.wikipedia.org/wiki/ABA_problem)
- [Memory barrier - Wikipedia](https://en.wikipedia.org/wiki/Memory_barrier)
- [Out-of-order execution - Wikipedia](https://en.wikipedia.org/wiki/Out-of-order_execution)
- [c - Working of `__asm__ __volatile__ ("" : : : "memory")` - Stack Overflow](https://stackoverflow.com/questions/14950614/working-of-asm-volatile-memory)
- [Memory ordering - Wikipedia](https://en.wikipedia.org/wiki/Memory_ordering)
- [GCC-Inline-Assembly-HOWTO](https://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html#ss5.4)
- [concurrency - What is a memory fence? - Stack Overflow](https://stackoverflow.com/questions/286629/what-is-a-memory-fence)
- [c - Why do we need both read and write barriers? - Stack Overflow](https://stackoverflow.com/questions/61307639/why-do-we-need-both-read-and-write-barriers)
- [assembly - Does the Intel Memory Model make SFENCE and LFENCE redundant? - Stack Overflow](https://stackoverflow.com/questions/32705169/does-the-intel-memory-model-make-sfence-and-lfence-redundant)
- [Cache coherence - Wikipedia](https://en.wikipedia.org/wiki/Cache_coherence)
- [assembly - LOCK prefix vs MESI protocol? - Stack Overflow](https://stackoverflow.com/questions/29880015/lock-prefix-vs-mesi-protocol)
- [Double-checked locking - Wikipedia](https://en.wikipedia.org/wiki/Double-checked_locking)
- [std::memory_order - cppreference.com](https://en.cppreference.com/w/cpp/atomic/memory_order)
- [c++ - Singleton instance declared as static variable of GetInstance method, is it thread-safe? - Stack Overflow](https://stackoverflow.com/questions/449436/singleton-instance-declared-as-static-variable-of-getinstance-method-is-it-thre)
- [Read–modify–write - Wikipedia](https://en.wikipedia.org/wiki/Read%E2%80%93modify%E2%80%93write)
- [Non-blocking algorithm - Wikipedia](https://en.wikipedia.org/wiki/Non-blocking_algorithm)
- [assembly - x86 LOCK question on multi-core CPUs - Stack Overflow](https://stackoverflow.com/questions/3339141/x86-lock-question-on-multi-core-cpus)
- [c++ - Memory fences: acquire/load and release/store - Stack Overflow](https://stackoverflow.com/questions/36824811/memory-fences-acquire-load-and-release-store)
- [Acquire and Release Semantics](https://preshing.com/20120913/acquire-and-release-semantics/)
- [Acquire and Release Fences](https://preshing.com/20130922/acquire-and-release-fences/)
- [assembly - Does it make any sense to use the LFENCE instruction on x86/x86_64 processors? - Stack Overflow](https://stackoverflow.com/questions/20316124/does-it-make-any-sense-to-use-the-lfence-instruction-on-x86-x86-64-processors)
- [Intel's Software Developers Manual, volume 3, section 8.2.2](https://www-ssl.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-system-programming-manual-325384.pdf)
- [c++ - Why GCC does not use LOAD(without fence) and STORE+SFENCE for Sequential Consistency? - Stack Overflow](https://stackoverflow.com/questions/19047327/why-gcc-does-not-use-loadwithout-fence-and-storesfence-for-sequential-consist)
- [std::atomic_thread_fence - cppreference.com](https://en.cppreference.com/w/cpp/atomic/atomic_thread_fence)
- [multithreading - When are x86 LFENCE, SFENCE and MFENCE instructions required? - Stack Overflow](https://stackoverflow.com/questions/27595595/when-are-x86-lfence-sfence-and-mfence-instructions-required)
- [c - GCC memory barrier __sync_synchronize vs asm volatile("": : :"memory") - Stack Overflow](https://stackoverflow.com/questions/19965076/gcc-memory-barrier-sync-synchronize-vs-asm-volatile-memory)
- [Test-and-set - Wikipedia](https://en.wikipedia.org/wiki/Test-and-set)
- [synchronization - When should one use a spinlock instead of mutex? - Stack Overflow](https://stackoverflow.com/questions/5869825/when-should-one-use-a-spinlock-instead-of-mutex)
- [multithreading - Implementing a mutex with test-and-set atomic operation: will it work for more than 2 threads? - Stack Overflow](https://stackoverflow.com/questions/56725078/implementing-a-mutex-with-test-and-set-atomic-operation-will-it-work-for-more-t)
- [language agnostic - How are mutexes implemented? - Stack Overflow](https://stackoverflow.com/questions/1485924/how-are-mutexes-implemented)
- [Mutual exclusion - Wikipedia](https://en.wikipedia.org/wiki/Mutual_exclusion)
