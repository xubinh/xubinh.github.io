---
title: "Git 入门"
description: "十分钟熟悉 Git 基本操作"
summary: "十分钟熟悉 Git 基本操作"
date: 2024-07-26T23:36:23+08:00
draft: false
tags: ["Tutorial", "xv6", "MIT 6.S081", "Git"]
series: ["Tutorial", "xv6", "MIT 6.S081", "Git"]
author: ["xubinh"]
type: posts
---

## Git 底层数据模型

- blob: 数据对象, 表示文件.
- tree: 树对象, 表示目录, 包含子 tree 和 blob.
- snapshot: 顶层 tree.
- history: 由所有 commit 以及它们之间的依赖关系构成的图.
- commit: 提交对象, 记录了一个 snapshot 及其 author, message 以及所有父 commit.
- object: 抽象对象, 对数据对象, 树对象和提交对象的统一.
- reference: 引用, 相当于C语言中的指针, 可以直接指向一个 commit,  也可以通过指向另一个引用间接指向一个 commit.

参考资料:

- [Git - Git Objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)

## working tree, index, commit, `git checkout`, 以及 `git reset`

- working tree (工作目录) 是当前项目所在的本地文件系统中的目录, 其中包含了所有项目的文件, 程序员可以对这些文件进行编辑, 删除, 以及添加等操作. 工作目录中的文件可以处于三种状态, 分别是 "未跟踪 (untracked)", "已暂存 (staged)", 以及 "已修改 (modified)".
- index (暂存区) 是一个介于工作目录和版本库之间的缓冲区. 暂存区保存了即将提交到版本库的文件快照, 程序员通过执行 `git add` 命令将文件从工作目录添加至暂存区, 表示这些文件已经准备好进行提交.
- commit (提交) 是对暂存区中文件快照的持久性保存. 提交操作会将暂存区的内容保存到版本库中, 并生成一个唯一的提交 ID (SHA-1 哈希值) 用于标识该提交. 每个提交都可以附加一个提交信息 (commit message), 用于描述本次提交的内容和目的. 提交后的文件状态会保存在版本库中并构成项目的历史记录.
- 可以看到 working tree, index 以及 commit 从小到大表示了三种不同的修改粒度. 如果将 index 比喻为正在生长的竹竿, 那么 commit 就是竹节, 而 working tree 则是竹叶. 由于 commit 已经保存至版本库中并且可以随时切换并查看, index 的更新实际上是和整个仓库的 commit 树相解耦的; 同时相对于 working tree 而言 index 又是相对固定的, 因此 working tree 的更新也可看作是和 index 相解耦的. 在这种三层结构模型下, `git checkout` 主要关注于对 index 和 working tree 的更新, 并不直接改变 commit 树本身, 而 `git reset` 则主要关注对 commit 树本身的更新. 相应地, 前者将使得 `HEAD` 引用自身指向另一个不同的引用或 commit, 后者则更新 `HEAD` 引用**所指向的引用** (例如 `master`) 的状态, 而不改变 `HEAD` 引用自身. 下面引用 [Git - Reset Demystified](https://git-scm.com/book/en/v2/Git-Tools-Reset-Demystified) 中的一幅插图作为例子:
  
  ![reset-checkout](https://git-scm.com/book/en/v2/images/reset-checkout.png)

参考资料:

- [Git - git-checkout Documentation](https://git-scm.com/docs/git-checkout)
- [Git - git-reset Documentation](https://git-scm.com/docs/git-reset)
- [What's the difference between "git reset" and "git checkout"? - Stack Overflow](https://stackoverflow.com/questions/3639342/whats-the-difference-between-git-reset-and-git-checkout)
- [Git - Reset Demystified](https://git-scm.com/book/en/v2/Git-Tools-Reset-Demystified)

## branch, reference, `master`, 以及 `HEAD`

- 在 Git 中每个 commit 由其 SHA-1 哈希值唯一标识, 直接使用哈希值索引某个 commit 对计算机而言十分简单直接, 但对程序员而言是反人类的. 引用的作用就是为 commit (的哈希值) 起一个别名. 引用本质上就是一个文件, 其文件名就是引用的名称, 而存储的内容则是其所引用的 commit 的哈希值.
  - 可以在当前仓库的 `.git/refs` 目录下找到所有引用.
- Git 中所说的分支 (branch) 是指 history 中从叶 commit 开始回溯一直回溯至根 commit 的过程中遍历到的所有 commit 构成的路径, 该路径由一个指向叶结点的引用唯一标识. 默认情况下 Git 将仓库中的首个分支命名为 `master` 或 `main`.
- 由于程序员常常需要在不同的分支间进行切换, 相对于固定的分支而言, 程序员当前的位置并不是固定不变的. 为了对程序员当前所在的位置进行标识, Git 使用另一个引用 `HEAD` 指向程序员当前所在的分支 (实际上是指向标识该分支的引用, 例如 `master`).
- 正常情况下 `HEAD` 引用指向的是标识分支的引用, 而不是该分支的叶 commit, 这么做有一个好处, 那就是当程序员使用 `git commit` 进行提交时除了更新 `master` 至新的叶 commit, 其实还使得 `HEAD` 也间接指向了新的叶 commit. 但在某些情况下程序员需要移动至内部非叶 commit (这可以通过 `git checkout` 命令做到), 此时 `HEAD` 将直接指向所移动至的 commit, 进入所谓的 "detach (脱离)" 状态.
- 在 detach 状态下, 程序员仍然可以执行提交, 并且新的 commit 将按照期望从 `HEAD` 处开始生长. 但需要注意的是此时新生长出来的 commit 并未形成任何的分支, 而是处于一种 "未被引用" 的状态. 如果程序员此时选择切换至其他分支, 处于 "未被引用" 状态的 commit 将会被 Git 的垃圾回收机制删除. 为了避免这种情况的发生, 可以通过执行 `git checkout` 命令或 `git branch` 命令为该 commit 创建一个新分支使其恢复为引用状态.

参考资料:

- [Git - git-checkout Documentation](https://git-scm.com/docs/git-checkout)
- [Git - Git References](https://git-scm.com/book/en/v2/Git-Internals-Git-References)

## 指定 commit 的常见方式

在日常工作中经常会遇见如 `git reset HEAD~` 这样的 "特殊形式" 的命令 (其中的 `HEAD~` 指的是 `HEAD` 当前所指向的 commit 的父 commit 对象). 下面是一些常见的指定 commit 对象的方式:

- `<sha1>`: 使用 SHA-1 哈希进行指定, 可以使用完整的 40 字符十六进制串 (例如 `dae86e1950b1277e545cee180551750029cfe735`) 也可以使用某一长度的子串 (例如 `dae86e`) (前提是仅存在唯一一个 commit 的 SHA-1 以该子串开头, 否则会产生歧义).
- `<refname>`: 使用引用的名称进行指定, 例如 `master`, `HEAD` 等等. 产生歧义时的具体解析规则请参阅文档.
- `@`: 等价于 `HEAD`.
- `<rev>^[<n>]`: 表示 `<rev>` 对象的第 `n` 个父结点 (一个 commit 可能具有多个父结点), 类似于广度优先搜索.
  - 仅使用 `<rev>^` (省略 `n`) 表示该对象的第 1 个父结点.
- `<rev>~[<n>]`: 表示 `<rev>` 对象的第 `n` 级祖先结点, 其中每次向上一级回溯时总是选择第 1 个父结点. 例如 `<rev>~3` 等价于 `<rev>^^^` 或 `<rev>^1^1^1`.
  - 仅使用 `<rev>~` (省略 `n`) 表示该对象的第 1 个父结点 (即 `<rev>~` 和 `<rev>^` 互相等价).

参考资料:

- [Git - git-rev-parse Documentation](https://git-scm.com/docs/git-rev-parse#_specifying_revisions)

## 特殊文件: `.gitignore`

简介:

- `.gitignore` 文件为文本文件, 用于显式指定用户不愿跟踪的文件, 其中每一行都代表一个模式 (pattern).
- Git 根据以下顺序确定所有模式:
  1. 命令行中读取到的模式.
  1. 从当前目录开始向上追溯至根目录为止找到的所有 `.gitignore` 文件.
  1. `$GIT_DIR/info/exclude` 中的模式.
  1. 配置 `core.excludesFile` 中的模式.
- 不同场景下的模式的存放位置:
  - 跟随整个项目一起分发的模式 (即一般情况下需要忽略的模式) 放在 `.gitignore` 文件中.
  - 与项目有关, 但仅限某个开发者自己使用的模式放在 `$GIT_DIR/info/exclude` 中.
  - 任何情况下都需要忽略的模式 (例如备份文件或者编辑器所产生的临时文件) 放在配置文件 `~/.gitconfig` 的条目 `core.excludesFile` 中.

具体规则:

- 空行不匹配任何文件 (可用作分隔行).
- 以井号 `#` 开头的行视作注释, 要获取字面值可以使用反斜杠 `\` 进行转义.
- 尾随的空格 (trailing space) 将被忽略, 除非它们被反斜杠 `\` 转义.
- 为模式前缀一个感叹号 `!` 将进行反匹配.
- 斜杠 `/` 用作目录分隔符, 可以出现在模式的开头, 中间或末尾:
  - 如果 `/` 出现在开头或中间, 则该模式将相对于 `.gitignore` 文件所在的目录进行匹配; 否则该模式将相对于 `.gitignore` 所处目录下的任意子目录进行匹配;
  - 如果模式末尾存在 `/`, 则该模式仅匹配目录; 否则该模式既可匹配文件也可匹配目录;
- 星号 `*` 匹配不包含 `/` 的任意长度的字符串;
- 问号 `?` 匹配不包含 `/` 的任意单个字符;
- 范围记号 `[a-zA-Z]` 可以匹配范围内的任意一个字符;
- 两个连续星号 `**` 可以有多种含义:
  - 前缀的 `**` 将会浮动匹配任何子目录, 例如模式 `**/foo` 和 `foo` 等价, 而模式 `**/foo/bar` 则会匹配任何形如 `foo/bar` 的子目录 (直接使用模式 `foo/bar` 无法做到这一点);
  - 后缀的 `**` 表示匹配目录下的任意文件, 例如模式 `abc/**` 匹配任意的名为 `abc` 的子目录中的所有文件, 注意此处 `**` 之前的斜杠 `/` 并不发挥其语义, 而是与 `**` 搭配作为一个整体;
  - 中间的 `**` 表示匹配一个或多个目录, 例如模式 `a/**/b` 能够匹配 `a/b`, `a/x/b` 或 `a/x/y/b`.
  - 如果存在连续多个星号, 除了一开始的两个星号按照上述规则解析以外剩下的星号将被视为普通星号.

参考资料:

- [Git - gitignore Documentation](https://git-scm.com/docs/gitignore)

## 配置 Git 默认编辑器

执行下列命令进行配置 (以 VS Code 为例):

```bash
git config --global core.editor "code --wait --new-window"
```

参考资料:

- [How to use Visual Studio Code as default editor for git? - Stack Overflow](https://stackoverflow.com/questions/30024353/how-to-use-visual-studio-code-as-default-editor-for-git)
- [Git - git-commit Documentation](https://git-scm.com/docs/git-commit)

## bare 仓库

- bare 仓库不同于一般的 Git 仓库, 其中并不含有任何能够用于编辑的源文件, 并且原先位于一般仓库中的 `.git` 目录下的文件在 bare 仓库中被移动至根目录下. 因此 bare 仓库并不是设计为让程序员使用的, 而是给服务器使用的.
- 可以在 `git clone` 时指定构建为一个 bare 仓库, 也可以在 `git init` 时指定新建为一个 bare 仓库.

参考资料:

- [git - What is a bare repository and why would I need one? - Stack Overflow](https://stackoverflow.com/questions/37992400/what-is-a-bare-repository-and-why-would-i-need-one)

## 命令速查表 / Cheat Sheets

### git add

> - `git-add` - Add file contents to the index

简介:

- `git add` 命令用于将 working tree 中的改动添加至 index 中 (如果反过来要复原 index 至某一状态可以使用 `git reset` 命令). 在一次提交之前可以重复使用 `git add` 添加改动.
- 可以使用通配符为 `git add` 命令指定要添加改动的文件.
- 对于那些被 `.gitignore` 显式忽略的文件, `git add` 默认不会添加. 即使这些文件被目录递归过程扫到了, `git add` 也会静默地忽略 (silently ignore) 这些文件.
- 如果显式在 `git add` 命令中指定添加一个被忽略的文件, 那么 `git add` 命令会报错. 但是如果在显式指定添加一个被忽略的文件的同时指定了 `-f` (force) 选项, 那么 `git add` 命令将无视该文件的被忽略属性, 正常将其添加至 index 中.

基本命令:

- `git add <pathspec>...`: 添加, 更新, 以及删除文件.
  - 参数 `pathspec` 用于指定要添加改动的文件的范围.
    - `pathspec` 总是相对于当前目录开始, 并且可以使用通配符 `*` 和 `?` 进行匹配, 前者匹配零个或多个字符, 后者匹配零个或一个字符 (二者均为贪婪模式).
      - 注意 `*` 和 `?` 不仅匹配常规字符, 还会匹配目录分隔符 `/`, 因此单个星号 `*` 能够匹配**任意长度且包含任意字符的字符串**, 而单个问号 `?` 能够匹配**任意字符** (或空字符串).
      - 此外在 shell 中键入命令时需要为通配符两边括上引号 (单引号双引号均可) 避免 shell 提前消耗通配符.
    - `pathspec` **不仅匹配 working tree 中的文件, 还会匹配 index 中的文件**. 如果某个文件在 index 中匹配到但在 working tree 中没有匹配到, 说明在 working tree 中该文件已被删除, `git add` 命令会将其从 index 中也删除.
    - 可以使用一种特殊的前缀进一步限定 `pathspec` 的匹配, 这种前缀包含一长一短两种形式, 较短的简洁形式由一个冒号 `:` 后接若干个魔数 (magic signature) 以及一个可选的冒号作为结尾 (如果 `pathspec` 中剩下的模式不以魔数字符开头那么这个结尾的冒号可以省略) 构成, 较长的精确形式为一个冒号 `:` 后接一个用括号 `()` 括起来的逗号分隔的魔法词语 (magic word) 列表. 可用的魔法词语和对应的魔数 (若有) 包括:
      - `top` (魔数: `/`): 显式指定 `pathspec` 从项目的根目录开始匹配.
      - `literal`: 禁用字符 `*` 和 `?` 的通配符作用, 二者将被作为字面量进行匹配.
      - `icase`: 进行大小写不敏感的匹配.
      - `glob`: 将通配符 `*` 和 `?` 按照 shell 的方式进行解析 (例如二者将不再匹配目录分隔符 `/`, 具体请参阅文档).
      - `attr`: 具体请参阅文档.
      - `exclude` (魔数: `!` 或 `^`): 指定进行反匹配. 例如 `git add "he*" ":^:he*.o"` 将在匹配 `he*` 的同时忽略被 `he*.o` 匹配到的文件 (即前一个集合对后一个集合作差).

    例如 `*hello.c` 和 `*.c` 将匹配像 `hello.c` (位于当前目录下), `sub1/hello.c`, `sub1/sub2/hello.c` 这样的任意目录深度的文件; `*/hello.c` (注意中间的斜杠 `/`) 将匹配像 `sub1/hello.c`, `sub1/sub2/hello.c` 等至少位于某个子目录下的文件; `hello.*` 将匹配当前目录下的所有以 `hello.` 作为前缀的文件, 但无法匹配像 `sub1/hello.c` 这样的位于其他子目录下的文件.
  - Git 1.x 旧版本中本命令只进行添加和更新, 不进行删除. 新版本中由于多了删除操作, 本命令已经等价于 `git add -A <pathspec>...`. 如果有需求可以使用 `--no-all` 选项显式告诉 `git add` 只进行添加和更新, 不进行删除操作.
- `git add (-A | --all | --no-ignore-removal) [<pathspec>...]`: 添加, 更新, 以及删除文件.
  - 如果没有指定任何 `pathspec`, 那么默认使用**项目的根目录**作为 `pathspec`.
- `git add (-u | --update) [<pathspec>...]`: 更新和删除文件, 不进行添加.
  - 仅对 index 中被 `pathspec` 匹配到的文件进行更新或删除, 不添加 working tree 中新增的文件.
  - 如果没有指定任何 `pathspec`, 那么默认使用**项目的根目录**作为 `pathspec`.

其他选项:

- `-f | --force`: 显式允许添加被 `.gitignore` 忽略的文件.

参考资料:

- [Git - git-add Documentation](https://git-scm.com/docs/git-add)
- [Git - gitglossary Documentation](https://git-scm.com/docs/gitglossary#def_pathspec)
- [git add - Difference between "git add -A" and "git add ." - Stack Overflow](https://stackoverflow.com/questions/572549/difference-between-git-add-a-and-git-add)
- [What does the 'git add .' ('git add' single dot) command do? - Stack Overflow](https://stackoverflow.com/questions/16969768/what-does-the-git-add-git-add-single-dot-command-do)
- [Git: Should the add/commit be run only at top level directory of a project? - Stack Overflow](https://stackoverflow.com/questions/58966403/git-should-the-add-commit-be-run-only-at-top-level-directory-of-a-project)

### git branch

> - `git-branch` - List, create, or delete branches

基本命令:

- `git branch --list [<pattern>...]`: 打印所有分支.
  - 需要注意的是 `pattern` 是用于 Shell 而不是 Git 进行 name globbing 的.
  - 尽管 `--list` 选项理论上在某些情况下可以省略, 为了避免 Git 将 `pattern` 解释为要创建的新分支名称, 还是建议显式指定 `--list` 选项.
- `git branch <new-branch-name>`: 创建新分支, 但并不关联 `HEAD` 引用至新分支.
  - 注意此命令并不使 `HEAD` 引用与新分支相关联, `HEAD` 引用此时仍然处于 detach 状态.
- `git branch (-d | --delete) <old-branch-name>`: 删除旧分支.
  - 旧分支在删除前必须完全 merge 到它的 upstream 分支或 `HEAD` 引用当前所指向的分支.
- `git branch (-m | --move) [old-branch-name] <new-branch-name>`: 重命名分支.

参考资料:

- [Git - git-branch Documentation](https://git-scm.com/docs/git-branch)

### git checkout

> - `git-checkout` - Switch branches or restore working tree files

基本命令:

- `git checkout [<branch-name>]`: 移动至指定的分支.
  - 执行本命令会将当前 index 更新为 (确切来说是复原为) 所指定的分支的叶 commit 的 index 的内容, 同时 `HEAD` 引用也会与指定分支相关联. 此前在工作目录中所做的改动仍然保留, 以便程序员在移动后继续提交 (如果不需要保留可以通过执行 `git reset` 命令进行删除).
  - 如果 `<branch-name>` 没有找到, 但是恰好有且仅有一个 remote 中含有一个同名的分支, 那么本命令等价于 `git checkout -b <branch> --track <remote>/<branch>`.
  - 如果没有指定 `<branch-name>`, 则默认为当前分支.
- `git checkout <commit-hash>`: 移动至指定的提交.
  - 执行本命令同样将修改 index, 同时 `HEAD` 引用将指向所指定的提交. 注意此时 `HEAD` 引用处于 detach 状态.
  - 移动后可以通过执行命令 `git checkout -b <new-branch-name>` 创建新分支.
- `git checkout -b new-branch`: 创建并移动至新分支.
  - 在当前 `HEAD` 所指处创建新分支并移动至该分支.
- `git checkout [<tree-ish>] [--] <pathspec>...`: 还原文件至某一提交中的状态.
  - 选项 `<tree-ish>` 一般为某个 commit, 如果未给出则默认为当前所在的 commit.
  - 选项 `--` 表示 "不要将位于 `--` 之后的参数解释为选项", 类似于 Python 中用于分隔函数参数列表中的普通参数与 keyword-only 参数的星号 `*`.
- `git checkout --orphan <new-branch-name>`: 创建并移动至一个孤立分支.
  - 新孤立分支没有任何提交, 并且程序员在该分支上所做的第一个提交将成为该分支的根结点. 此时新孤立分支与原有的所有分支构成一个森林.

参考资料:

- [Git - git-checkout Documentation](https://git-scm.com/docs/git-checkout)

### git clone

> - `git-clone` - Clone a repository into a new directory

基本命令:

- `git clone [--bare] <repository-name>`: 克隆仓库至当前目录.
  - 选项 `--bare` 表示构建一个 bare 仓库.

参考资料:

- [Git - git-clone Documentation](https://git-scm.com/docs/git-clone)

### git commit

> - `git-commit` - Record changes to the repository

撰写 commit message 的最佳实践:

```text
Capitalized, short (50 chars or less) summary

More detailed explanatory text, if necessary.  Wrap it to about 72
characters or so.  In some contexts, the first line is treated as the
subject of an email and the rest of the text as the body.  The blank
line separating the summary from the body is critical (unless you omit
the body entirely); tools like rebase can get confused if you run the
two together.

Write your commit message in the imperative: "Fix bug" and not "Fixed bug"
or "Fixes bug."  This convention matches up with commit messages generated
by commands like git merge and git revert.

Further paragraphs come after blank lines.

- Bullet points are okay, too

- Typically a hyphen or asterisk is used for the bullet, followed by a
single space, with blank lines in between, but conventions vary here

- Use a hanging indent
```

参考资料:

- [tbaggery - A Note About Git Commit Messages](https://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html)

基本命令:

- `git commit (-m <msg> | --message=<msg>)`: 生成一个新提交, 在命令行中直接附加注释.
- `git commit (-F <file> | --file=<file>)`: 生成一个新提交, 从文件中读取注释.
- `git commit (-C <commit> | --reuse-message=<commit>)`: 生成一个新提交, 重用此前的某个提交的注释和作者信息.
- `git commit (-c <commit> | --reedit-message=<commit>)`: 生成一个新提交, 重用**并修改**此前的某个提交的注释和作者信息.
  - 使用本命令将打开 Git 默认编辑器进行注释修改.
- `git commit --amend`: 生成一个新提交, 替换**当前分支的最新提交**.
  - 执行本命令时如果没有使用 `-m`, `-F`, 或 `-c` 等选项指定注释, 那么被替换的提交的注释将被用作新注释的初始值, 同时将打开 Git 默认编辑器进行注释修改.

参考资料:

- [Git - git-commit Documentation](https://git-scm.com/docs/git-commit)

### git config

> - `git-config` - Get and set repository or global options

基本命令:

- `git config --global init.defaultBranch main`: 设置默认主分支名称为 `main`.
- `git config --global http.proxy <protocol://ip:port>`: 配置全局代理.
- `git config --global https.proxy <protocol://ip:port>`: 配置全局代理.
- `git config (--system | --global | --local) <config-name>`: 查看某一配置.
- `git config --list [--show-origin]`: 查看所有配置 (包括系统级配置, 用户级配置以及仓库本地配置).
  - 选项 `--show-origin` 将打印配置的级别.

参考资料:

- [Git - git-config Documentation](https://git-scm.com/docs/git-config)

### git help

> - `git-help` - Display help information about Git

基本命令:

- `git help <command>`: 打印帮助文档.

参考资料:

- [Git - git-help Documentation](https://git-scm.com/docs/git-help)

### git init

> - `git-init` - Create an empty Git repository or reinitialize an existing one

基本命令:

- `git init [--bare]`: 新建仓库.
  - 新创建的仓库将包含一个不含有任何提交的初始分支.
  - 选项 `--bare` 表示新建为一个 bare 仓库.

参考资料:

- [Git - git-init Documentation](https://git-scm.com/docs/git-init)

### git log

> - `git-log` - Show commit logs

基本命令:

- `git log [--graph] --all`: 打印当前仓库的完整依赖拓扑.
  - 选项 `--all` 表示打印所有 commit 的依赖拓扑.
  - 选项 `--graph` 表示打印拓扑的图像表示 (基于 ASCII 字符).
- `git log [--graph] -- <path>...`: 从指定 commit 开始打印依赖拓扑.
  - `path` 可以是 commit 或指针, 表示 "打印从该 commit 或指针开始向前回溯过程中遍历到的所有 commit", 还可以在 commit 或指针前加上字符 `^` 表示不打印. 此外还可以使用特殊形式 `<commit1>..<commit2>`, 其等价于 `^<commit1> <commit2>`.

参考资料:

- [Git - git-log Documentation](https://git-scm.com/docs/git-log)

### git push

> - `git-push` - Update remote refs along with associated objects

基本命令:

- `git push [-u | --set-upstream] <remote-repo> <local-branch>[:<remote-branch>][+<local-branch>[:<remote-branch>]...]`: 在本地分支与远端分支之间进行同步.
  - 选项 `-u` 将给定 remote 设置为本地仓库的默认远端 (即 upstream reference). 本选项可以在第一次设置默认远端或是想要更改默认远端时使用, 此后执行 `git push` 或 `git pull` 命令时可以省去参数.
  - `remote-repo` 用于指定远端仓库, 可以是远端仓库的别名 (例如 `origin`) 也可以直接使用 URL.
  - `local-branch` 表示本地分支或对象.
  - `remote-branch` 表示远端分支或对象. 如果省略 `remote-branch` 则表示更新与 `local-branch` 同名的分支或对象.
- `git push [-u | --set-upstream] <remote-repo> --all`: 一次性推送所有本地分支.

参考资料:

- [Git - git-push Documentation](https://git-scm.com/docs/git-push)

### git remote

> - `git-remote` - Manage set of tracked repositories

基本命令:

- `git remote -v`: 列出现有的所有远程仓库.
- `git remote add <origin-name> <url>`: 添加远程仓库与 URL.
- `git remote set-url <origin-name> <new-url>`: 更改远程仓库关联的 URL.
- `git remote rename <origin-name> <new-origin-name>`: 更改远程仓库名称.
- `git remote rm <origin-name>`: 删除远程仓库.

参考资料:

- [Git - git-remote Documentation](https://git-scm.com/docs/git-remote)

### git reset

> - `git-reset` - Reset current HEAD to the specified state

基本命令:

- `git reset [<tree-ish>] <pathspec>...`: 将 index 中的指定文件复原为指定 commit 的 index 中的状态, 不改变工作目录, 也不移动 `HEAD` 引用.
  - `tree-ish` 默认为 `HEAD` 引用当前所指向的 commit.
  - 本命令可看作是命令 `git add <pathspec>...` 的反操作.
- `git reset [--mixed] [<commit>]`: 将整个 index 复原为指定 commit 中的 index 的状态, 不改变工作目录, 但会移动 `HEAD` 引用.
  - `commit` 默认为 `HEAD` 引用当前所指向的 commit.
  - 由于 mixed 是默认模式, 因此执行本命令时可以省略.
- `git reset --soft [<commit>]`: 既不复原 index 也不复原工作目录, 只移动 `HEAD` 引用.
  - `commit` 默认为 `HEAD` 引用当前所指向的 commit.
- `git reset --hard [<commit>]`: 同时复原 index 和工作目录, 并移动 `HEAD` 引用.
  - `commit` 默认为 `HEAD` 引用当前所指向的 commit.

参考资料:

- [Git - git-reset Documentation](https://git-scm.com/docs/git-reset)

### git rm

> - `git-rm` - Remove files from the working tree and from the index

基本命令:

- `git rm <some-file>`: 从 index 中删除, 同时删除物理文件.
- `git rm --cached <some-file>`: 仅从 index 中删除, 保留物理文件.

参考资料:

- [Git - git-rm Documentation](https://git-scm.com/docs/git-rm)

### git show-ref

> - `git-show-ref` - List references in a local repository

基本命令:

- `git show-ref [--head]`: 打印当前仓库中的所有引用以及所指向的提交.

参考资料:

- [Git - git-show-ref Documentation](https://git-scm.com/docs/git-show-ref)

### git status

> - `git-status` - Show the working tree status

基本命令:

- `git status`: 打印仓库的当前状态.

参考资料:

- [Git - git-status Documentation](https://git-scm.com/docs/git-status)

### git submodule

> - `git-submodule` - Initialize, update or inspect submodules

基本命令:

- `git submodule add <repository> <path>`: 在当前仓库中添加子仓库.
  - `repository` 表示要添加的子模块的 Git 仓库的 URL.
  - `path` 表示子模块在当前仓库中的存放路径.
  - 执行本命令后 Git 会在当前仓库的根目录中创建或更新一个名为 `.gitmodules` 的文件, 记录子模块的信息.
- `git submodule init`: 初始化所有子模块.
  - 可以选择初始化部分子模块, 具体参考文档.
- `git submodule update`: 克隆所有子模块至本地.
- `git submodule update --init --recursive`: 同时初始化并克隆所有子模块.

参考资料:

- [Git - git-submodule Documentation](https://git-scm.com/docs/git-submodule)
