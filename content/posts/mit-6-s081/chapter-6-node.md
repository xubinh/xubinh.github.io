---
title: "MIT 6.S081 操作系统 xv6 教材笔记 - 第 6 章"
description: "系统梳理教材知识点, 结合代码深入理解 xv6 核心工作原理"
summary: "系统梳理教材知识点, 结合代码深入理解 xv6 核心工作原理"
date: 2025-02-17T16:10:23+08:00
draft: false
tags: ["MIT 6.S081", "Operating System Engineering", "xv6"]
series: ["MIT 6.S081", "Operating System Engineering", "xv6"]
author: ["xubinh"]
type: posts
math: true
---

## 6.5 Locks and interrupt handlers

- 由于同一个线程的主执行流和中断执行流有可能会因为需要获取同一个互斥锁而导致相互死锁, 因此一般来讲如果一个 CPU 要获取一个可能会被中断处理函数获取的锁, 那么必须在获取之前关闭中断. xv6 则使用了一个更为保守的策略——只要 CPU 尝试获取任意互斥锁, 都必须确保在获取之前将中断关闭, 这也是 `acquire` 函数中第一行的 `push_off();` 的意义.

## 6.6 Instruction and memory ordering

- 为了确保临界区域中的代码不会由于编译器重排序或 CPU 的乱序执行而跳出临界区域外执行, xv6 在互斥锁的 `acquire` 函数和 `release` 函数中均使用了 C 编译器内置的 `__sync_synchronize();` 语句来手动禁用临界区域代码的重排序. 像 `__sync_synchronize();` 这样的语句被称为**内存屏障** (memory barrier).

## 6.7 Sleep locks

- 自旋锁 (spinlock) 要求获取它的进程使用一个循环不断轮询来等待外部事件的完成, 这对 CPU 来说是一种极大的浪费, 因为进程本可以在这段等待时间里做其他更有意义的事情. 另一方面自旋锁不允许获取到它的进程被抢占或主动让出执行权, 因为自旋锁的 `acquire` 函数将在获取到锁之前暂时禁用中断, 如果在当前进程睡眠期间**在同一个 CPU 上**有另外的进程尝试获取同一个自旋锁, 那么当前进程就会因为中断被禁用而无法被唤醒并释放锁, 从而与另外的进程形成死锁 (这一点和自旋锁不允许获取到它的进程启用中断类似). 最后 `yield` 函数 (或 `sleep` 函数) 本身就与 spinlock 互斥, 因为前者要求 CPU 启用中断以便其他进程能够被抢占, 而后者则要求获取者禁用中断以防止死锁.
- xv6 实现了 "sleep-lock" 来解决上述问题. sleep-lock 本质上就是一个使用自旋锁实现的初始值为 1 的信号量, 其中获取操作相当于 P 操作, 而释放操作则相当于 V 操作. 如果当前进程没有获取到 sleep-lock, 那么它将不会陷入自旋, 而是直接调用 `sleep` 函数释放 (sleep-lock 所使用到的) 自旋锁, 让出执行权, 并启用中断以便其他进程能够被抢占.
- 由于 sleep-lock 允许启用中断, 因此 sleep-lock 不能够在中断处理函数中进行使用, 否则将导致死锁. 另一方面由于 sleep-lock 有可能让出当前进程的执行权, 因此不允许 sleep-lock 在自旋锁的临界区域中被使用 (但反过来可以, 即自旋锁允许在 sleep-lock 的临界区域中使用).
- 自旋锁适合临界区域短暂的场景; sleep-lock 适合临界区域冗长的场景.
- 锁的获取是一种开销较大的操作, 因为它涉及到 memory model 的可见性问题. 一般的内存模型都具有高速缓存 cache, 如果一个 CPU 将锁的状态缓存在了该 CPU 的局部 cache 中, 那么另一个 CPU 要想获取同一个锁就必须先把锁的状态从该 CPU 的 cache 中复制到另一个 CPU 的 cache 中并使其他所有 CPU 的 cache 中的相关的 cache line 失效 (invalidate).
- 为了避免锁的开销, 许多操作系统使用了**无锁** (lock-free) 数据结构与算法, 但无锁数据结构与算法的设计和实现比直接使用互斥锁的实现来得更为复杂 (而后者本身就已经十分复杂了), 因此 xv6 并不使用无锁算法.
