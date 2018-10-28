title: 可装卸的内核 gcov
date: 2018-10-28 20:30:00
tags:
- Linux Kernel
- gcov
categories:
- Develop
---

即使是 Linux 内核模块，测试依然是开发过程重要的一环，目前有 [Linux Test Project](https://github.com/linux-test-project/ltp) 等围绕着 Linux  Kernel 测试项目可以来参考。在测试的过程中，其有效性可以通过代码的覆盖率来表现，例如 Linux Test Project 采用了 gcov 作为其测试覆盖率工具。gcov 不仅支持用户态程序，同样支持内核的代码。目前 gcov 的支持需要在内核编译时开启参数，除此之外对于编译内核或内核模块的 GCC 版本有着一定的兼容性要求。但这篇文章并不对 gcov 用法以及如何通过正常手段开启内核中的 gcov 支持做介绍，主要讨论的是如何在不重新编译内核的情况下，通过一些 dirty hack 的手段实现内核模块 gcov 的正常使用。

# 背景

参考 [Using gcov with the Linux kernel](https://01.org/linuxgraphics/gfx-docs/drm/dev-tools/gcov.html) 或者相似的文章，可以发现内核的 gcov 支持主要由以下的选项支持：

```
# enable GCOV
CONFIG_DEBUG_FS=y
CONFIG_GCOV_KERNEL=y

# select the gcc’s gcov format, default is autodetect based on gcc version
CONFIG_GCOV_FORMAT_AUTODETECT=y

# get coverage data for the entire kernel
CONFIG_GCOV_PROFILE_ALL=y
```

换句话说，相关功能代码由这些宏包裹，如果宏未定义，相关代码是不会编译进内核的。通过 `/proc/config.gz` 中的 `.config` 文件，可以获取编译内核时的相关参数，如果需要使用 gcov 而没有使用这些参数的话，按官方支持，就只能重新编译内核然后安装了。

如果很幸运开启了 gcov 相关选项，也并不一定是只用在编译内核模块时加上 `-fprofile-arcs -ftest-coverage`，就可以顺利测试的。在机器上内核版本不够新，但是 GCC 更新过的情况下，有可能你会遇到 [https://stackoverflow.com/questions/17093461/why-my-kernel-gcov-doesnt-work-on-linux-3-9](https://stackoverflow.com/questions/17093461/why-my-kernel-gcov-doesnt-work-on-linux-3-9) 中描述的问题，简单来说这个问题的原因是 GCC 的较新版本使用了 `.init_array` 段代替 `.ctors` 段作为其初始化的代码段，如果内核比较老的话，gcov 还是会依赖 `.ctors` 来初始化其相关功能，两者不能兼容，所以这种情况下的 gcov 无法正常工作。

# 改造

本篇文章的改造目标是将 gcov 独立为一个单独的可卸载的模块，给内核模块提供覆盖率测试的功能，同时修正之前提到的 GCC 版本与内核不协调的差异。

Kernel 部分 gcov 相关的代码可以在 [kernel/gcov](https://elixir.bootlin.com/linux/latest/source/kernel/gcov) 中找到，总体来看 gcov 相关的代码分为三个部分，base、fs 以及 GCC 相关的一些 helper：

1. base 部分：
    base 部分主要包括对内核模块相关事件的监听，这里主要就是监听内核模块的装入和卸载了。由于 gcov 本身是作为内核的一部分的编译的，所以虽然有初始化，但没有相关的卸载操作。这里要修改的是在内核模块被卸载时，需要调用 `unregister_module_notifier` 把之前的监听事件清理掉。

2. fs 部分：
    在 gcov 正常的使用中，会在 debugfs 下面生成一些文件，包括 reset 开关和相关的统计数据。`fs.c` 这部分主要初始化和处理 debugfs 相关的事件的，和 base 部分同样，fs 部分也是不存在卸载操作的。除去 `OBJTREE` 和 `SRCTREE` 两个宏在仅测试模块内代码暂且用不到依赖，需要增加的是 debugfs 的入口清除的相关操作。

3. GCC helper 部分：
    GCC 中有两种版本的与 gcov 相关的数据结构，一是 GCC 3.4+，另一个是 GCC 4.7+，源代码也分割为 API 基本一致的两个文件。如果需要兼容所有支持 gcov 的 GCC 版本，那么需要从 API 层面上进行改造；如果仅需要简单使用，选择编译环境 GCC 一致版本的文件即可。

除了 gcov 相关的功能改造，还需要在内核模块加载的部分用 kprobe 或者 jprobe 来 hook 一部分调用，hook 地方集中在 [kernel/module.c](https://elixir.bootlin.com/linux/latest/source/kernel/module.c#L3642) 的  `load_module` 和 `do_init_module` 内，主要目的是完成未开启的宏内包裹的代码的相关功能。由于编译相关优化，不一定函数内的调用能被 hook 到，可以参考 `/proc/kallsyms` 来寻找可用的 hook 点。

1. `.ctors` 和 `.init_array` 段的初始化：
    在正常的流程中，`load_module` 中会调用 `find_module_sections` 来寻找相关的段信息，如果当前内核版本较旧或因为一些宏未开启的原因，并不同时支持 `.ctors` 或 `.init_array`。可以参考[较新的内核代码](https://elixir.bootlin.com/linux/latest/source/kernel/module.c#L3075)，在 `.ctors` 为空时，再寻找 `.init_array` 作为 `.ctors` 段保存下来。

2. `.ctors` 和 `.init_array` 段的执行：
    在 `do_init_module` 中的 `do_mod_ctors` 可以看到，在 `CONFIG_CONSTRUCTORS` 这个宏没开启的时候，是不会运行内核模块 `.ctors` 或者 `.init_array` 段内相关指令的。但恰好 gcov 的初始化就是通过这些段的执行来支持的，那么为了 gcov 能够正常初始化，将之前初始化过的指令列表执行即可。

在这些改造结束之后，依然有一些杂项需要处理，包括：

1. 聚合之前所有的初始化函数和卸载函数，确保模块安装、卸载前后无影响。
2. 通过 `nm` 一类的工具查看符号表，在给内核模块增加编译选项 `-fprofile-arcs -ftest-coverage` 后编译时提示缺符号的时候，以 `Module.symvers` 中形式支持为外部的符号。
3. 编写一个 Makefile 编译 gcov 为单独的模块。

在这些改造完成后，gcov 就可以作为独立的模块被依赖和正常使用了，可以 follow 其他教程来生成覆盖率网页或火焰图。如果在生成代码覆盖率信息时，不想要内核相关的代码信息，可以在 `getinfo` 命令生成的 info 文件中清除非测试模块内代码相关的数据。因为 `getinfo` 生成统计信息的基本格式是代码路径加数据的形式，比较简单，这里清洗的具体操作就不再赘述。

--------------

参考资料：
1. https://01.org/linuxgraphics/gfx-docs/drm/dev-tools/gcov.html
2. https://elixir.bootlin.com/linux/latest/source/kernel/gcov
3. https://gcc.gnu.org/bugzilla/show_bug.cgi?id=46770