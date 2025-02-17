---
title: "MIT 6.S081 操作系统 xv6 教材笔记 - 第 8 章"
description: "系统梳理教材知识点, 结合代码深入理解 xv6 核心工作原理"
summary: "系统梳理教材知识点, 结合代码深入理解 xv6 核心工作原理"
date: 2025-02-17T16:12:23+08:00
draft: false
tags: ["MIT 6.S081", "Operating System Engineering", "xv6"]
series: ["MIT 6.S081", "Operating System Engineering", "xv6"]
author: ["xubinh"]
type: posts
math: true
---

## 8.1 Overview

- xv6 的文件系统拥有 7 层结构:
  1. Disk: 物理硬盘. 逻辑上被切分为等长的块 (block).
  1. Buffer cache: 物理块在内存中的缓存.
  1. Logging: 用于对接来自高层的操作, 负责将高层操作转换为底层的关于多个块的操作 (这些操作整体被称为**事务** (transaction)), 并确保转换后的这些底层操作仍然是原子性的 (这里的原子性是关于断电等物理情况而言的).
  1. Inode: 对文件的抽象. 一个 inode 由一个唯一的 i-number 进行索引, 其内容为所代表的文件的数据所在的物理块的索引.
  1. Directory: 特殊的文件, 其内容为目录项 (directory entry) 的序列, 其中每个目录项由某个文件的文件名和 i-number 构成.
  1. Pathname: 文件路径, 用于对文件进行递归查询.
  1. File descriptor: 用于对各种资源进行统一抽象, 包括管道, 设备, 以及文本文件等等.
- 文件系统在硬盘中的组织结构按物理块 block 的下标从小到达分别组织为:
  1. boot: 第 0 块, 用于存储开机相关的数据. 文件系统总是不会使用该块.
  1. superblock: 第 1 块, 用于存储文件系统的元数据, 包括当前文件系统中单个 block 的大小, block 的总数, inode 的总数, 以及 log 中的 block 的总数等等.
  1. log: 第 2 块开始.
  1. inode: log 区域后开始.
  1. bitmap: inode 区域后开始, 用于标记所有空闲的 block.
  1. data block: bitmap 区域后开始直至最后.
