title: Fresh to SystemTap
date: 2018-10-04 12:27:00
tags:
- Linux Kernel
- SystemTap
categories:
- Develop
---

最近在围绕着 Linux Kernel 在做一些相关的开发，鉴于个人水平，在开发过程中难免遇到一些 BUG 需要调试。苦于 GDB + QEMU 的调试方案配起来有点折腾，需要一个更易用的、不太干扰原有环境的方案。搜索整理后最后选用了 SystemTap 来进行动态调试，总体来说体验非常地好，这篇文章姑且算是 SystemTap 的一个简单笔记。

# SystemTap 简介

从官方上介绍来说，SystemTap 是运行在 Linux 环境下的用于信息获取的基础工具，通过获取到的信息，可以用来进行性能的检测、动态的调试等。对于调试方面来说，区别于 GDB 需要暂停程序单步，SystemTap 不会暂停程序运行，而是通过类似 Hook 形式插入原有逻辑，运行用户编写的脚本。

SystemTap 的实现结构可以参考官方的[架构文档](https://sourceware.org/systemtap/archpaper.pdf)。从简单来说 stap 脚本通过解析，在 SystemTap 本身的库以及 ELF Object 的 debug 等信息的帮助下，翻译为 C 源码，源码编译为 Loadable Kernel Modules，基于 kprobe 探测相关信息并运行脚本，通过 relayfs 将信息输出用户。

# 安装和运行

具体根据 Linux 的发行版以及包管理器的不同，SystemTap 安装的方式有所不同，具体安装方式参考各发行版 WIKI。

除 SystemTap 本体以外，由于 SystemTap 需要内核相关信息来对函数打点，还需安装当前内核版本的 `kernel-debuginfo`、`kernel-debuginfo-common` 以及 `kernel-devel`，具体内核版本可以通过 `uname -r` 查看。

安装完毕后可以通过 `stap -v -e 'probe vfs.read {printf("read performed\n"); exit()}'` 来测试一下 stap 是否可以正常运行，该例子主要挂在 vfs 的读操作上，在读操作被触发后输出 `read performed\n` 并退出。除了这种类似内联的一次输入，SystemTap 是支持通过文件脚本的形式来运行的，例如 `stap example.stap`。

# 探测点

SystemTap 的探测点相关的基本语法是 `probe event {statements}`，其中 event 是我们需要探测的事件，简单来说 event 分为同步事件和异步事件两种类型，同步事件包括一些内置的事件（例如之前举例的 VFS 读操作）、系统调用、内核模块函数、内核事件以及内核函数等，异步事件包括脚本本身的启动和结束行为、以及时间相关事件。

SystemTap 的探测点可以通过 `stap -l 'event'` 的形式查询，查到的信息还包含参数的 tag，例如 `stap -l 'kernel.function("tcp_v4_*")'`，值得一提的是 function 内支持通配符，这同样适用 SystemTap 本身的脚本，即可以通过通配符的方式在同一个 probe 内监控多个事件。function 相关的探测点默认 hook 在函数调用的开始阶段，这时可以通过脚本访问相关的参数。如果需要 hook 在函数的 return 阶段，则需要 `function("foo").return`，这时脚本可以访问函数的返回值。除此之外 `function("foo").exported` 过滤了可导出的函数，`function("foo").inline` 对应内联调用的函数，`function("foo").call` 对应非内联调用的函数。

# 变量、语法及内置变量与函数

除了 global 的变量外，基本 SystemTap 的变量是无类型声明直接使用的，在语法上也和现代的计算机语言差不太多，支持循环、判断、宏一类的语法，向外交互主要已输出为主。

因为 SystemTap 本身常用于收集数据，所以内置了一些方便用于使用的变量及函数，例如为了方便访问变量，除逐个访问函数内变量外，内置了如可以输出作用域内所有变量的 `$$vars`、本地变量的 `$locals`、函数参数的`$$parms` 以及返回值的`$$return`。在统计方面也有许多方便的基础设施，在通过 `<<<` 来将数据加入集合内后，可以通过 `@count`、`@sum` 等函数方便地统计，同时非常方便地可以用 `@hist_linear` 和 `@hist_info` 输出 ASCII 形式的直方图。

更具体的可参考[文档](https://sourceware.org/systemtap/langref.pdf)语法部分，此处不再赘述。

# Embedded C

除了本身的脚本语法以外，systemtap 还支持在脚本中嵌入 C 语言，方便了复杂逻辑或复杂结构的表示。

目前可以通过类似 `%{ #include <linux/ip.h> %}` 的方式引入头文件、 `function foo_bar:type(var: type) %{ %}` 的形式定义函数，简单的说，可以理解为被括号包裹住的部分可以使用 C 语言相关的语法。

参数及返回值支持的类型主要包括 `long` 和 `string`，需要用户通过操作 `STAP_RETVALUE` 来操作返回值。`long` 类型的返回值可以当作 C 语言的 `long` 型变量直接操作；对于 `string` 类型的变量，则对应 C 语言中的 `char[]` 类型，可通过内嵌的宏定义 `STAP_PRINTF` 来生成返回的字符串。如果需要类似 C 语言的 `return` 语句，可以通过内嵌的宏定义 `STAP_RETURN` 开实现。

参数同样无法直接在 C 语言部分被访问，而是通过 `STAP_ARG_var` 这样的宏来处理，举个例子：

```stap
function foo: long (val: long) %{
    long _val = (long)(STAP_ARG_val);
    STAP_RETVALUE = _val;
%}
```

同时内嵌的 C 语言部分可以通过注释的方式支持一部分断言和安全特性，具体可以参见 [Embedded C pragma comments](https://sourceware.org/systemtap/langref/Components_SystemTap_script.html#SECTION00047000000000000000)

嵌入了 C 语言的脚本区别于普通的 stap 脚本，多数的在运行时通过 `-g` 参数开启 guru 模式来关闭 stap 的安全检查，来启用对 Embedded C 的支持。

# 简单实例

SystemTap 在排查 TCP 数据包的处理逻辑中非常有效，例如通过 `tcpdump` 可以抓到需要的数据包，但应用层上无法成功建立连接或通信的情况，可以通过 SystemTap 挂载在相关的函数上，通过相关的调用栈相关信息以及相关变量值判断数据包在哪个环节被丢弃，例如脚本：

```stap
function lookup_tcp_v4_do_rcv_detail: string (sk: long, skb: long) %{
    struct sock *sk = (strcut sock *)(STAP_ARG_sk);
    struct sk_buff skb* = (struct sock *)(STAP_ARG_skb);
    // do something with sk and skb.
%}

probe kernel.function("tcp_v4_do_rcv").return
{
    printf("backtrace:\n%s", sprint_backtrace());
    lookup_tcp_v4_do_rcv_detail($sk, $skb);
}
```

--------------

参考资料：
1. https://sourceware.org/systemtap/archpaper.pdf
2. https://sourceware.org/systemtap/SystemTap_Beginners_Guide/