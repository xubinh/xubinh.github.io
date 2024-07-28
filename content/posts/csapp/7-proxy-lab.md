---
title: "CSAPP 实验笔记 - 7. Proxy Lab"
description: "实现一个简单的具有缓存功能的并发网络代理服务器"
summary: "实现一个简单的具有缓存功能的并发网络代理服务器"
date: 2024-07-11T20:08:55+08:00
draft: false
tags: ["CSAPP", "CMU 15-213", "ICS"]
series: ["CSAPP", "CMU 15-213", "ICS"]
author: ["xubinh"]
type: posts
---

> - **具体代码请移步至[GitHub](https://github.com/xubinh/csapp/tree/main/7-proxy-lab).**

## Proxy Lab

### 实验目的

实现一个简单的代理服务器 (proxy), 使其能够并发处理来自客户端的请求并且能够缓存 (cache) 所请求的对象.

### 实验框架

#### `csapp.h` - 帮手库头文件

#### `csapp.c` - 帮手库

源文件 `csapp.c` 为 CS:APP 官方提供的源文件, 其中包括各种系统函数的包装函数, 以及 RIO 包的具体代码.

#### `proxy.c` - 要实现的代理服务器的源文件

本实验要求实现一个拥有缓存功能的 HTTP 代理服务器, 整个实验一共分为三个循序渐进的阶段:

1. 实现一个顺序 (sequential) 处理请求的代理服务器;
2. 添加并行机制, 并行化代理服务器;
3. 添加缓存机制.

一个代理服务器的功能:

- 用于防火墙, 防止内部客户端访问不允许的网站.
- 用作匿名器, 剔除客户端请求中的任何可识别信息, 代替客户端向服务器请求数据.
- 缓存客户端近期已请求过的对象, 提高请求速率.

第一个阶段需要实现一个顺序处理请求的代理服务器. 具体细节:

- 通过命令行指定代理服务器将要监听的端口.
- 启动代理服务器之后, 代理服务器应监听来自客户端的请求, 读取整个请求的内容, 解析请求内容, 并代替客户端向目标服务器建立连接. 最后代理服务器负责接收来自目标服务器的响应内容并将其转发回客户端.
- 假设客户端的请求行的内容为
  
  ```text
  GET http://www.cmu.edu/hub/index.html HTTP/1.1
  ```

  代理服务器应爬取域名 `www.cmu.edu` 和所请求的对象路径以及可选参数 (本例没有参数) `/hub/index.html`, 然后向服务器发送如下请求:

  ```text
  GET /hub/index.html HTTP/1.0
  ```

- HTTP 请求中的每一行都以 `\r\n` 结尾, 并且以一个空行结束整个请求.
- 代理服务器需要能够正确解析来自客户端的 HTTP/1.0 和 HTTP/1.1 GET 请求并向服务器统一发送 HTTP/1.0 GET 请求.
- 代理服务器的请求解析器不需要处理跨越多行的请求字段.
- 代理服务器不应由于请求不合法这样的简单错误而提前终止. 
- 代理服务器应该总是在请求头中添加 `Host` 字段. 不过如果客户端已经在请求头中添加了该字段, 代理服务器就不需要再添加了.
- 代理服务器可以在请求头中添加如下的 `User-Agent` 字段:
  
  ```text
  User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:10.0.3) Gecko/20120305 Firefox/10.0.3
  ```

- 代理服务器应该总是在请求头中添加固定的 `Connection: close` 字段.
- 代理服务器应该总是在请求头中添加固定的 `Proxy-Connection: close` 字段.
- 对于客户端在请求头中添加的其他任何字段, 代理服务器都应该能够正确解析并原样转发给目标服务器.
- 代理服务器应能够正确解析 HTTP 请求行中的可选端口号, 例如 `http://www.cmu.edu:8080/hub/index.html` 中的 `8080`, 并在此后向该端口 (而不是默认端口) 发起连接.
- 代理服务器需要正确处理 `write` 函数, `SIGPIPE` 信号, 以及 `EPIPE` 错误, 具体参考教材 11.6 节末的 Aside 部分.
- 代理服务器需要正确处理客户端提前关闭连接时 `read` 函数报 `ECONNRESET` 错误的问题.
- 代理服务器需要正确处理不同的内容类型 (文本内容如 HTML, 二进制内容如图像或视频).

第二个阶段需要添加并行机制, 并行化代理服务器. 具体细节:

- 可选的实现方案包括实时创建线程和使用线程池等等.
- 创建的线程应立即 detach, 减少内存泄露风险.
- 可以在线程中安全地使用 `open clientfd` 函数和 `open listenfd` 函数, 因为这两个函数是基于 `getaddrinfo` 函数的, 因此是线程安全的.

第三个阶段需要添加缓存机制. 具体细节:

- **缓存**指的是代理服务器将来自目标服务器的内容转发回客户端时应同时在内存中保留该内容的一个副本. 这样如果之后来自某个客户端的某个请求要求相同的内容, 那么代理服务器可以直接发送该副本而不用再次向目标服务器请求内容.
- 由于缓存不可能无限大, 并且所请求的内容的大小也无法预知, 缓存需要同时设置一个最大容量 `MAX_CACHE_SIZE` 和一个最大内容大小 `MAX_OBJECT_SIZE`. 每个内容的大小不应超过最大内容大小, 而全部内容加起来不应该超过最大缓存容量大小.
- 缓存的驱逐 (eviction) 策略可以近似于 LRU, 但没必要严格按照 LRU 策略来实现, 因为缓存还需要处理并发问题.
- 缓存的并发读写问题是一个经典的读者-写者问题, 解决方案包括对缓存进行分块, 使用现成的 Pthreads 读者-写者锁, 以及使用信号量来实现一个自定义方案等等.

#### `Makefile`

构建项目.

#### `port-for-user.pl` - 随机生成端口号

脚本 `port-for-user.pl` 使用哈希函数为用户名生成 (重复的概率很小但不为零) 随机端口号.

#### `free-port.sh` - 探测并返回一个可用的端口号

脚本 `free-port.sh` 探测并返回一个可用的 TCP 端口号.

#### `driver.sh` - 评分工具

脚本 `driver.sh` 用于对代理服务器实现进行自动评分, 使用方法:

```text
./driver.sh
```

#### `nop-server.py` - 评分工具的帮手脚本

脚本 `nop-server.py` 用于在测试代理服务器的并发功能时模拟阻塞环境.

#### `tiny/` - CS:APP 官方实现的 TINY 服务器

目录 `tiny/` 下存放的是官方实现的 TINY 服务器的源代码.

#### 其他

- 关于调试, 本实验并没有提供任何测试用例或是测试程序. 为了能够进行调试需要构建自己的测试用例框架 (testing harness). 可选的调试工具有:
  - CS:APP 官方实现的 TINY 服务器. 实际上 TINY 服务器本就用于在最终评分过程中充当目标服务器的角色.
  - Linux 程序 `telnet`. 如教材 11.5.3 小节中所示, `telnet` 可被用于与代理服务器建立 TCP 连接并向其发送 HTTP 请求.
  - Linux 程序 `curl`. 使用 `curl` 可以通过显式指定代理服务器来发送请求, 例如:
    
    ```text
    curl -v --proxy http://localhost:15214 http://localhost:15213/home.html
    ```
  
  - Linux 程序 `netcat` (或等价的 `nc`). `nc` 不仅可用于建立与服务器的连接并手动发送 HTTP 请求, 例如 `nc catshark.ics.cs.cmu.edu 12345` 将连接至服务器 `catshark.ics.cs.cmu.edu` 的 `12345` 端口, 也可以用于充当一个目标服务器来探查代理服务器的请求内容, 例如 `nc -l 12345` 将在本机建立一个服务器并监听 `12345` 端口. 代理服务器可以向 `nc` 请求任意伪对象, 而 `nc` 将能够探查代理服务器的请求内容.
  - 现代浏览器 (Google Chrome, Mozilla Firefox 等等). 有两个点要注意, 一个是需要在浏览器的设置中配置好代理服务器, 另一个是如果要测试代理服务器的缓存功能, 需要先关闭浏览器本身自带的缓存功能.
- 使用 `csapp.c` 中提供的 RIO 函数进行套接字 I/O.
- `csapp.c` 中的错误处理函数在检测到错误之后会立即关闭整个进程, 不适合用于代理服务器这样的长时间运行的程序. 可以选择修改或重写新的错误处理函数.
- 本实验允许对 handout 目录进行任意修改. 例如为了模块化可以将缓存实现在文件 `cache.c` 和对应的头文件 `cache.h` 中 (记得同时修改 `Makefile` 文件).
- 不论是关于程序错误, 关于任何不合法或可疑的输入, 还是关于段错误和内存/文件描述符泄露, 代理服务器都应该非常鲁棒.
- 关于 HTTP/1.0 请求的规范, 具体参考 [RFC 1945](https://datatracker.ietf.org/doc/html/rfc1945)

### 评价标准

实验针对实验的三个阶段分别进行评分, 评分标准:

- Basic Correctness (阶段一): 40 分.
- Concurrency (阶段二): 15 分.
- Cache (阶段三): 15 分.

### 实验思路与总结

#### 大纲

- 主线程:
  - 初始化线程池;
  - 初始化缓存;
  - 初始化 HTTP 请求解析器 (即编译全局正则表达式对象);
  - 打开监听套接字描述符, 不断监听客户端请求;
  - 对于每个客户端请求, 主线程负责将客户端已连接套接字描述符送入线程池.
- 副线程:
  - 分离自身;
  - 不断从队列中获取客户端的已连接套接字描述符.
  - 每当成功获取到一个描述符:
    - 初始化 HTTP 请求解析对象;
    - 使用 HTTP 请求解析器读取并解析该请求:
      - 如果请求不合法, 则触发错误处理程序 (可以选择向客户端回送一个错误页面, 或者什么也不做);
      - 如果所请求的是静态内容, 则检查缓存中是否已经存在该对象的副本, 若存在则直接将内容发送回客户端;
      - 如果所请求的是静态内容但未缓存, 或者所请求的是动态内容, 则建立与目标服务器的连接, 向目标服务器请求内容;
      - 如果所请求的是静态内容, 在获取到对象内容后需要将内容存入缓存 (在此之前需要检查内容大小以及缓冲区剩余容量);
      - 将内容发送回客户端.
    - 释放存储所请求对象内容的缓冲区;
    - 释放 HTTP 请求解析对象.

#### 要点

##### 一般要点

- 实验前请务必关闭任何代理. `curl` 默认从环境变量中获取可能的代理设置, 这会导致连接服务器失败并返回空内容.
- 每当遇到原因未知的 BUG 时记得打开调试工具的 `--verbose` 选项.
- 需要将辅助脚本 `nop-server.py` 的 shebang 行改为 `#!/usr/bin/env python` 并在下方添加一行编码行 `# -*- coding: utf-8 -*-` 以便在任何场合下都能正确运行.
- 确保所有 HTTP 请求合法但不在能力范围内的分支都得到处理.
- 正确处理 `write` 函数, `SIGPIPE` 信号, 以及 `EPIPE` 错误, 具体参考教材 11.6 节末的 Aside 部分.
- 正确处理客户端提前关闭连接时 `read` 函数报 `ECONNRESET` 错误的问题.
- 确保所有以指针引用传入的实参的类型正确匹配形参的类型.
  - 在编译选项中启用 `-Wall` 即可保证.
- 为所有自定义的数据结构定义相应的构造 (初始化) 函数与析构 (释放) 函数, 并确保正确调用.
- 确保所有套接字描述符在读取完毕之后都被关闭.
- 确保**替换**所有固定头部 (例如将 `Proxy-Connection: Keep-Alive` 替换为 `Proxy-Connection: close`).
- 确保请求的锁都已经按相反方向全部释放.
- 请求头部是大小写**不**敏感的, 确保正确处理头部名称的对应问题.
- 确保所有动态分配的内存在任何可能的函数退出路径上都被正确释放.
- 确保正确释放多级指针 (例如对于 `char **object` 应先释放 `*object` 再释放 `object`).
- 使用合适的逻辑处理各种可能的报错分支.
- 编写充分的注释.
- 可以使用非本地跳转实现报错.

##### 字符串和缓冲区处理要点

- 函数 `sscanf` 的 `%s` 会自动在传入的缓冲区末尾添加空字符 `\0`.
- 函数 `strlen` 返回的字符串长度不包括空字符 `\0`.
- 函数 `strcpy` 将源字符串复制到目的字符串, 直至遇到空字符 `\0` (空字符也会被复制).
  - 大小写**不**敏感版本: `strcasecmp`.
- 函数 `strcat` 将原字符串追加到目的字符串之后, 目的字符串的结尾空字符被原字符串的首字符所替代, 而原字符串的结尾空字符则称为二者所连接形成的新字符串的结尾空字符.
- 函数 `strstr` 返回子串在原串的首次出现位置的指针, 若无则返回 `NULL`.
- 函数 `memcpy` 将源缓冲区的 `n` 个字节的数据直接复制到目的缓冲区, 不考虑空字符, 并且假设两个缓冲区不会重叠.
- 函数 `Rio_readlineb` 每次读取一行文本数据, 直至遇到字符 `\n` (该字符也会被读入缓冲区), 并自动添加空字符 `\0` 结尾.
- 函数 `Rio_readnb` 每次读取 `n` 个字节的数据, 除非遇到 `EOF`.
- 函数 `atoi` 默认忽略任何前导空白符.

##### Posix 正则表达式 (`<regex.h>`) 要点

- 反斜杠 `\` 在字符类模式中失去转义作用.
- 如果要在方括号字符类模式 `[]` 内实现对字符 `]` 的排他匹配, 由于反斜杠 `\` 在字符类模式中失去转义作用, 因此必须将 `]` 置于排他字符 `^` 之后, 即 `[^]]`.
- 由于 C 语言自身对字符串具有转义处理, 对 regex 中的转义字符需要双重转义, 例如若要转义 `[` 字符, regex 模式为 `\[`, 对应的 C 语言字符串模式应为 `\\[`.
- 无法在一个模式的字符类中嵌套另一个相反模式的字符类.
- 无法使用 `\xHH` 的语法直接匹配 ASCII 字面量, 其中 `H` 表示一个十六进制字符.

##### Telnet 要点

- 当 `telnet` 处于逐行模式的时候, 直接输入 `eof` 将会发送 EOF 给对方.
- 在 `telnet` 中默认的 newline 为 CRLF, 每当键入换行符时将自动转换为 CRLF. 更好的方法是使用管道直接将一个行结束字符为 LF 的文本文件输入到 `telnet`.
- `telnet` 无法在标准输入重定向至文件的同时将服务器返回的响应通过重定向标准输出存储到文件中, 猜测一个可能的原因是标准输入重定向至文件之后 `telnet` 便不再进入逐行模式, 而响应只会在逐行模式下才会打印. 因此唯一的做法是放弃将标准输入重定向至文件的方法, 直接手动输入请求 (复制粘贴即可), 并使用 `tee` 将响应同时输出至屏幕以及文件.

#### 实现

##### HTTP 解析器 (位于 `http_request_parser.h`)

HTTP 请求的定义:

- 一个 HTTP 请求要么为简单请求, 要么为完整请求. HTTP 请求的定义:

  ```text
       Request        = Simple-Request | Full-Request
  ```

- 完整请求的定义:

  ```text
       Full-Request   = Request-Line
                        *( General-Header
                         | Request-Header
                         | Entity-Header )
                        CRLF
                        [ Entity-Body ]
  ```

  - 本实验不需要处理 Simple-Request.

- 请求行的定义:

  ```text
       Request-Line = Method SP Request-URI SP HTTP-Version CRLF
  ```

- 方法的定义:

  ```text
       Method         = "GET"
                      | "HEAD"
                      | "POST"
                      | extension-method

       extension-method = token
  ```

  - `token` 的定义见[RFC1945#2.2](https://datatracker.ietf.org/doc/html/rfc1945#section-2.2).

- 请求 URL 的定义:

  ```text
       Request-URI    = absoluteURI | abs_path
  ```

  - `absoluteURI` 的定义见[RFC1945#3.2.1](https://datatracker.ietf.org/doc/html/rfc1945#section-3.2.1).
  - `abs_path` 的定义见[RFC1945#3.2.1](https://datatracker.ietf.org/doc/html/rfc1945#section-3.2.1).
  - 文档中指出绝对 URI `absoluteURI` 只能被用于请求代理服务器, 而绝对路径 `abs_path` 则需要在与目标服务器建立连接的前提下使用.
  - 包含绝对 URI 的请求行例子:

    ```text
    GET http://www.w3.org/pub/WWW/TheProject.html HTTP/1.0
    ```

    对应的绝对路径的请求行的例子:

    ```text
    GET /pub/WWW/TheProject.html HTTP/1.0
    ```

- HTTP 中的 URI (即 `absoluteURI`) 的定义:

  [RFC 1945](https://datatracker.ietf.org/doc/html/rfc1945#section-3.2.2):

  ```text
       http_URL       = "http:" "//" host [ ":" port ] [ abs_path ]

       host           = <A legal Internet host domain name
                         or IP address (in dotted-decimal form),
                         as defined by Section 2.1 of RFC 1123>

       port           = *DIGIT

   If the port is empty or not given, port 80 is assumed. The semantics
   are that the identified resource is located at the server listening
   for TCP connections on that port of that host, and the Request-URI
   for the resource is abs_path. If the abs_path is not present in the
   URL, it must be given as "/" when used as a Request-URI (Section
   5.1.2).

      Note: Although the HTTP protocol is independent of the transport
      layer protocol, the http URL only identifies resources by their
      TCP location, and thus non-TCP resources must be identified by
      some other URI scheme.

   The canonical form for "http" URLs is obtained by converting any
   UPALPHA characters in host to their LOALPHA equivalent (hostnames are
   case-insensitive), eliding the [ ":" port ] if the port is 80, and
   replacing an empty abs_path with "/".
  ```

  [RFC 1738](https://datatracker.ietf.org/doc/html/rfc1738#section-3.3):

  ```text
   An HTTP URL takes the form:

      http://<host>:<port>/<path>?<searchpart>

   where <host> and <port> are as described in Section 3.1. If :<port>
   is omitted, the port defaults to 80.  No user name or password is
   allowed.  <path> is an HTTP selector, and <searchpart> is a query
   string. The <path> is optional, as is the <searchpart> and its
   preceding "?". If neither <path> nor <searchpart> is present, the "/"
   may also be omitted.

   Within the <path> and <searchpart> components, "/", ";", "?" are
   reserved.  The "/" character may be used within HTTP to designate a
   hierarchical structure.
  ```

  - 请求 URI 要么是以 `http` 开头的绝对 URI 形式, 要么是以斜杠 `/` 开头的绝对路径形式. 如果是绝对 URI 形式, 那么其后的路径和查询字符串均可省略. 在路径和查询字符串都省略的情况下根目录 `/` 也可以省略.
  - HTTP URL 的标准形式中的域名部分 `host` 总是小写的 (域名是大小写不敏感的), 如果给定的端口号为 80 则可以省略端口号部分, 最后在 HTTP 请求的场景下若 `abs_path` 部分为空则必须将其设置为默认的 `/`.

- host 的定义:

  [RFC 952](https://datatracker.ietf.org/doc/html/rfc952):

  ```text
   1. A "name" (Net, Host, Gateway, or Domain name) is a text string up
   to 24 characters drawn from the alphabet (A-Z), digits (0-9), minus
   sign (-), and period (.).  Note that periods are only allowed when
   they serve to delimit components of "domain style names". (See
   RFC-921, "Domain Name System Implementation Schedule", for
   background).  No blank or space characters are permitted as part of a
   name. No distinction is made between upper and lower case.  The first
   character must be an alpha character.  The last character must not be
   a minus sign or period.  A host which serves as a GATEWAY should have
   "-GATEWAY" or "-GW" as part of its name.  Hosts which do not serve as
   Internet gateways should not use "-GATEWAY" and "-GW" as part of
   their names. A host which is a TAC should have "-TAC" as the last
   part of its host name, if it is a DoD host.  Single character names
   or nicknames are not allowed.

   2. Internet Addresses are 32-bit addresses [See RFC-796].  In the
   host table described herein each address is represented by four
   decimal numbers separated by a period.  Each decimal number
   represents 1 octet.
  ```

  [RFC 1123](https://datatracker.ietf.org/doc/html/rfc1123#page-13):

  ```text
   2.1  Host Names and Numbers

      The syntax of a legal Internet host name was specified in RFC-952
      [DNS:4].  One aspect of host name syntax is hereby changed: the
      restriction on the first character is relaxed to allow either a
      letter or a digit.  Host software MUST support this more liberal
      syntax.

      Host software MUST handle host names of up to 63 characters and
      SHOULD handle host names of up to 255 characters.

      Whenever a user inputs the identity of an Internet host, it SHOULD
      be possible to enter either (1) a host domain name or (2) an IP
      address in dotted-decimal ("#.#.#.#") form.  The host SHOULD check
      the string syntactically for a dotted-decimal number before
      looking it up in the Domain Name System.

      DISCUSSION:
           This last requirement is not intended to specify the complete
           syntactic form for entering a dotted-decimal host number;
           that is considered to be a user-interface issue.  For
           example, a dotted-decimal number must be enclosed within
           "[ ]" brackets for SMTP mail (see Section 5.2.17).  This
           notation could be made universal within a host system,
           simplifying the syntactic checking for a dotted-decimal
           number.

           If a dotted-decimal number can be entered without such
           identifying delimiters, then a full syntactic check must be
           made, because a segment of a host domain name is now allowed
           to begin with a digit and could legally be entirely numeric
           (see Section 6.1.2.4).  However, a valid host name can never
           have the dotted-decimal form #.#.#.#, since at least the
           highest-level component label will be alphabetic.
  ```

  - 域名是大小写不敏感的.
  - 域名由字母 (A-Z), 数字 (0-9), 减号 (-), 以及句号 (.) 组成, 长度至少为 2 个字符且不超过 255 个字符.
  - 域名的首字符可以是字母或数字, 末尾字符不能是减号或句号.
  - IP 是一个 32 位二进制模式, 可以分为连续四个字节并以句号分隔的四个十进制整数的形式来表示, 每个整数表示一个字节的二进制模式的无符号字面大小 (点分十进制格式).
  - host 要么是一个域名, 要么是一个点分十进制格式的 IP.

- HTTP 版本的定义:
  
  ```text
       HTTP-Version   = "HTTP" "/" 1*DIGIT "." 1*DIGIT
  ```

  - 注意要处理任何可能的前导零.

- 一般头部的定义:

  ```text
       General-Header = Date
                      | Pragma
  ```

- 请求头部的定义:

  ```text
       Request-Header = Authorization
                      | From
                      | If-Modified-Since
                      | Referer
                      | User-Agent
  ```

- 实体头部的定义:

  ```text
       Entity-Header  = Allow
                      | Content-Encoding
                      | Content-Length
                      | Content-Type
                      | Expires
                      | Last-Modified
                      | extension-header

       extension-header = HTTP-header
  ```

- 通用头部的定义:

  ```text
       HTTP-header    = field-name ":" [ field-value ] CRLF

       field-name     = token
       field-value    = *( field-content | LWS )

       field-content  = <the OCTETs making up the field-value
                        and consisting of either *TEXT or combinations
                        of token, tspecials, and quoted-string>
  ```

  - `LWS` 为线性空白符 (linear whitespace), 包括 `CRLF` 和空格, 用于生成跨越多行的头部值. 由于本实验不需要处理跨行头部值, 因此 `LWS` 可被忽略.

解析内容:

- 方法名 (本实验只考虑 `GET` 方法, 不考虑其他任何方法)
- URL (本实验只考虑 `http_URL` 形式与 `abs_path` 形式, 不考虑其他任何形式)
  - 域名
  - 端口号
  - 所请求对象的绝对路径
- HTTP 版本 (本实验只考虑 `HTTP/1.0` 版本与 `HTTP/1.1` 版本, 不考虑其他任何版本)
- 请求头部 (本实验只考虑各种特定头部, 不考虑任何扩展头部)
- 由于只考虑 `GET` 方法, 解析器将忽略 `Entity-Body` 部分.

实现细节:

- 使用正则表达式解析 HTTP 请求.
- 所有解析内容使用指针+缓冲区以参数形式传入解析器, 解析器返回非零值 (True) 当且仅当**请求位于解析器的能力范围内并且格式正确**, 此时各个缓冲区中包含对应的属性值, 否则解析器返回零 (False).
- `Entity-Body` 部分总是返回空指针.

逻辑:

- 对于 request line, 使用空格将其分割为 method, URL, 以及 HTTP version 三个部分. 若分割失败则返回错误页面.
  - 对于 method, 使用正则表达式检查其合法性. 若不合法或不是 `GET` 方法则返回错误页面.
  - 对于 URL (代理服务器只需要处理来自客户端的请求, 因此必然是 `absoluteURI` 形式的 URL), 使用正则表达式将其分割为 host, port, path, 以及 query string 四个部分. 若分割失败则返回错误页面.
    - 对于 host, 检查其合法性. 若不合法则返回错误页面.
    - 对于 port, 分割等价于检查合法性, 因此不需要检查合法性.
    - 对于 path, 分割等价于检查合法性, 因此不需要检查合法性.
    - 对于 query string, 分割等价于检查合法性, 因此不需要检查合法性.
  - 对于 HTTP version, 使用正则表达式检查其合法性. 若不合法或不是 HTTP/1.0 或 HTTP/1.1 则返回错误页面.
- 对于 headers, [TODO]

##### 线程池 (位于 `integer_queue.h`)

线程池是通过实现一个同步工作队列来实现的.

##### 并发缓存 (位于 `cache.h`)

对缓存进行分区, 各个分区独立执行 LRU 驱逐策略, 拥有独立的锁, 并应用读者-写者模型实现同步. 缓存的分区策略类似于低位交叉编址, 目的是为了更好的可拓展性并降低实现难度.

要点:

- 读和写都算作修改动作, 都需要将对应的对象向前移动至链表头.
- 读操作将结点中的缓冲区复制到调用者提供的外部缓冲区中.
- 写操作将覆盖旧值. 确保将来仍可能使用的值不被意外覆盖是调用者的责任.
- 由于同一时间允许存在多个读者, 可能会出现某些读者在另一些读者还在查找的过程中就想要修改链表. 为了解决这一冲突还需要将读者群体进一步分为 "只读" 读者群体和 "欲修改" 读者群体, 并再次应用读者-写者模型解决冲突.
- 使用双向链表实现 LRU, 这是因为读者的只读操作和欲修改操作是解耦的, 采用单链表无法保证只读阶段获取到的链表结构在欲修改阶段仍然合法.

定义:

- 缓存的成员:
  - 分区数量
  - 链表数组
  - 读者数量数组
  - 读者锁数组
  - 读者-写者互斥锁数组
  - 只读读者数量数组
  - 只读读者锁数组
  - 只读读者-欲修改读者互斥锁数组
  - 初始化函数
  - 放置缓存对象的函数
    - 传入参数包括一个字符串, 用作缓存对象的键, 以及缓存对象的数据缓冲区指针.
  - 取出缓存对象的函数
    - 传入参数包括一个字符串, 用作缓存对象的键. 若找到缓存对象, 则返回值为所缓存对象的数据缓冲区指针, 否则返回 `NULL`.
  - 释放函数
- 链表结点的成员:
  - 一个表示所缓存的对象的键的字符串
  - 一个指向所缓存的对象的数据缓冲区的指针
  - 下一个链表结点的指针

#### Cheat Sheets

##### Unix I/O

头文件:

1. `<sys/types.h>`
2. `<sys/stat.h>`
3. `<fcntl.h>`
4. `<unistd.h>`
5. `<dirent.h>`

I/O 函数:

- 打开或创建文件: `int open(char *filename, int flags, mode_t mode)` [1,2,3]
- 设置文件的默认权限禁用位: `mode_t umask(mode_t cmask);` [2]
- 关闭文件描述符: `int close(int fd);` [4]
- 读文件: `ssize_t read(int fd, void *buf, size_t n);` [4]
- 写文件: `ssize_t write(int fd, const void *buf, size_t n);` [4]
- 重新设置文件描述符的位置: `off_t lseek(int fildes, off_t offset, int whence);` [4]

获取文件元数据:

- 使用文件名检索: `int stat(const char *filename, struct stat *buf);` [2,4]
- 使用文件描述符检索: `int fstat(int fd, struct stat *buf);` [2,4]

获取目录内容:

- 打开目录流: `DIR *opendir(const char *name);` [1,5]
- 遍历目录流: `struct dirent *readdir(DIR *dirp)` [5]
- 关闭目录流: `int closedir(DIR *dirp);` [5]

I/O 重定向:

- 复制文件描述符: `int dup2(int oldfd, int newfd);` [4]

##### 标准 I/O

头文件: `<stdio.h>`

I/O 函数:

- 根据文件名打开文件 (返回文件流): `FILE *fopen(const char *restrict filename, const char *restrict mode);`
- 根据文件描述符打开文件 (返回文件流): `FILE *fdopen(int fildes, const char *mode);`
- 关闭文件流: `int fclose(FILE *stream);`
- 读文件: `size_t fread(void *restrict ptr, size_t size, size_t nitems, FILE *restrict stream);`
- 写文件: `size_t fwrite(const void *restrict ptr, size_t size, size_t nitems, FILE *restrict stream);`
- 读文本行: `char *fgets(char *restrict s, int n, FILE *restrict stream);`
- 写文本行: `int fputs(const char *restrict s, FILE *restrict stream);`
- 格式化输入 (从文件流): `int fscanf(FILE *restrict stream, const char *restrict format, ... );`
- 格式化输入 (从标准输入): `int scanf(const char *restrict format, ... );`
- 格式化输入 (从字符串): `int sscanf(const char *restrict s, const char *restrict format, ... );`
- 格式化输出 (至文件流): `int fprintf(FILE *restrict stream, const char *restrict format, ...);`
- 格式化输出 (至标准输出): `int printf(const char *restrict format, ...);`
- 格式化输出 (至字符串): `int sprintf(char *restrict s, const char *restrict format, ...);`
- 清空缓冲区: `int fflush(FILE *stream);`
- 设置文件读写偏移: `int fseek(FILE *stream, long offset, int whence);`

##### CS:APP 所实现的 RIO

内部缓冲区类型定义:

```c
#define RIO_BUFSIZE 8192
typedef struct {
    int rio_fd;                // 文件描述符
    int rio_cnt;               // 当前内部缓冲区中的数据大小
    char *rio_bufptr;          // 当前内部缓冲区的读/写位置
    char rio_buf[RIO_BUFSIZE]; // 内部缓冲区
} rio_t;
```

无缓冲 I/O 函数:

- 无缓冲读: `ssize_t rio_readn(int file_descriptor, void *user_buffer, size_t total_bytes_number)`
- 无缓冲写: `ssize_t rio_writen(int file_descriptor, void *user_buffer, size_t total_bytes_number)`

带缓冲 I/O 函数:

- 初始化缓冲区, 关联文件描述符: `void rio_readinitb(rio_t *rp, int fd)`
- Unix I/O `read` 函数的带缓冲版本: `static ssize_t rio_read(rio_t *rp, char *usrbuf, size_t n)`
- 带缓冲读: `ssize_t rio_readnb(rio_t *rp, void *user_buffer, size_t total_bytes_number)`
- 带缓冲读 (文本行): `ssize_t rio_readlineb(rio_t *rp, void *user_buffer, size_t max_line_length)`

##### 网络

头文件:

1. `<arpa/inet.h>`
2. `<sys/types.h>`
3. `<sys/socket.h>`
4. `<netdb.h>`

网络字节序转换:

- 主机至网络 (long): `uint32_t htonl(uint32_t hostlong);` [1]
- 主机至网络 (short): `uint16_t htons(uint16_t hostshort);` [1]
- 网络至主机 (long): `uint32_t ntohl(uint32_t netlong);` [1]
- 网络至主机 (short): `uint16_t ntohs(uint16_t netshort);` [1]

IP 的整数形式与点分十进制字符串形式之间的转换:

- 字符串至整数: `int inet_pton(AF_INET, const char *src, void *dst);` [1]
- 整数至字符串: `const char *inet_ntop(AF_INET, const void *src, char *dst, socklen_t size);` [1]

套接字接口:

- 创建套接字描述符: `int socket(int domain, int type, int protocol);` [2,3]
- 客户端连接服务器: `int connect(int client_sockfd, const struct sockaddr *addr, socklen_t addrlen);` [3]
- 服务器将套接字描述符与套接字地址相关联: `int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);` [3]
- 服务器将主动套接字描述符转换为监听套接字描述符: `int listen(int sockfd, int backlog);` [3]
- 服务器接受连接: `int accept(int listenfd, struct sockaddr *addr, int *addrlen);` [3]
- 给定主机与服务, 查询套接字地址结构: `int getaddrinfo(const char *host, const char *service, const struct addrinfo *hints, struct addrinfo **result);` [2,3,4]
- 释放查询到的服务所占用的内存: `void freeaddrinfo(struct addrinfo *result);` [2,3,4]
- 将调用函数 `getaddrinfo` 报错所产生的错误代码转换为字符串: `const char *gai_strerror(int errcode);` [2,3,4]
- 给定套接字地址结构, 返回主机与服务信息: `int getnameinfo(const struct sockaddr *sa, socklen_t salen, char *host, size_t hostlen, char *service, size_t servlen, int flags);` [3,4]

CS:APP 官方提供的包装函数:

- 客户端主动建立连接: `int open_clientfd(char *hostname, char *port)`
- 服务器创建监听描述符: `int open_listenfd(char *port)`

##### I/O 多路复用

头文件: `<sys/select.h>`

函数:

- 统一监听一组描述符的状态: `int select(int n, fd_set *fdset, NULL, NULL, NULL);`

配套的宏:

- `FD_ZER0(fd_set *fdset);`
- `FD_CLR(int fd, fd set *fdset);`
- `FD_SET(int fd, fd set *fdset);`
- `FD_ISSET(int fd, fd set *fdset);`

##### Posix 线程 Pthreads

头文件: `<pthread.h>`

函数:

- 创建线程: `int pthread_create(pthread t *tid, pthread_attr_t *attr, func *f, void *arg);`
- 返回当前线程自身的 TID: `pthread_t pthread_self(void);`
- 当前进程主动终止自身: `void pthread_exit(void *thread_return);`
- 取消任意线程: `int pthread_cancel(pthread_t tid);`
- 回收任意线程: `int pthread_join(pthread_t tid, void **thread_return);`
- 当前线程分离自身: `int pthread_detach(pthread_t tid);`
- 首个线程执行初始化: `int pthread_once(pthread_once_t *once_control, void (*init_routine)(void));`

##### Posix 信号量

头文件: `<semaphore.h>`

函数:

- 初始化信号量: `int sem_init(sem_t *sem, 0, unsigned int value)`
- P 操作: `int sem_wait(sem_t *s);`
- V 操作: `int sem_post(sem_t *s);`

### 实验结果展示

```text
Building the tiny executable.

*** Basic ***
Starting tiny on 23735
Starting proxy on 14186
1: home.html
   Fetching ./tiny/home.html into ./.noproxy directly from Tiny
   Fetching ./tiny/home.html into ./.proxy using the proxy
   Comparing the two files
   Success: Files are identical.
2: csapp.c
   Fetching ./tiny/csapp.c into ./.noproxy directly from Tiny
   Fetching ./tiny/csapp.c into ./.proxy using the proxy
   Comparing the two files
   Success: Files are identical.
3: tiny.c
   Fetching ./tiny/tiny.c into ./.noproxy directly from Tiny
   Fetching ./tiny/tiny.c into ./.proxy using the proxy
   Comparing the two files
   Success: Files are identical.
4: godzilla.jpg
   Fetching ./tiny/godzilla.jpg into ./.noproxy directly from Tiny
   Fetching ./tiny/godzilla.jpg into ./.proxy using the proxy
   Comparing the two files
   Success: Files are identical.
5: tiny
   Fetching ./tiny/tiny into ./.noproxy directly from Tiny
   Fetching ./tiny/tiny into ./.proxy using the proxy
   Comparing the two files
   Success: Files are identical.
Killing tiny and proxy
basicScore: 40/40

*** Concurrency ***
Starting tiny on port 2622
Starting proxy on port 27710
Starting the blocking NOP server on port 13446
Sleep for 1 seconds waiting for port to be used...
Trying to fetch a file from the blocking nop-server
Fetching ./tiny/home.html into ./.noproxy directly from Tiny
Fetching ./tiny/home.html into ./.proxy using the proxy
Checking whether the proxy fetch succeeded
Success: Was able to fetch tiny/home.html from the proxy.
Killing tiny, proxy, and nop-server
concurrencyScore: 15/15

*** Cache ***
Starting tiny on port 2713
Starting proxy on port 29202
Fetching ./tiny/tiny.c into ./.proxy using the proxy
Fetching ./tiny/home.html into ./.proxy using the proxy
Fetching ./tiny/csapp.c into ./.proxy using the proxy
Killing tiny
Fetching a cached copy of ./tiny/home.html into ./.noproxy
Success: Was able to fetch tiny/home.html from the cache.
Killing proxy
cacheScore: 15/15

totalScore: 70/70
```

### 相关资料

- HTTP/1.0 规范: [RFC 1945](https://datatracker.ietf.org/doc/html/rfc1945).
- C 语言下使用正则表达式:
  - [8.5.10 ‘posix-extended’ regular expression syntax - GNU Findutils 4.9.0](https://www.gnu.org/software/findutils/manual/html_node/find_html/posix_002dextended-regular-expression-syntax.html)
  - [Regular expression - Wikipedia](https://en.wikipedia.org/wiki/Regular_expression)
  - [Regular expressions in C: examples? - Stack Overflow](https://stackoverflow.com/questions/1085083/regular-expressions-in-c-examples)
