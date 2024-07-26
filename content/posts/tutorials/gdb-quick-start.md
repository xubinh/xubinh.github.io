---
title: "GDB 入门"
description: "十分钟熟悉 GDB 基本操作"
summary: "十分钟熟悉 GDB 基本操作"
date: 2024-07-11T20:09:55+08:00
draft: false
tags: ["tutorials"]
series: ["tutorials"]
author: ["xubinh"]
type: posts
---

## 进入 gdb 环境

- `gdb` : 直接启动环境
- `gdb <executable>` : 指定可执行文件并启动环境

> - 常用的是 `gdb <executable>` 命令, 以直接载入可执行文件.

## 设置断点

- `break <function-name>` : 将断点设置在函数入口处
- `break *<memory-address>` : 将断点设置在指定内存位置

> - `<function-name>` 可以是函数名称, 例如 `my_func`; 而 `<memory-address>` 可以是某个具体的内存地址, 例如 `0x4328afe`, 使用内存地址设置断点时务必前缀一个星号 `*`.

## 禁用和启用断点

- `disable n` : 暂时禁用断点
- `enable n` : 重新启用断点

> - `n` 指 gdb 为该断点分配的序号. gdb 在设置断点时会为每一个断点分配一个正整数序号, 从 1 开始从小到大分配. 可以直接使用序号对断点进行操作.

## 删除断点

- `delete n` : 删除指定断点
- `delete` : 删除当前设置的所有断点

## 运行

- `run <arg1> <arg2> <arg3>` : 传入参数并运行可执行文件
- `stepi` : 单步进入 (以指令为单位)
- `stepi n` : 连续执行 `n` 次单步进入 (以指令为单位)
- `nexti` : 单步跳过 (以指令为单位)
- `nexti n` : 连续执行 `n` 次单步跳过 (以指令为单位)
- `step` : 单步进入 (以 C 语句为单位)
- `continue` : 继续执行, 直至遇见下一个断点
- `finish` : 继续执行, 直至当前函数返回
- `kill` : 终止当前运行的可执行文件
- `exit` : 退出 gdb

## 反汇编函数

- `disas` : 反汇编当前函数
- `disas <function-name>` : 反汇编指定函数

## 打印数据

- `print (char *) <memory-address>` : 打印字符串
- `print *(int *) <memory-address>` : 打印整数

> - `<memory-address>` 可以是一般的内存地址, 例如 `0x423fabd`, 也可以是某个表达式, 例如 `($rip + 0x423fabd)`.

## 打印二进制内容

- `x/[n][s][f] <memory-address>` : 打印指定内存地址附近的二进制内容
- `x/[n][s][f] <function-name>` : 打印函数入口附近的二进制内容
- `x/[n]i <function-name>` : 打印函数入口附近的若干行机器指令
- `x/s <memory-address>` : 打印指定地址处的字符串
- `x/a <memory-address>` : 打印指定地址处所存储的地址 (以当前位置到前一个最近的全局符号的偏移的形式打印)

> 命令 `x` 以字节块的形式组织并打印二进制内容, 上述命令中的 `[n]`, `[s]` 和 `[f]` 分别代表要显示的字节块的 "数量 (number)", "大小 (size)" 以及 "格式 (format)". 其中:
>
> - 数量 `[n]` 为正整数;
> - 大小 `[s]` 可以是:
>   - `b` : 字节
>   - `h` : 双字节
>   - `w` : 四字节 (32 bits)
>   - `g` : 八字节 (64 bits)
> - 格式 `[f]` 可以是:
>   - `d` : 十进制
>   - `x` : 十六进制
>   - `o` : 八进制
>
> 常见例子:
>
> - `x/2gx $rsp` : 以十六进制格式打印从 `$rsp` 中存储的地址处开始的 2 个八字节块

## 打印环境信息

- `info breakpoints` : 打印所有断点
- `info registers` : 打印所有寄存器内容
