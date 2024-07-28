---
title: "CSAPP 实验笔记 - 5. Shell Lab"
description: "模仿 Unix shell 使用 C 语言实现一个简单版 shell"
summary: "模仿 Unix shell 使用 C 语言实现一个简单版 shell"
date: 2024-07-11T20:06:55+08:00
draft: false
tags: ["CSAPP", "CMU 15-213", "ICS"]
series: ["CSAPP", "CMU 15-213", "ICS"]
author: ["xubinh"]
type: posts
---

> - **具体代码请移步至[GitHub](https://github.com/xubinh/csapp/tree/main/5-shell-lab).**

## Shell Lab

### 实验目的

模仿 Unix shell 程序, 使用 C 语言实现一个简单版本的 tiny shell (简称 tsh).

### 实验框架

{{< notice tip >}}
如果在函数 `Signal` 中遇到结构类型 `struct sigaction` 报 `incomplete type is not allowed` 错误, 可以尝试将 VS Code 中的 `c_cpp_properties.json` 中的 `cStandard` 降级为 `gnu89`. 参考资料: [link](https://stackoverflow.com/questions/6491019/struct-sigaction-incomplete-error).
{{< /notice >}}

#### `Makefile` - 编译 tsh 和一些帮手程序, 以及自动化测试

编译所实现的 `tsh.c`, 以及一些用于手动执行合理性测试的帮手程序 `myspin.c`, `mysplit.c`, `mystop.c` 和 `myint.c`. 同时包括一系列自动化测试 `test{01~16}`, `rtest{01~16}`.

#### `my<xxx>.c` - 帮手程序

一些用于手动执行合理性测试的帮手程序, 包括 `myspin.c`, `mysplit.c`, `mystop.c` 和 `myint.c`.

#### `sdriver.pl` - 驱动程序

用于执行自动化测试的驱动程序.

#### `trace{01-16}.txt` - 自动化测试序列

用于固定自动化测试过程的序列文件, 测试复杂度随文件序号增加逐步提升.

#### `tsh.c` - tsh 源文件

`tsh.c` 为本实验需要实现的 tsh 的源文件, 其中包含 tsh 的简单骨架 (`main` 函数) 和一些提前已经实现好的基本函数, 以及 7 个待填充的重要函数.

shell 程序概览:

- shell 程序是一个交互性的命令行解释器, 其作用是解释命令行并代表用户运行其他程序. 最早的 shell 程序的实现是 `sh`, 后来出现了一些变种如 `csh`, `tcsh`, `ksh` 和 `bash` 等等. shell 程序的执行过程总结起来就是打印提示信息, 等待用户通过 `stdin` 输入命令行, 然后解释并执行命令行, 循环往复.
- 一个命令行 (command line) 是指一个由空格分隔的一系列由 ASCII 字符构成的单词序列, 序列中第一个单词为某个内置 shell 命令或是某个可执行文件的路径, 剩下的所有单词均为参数, 若要单词包含空格可以使用单引号括起来.
  - 如果首个单词为内置命令, 那么 shell 会在当前进程中立即执行该内置命令;
  - 如果首个单词表示可执行文件, 那么 shell 会新建一个子进程并在新的子进程的上下文中加载并执行该程序.
- shell 的命令行还支持使用管道串联多个子进程, 因此为了执行一个命令行可能需要创建不止一个子进程. 关于一个命令行所创建的所有子进程构成的集合称为作业 (job).
- 如果一个命令行以字符 `&` 结尾, 那么说明用户指定该作业运行于后台 (background), 这意味着 shell 并不等待该作业完成就立即继续打印提示信息并等待下一命令行; 反之如果一个命令行不以 `&` 结尾则说明用户要求该作业运行于前台 (foreground), 这意味着 shell 将等待该作业运行结束之后再继续运作. 因此前台作业在同一时间至多只有一个, 而后台作业则可以有任意多个.
- shell 还支持作业控制 (job control) 的概念, 允许用户通过内置命令以及信号机制改变作业的前后台设置以及进程的运行状态:
  - 内置命令:
    - `jobs`: 列出所有后台作业, 包括正在运行的以及暂时停止的作业.
    - `bg <job>`: 唤醒一个停止的后台作业.
    - `fg <job>`: 将一个停止的/正在运行的后台作业提升为前台作业.
    - `kill <job>`: 终止一个作业.
  - 信号:
    - `SIGINT`: 默认行为是终止进程. 通过在键盘上键入 `ctrl-c` 来向前台作业中的每一个进程发送该信号.
    - `SIGTSTP`: 默认行为是停止进程. 通过在键盘上键入 `ctrl-z` 来向前台作业中的每一个进程发送该信号. 停止的进程会在接收到 `SIGCONT` 信号时被唤醒.

本实验要求 tsh 程序需要满足的特性:

- 提示信息应为字符串 `tsh>_` (下划线表示空格).
- 用户输入的命令行由一个名称 `name` 和零个或多个参数构成, 单词之间由一个或多个空格分隔. 其中 `name` 表示一个内置命令或是一个可执行文件的路径, 如果是前者, 那么 tsh 将立即解释该内置命令并继续等待下一命令行; 如果是后者, 那么 tsh 将会新建一个初始子进程, 加载并执行该可执行文件 (这种情况下前台作业等价于该初始子进程, 毕竟作业 ID 一般就等于第一个进程的 PID).
- tsh 不需要实现管道运算符 `|` 和 I/O 重定向运算符 `>` 和 `<`.
- 输入 `ctrl-c`/`ctrl-z` 将会向前台作业中的所有进程以及这些进程所创建的所有后代子进程发送一个 `SIGINT`/`SIGTSTP` 信号. 如果没有前台作业, 那么所发送的信号不应产生任何影响.
- 如果一个命令行以字符 `&` 结尾, 那么 tsh 应该在后台运行该作业, 否则应该在前台运行该作业.
- 每个作业可以通过一个进程 ID (PID) 或一个作业 ID (JID) 来进行引用. 其中 JID 是由 tsh 所分配的一个正整数. 在命令行中引用 JID 应前缀一个字符 `%`, 例如 `%5` 表示 JID 5 而 `5` 就表示 PID 5.
- tsh 需要支持下列内置命令:
  - `quit`: 终止 tsh.
  - `jobs`: 列出所有后台作业.
  - `bg <job>`: 通过发送信号 `SIGCONT` 唤醒一个停止的作业, 并于后台运行该作业. 参数 `<job>` 既可以是 PID 也可以是 JID.
  - `fg <job>`: 通过发送信号 `SIGCONT` 唤醒一个停止的作业, 并于前台运行该作业. 参数 `<job>` 既可以是 PID 也可以是 JID.
- tsh 应该回收其所有僵死子进程, 如果存在子进程因为接收信号而提前终止, 那么 tsh 应能识别该事件并打印一条包含该进程 PID 以及相关信号描述的信息.

#### `tshref` - tsh 的参考实现

tsh 的参考实现程序.

#### `tshref.out` - tsh 参考实现的自动化测试输出

对 tsh 参考实现执行自动化测试所得到的输出.

#### 其他

关于本实验的一些建议与提示:

- **一字不落地阅读 CS:APP 第 8 章的内容**.
- **按 trace 文件的难度顺序来开发 tsh**, 例如首先从 `trace01.txt` 开始, 确保所实现的 tsh 和 tsh 的参考实现的输出一致, 然后继续 `trace02.txt`, 以此类推.
- 函数 `waitpid`, `kill`, `fork`, `execve`, `setpgid` 以及 `sigprocmask` 用起来很方便. `waitpid` 函数的 `WUNTRACED` 和 `WNOHANG` 选项也很有帮助.
- 在实现信号处理机制的时候注意将信号通过 `kill` 函数以及负的参数 `-pid` 发送给整个进程组中的每个进程. `sdriver.pl` 驱动程序是会检查这个错误的.
- 最好将子进程的回收逻辑全部放在信号处理程序 (即 `sigchld_handler`) 中. 如果在主进程 (即 `waitfg`) 和信号处理程序中同时实现回收逻辑会使得思维难度上升 (因为主进程只需要回收前台进程, 却有可能接收到后台进程的信号).
- 在 `eval` 函数中, 为了避免竞争, 记得在调用 `fork` 之前先使用 `sigprocmask` 阻塞 `SIGCHLD` 信号, 然后在调用 `addjob` 之后再调用 `sigprocmask` 恢复 `SIGCHLD` 信号. 由于新 fork 出来的子进程会继承父进程的 `blocked` 阻塞向量, 因此还需要记得在子进程执行程序之前取消阻塞 `SIGCHLD` 信号.
- 不要在 tsh 中运行如 `more`, `less`, `vi` 以及 `emacs` 等需要特殊终端设置的程序. 尽量只运行简单的文本程序, 例如 `/bin/ls`, `/bin/ps` 以及 `/bin/echo` 等等.
- 由于 tsh 实际上是作为一个前台进程运行在标准 Unix shell 中的, 因此 tsh 所创建的所有子进程如果不加以额外设置默认都是同属于 tsh 的前台进程组之中. 为了模拟标准 shell 的行为需要注意在 tsh fork 出来的子进程中调用 `setpgid(0, 0)` 将其与 tsh 的进程组解耦并形成自己的独立进程组, 然后 tsh 自身负责向新产生的进程组转发用户输入的所有信号.

### 评价标准

- 每个 trace 文件占 5 分, 共 80 分.
  - 满分当且仅当本实验实现的 tsh 和参考实现的 tshref 在执行过程中的输出**完全一致** (除了 PID 和官方 `/bin/ps` 程序的输出以外, 因为这两种输出随每次运行而不同).
- 代码具有良好注释占 5 分.
- 在代码中检查了所有系统调用的返回值占 5 分.

### 实验思路与总结

#### 用于自动化测试的脚本

首先利用 `sdriver.pl` 和 `Makefile` 编写了一个自动化测试的 bash 脚本 `run.sh`, 代码如下:

```bash
#!/usr/bin/env bash

error_msg='Invalid input. Please enter test type (test/rtest) and a number between 1 and 16 for one test or '"'"'all'"'"' to run all tests.'

test_type="$1"
test_choice="$2"

if [[ $test_type != "test" && $test_type != "rtest" ]]; then
    echo "$error_msg"
fi

output_file='output_'"$test_type"'.txt'

if [[ $test_choice == "all" ]]; then
    true >"$output_file"

    for i in $(seq 1 16); do
        if [[ $i -lt 10 ]]; then
            i="0$i"
        fi

        make "$test_type$i" 2>&1 | tee -a "$output_file"
    done
else
    if [[ $test_choice =~ ^[1-9]$|^1[0-6]$ ]]; then
        make "$test_type$test_choice" >"$output_file" 2>&1
    else
        echo "$error_msg"
    fi
fi
```

通过命令 `./run.sh test 1` 或 `./run.sh rtest all` 自动测试并将结果输出至文本文件 (即 `output_test.txt` 和 `output_rtest.txt`) 中.

注意事项:

- 输出结果可以通过使用 `Ctrl+Tab` 在两个结果之间快速翻页来肉眼检查.
- 不失一般性, 最终输出结果中删除了程序 `ps` 的输出结果.

#### 所需填充的函数的思路

为了实现完整的 tsh 所需填充的 7 个重要函数的思路如下 (括号中数字代表预计代码行数 (包括注释)):

- `eval`: 读取, 解析并执行命令行. [70 行 | 实际 68 行]
  - 如果是前台作业, 需要调用函数 `waitfg` 来等待前台作业终止或停止.
- `builtin_cmd`: 识别并执行内置命令, 包括 `quit`, `fg`, `bg` 和 `jobs`. [25 行 | 实际 21 行]
  - 需要将 `fg` 和 `bg` 的逻辑转发给函数 `do_bgfg`.
- `do_bgfg`: 内置命令 `bg` 和 `fg` 的具体实现. [50 行 | 实际 74 行]
  - 需要区分 JID 和 PID, 前者有 `%` 前缀, 后者没有.
  - 调用 `kill` 函数向整个进程组发送 `SIGCONT` 信号.
  - 如果从后台作业转为前台作业, 需要调用函数 `waitfg` 来等待前台终止或停止.
  - 需要更改作业的执行状态为 `FG` 或 `BG`.
- `waitfg`: 等待一个前台作业终止或停止. [20 行 | 实际 27 行]
  - 需要使用 `sigsuspend`, 配合循环, 不断检查前台作业的执行状态, 只要状态仍然为 `FG` 就继续循环.
  - 需要打开信号 `SIGQUIT`, `SIGINT`, `SIGTSTP` 和 `SIGCHLD` 的监听功能.
  - 只需要监听信号并检查状态, 不需要例如从作业列表中删除作业等等工作, 这些工作交给信号处理程序完成.
  - 由于 `sigsuspend` 等待本次接收到的信号所对应的信号处理程序返回之后再返回, 而返回之后主进程会立即打印 prompt, 因此要注意正确安排信号处理程序的输出和主进程的输出之间的顺序, 以确保执行过程的输出和参考实现的输出完全一致.
  - 由于 `waitfg` 通过在信号处理程序返回之后立即检查作业状态来决定是否退出循环, 因此除了输出, 还要注意在信号处理程序中实时更新作业的状态:
    - 对于 `sigchld_handler`, 在监听到终止时直接删除作业, 而在监听到停止时需要修改作业状态;
    - 对于 `sigtstp_handler`, 需要在发送停止信号之后立即修改作业状态.
    - 需要注意的是 `sigchld_handler` 和 `sigtstp_handler` 中都需要针对停止事件修改作业状态, 因为 `sigchld_handler` 中监听到的停止事件有可能不是由用户通过键盘输入发起的, 还有可能是由子进程自己发起的.
- `sigchld_handler`: 信号处理程序, 负责接收 `SIGCHLD` 信号. [80 行 | 实际 71 行]
  - 全权负责接收 `SIGCHLD` 信号, 回收子进程, 并从作业列表中删除对应作业.
  - 需要使用循环实现, 即接收到一个信号之后需要不断循环直至没有子进程需要回收为止.
  - 如果是终止, 那么直接回收; 如果是停止, 那么还需要负责更改作业的执行状态为 `ST`.
  - 对于 `SIGTSTP` 信号还需要区分该信号的来源, 有可能是用户在终端上通过键盘发送并被主进程接收的, 也有可能是子进程自己发给自己的, 之所以要进行区分是因为这两个来源的接收方不同, 前一个是主进程接收并且是会额外触发信号处理程序 `sigtstp_handler` 的, 而后者则是子进程自己接收, 只会触发 `sigchld_handler`. 区分方法也很简单, 如果是主进程接收的信号那么在信号处理程序中已经对作业的状态进行了更改, 而如果是子进程接收的信号则作业状态还未改变, 因此只需要在 `sigchld_handler` 中检查作业状态是否已经更改为 `ST` 即可.
- `sigint_handler`: 信号处理程序, 负责接收 `SIGINT` (ctrl-c) 信号. [15 行 | 实际 19 行]
  - 向前台作业发送终止信号.
- `sigtstp_handler`: 信号处理程序, 负责接收 `SIGTSTP` (ctrl-z) 信号. [15 行 | 实际 25 行]
  - 向前台作业发送停止信号, 并打印提示信息.

#### Cheat Sheets

##### `tsh.c` 中已实现的函数

`tsh.c` 中已经实现的函数归纳如下 (按调用顺序拓扑排序, 未调用的函数按出现顺序排序):

- `usage`: 打印帮助信息并退出 (状态码为 1).
- `sigquit_handler`: 信号处理程序, 打印提示信息然后直接退出, 非常简单的一个函数.
- `clearjob`: 清空 `struct job_t` 类型的变量的内容 (起到初始化的作用).
- `initjobs`: 调用 `clearjob` 函数对预定义的作业列表 `jobs` (全局数组, 长度为 `MAXJOBS = 16`) 中的每个作业变量进行初始化.
- `app_error`: 应用风格 (application-style) 的报错函数, 打印提示信息并退出 (状态码为 1).
- `main`: 安装信号处理程序, 初始化作业列表 `jobs`, 之后开始打印提示信息, 读取命令行, 调用 `eval` 函数执行命令行, 循环往复.
- `parseline`: 解析命令行参数, 明确参数的数量, 明确程序是否后台执行. 命令行读取之后其内容副本保存在一个静态变量中, 返回的字符串数组 `argv` 中包含的是指向副本中字符位置的指针.
- `maxjid`: 遍历作业列表 `jobs`, 返回最大的 JID.
- `addjob`: 遍历作业列表 `jobs`, 将作业添加进第一个遍历到的空槽中. JID 由 1 开始分配, 每分配一个作业增加 1, 超过 `MAXJOBS` 之后重置为 1 (这里他代码应该打错了, 应该是超过 `MAXJID` 而不是 `MAXJOBS`, 前者才是最大 JID).
- `deletejob`: 清空指定 PID 的作业, 并将下一个要分配的 JID 设置为作业列表中最大的 JID + 1 (这里他忘记加 1 之后如果超过 `MAXJID` 需要重置为 1).
- `fgpid`: 遍历作业列表 `jobs`, 返回前台作业的 PID, 如果没有前台作业则返回 0.
- `getjobpid`: 遍历作业列表 `jobs`, 返回 PID 等于指定 PID 的作业, 如果没有则返回空指针.
- `getjobjid`: 遍历作业列表 `jobs`, 返回 JID 等于指定 JID 的作业, 如果没有则返回空指针.
- `pid2jid`: 遍历作业列表 `jobs`, 返回 PID 等于指定 PID 的作业的 JID, 如果没有则返回 0.
- `listjobs`: 遍历作业列表 `jobs`, 打印所有作业的信息.
- `unix_error`: Unix 风格 (unix-style) 的报错函数, 打印提示信息与错误信息 (`strerror(errno)`) 并退出 (状态码为 1).
- `Signal`: 书中提到的针对 `sigaction` 的包装函数, 作用是统一信号语义, 其功能与 `signal` 函数相同, 都是为给定的信号安装信号处理程序.

##### C 标准函数

头文件:

1. `sys/types.h`: 定义了一系列系统/硬件相关的类型, 例如 `pid_t`, `uid_t` 等等, 将具体硬件细节与 C 语言代码解耦, 主要是为了跨平台一致性.
2. `unistd.h`: 定义了对 Unix 系统调用的封装, 例如 `fork`, `read` 和 `write` 等等.
3. `stdlib.h`: 定义了一系列通用的函数, 包括动态内存管理 (`malloc`), 整数算术 (`atoi`), 搜索排序 (`bsearch`, `qsort`) 等等.
4. `sys/wait.h`: 定义了有关等待进程状态变化的函数与宏, 例如 `wait`, `waitpid` 等等, 一般与子进程相关的函数配合使用.
5. `signal.h`: 定义了用于处理信号 (signal) 的函数与宏, 例如 `signal`, `sigaction` 等等.
6. `setjmp.h`: 定义了用于非本地跳转的函数, 例如 `setjmp`, `longjmp` 等等.

函数:

- `pid_t getpid(void)`: 返回本进程 PID. [1, 2]
- `pid_t getppid(void)`: 返回父进程的 PID. [1, 2]
- `void exit(int status)`: 以 `status` 为状态代码退出程序. [3]
- `pid_t fork(void)`: 创建子进程. [1, 2]
- `pid_t waitpid(pid_t pid, int *statusp, int options)`: 以挂起/无挂起的方式等待进程的终止/停止/恢复, 并且可以通过指针 `statusp` 所指向的变量中的值了解进程被终止/停止/恢复的原因, 若进程终止则还会将其回收. [1, 4]
- `pid_t wait(int *statusp)`: `waitpid` 函数的简化版本, 等价于 `waitpid(-1, statusp, 0)`. [1, 4]
- `unsigned int sleep(unsigned int secs)`: 挂起当前进程直到时间结束或者被强制唤醒 (信号 `SIGCONT`). [2]
- `int pause(void)`: 挂起当前进程直到接收到任何一个信号. [2]
- `int execve(const char *filename, const char *argv[], const char *envp[])`: 加载并执行一个程序. [2]
- `char *getenv(const char *name)`: 寻找环境变量. [3]
- `int setenv(const char *name, const char *newvalue, int overwrite)`: 设置环境变量. [3]
- `void unsetenv(const char *name)`: 删除环境变量. [3]
- `pid_t getpgrp(void)`: 返回当前进程的进程组 ID. [2]
- `int setpgid(pid_t pid, pid_t pgid)`: 设置进程组. [2]
- `int kill(pid_t pid, int sig)`: 向任意进程 (包括调用进程自身) 发送任意信号. [1, 5]
- `unsigned int alarm(unsigned int secs)`: 在 `secs` 秒之后给调用进程自身发送 `SIGALRM` 信号. [2]
- `sighandler_t signal(int signum, void (*handler)(int))`: 设置自定义的信号处理函数. [5]
- `int sigprocmask(int how, const sigset_t *set, sigset_t *oldset)`: 修改阻塞信号集. [5]
- `int sigemptyset(sigset_t *set)`: 初始化信号集 `set`. [5]
- `int sigfillset(sigset_t *set)`: 将所有信号全部添加进信号集 `set`. [5]
- `int sigaddset(sigset_t *set, int signum)`: 将特定信号 `signum` 添加进信号集 `set`. [5]
- `int sigdelset(sigset_t *set, int signum)`: 将特定信号 `signum` 从信号集中删除. [5]
- `int sigismember(const sigset_t *set, int signum)`: 检查特定信号 `signum` 是否是信号集 `set` 中的成员. [5]
- `int sigaction(int signum, struct sigaction *act, struct sigaction *oldact)`: 用于统一不同 Unix 系统之间的信号处理语义. [5]
- `int sigsuspend(const sigset_t *mask)`: 无限期挂起当前进程, 并在幕后做一些额外工作使得当前进程会在接收到特定信号并触发信号处理程序并且信号处理程序正常返回之后被唤醒. 函数 `sigsuspend` 的预期行为就是无限期挂起当前进程, 也就是说当前进程调用 `sigsuspend` 之后被唤醒当且仅当 `sigsuspend` 报错 (即返回值为 -1). [5]
- `int setjmp(jmp_buf env)`: 设置非本地跳转锚点. [6]
- `void longjmp(jmp_buf env, int retval)`: 执行非本地跳转至最近设置的锚点. [6]
- `int sigsetjmp(sigjmp_buf env, int savesigs)`: 函数 `setjmp` 的信号处理版本. [6]
- `void siglongjmp(sigjmp_buf env, int retval)`: 函数 `longjmp` 的信号处理版本. [6]

### 实验结果展示

见文件 [output_test.txt](https://github.com/xubinh/csapp/tree/main/5-shell-lab/output_test.txt) 与 [output_rtest.txt](https://github.com/xubinh/csapp/tree/main/5-shell-lab/output_rtest.txt). 复现命令:

```bash
./run.sh test all
./run.sh rtest all
```

### 相关资料

无
